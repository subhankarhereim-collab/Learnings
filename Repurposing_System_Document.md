# Drug Repurposing System — Design Document

**Project:** Literature-based drug repurposing
**Prepared for:** review

---

## 1. What the system does

**It reads research papers and produces a ranked list of diseases that a drug might be repurposed for — with a score for each, and the exact sentences that produced that score.**

The whole system is built around one unit: **a statement**. A statement is one claim, taken from one sentence, in one paper. Every statement gets a number. Every score is the sum of numbered statements. Nothing in the output exists without a sentence behind it.

---

## 2. What the user sees

One table per drug.

| Rank | Disease | Evidence | Confidence | Risk | Statements | Papers | Status |
|---|---|---|---|---|---|---|---|
| 1 | Coronary artery disease | 0.74 | 0.81 | Low | 11 | 7 | Well supported |
| 2 | Sjögren's syndrome | 0.61 | 0.72 | Low | 6 | 5 | Well supported |
| 3 | Type 2 diabetes | 0.44 | 0.38 | High | 8 | 3 | Contested |
| 4 | Pulmonary fibrosis | 0.29 | 0.66 | Low | 2 | 2 | Thin |

Click any row and you get the statements behind it:

```
Coronary artery disease  —  evidence 0.74  from 11 statements

  S-000142   support / moderate / human observational      [PMID 29875348, 2018]
             "Hydroxychloroquine use was associated with a
              lower incidence of cardiovascular events in
              this cohort."

  S-000210   support / strong / human trial                [PMID 31204410, 2019]
             "..."

  S-000377   contradict / weak / review                    [PMID 30117722, 2018]
             "..."
```

**That traceability is the product.** A number on its own is worth little. A number that opens into eleven cited sentences is something a researcher can act on.

---

## 3. What is drug repurposing?

Finding a new disease that an existing, already-approved medicine can treat.

A brand-new medicine takes 10–15 years and enormous cost, and most fail. An approved medicine has already passed safety testing. Showing it might help a different disease removes years of work and most of the risk.

Known examples: sildenafil (heart condition → erectile dysfunction), thalidomide (sedative → cancer), metformin (diabetes → now studied for cancer and ageing).

---

## 4. The pipeline

```
   ┌──────────────────────────────────────────────────────────────┐
   │  1  PAPERS                                                   │
   │     PDFs or abstracts. Each gets a paper_id and a year.      │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  2  CHUNKING                                                 │
   │     Split into overlapping chunks. Each gets a chunk_id.     │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  3  LLM EXTRACTION                                           │
   │     One call per chunk. Returns a JSON list of statements,   │
   │     or an empty list. Most chunks return empty.              │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  4  VALIDATION AND NUMBERING                                 │
   │     Check the JSON. Check the quoted sentence is really in   │
   │     the chunk. Remove duplicates. Assign S-000001 onwards.   │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  5  STATEMENT STORE                                          │
   │     Every accepted statement, with its source.               │
   │     This is the heart of the system.                         │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  6  AGGREGATE AND SCORE                                      │
   │     Group by disease. Compute evidence, confidence, risk.    │
   │     Rank.                                                    │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  7  NEO4J                                                    │
   │     Write the final results and their statement links.       │
   └────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────▼─────────────────────────────────┐
   │  8  DISPLAY                                                  │
   │     The table above, with click-through to statements.       │
   └──────────────────────────────────────────────────────────────┘
```

Steps 1–6 run outside the database. Neo4j is written to at step 7 and read from at step 8. **The scoring never reads from Neo4j.** This keeps the pipeline reproducible and makes it impossible for previous results to influence new ones.

---

## 5. Step 3 — what the LLM extracts

One call per chunk. The LLM is asked to find claims linking a drug to a disease, and to return them in JSON. JSON is used rather than free text because published work has found LLMs produce structured JSON more reliably than other formats, and because it maps directly onto Python objects.

### The fields

| Field | What the LLM answers | Notes |
|---|---|---|
| `drug` | name as written in the text | |
| `disease` | name as written in the text | normalised later |
| `relation` | may_treat / no_effect / worsens / already_used_for | |
| `stance` | support / neutral / contradict | |
| `strength` | strong / moderate / weak | |
| `study_type` | human_trial / human_observational / animal / cell / review | |
| `clarity` | clear / partial / vague | how directly the sentence states it |
| `sentence` | the exact sentence, copied word for word | **verified in step 4** |

If the chunk contains no such claim, the LLM returns an empty list. **Most chunks will.** That is expected and correct — a system that finds a statement in every chunk is hallucinating.

### Three design points that matter

**The LLM never produces a score.** It reports what the text says. All arithmetic happens in our code. This is deliberate: published evaluations have found that when an LLM is asked to rate its own confidence, it returns roughly the same high value regardless of how strong the evidence actually is. Self-rated confidence carries no information. Factual answers do.

**`relation = worsens` must be possible.** If the extraction can only report positive findings, every disease scores well and nothing is ever ruled out. Negation handling is treated as a first-class requirement, not an afterthought — recent relation-extraction work makes explicit negation handling a named pipeline stage for exactly this reason.

**`already_used_for` separates old from new.** If a paper says "the drug is indicated for malaria", that is the drug's current job, not a repurposing idea. These statements are stored but the disease is excluded from the ranking. Without this, the top of every list is permanently the drug's existing label.

### Asking three times

Each chunk is sent three times and the majority answer is taken. Repeated sampling gives noticeably more stable output than a single pass. The calls are cheap; the stability is worth it.

---

## 6. Step 4 — validation and numbering

Nothing enters the store without passing four checks.

| Check | What it does | On failure |
|---|---|---|
| **Schema** | Required fields present; values from the allowed list | Reject, log |
| **Sentence** | The quoted sentence must actually appear in the chunk | Reject, log |
| **Duplicate** | Same drug, disease and sentence from the same chunk | Keep one |
| **Disease name** | Map to a canonical disease and a `disease_id` | Flag if unmatched |

### The sentence check is the most important line of code in the system

The LLM must copy the sentence that justifies its answer. Our code then searches for that sentence in the original chunk, allowing for whitespace and punctuation differences. If it is not there, the entire statement is thrown away.

This does two jobs:

1. **Catches invented claims.** A finding the model made up has no sentence to point at.
2. **Stops the model answering from memory.** LLMs know a great deal about drugs from their training. Requiring a quote from *this chunk* forces the answer to come from the text in front of it rather than from what the model already believes.

It costs a string search. It is the single cheapest reliability measure available.

**The rejection rate is a monitored number.** If it starts climbing, something has changed — a prompt edit, a model change, or a corpus the model is more opinionated about.

### Why statements are numbered

Every accepted statement gets a sequential ID: `S-000001`, `S-000002`, and so on.

| Reason | What it enables |
|---|---|
| **Audit** | Any score opens into the exact statements that produced it |
| **Deduplication** | The same sentence extracted twice is caught by ID |
| **Reproducibility** | Two runs on the same corpus can be compared statement by statement |
| **Debugging** | "Why did this disease rank first?" is answered by listing its IDs |
| **Incremental runs** | Add a paper, and only its new statements need processing |

Without numbering, a score is an opinion. With numbering, it is a claim that can be checked.

---

## 7. Step 5 — the statement store

Three tables. Plain JSON or SQLite on disk — no database server needed at this stage.

### Table 1 — Statements

```
{
  "statement_id"   : "S-000142",
  "paper_id"       : "PMID:29875348",
  "paper_year"     : 2018,
  "chunk_id"       : "PMID:29875348#c07",
  "drug"           : "hydroxychloroquine",
  "disease_id"     : "D-0012",
  "disease_raw"    : "coronary artery disease",
  "relation"       : "may_treat",
  "stance"         : "support",
  "strength"       : "moderate",
  "study_type"     : "human_observational",
  "clarity"        : "clear",
  "sentence"       : "Hydroxychloroquine use was associated with ...",
  "score"          : 0.48,
  "run_id"         : "run_2026_03_11"
}
```

### Table 2 — Chunks

```
{
  "chunk_id"         : "PMID:29875348#c07",
  "paper_id"         : "PMID:29875348",
  "text"             : "...",
  "processed"        : true,
  "statements_found" : 2,
  "statements_rejected": 1
}
```

Tracking chunks separately means a crashed run can resume, and we can report what fraction of the corpus actually produced anything.

### Table 3 — Diseases

```
{
  "disease_id"     : "D-0012",
  "canonical_name" : "coronary artery disease",
  "aliases"        : ["CAD", "coronary heart disease", "ischaemic heart disease"]
}
```

**This table decides whether the system works.** If "type 2 diabetes", "T2DM" and "diabetes mellitus type 2" stay separate, the same disease splits three ways and all three fragments score low. The alias list starts small and grows as unmatched names appear.

The pipeline reports **how many raw names collapsed into how many diseases.** If that ratio looks wrong, it is visible immediately rather than silently damaging every score.

---

## 8. Step 6 — scoring

### Statement score

Each statement becomes one number: **what it claims × how good the study was.**

**Claim value** — from stance and strength:

| Stance | Strength | Value |
|---|---|---|
| support | strong | +1.0 |
| support | moderate | +0.6 |
| support | weak | +0.3 |
| neutral | any | 0 |
| contradict | weak | −0.3 |
| contradict | moderate | −0.6 |
| contradict | strong | −1.0 |

**Study weight** — from study type:

| Study type | Weight |
|---|---|
| Human clinical trial | 1.0 |
| Human observational | 0.8 |
| Animal | 0.5 |
| Cell / lab | 0.3 |
| Review or opinion | 0.2 |

```
   statement score  =  claim value  ×  study weight
```

Example: `support / moderate` in a human observational study → `0.6 × 0.8 = 0.48`.

The study-type ordering is not invented. It follows the evidence hierarchy used throughout medicine: trials outrank observational studies, which outrank animal work, which outranks cell work. The exact numbers are our choice and live in one settings file.

### Evidence score

For each disease, two things matter: **how positive the statements are**, and **how many there are.**

```
                   average statement score  +  1
   direction  =  ──────────────────────────────       →  0 to 1
                              2

                          n
   volume     =  ──────────────                       →  0 to 1
                       n  +  3

   evidenceScore  =  direction  ×  volume
```

- `direction` is 0.5 when the evidence is neutral, 1.0 when everything strongly supports, 0 when everything strongly contradicts.
- `volume` rises with more statements and levels off: 1 statement gives 0.25, 3 gives 0.50, 10 gives 0.77, 20 gives 0.87.

**Why volume is a separate factor:** without it, a single strong sentence produces a perfect score, indistinguishable from twenty converging papers. One sentence is not proof, and the score should say so.

### Confidence score

```
   consistency   =  1  −  (contradicting statements / n)
   independence  =  distinct papers / n
   clarity       =  average clarity   (clear 1.0, partial 0.6, vague 0.3)

   confidenceScore  =  0.5 × consistency
                     + 0.3 × independence
                     + 0.2 × clarity
```

**Independence is the part that earns its place.** If ten statements all come from one paper, independence is 0.1 and confidence drops sharply. If they come from ten different papers, it is 1.0. This catches a real failure mode: one enthusiastic review generating a pile of statements that look like a consensus.

### Risk level

```
   contradiction ratio  =  contradicting statements / n

   < 0.2   →  Low
   < 0.5   →  Medium
   ≥ 0.5   →  High
```

### Status

| Status | Condition |
|---|---|
| **Well supported** | 3+ statements, risk Low |
| **Promising** | 3+ statements, risk Medium |
| **Contested** | risk High |
| **Thin** | fewer than 3 statements |
| **Known use** | has `already_used_for` statements — excluded from ranking |

### Ranking

Sort by `evidenceScore`, with `confidenceScore` as the tiebreak. Exclude anything marked **Known use**.

Both scores are shown separately in the output. They are never combined into a single number, because doing so would require weights we cannot justify, and because a manager or reviewer can read two numbers just as easily as one.

---

## 9. Step 7 — Neo4j

Written to only at the end. Nothing in the scoring reads from it.

```
   (:Drug {name})
        │
        │ HAS_CANDIDATE
        ▼
   (:Candidate {candidate_id, evidenceScore, confidenceScore,
                riskLevel, status, rank,
                statementCount, paperCount, run_id})
        │                              │
        │ FOR_DISEASE                  │ BASED_ON
        ▼                              ▼
   (:Disease {disease_id,        (:Statement {statement_id, sentence,
              canonical_name})                 stance, strength,
                                               study_type, score})
                                              │
                                              │ FROM_PAPER
                                              ▼
                                        (:Paper {paper_id, year, title})
```

**Why Candidate is a node and not a relationship:** Neo4j cannot attach nodes to a relationship. Making the candidate its own node lets every supporting statement hang off it, so the click-through view in Section 2 is a single graph query.

**Why every Paper carries a year:** required for the accuracy check in Section 11.

**One rule:** Candidate nodes are output. Scoring must never read them back in. Predictions feeding into their own inputs is silent and fatal.

---

## 10. Files

Eleven files, four folders.

```
repurposing/
│
├── config.py
├── main.py
│
├── extraction/
│   ├── chunker.py
│   ├── prompt.py
│   ├── llm_client.py
│   └── validator.py
│
├── store/
│   ├── statement_store.py
│   └── disease_registry.py
│
├── scoring/
│   ├── statement_scorer.py
│   └── aggregator.py
│
└── output/
    ├── neo4j_writer.py
    └── report.py
```

### Setup

**`config.py`** — every setting in one place: LLM model, the two weight tables, the `+3` volume constant, the confidence weights, risk thresholds, status thresholds, Neo4j connection. When someone asks "what weight did animal studies get?", the answer is one file, one line.

**`main.py`** — single entry point. Takes a drug name and an optional cutoff year, runs all eight steps in order, prints and saves the table.

### `extraction/`

**`chunker.py`** — splits papers into overlapping chunks and assigns `chunk_id`. Overlap matters: a claim split across a chunk boundary is otherwise lost.

**`prompt.py`** — the actual prompt text and the JSON schema, with three or four worked examples including one where the correct answer is an empty list. Kept separate because prompt wording is the thing most likely to need tuning.

**`llm_client.py`** — sends a chunk three times, takes the majority answer, handles retries and rate limits. Caches every response so re-runs cost nothing.

**`validator.py`** — the four checks from Section 6. Assigns statement IDs. Maintains the rejection log and rejection-rate counter.

### `store/`

**`statement_store.py`** — read and write the three tables. Supports resuming an interrupted run by skipping chunks already marked processed.

**`disease_registry.py`** — maps raw disease names to canonical diseases and IDs. Reports the collapse ratio. Where a name cannot be matched, it is added as a new disease and logged for review.

### `scoring/`

**`statement_scorer.py`** — looks up the two weights, multiplies, writes the score back onto the statement.

**`aggregator.py`** — groups statements by disease, computes evidence, confidence and risk, assigns status, ranks. Attaches the list of contributing statement IDs to each candidate.

### `output/`

**`neo4j_writer.py`** — creates Drug, Disease, Candidate, Statement and Paper nodes and their relationships.

**`report.py`** — builds the table in Section 2 and the click-through view.

---

## 11. How we know the output is correct

Three checks, in order of how quickly they can be done.

### Check 1 — extraction accuracy (half a day)

Take 50 accepted statements at random. Read each one against its source chunk. Count how many are correct.

**This is the most valuable half-day in the project.** If extraction is 60% accurate, no amount of clever scoring downstream will fix it. If it is 90%, everything above it is worth building.

Report: extraction precision, plus the rejection rate from the sentence check.

### Check 2 — known-indication recovery (one day)

Run the system on a drug whose approved indications are known. Do those indications come out near the top, marked **Known use**?

If they do, the extraction and scoring are picking up real signal. If a drug's own well-documented indications do not surface, something is wrong upstream.

### Check 3 — time-split prediction (one week)

The real test. Choose a cutoff year. Ask the system to predict something it should not know yet.

**Two things must be hidden, not one.**

1. **Hide the known outcomes.** Set aside every drug–disease approval dated after the cutoff.
2. **Hide the papers too.** Exclude every paper published after the cutoff from the corpus.

The second is the one that gets missed, and it quietly ruins the result. If a drug was approved for a disease in 2018 and the cutoff is 2015, hiding the 2018 approval but leaving 2016–17 papers in the corpus means the LLM reads *"promising phase 2 results in this condition"* and scores the disease highly. That is not prediction — it is reading the answer sheet with the cover page removed.

Since papers are the system's only input, the fix is simple: filter the corpus by `paper_year` before the run, and assert that the newest paper used is not later than the cutoff. **Without this check the accuracy numbers will look excellent and mean nothing.**

**One residual limitation, stated openly:** the LLM's training data already contains post-cutoff knowledge. It may simply know about a later approval. The sentence check is the main defence — an answer drawn from memory has no sentence in the chunk to point at, so it is discarded. The rejection rate is the warning signal. This cannot be fully eliminated with a general-purpose model and should be acknowledged rather than glossed over.

Report: how often a hidden correct answer appears in the top 10 and top 50.

---

## 12. Plan

| Stage | Work | Time | Outcome |
|---|---|---|---|
| 1 | Chunker, prompt, LLM client. Run on 20 chunks and read the output by hand | 1–2 weeks | Confidence extraction works |
| 2 | Validator, sentence check, statement store with numbering | 1–2 weeks | Statements accumulating with IDs |
| 3 | Disease registry and name grouping | 1 week | Names collapsing correctly |
| 4 | Scoring and aggregation | 1–2 weeks | **First ranked table** |
| 5 | Neo4j writer, report with click-through | 1–2 weeks | Presentable system |
| 6 | Checks 1 and 2 from Section 11 | 1 week | Evidence it is accurate |
| 7 | Check 3, time-split | 1 week | Evidence it predicts |

**Stage 1 deserves patience.** Run the prompt on 20 chunks, read the output against the source text yourself, and fix the prompt before building anything on top. Everything downstream inherits whatever the extraction gets wrong, and this is the cheapest possible moment to discover a problem.

Stage 4 is the first demonstrable milestone.

---

## 13. Risks

| Risk | What goes wrong | Handling |
|---|---|---|
| **Poor extraction accuracy** | Every score built on wrong statements | Check 1 in Stage 1, before building further |
| **Disease names not grouped** | One disease split three ways, all scoring low | Registry reports collapse ratio; unmatched names logged |
| **Post-cutoff papers in the corpus** | Accuracy looks excellent, means nothing | Filter by `paper_year`; assert newest paper used |
| **Model answering from memory** | Claims with no source | Sentence check; rejection rate monitored |
| **Only positive findings extracted** | Nothing is ever ruled out | `relation = worsens` and `stance = contradict` are first-class |
| **Existing indications rank top** | Output states the obvious | `already_used_for` marks them; excluded from ranking |
| **One paper dominating** | Ten statements from one review look like consensus | Independence term in confidence score |
| **LLM cost** | Three calls per chunk across a large corpus | Cache every response; abstracts before full text |
| **Interrupted runs** | Hours of work lost | Chunk table marks processed chunks; runs resume |

---

## 14. What this system is, and is not

**It is** a way to read a large body of literature systematically and produce a short, ranked, fully cited list of diseases worth investigating for a given drug.

**It is not** proof that a drug works. It ranks published claims. Only clinical trials establish efficacy. Hydroxychloroquine and COVID-19 is the standing reminder — many computational studies supported it, and the trials did not.

The output should always be described as **"suggestions worth investigating"**, never as discoveries. What makes it useful is not the ranking alone but that every row opens into the sentences behind it, so a researcher can judge the evidence rather than trust a number.

---

## 15. Choices considered and rejected

| Rejected | Why |
|---|---|
| **Asking the LLM for a 0–1 score** | Self-rated LLM confidence is roughly constant regardless of evidence strength. Replaced by factual questions with code-side arithmetic. |
| **Free-text LLM output** | Unreliable to parse. JSON with a fixed schema is more consistent and maps directly to code. |
| **Extracting only positive claims** | Nothing could ever be ruled out; every disease scores well. Negative and neutral relations are first-class. |
| **Combining evidence and confidence into one number** | Requires weights we cannot justify. Both are shown separately. |
| **Scoring straight from Neo4j** | Makes runs irreproducible and risks previous results influencing new ones. The statement store sits outside the database; Neo4j is written to at the end. |
| **Hiding only known outcomes in the time-split check** | Left post-cutoff papers in the corpus, so the system could read the answer. Papers are filtered too. |

---

## 16. Open questions

1. Chunk size and overlap — needs one experiment in Stage 1.
2. Which cutoff year gives enough history and enough test cases for Check 3.
3. Whether `clarity` adds enough to justify the extra field, or whether stance and strength are sufficient.
4. Whether to extend to protein-mediated statements (drug → protein, protein → disease) in a later phase, which would let the system connect claims that no single paper makes.

---

## 17. Related published systems

| System | Approach | Relevance |
|---|---|---|
| **TheraMind** *(npj Precision Oncology, 2026)* | Fixed questions per paper, then hard-coded logic over the answers | Closest match to our extraction-then-arithmetic design |
| **DrugReX** *(Briefings in Bioinformatics)* | Literature knowledge graph with LLM-generated explanations | Similar end goal, heavier machinery |
| **RELATE** *(2025)* | LLM relation extraction with ontology constraints and explicit negation handling | Source of the negation-handling requirement |
| **LCoDR-KE** *(JMIR Medical Informatics, 2025)* | Schema-driven extraction for drug repositioning, 11 entity types and 18 relations | Precedent for schema-constrained extraction |

Our design is deliberately lighter than these. The distinguishing features are the numbered statement store, the sentence-verification guard, and keeping all scoring outside the database.

---

*End of document.*
