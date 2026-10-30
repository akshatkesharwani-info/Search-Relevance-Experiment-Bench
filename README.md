# Search Relevance Experiment Bench

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![Platform](https://img.shields.io/badge/Run%20on-Google%20Colab-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Cost](https://img.shields.io/badge/Cost-Free-success)

A small experiment bench to compare keyword, semantic and hybrid search on documentation pages, with judged queries, a held-out set and a report that shows regressions, not only wins.

## Table of Contents
1. [Problem](#problem)
2. [What I Built](#what-i-built)
3. [Results](#results)
4. [Choosing the Hybrid Weight](#choosing-the-hybrid-weight)
5. [Ranking Differences](#ranking-differences)
6. [Acceptance Tests](#acceptance-tests)
7. [Tech Stack](#tech-stack)
8. [How to Run](#how-to-run)
9. [Limitations](#limitations)
10. [About Me](#about-me)

## Problem
A documentation site needs better search. Changing the ranking by feeling can make exact things (error codes, near-identical product names) harder to find. So instead of picking a method, I built a bench: judged queries, three retrievers, and a report per query type.

## What I Built
- **100 fictional documentation pages**: 48 target pages (one per product and topic), 18 bridge pages (use all the ways of saying a topic), 25 error code pages and 9 release notes. Product names are near-identical on purpose (`Nimbus Gateway`, `Gateway Pro`, `Gateway Lite`)
- **40 judged queries** with graded relevance (2 = best page, 1 = related page, 0 = not relevant): 12 exact error codes, 8 near-identical product names, 12 paraphrases (different wording than the page) and 8 with **no right answer**
- A **dev set (28 queries)** to choose settings and a **held-out set (12 queries)** used only for reporting
- **3 retrievers**:
  1. **Keyword**: BM25 (k1 = 1.5, b = 0.75). Tokens keep hyphens, so `ERR-4021` stays one exact token
  2. **Semantic**: LSA (TF-IDF then SVD, 12 dimensions). This is an offline stand-in for vector search, no API needed
  3. **Hybrid**: alpha x BM25 + (1 - alpha) x LSA, each min-max scaled per query
- **Metrics**: recall@5, MRR (reciprocal rank of the first relevant page) and no-answer handling, split by query type
- A **ranking difference report** with the regressions listed, a deleted-pages safety test, and a versioned config

## Results
| Retriever | Split | recall@5 | MRR | Exact-ID MRR | Name MRR | Paraphrase MRR | No-answer correct |
|---|---|---|---|---|---|---|---|
| Keyword | dev | 0.837 | 0.773 | 1.000 | 1.00 | 0.348 | 1/5 |
| Keyword | held-out | 0.778 | 0.704 | 1.000 | 0.75 | 0.458 | 1/3 |
| Semantic | dev | 0.768 | 0.870 | 0.889 | 1.00 | 0.750 | 1/5 |
| Semantic | held-out | 0.694 | 0.889 | 1.000 | 0.75 | 0.875 | 1/3 |
| Hybrid | dev | 0.859 | 0.884 | 1.000 | 1.00 | 0.667 | 1/5 |
| Hybrid | held-out | 0.778 | 0.889 | 1.000 | 0.75 | 0.875 | 1/3 |

What the table says:
- **Keyword** is perfect on exact error codes, but weak on paraphrases (MRR 0.348 on dev).
- **Semantic** is much better on paraphrases (0.750) but lost some exact-ID accuracy on dev (0.889).
- **Hybrid** keeps exact lookup at 1.000 and still lifts paraphrase MRR from 0.348 to 0.667 on dev and from 0.458 to 0.875 on held-out.
- On held-out, hybrid raises MRR (0.704 to 0.889) but recall@5 stays the same as keyword (0.778).

## Choosing the Hybrid Weight
Alpha was chosen on the **dev set only**, with one rule: exact-ID MRR must not drop below the keyword-only value.

| alpha | Dev MRR | Exact MRR | Name MRR | Paraphrase MRR |
|---|---|---|---|---|
| 0.00 (semantic only) | 0.870 | 0.889 | 1.0 | 0.750 |
| **0.25** | **0.884** | **1.000** | 1.0 | 0.667 |
| 0.50 | 0.877 | 1.000 | 1.0 | 0.646 |
| 0.75 | 0.804 | 1.000 | 1.0 | 0.438 |
| 1.00 (keyword only) | 0.773 | 1.000 | 1.0 | 0.348 |

Chosen alpha: **0.25**. Across all 12 exact-ID queries, semantic-only search scored 0.917 exact-ID MRR against 1.000 for keyword. That is why pure semantic search was rejected as the default, even though it is good on paraphrases.

## Ranking Differences
Keyword vs hybrid on the 32 answerable queries: **9 improved, 23 unchanged, 0 regressed.**

Biggest wins (rank of the first relevant page):
- `Nimbus Camera: the battery drains fast`: rank 6 to rank 1
- `Nimbus Gateway Lite: the charge does not last`: rank 6 to rank 2
- `Nimbus Sensor X: flash a newer version`: rank 5 to rank 2

All the wins are paraphrase queries. The full list is saved in `ranking_differences.csv`.

## Acceptance Tests
| Test | Result |
|---|---|
| Same corpus + same config gives the same rankings | PASS |
| Exact-ID queries tested on their own: keyword 1.000, semantic 0.917, hybrid 1.000, none with zero score | PASS |
| Deleted pages (3 tested) never appear in fresh results or in snippets built from cached rankings | PASS |

## Tech Stack
| Area | Tools |
|---|---|
| Language | Python |
| Keyword search | BM25 written with numpy |
| Semantic search | scikit-learn (TF-IDF, TruncatedSVD / LSA) |
| Data | pandas |
| Charts | matplotlib |
| Environment | Google Colab |

## How to Run
1. Open `Search_Relevance_Experiment_Bench.ipynb` in Google Colab
2. Click **Runtime > Run all**
3. No dataset or API key is needed, everything is generated inside the notebook (seed 42)

Or locally: `pip install -r requirements.txt` and open the notebook in Jupyter.

## Limitations
- **No-answer handling is weak.** The abstain threshold was picked on only 5 dev queries, and every retriever got only 1 of 5 (dev) and 1 of 3 (held-out) right. Most queries with no answer still return results
- The held-out set is small: 12 queries, 9 of them answerable. Small differences between methods can be noise
- The semantic part is LSA on a tiny made-up corpus (123 vocabulary tokens). Real embeddings and real user queries could behave differently
- Relevance judgments come from the generator, not from real people

## About Me
**Akshat Kesharwani** - Data Analyst / Data Scientist (fresher)

- Portfolio: https://akshatkesharwani-info.github.io/
- GitHub: https://github.com/akshatkesharwani-info

Idea source: "Top 50 Projects to Build in 2026" (Yasir Arafat Sharfi). The build, data and code here are my own.
