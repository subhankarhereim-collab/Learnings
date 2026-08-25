# Drug Repurposing System — Design (Version 2)

**Project:** EviGraph
**New module:** Repurposing
**Status:** Design for review

> This version replaces an earlier draft. Section 15 lists the design choices that were tested and rejected, and why.

---

## 1. What the system will show

Not one ranked list. **Two lists**, because the system does two different jobs.

### Output A — Hypotheses (the main result)

Diseases where the biology fits but few papers have looked yet.

| Drug | Disease | Mechanism | Evidence | Reason |
|---|---|---|---|---|
| Hydroxychloroquine | Disease X | 0.78 | 0.12 (2 papers) | Blocks 3 targets that are over-active in this disease. Almost nothing published |
| Hydroxychloroquine | Disease Y | 0.71 | — (0 papers) | Blocks 2 over-active targets. No literature at all |

### Output B — Validation (proof the system works)

Diseases where the biology fits **and** the papers already agree.

| Drug | Disease | Mechanism | Evidence | Reason |
|---|---|---|---|---|
| Hydroxychloroquine | Coronary artery disease | 0.82 | 0.74 (9 papers) | Blocks 4 over-active targets. 8 papers support, 1 opposes. Strongest: human study |

If Output B recovers indications that are already known, the pipeline works. Output A is then worth taking seriously.

Every row traces back to exact sentences in exact papers, stored in Neo4j.

---

## 2. What is drug repurposing?

Finding a new disease that an existing, already-approved medicine can treat.

A brand-new medicine takes 10–15 years and enormous cost, and most fail. An approved medicine has already passed safety testing. Showing it might help a different disease skips years of work.

Known examples: sildenafil (heart → erectile dysfunction), thalidomide (sedative → cancer), metformin (diabetes → studied for cancer and ageing).

**How this differs from what we built already:**

| | Target discovery (done) | Repurposing (new) |
|---|---|---|
| Question | Which proteins does this drug act on? | Which other disease can this drug treat? |
| Answer | Ranked proteins | Two lists of diseases |

---

## 3. The core design decision: two jobs, two scores

This is the most important section. It fixes a flaw that would otherwise make the whole system misleading.

### The problem with a single score

The obvious design is one number combining biology and literature. It does not work, and the reason is subtle.

Literature evidence requires papers that mention **both** the drug and the disease. So:

- **Lots of such papers** → someone has already thought of this. We measured how much research exists, not whether the idea is good.
- **No such papers** → no evidence score at all. But those are exactly the genuinely new ideas we want.

**A literature score is therefore highest for ideas that are least novel.** Combine it into one number and the ranking is dominated by how well-studied a pair already is.

This is a documented failure. A 2026 benchmark analysis of a literature-heavy repurposing system found that because evaluation datasets are built from established drug–disease pairs with confirmed indications, a literature-focused approach naturally scores well on well-documented relationships — and that this becomes a real limitation for novel candidates with little existing literature.

### The fix

Keep the two scores apart and let each do the job it is suited for.

```
                          Evidence score (from papers)
                       LOW                        HIGH
                 ┌────────────────────┬────────────────────┐
                 │                    │                    │
                 │   HYPOTHESES       │   VALIDATION       │
         HIGH    │                    │                    │
                 │  Biology fits,     │  Biology fits,     │
                 │  few papers yet    │  papers agree      │
Mechanism        │                    │                    │
  score          │  ** OUTPUT A **    │  ** OUTPUT B **    │
                 │  worth testing     │  proves it works   │
                 ├────────────────────┼────────────────────┤
                 │                    │                    │
                 │   IGNORE           │   REVIEW           │
         LOW     │                    │                    │
                 │  Nothing supports  │  Papers support it │
                 │  it either way     │  but we cannot see │
                 │                    │  the mechanism     │
                 └────────────────────┴────────────────────┘
```

**Mechanism generates candidates.** It works from the graph alone, so it can reach drug–disease pairs that nobody has written about. That is where novelty lives.

**Evidence grades candidates.** It reads what has been published about the ones mechanism proposed.

The "Review" quadrant is informative too: it means the literature supports something our biology data cannot explain, which usually points to a gap in our target or disease coverage.

**A useful side effect:** this removes the need to invent weights for combining the scores. There is no `0.5 × A + 0.3 × B` any more. That formula was unjustified and is now gone.

---

## 4. Mechanism Score — generating candidates

This runs first, uses only the graph, and needs no papers.

### The idea

A disease upsets the balance of proteins. Some become too active, some too quiet. A medicine also changes proteins — it blocks some and stimulates others.

**A medicine may help if it pushes the proteins the opposite way.**

| Protein | Disease does | Drug does | Match? |
|---|---|---|---|
| Protein A | Too active ↑ | Blocks ↓ | Good |
| Protein B | Too active ↑ | Blocks ↓ | Good |
| Protein C | Too quiet ↓ | Blocks ↓ | Wrong direction |
| Protein D | Too active ↑ | Stimulates ↑ | Wrong direction |

*(Illustration only.)*

We already know what the drug does to each protein — our extraction pipeline reads it from the papers, along with a confidence score. Open Targets provides the disease side.

### The formula

```
              sum of (matching targets × our confidence in them)
   MS  =  ───────────────────────────────────────────────────────
                  sum of (all shared targets × confidence)
```

Range: 0 to 1. Higher means more of the drug's actions push against the disease.

### Honest limitation — coverage

Open Targets does not have a direction for every disease–protein link. Where direction is missing we fall back to plain overlap (do the drug and disease share this protein at all) and **mark that row as unsigned**.

Two numbers are reported for every result:

- `MS` — the score
- `direction_coverage` — what fraction of shared targets had direction data

A score of 0.9 with 10% coverage is much weaker than 0.7 with 90% coverage. Hiding this would be dishonest, so it appears in the output table.

**Measuring this coverage is part of Stage 0** (Section 11). If it turns out to be very low, the signed version is not viable and we say so.

### One technical constraint

We only apply direction to proteins the drug acts on **directly**. Knowing that two proteins interact does not tell us whether one activates or blocks the other, so direction cannot be safely followed through the protein network. This limit should be stated up front — it is a predictable question from any reviewer.

---

## 5. Evidence Score — grading candidates

This runs second, only on the top candidates from Section 4.

### The core principle

**The LLM answers simple factual questions. Our code does the arithmetic.**

The obvious approach — show the LLM a paper and ask "rate this 0 to 1" — does not work. Published evaluations found that LLM self-reported confidence clusters around 0.95 almost regardless of the actual difficulty, in one study at exactly 0.95 in 71% of cases. The number carries no information.

So we never ask for a score. We ask factual questions and convert the answers ourselves.

### How one paper becomes a number

```
   ┌──────────────────────────────────────────────────────────┐
   │  Input: one paper chunk about this drug and disease      │
   └────────────────────────┬─────────────────────────────────┘
                            │
   ┌────────────────────────▼─────────────────────────────────┐
   │  LLM answers 4 fixed questions                           │
   │                                                          │
   │   Q1. Does this text support or oppose the link?         │
   │       → Supports / Opposes / Neutral                     │
   │                                                          │
   │   Q2. What kind of study does this text describe?        │
   │       → Human trial / Human study / Animal /             │
   │         Cell / Review                                    │
   │                                                          │
   │   Q3. Direct finding or indirect reasoning?              │
   │       → Direct / Indirect                                │
   │                                                          │
   │   Q4. Copy the exact sentence that shows this.           │
   │       → verbatim text                                    │
   │                                                          │
   │   Asked 3 times; majority answer is used                 │
   └────────────────────────┬─────────────────────────────────┘
                            │
   ┌────────────────────────▼─────────────────────────────────┐
   │  GUARD: is the copied sentence actually in the chunk?    │
   │    Yes → continue      No → discard and log              │
   └────────────────────────┬─────────────────────────────────┘
                            │
   ┌────────────────────────▼─────────────────────────────────┐
   │  Our code looks up weights and multiplies                │
   │    paper score = direction × study weight × directness   │
   └──────────────────────────────────────────────────────────┘
```

### The weight tables

**Direction**

| Answer | Value |
|---|---|
| Supports | +1 |
| Opposes | −1 |
| Neutral | 0 |

**Study type**

| Answer | Weight |
|---|---|
| Human clinical trial | 1.0 |
| Human observational study | 0.8 |
| Animal study | 0.5 |
| Cell / lab study | 0.3 |
| Review or opinion | 0.2 |

**Directness**

| Answer | Weight |
|---|---|
| Direct finding | 1.0 |
| Indirect reasoning | 0.6 |

**Example:** supporting, human trial, direct → `+1 × 1.0 × 1.0 = +1.0`. Opposing review reasoning indirectly → `−1 × 0.2 × 0.6 = −0.12`.

### Where these weights come from, and how we defend them

The study-type ordering is not invented. It follows the standard evidence hierarchy used throughout evidence-based medicine: trials outrank observational studies, which outrank animal work, which outranks cell work. The *ordering* is conventional. The *exact numbers* are a choice.

**So we test whether the exact numbers matter.** We run the ranking with three different weight sets — the default, a flatter one, and a steeper one — and measure the rank correlation between the resulting lists. If the correlation is high, the ranking is robust and the precise values are not doing the work. If it is low, we report that the results are weight-sensitive.

This takes about a day and turns an arbitrary choice into a measured one.

### Rolling up to the Evidence Score

```
                 sum of all paper scores
   ES  =  ─────────────────────────────────────
              number of papers  +  k
```

`k` is a small constant (start at 3). It prevents a pair with one paper from reaching a confident-looking score. Range: roughly −1 to +1.

### Evidence label — not a score

Alongside ES, each pair gets a plain-language label:

| Label | Condition |
|---|---|
| **Consistent** | 3+ papers, fewer than 20% opposing |
| **Contested** | 3+ papers, 20% or more opposing |
| **Thin** | 1–2 papers |
| **None** | 0 papers |

This is deliberately a **label, not a number**. Contradiction is already reflected in ES, because opposing papers contribute negative values to the sum. Adding a separate agreement score on top would count the same information twice.

---

## 6. Cost control

Fetching papers and running LLM calls is the expensive part. The two-axis design controls this naturally:

1. Mechanism score is computed for **all** candidates — cheap, graph only.
2. Papers are fetched and scored only for the **top N by mechanism** (start with N = 50).

So LLM cost scales with N, not with the full candidate set. This is why mechanism must run first.

---

## 7. How it is stored in Neo4j

The existing graph is untouched. We add a candidate layer.

```
        ┌──────────┐                          ┌──────────┐
        │  Drug    │                          │ Disease  │
        └────┬─────┘                          └────▲─────┘
             │                                     │
             │ HAS_CANDIDATE                       │ FOR_DISEASE
             │                                     │
        ┌────▼─────────────────────────────────────┴─────┐
        │              Candidate                         │
        │   ms, direction_coverage, es, label,           │
        │   quadrant, run_id, cutoff_year                │
        └────────────────────┬───────────────────────────┘
                             │ SUPPORTED_BY
                             │
        ┌────────────────────▼───────────────────────────┐
        │              Evidence                          │
        │   sentence, direction, study_type,             │
        │   directness, paper_score                      │
        └────────────────────┬───────────────────────────┘
                             │ FROM_PAPER
                             │
                       ┌─────▼──────┐
                       │   Paper    │
                       │ pmid, year │
                       └────────────┘
```

**Why a Candidate node rather than a relationship:** Neo4j cannot attach nodes to relationships. Making the candidate its own node lets every piece of evidence hang off it, so the "Reason" column is a direct graph query rather than something reconstructed later.

**Two fields that exist for correctness, not display:**

- `cutoff_year` on Candidate — records which time-filtered run produced this row. Without it, results from different runs get mixed and become uninterpretable.
- `year` on Paper — required for the time filtering in Section 10.

**Rule:** Candidate nodes are output. They are never read back in during scoring. Predictions feeding into their own inputs is a silent and fatal error.

---

## 8. Outside data

| What | Source | Used for |
|---|---|---|
| Which proteins matter in which disease, and in which direction | Open Targets Platform (free) | Mechanism score |
| Papers about drug–disease pairs | PubMed (free) | Evidence score |
| Known drug–disease treatments, with years | repoDB (free) | Accuracy check |

repoDB matters because it lists both **approved** and **failed** drug–indication pairs. Real failures, not just successes, make an accuracy check meaningful.

---

## 9. Files to be written

Thirteen files, four folders, in run order.

```
repurposing/
│
├── config.py
├── run_repurposing.py
│
├── data/
│   ├── load_diseases.py
│   ├── name_matcher.py
│   └── fetch_papers.py
│
├── scoring/
│   ├── find_candidates.py
│   ├── score_mechanism.py
│   ├── ask_llm.py
│   ├── score_one_paper.py
│   ├── score_evidence.py
│   └── build_output.py
│
└── graph/
    ├── save_to_neo4j.py
    └── queries.py
```

### Setup

**`config.py`**
Every setting in one place: Neo4j connection, LLM model, the three weight tables, `k`, the top-N limit, quadrant thresholds, and the cutoff year for evaluation runs.
*Why:* when someone asks "what weight did animal studies get?", the answer is one file, one line.

**`run_repurposing.py`**
Single entry point. Takes a drug name and an optional cutoff year, runs everything in order, produces both output tables.
*Why:* one command runs the system. Nobody needs to understand the internals to use it.

### Folder: `data/`

**`load_diseases.py`**
Downloads Open Targets data and creates `Disease` nodes linked to our existing protein nodes, **including the direction of effect where available**.
*Output:* Disease nodes, protein–disease links, and a report of how many links carry direction. That report is the input to the Stage 0 decision in Section 11.

**`name_matcher.py`**
Cleans names so they match across systems.
*The problem:* our pipeline extracts protein names from paper text, so we get "TNF-alpha", "TNFα" and "tumour necrosis factor alpha" for one protein. Open Targets uses one official ID.
*Output:* a mapping table plus a list of unmatched names.
*Why this is the riskiest file:* if names do not match, Neo4j joins return nothing and the system looks like it found no signal when it actually has a name bug. **This file must print the failure count** so the problem is visible.

**`fetch_papers.py`**
Searches PubMed for a drug–disease pair and returns abstracts, with local caching.
*Critical behaviour:* accepts a cutoff year and **excludes every paper published after it**. See Section 10 — this is what prevents the system reading the answer sheet.
*Output:* text chunks per pair, each tagged with its publication year.

### Folder: `scoring/`

**`find_candidates.py`**
Takes the drug's targets, finds every disease linked to any of them, removes diseases the drug is already approved for.
*Output:* candidate list, usually a few hundred.
*Why:* there are around 25,000 diseases. This narrows to those with a biological reason to look.

**`score_mechanism.py`**
Computes the mechanism score from Section 4 for every candidate, using our existing target rankings and the Open Targets directions.
*Output:* MS, direction coverage, and the list of matching targets that produced the score.
*Note:* this runs on all candidates because it is cheap. It also decides which candidates get the expensive treatment.

**`ask_llm.py`**
All LLM communication. Sends the four questions for one chunk, three times, returns the majority answer.
*Contains:* the exact prompt text, retry logic, majority-vote logic, and the instruction restricting the model to the supplied text only.
*Why separate:* one place to swap models, change prompts, or log every call without touching the scoring maths.

**`score_one_paper.py`**
Turns one answer set into one number.
*Does:* looks up the weights, multiplies, and **verifies the quoted sentence appears in the source chunk**. If not found, the judgement is discarded and logged.
*Why the quote check matters:* it is our defence against invented evidence and costs almost nothing — a string search. It also blocks the model from answering out of its own memory rather than the text.

**`score_evidence.py`**
Rolls paper scores into ES and assigns the evidence label from Section 5.
*Output:* ES, label, supporting count, opposing count.

**`build_output.py`**
Places each candidate on the two-axis grid and produces the two tables.
*Also:* writes the human-readable reason string, and carries `direction_coverage` through to the output so it is visible.

### Folder: `graph/`

**`save_to_neo4j.py`**
Writes results back: a `Candidate` node per pair with its `cutoff_year`, every `Evidence` node attached, each linked to its `Paper`.

**`queries.py`**
All Cypher in one file — get targets, get diseases for a protein set, save a candidate, fetch reasons.
*Why:* keeps database code separate from logic and easy to review.

---

## 10. Checking whether it works — and the leaks we must close

The check uses repoDB. Pick a cutoff year, hide what came after, and see whether the system predicts it.

**This is simple to describe and easy to get wrong. There are three ways information about the answer leaks backwards, and all three must be closed.**

### Leak 1 — the approvals

Hide every repoDB drug–disease approval dated after the cutoff.

This one is obvious and is usually the only one people close.

### Leak 2 — the papers

**Also exclude every paper published after the cutoff.**

This is the leak that quietly ruins the result. Suppose a drug was approved for a disease in 2018 and we set the cutoff at 2015. If we hide the 2018 approval but leave the 2016 and 2017 papers in the corpus, those papers say things like "promising phase 2 results." The LLM reads them and scores the pair highly.

That is not prediction. It is reading the answer sheet with the cover page removed.

`fetch_papers.py` must filter by publication year, and the run must assert that the newest paper used is not later than the cutoff. **If this check is missing, the accuracy numbers are worthless and will look excellent.** That combination — meaningless but impressive — is the most dangerous kind of result.

### Leak 3 — the model's own memory

The LLM was trained on text that includes post-cutoff knowledge. Asked about a 2018 approval, it may simply know.

Full prevention is not possible with a general model, but three things reduce it substantially:

1. The four questions ask only what **the supplied text** says, never what is true in general.
2. The quote-verification guard requires a sentence from the chunk. A judgement drawn from memory has no sentence to point at and is discarded.
3. We record the discard rate. A sudden rise on post-cutoff pairs is a signal that the model is answering from memory.

This residual limitation should be stated in any write-up rather than left for someone else to notice.

### What we report

- How often a hidden correct answer appears in the top 10 and top 50
- The same figures for **Output B** (validation quadrant) — this is where recovery should happen
- Number of candidates in **Output A** (hypotheses quadrant), which by design cannot be scored against repoDB
- Quote-rejection rate
- Direction coverage

Output A is not measurable against a gold standard, because a hypothesis nobody has tested has no recorded outcome. We should say this plainly rather than construct a metric that appears to measure it.

---

## 11. Stage 0 — the test that decides whether this design is viable

**Do this before writing pipeline code. It takes about two days.**

### Test 1 — literature coverage

Take 20 known drug–disease pairs and 20 random ones. For each, count PubMed papers mentioning both.

| Result | What it means |
|---|---|
| Most pairs return several papers | Evidence score is viable as designed |
| Most return 0–1 papers | Evidence score cannot grade most candidates. Most results land in "Thin" or "None" |

If coverage is poor, the evidence side becomes a filter on a handful of candidates rather than a scoring layer, and Section 5 must be scaled back. Better to know now than after six weeks.

### Test 2 — direction coverage

Pull Open Targets data for the diseases linked to our drug's targets. Count what fraction of protein–disease links carry a direction of effect.

| Result | What it means |
|---|---|
| Good coverage | Signed mechanism score works as designed |
| Poor coverage | Mechanism score falls back to plain overlap, and is correspondingly weaker |

**These two numbers determine whether the architecture stands as written.** Everything downstream depends on them, so they are measured first.

---

## 12. Plan

| Stage | Work | Time | Result |
|---|---|---|---|
| **0** | Two coverage tests above | 2 days | **Go / adjust decision** |
| 1 | Load diseases, name matching, candidate finder | 2–3 weeks | Candidate lists working |
| 2 | Mechanism score with direction and coverage reporting | 2 weeks | Output A becomes possible |
| 3 | Four questions, paper scoring, quote guard, ES | 2–3 weeks | Output B becomes possible |
| 4 | Two-axis output, save to Neo4j | 1–2 weeks | **Full working system** |
| 5 | Time-filtered accuracy check, weight sensitivity | 2 weeks | Evidence it works |

Stage 4 is the demonstrable milestone. Stage 0 is non-negotiable.

---

## 13. Things to watch

| Concern | What could go wrong | How we handle it |
|---|---|---|
| **Time leak via papers** | Post-cutoff papers left in corpus; accuracy looks excellent and means nothing | Year filter in `fetch_papers.py` plus an assertion on the newest paper used |
| **Time leak via model memory** | LLM recalls a post-cutoff approval | Questions restricted to supplied text; quote guard; discard rate monitored |
| Name matching | Joins return empty and look like "no signal" | `name_matcher.py` reports failure count; checked before trusting results |
| Low direction coverage | Mechanism score weaker than designed | Measured in Stage 0; coverage reported in every output row |
| Low literature coverage | Most pairs unscoreable on evidence | Measured in Stage 0; design scaled back if needed |
| LLM inconsistency | Same paper scored differently across runs | Ask three times, take majority |
| Invented evidence | Model claims a paper says something it does not | Quote must be found in source or judgement discarded |
| Weight arbitrariness | Results depend on numbers we chose | Sensitivity test across three weight sets, correlation reported |
| Prediction feedback | Candidate nodes read back in during scoring | Candidates are output-only; scoring queries never touch them |

---

## 14. Honest framing

The output should be described as **"suggestions worth investigating"**, never as discoveries.

The system ranks possibilities from published evidence and known biology. It cannot prove a medicine works — only clinical trials can. Hydroxychloroquine and COVID-19 is the standing reminder: many computational studies supported it, and the trials did not.

What the system does well is narrow a long list to a short, explained one, where every claim traces to a sentence in a real paper, and where the difference between "already known" and "genuinely new" is visible on the face of the output rather than hidden inside a single number.

---

## 15. Designs considered and rejected

Included because a reviewer will ask, and because these were tested rather than assumed.

| Rejected | Why |
|---|---|
| **Single combined score** `0.5×ES + 0.3×MS + 0.2×AS` | The weights were unjustified, and combining them buried the novelty problem in Section 3. Novel candidates were penalised for having little literature — the opposite of what is wanted. Replaced by the two-axis output. |
| **Asking the LLM for a 0–1 confidence** | Self-reported LLM confidence clusters near 0.95 regardless of difficulty. Replaced by factual questions with code-side arithmetic. |
| **Separate Agreement Score** | Double-counted contradiction, since opposing papers already contribute negative values to ES. A single supporting paper also produced a perfect agreement score, which is meaningless. Replaced by a four-level label. |
| **Unsigned mechanism score** | Plain target overlap is dominated by promiscuous drugs and well-studied diseases, and carries no biological direction. Replaced by the signed version, with unsigned retained only as a marked fallback. |
| **Following direction through the protein network** | A protein interaction does not indicate whether one protein activates or blocks another. Direction is applied to direct targets only. |
| **Hiding only post-cutoff approvals in evaluation** | Left post-cutoff papers in the corpus, so the system read the answer. Both approvals and papers are now filtered. |

---

## 16. Points to discuss

1. Stage 0 gives us two coverage numbers. What thresholds should trigger a redesign rather than proceeding?
2. Which cutoff year balances enough training history against enough test cases in repoDB?
3. Should the top-N limit be 50, or should it scale with how many candidates clear a mechanism threshold?
4. Output A cannot be validated against any gold standard. Is expert review of the top 10 an acceptable substitute?

---

## Reference systems

1. **TheraMind** — asks fixed questions per paper, then applies hard-coded logic over the answers. Closest published match to our evidence scoring. *npj Precision Oncology, 2026.*
2. **DrugReX** — literature knowledge graph with LLM-generated explanations and a composite repurposing score. *Briefings in Bioinformatics.*
3. **DrugAgent** — separate agents produce knowledge-graph, literature and ML scores, merged with weights. *Inoue et al.*
4. **DrugKLM** — assigns a 0–100 score using a fixed expert-designed guideline written into the prompt.
5. **repoDB** — Brown AS, Patel CJ. *A standard database for drug repositioning.* Scientific Data, 2017.

Our evidence layer resembles TheraMind's. What is added: the signed mechanism score built from our own extraction output, the quote-verification guard, and the separation of novelty from popularity in Section 3.

---

*End of document.*
