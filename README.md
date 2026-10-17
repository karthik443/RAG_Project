# RAG Assignment: Baseline Retrieval vs. Cross-Encoder Re-Ranking

Single notebook: [`RAG_Baseline_vs_Reranking.ipynb`](RAG_Baseline_vs_Reranking.ipynb) (outputs included).

- **Document:** *CATAN Game Rules & Almanac* (`data/catan_rules.pdf`, 16 pages)
- **Bi-Encoder:** `all-MiniLM-L6-v2` + FAISS inner product
- **Cross-Encoder:** `cross-encoder/ms-marco-MiniLM-L6-v2`
- **LLM:** `Qwen2.5-3B-Instruct` (local, CPU, greedy decoding)
- `results/` holds the raw numbers (rank shifts, latency, scaling, per-query answers) cited in the report.

## Run
```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace RAG_Baseline_vs_Reranking.ipynb
```
No API keys needed. Full run takes ~20-30 min on CPU (most of it LLM generation); needs ~8 GB RAM.
