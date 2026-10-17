# Report: Baseline Retrieval vs. Cross-Encoder Re-Ranking (Catan Rules & Almanac)

> **DRAFT NOTES, NOT THE FINAL REPORT.** The assignment requires the report to be in your own words. Every number below
> comes from the executed notebook / `results/`. Use this as a fact sheet, re-write the explanations yourself, then delete this box.

**Setup:** 82 chunks (~112 words), `all-MiniLM-L6-v2` + FAISS inner product, `ms-marco-MiniLM-L6-v2` re-ranker, Qwen2.5-3B-Instruct (greedy), same prompt for both pipelines.

## 1. Architecture comparison  (write in your own words)
Points to cover:
- Bi-Encoder: query and chunk are encoded **separately** into one vector each; similarity = cosine. Chunk vectors are pre-computed once and indexed, so search is fast, but each text is compressed into one vector with no knowledge of the other, so fine details (negations, "except", who/what role) get blurred.
- Cross-Encoder: query and chunk are concatenated into **one sequence**; self-attention lets every query token attend to every chunk token, and it outputs a single relevance score. More accurate, but nothing can be pre-computed; each pair needs its own forward pass.

## 2. Rank-shift analysis
| Query | Failure mode | Answer chunk | Baseline rank | Re-ranked | In top-3 after? |
|---|---|---|---|---|---|
| Q1 Buy a VP card that reaches 10 points, must I wait? | Exception | "Exception: ... victory point card" (29) | 7 | 3 | Yes |
| Q2 Replay a progress card? | Exception | "removed from the game" (19) | **10** | **2** | Yes |
| Q3 Resources if a robber-blocked number is rolled? | Exception | robber example (54) | 7 | 4 | No (partial) |
| Q4 Play two knight cards in one turn? | Negation / limit | "only play 1 development card" (29) | 6 | **1** | Yes |
| Q5 Build a sixth settlement? | Negation / limit | "all 5 settlements ... must upgrade" (60) | 7 | 2 | Yes |
| Q6 More than one road on one path? | Negation / limit | "Only 1 road per path" (13) | 8 | 2 | Yes |
| Q7 Which hex never produces anything? | Keyword trap | Desert (28) | 4 | **1** | Yes |
| Q8 Who gets starting resources, from which settlement? | Buried chunk | second-settlement rule (62) | **10** | **1** | Yes |
| Q9 Do roads count toward my score? | Limitation | "Roads provide VP only if..." (53) | >10 | >10 | No, both fail |
| Q10 Win during an opponent's turn? (control) | none | (32) | 1 | 1 | already top-3 |

Answer 1 vs Answer 2 (check full text in the notebook):
- **Q8:** Answer 1 (baseline) hallucinated: it described normal dice-roll production by number, not starting resources. Answer 2 correctly said you get them after building your *second* settlement, one card per adjacent hex. Clearest hallucination case.
- **Q7:** Answer 1 said *the hex marked "7"* never produces (wrong, there is no 7 token); Answer 2 said *desert* (right).
- **Q1:** Both said no waiting is needed, but Answer 1 only quoted the general "10 points on your turn" rule; Answer 2 states the actual exception (a purchased VP card may be revealed immediately).
- **Q2:** Both "no"; Answer 1's reason is the wrong rule (one dev card per turn). Answer 2 gives the right reason (card removed from game) but pads it with an unrelated knight-card quote.
- **Q5:** Both "no"; Answer 1 is muddled about settlement supply, Answer 2 quotes the exact rule (must upgrade one to a city first).
- **Q4, Q6:** the answer chunk was promoted (6 -> 1, 8 -> 2) but both answers were already correct, so the LLM could cope even with partial context; the benefit shows in rank, not answer text.
- **Q3:** helped but stayed at rank 4, just outside the top 3; both answers wrong/unchanged.
- **Q9:** answer chunk not in baseline top-10, so re-ranking cannot recover it; both wrongly said roads give points. Re-ranking only reorders candidates, so stage-1 recall is the ceiling.
- **Q10:** control, both right.

## 3. Latency (CPU, mean of 20 runs per query, 10 queries; no GPU)
| Stage | Time |
|---|---|
| Pipeline 1: query embedding + FAISS top-10 | 9.2 ms |
| Extra for Pipeline 2: Cross-Encoder on 10 pairs | 276 ms |
| Pipeline 2 total retrieval | 285 ms (~31x slower than baseline retrieval) |
| LLM generation (3B on CPU), Answer 1 / Answer 2 | ~99 s / ~109 s (3 queries) |

Re-ranking is much slower than plain retrieval in relative terms, but small compared with the LLM call. (LLM time is only a rough figure: CPU-only, shared machine.)

## 4. Scaling question  (write in your own words)
Points to cover: Cross-Encoder cost is linear in the number of chunks (one forward pass per query-chunk pair, nothing precomputable, ~52 ms/pair measured here on CPU -> ~52 s for 1,000 chunks, ~14 hours for 1M, per query), whereas bi-encoder search needs only one query embedding plus a (possibly approximate) vector search (flat search at 1M vectors: ~113 ms measured). Hence the two-stage design: the cheap bi-encoder gives high recall on top-k, the expensive cross-encoder gives high precision on just those k. See `results/scaling.csv`.
