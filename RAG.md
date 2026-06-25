*This project has been created as part of the 42 curriculum by gvan-box.*

---

# RAG against the Machine

A **Retrieval-Augmented Generation** (RAG) system that answers questions about the [vLLM](https://github.com/vllm-project/vllm) codebase. Given a natural-language question, the system retrieves the most relevant source passages and optionally generates a grounded answer using a local language model.

---

## Description

RAG is a technique that grounds language model answers in a specific document corpus, reducing hallucination and enabling question-answering over private or specialised knowledge bases.

This project implements a full RAG pipeline with three interchangeable retrieval strategies:

| Strategy | Method |
|---|---|
| `bm25` | Sparse keyword search (Okapi BM25) |
| `semantic` | Dense vector search (cosine similarity over sentence embeddings) |
| `hybrid` | Reciprocal Rank Fusion of BM25 + semantic results |

The corpus is the vLLM 0.10.1 source tree (~36 MB, ~2 900 files of Python and Markdown). Two document chunkers are available: a custom AST/header-based chunker and a LangChain-based splitter.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Corpus (vLLM src)                    │
│                   .py files   +   .md files                 │
└───────────────────────────┬─────────────────────────────────┘
                            │  CorpusReader
                            ▼
                    ┌───────────────┐
                    │   Chunker     │  PythonChunker / MarkdownChunker
                    │               │  LCPythonChunker / LCMarkdownChunker
                    └──────┬────────┘
                           │  list[Chunk]
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        BM25Index   SemanticIndex  (both, for hybrid)
        (rev_index  (embeddings
         .json)      .npy + chunks
                     .json)
              │            │            │
              └────────────┼────────────┘
                           │  query
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        BM25Retriever SemanticRetriever HybridRetriever
                                        (RRF fusion)
                           │
                           ▼
                      list[Chunk]   ←  top-k context
                           │
                           ▼
                  ┌────────────────┐
                  │   Generator    │  Qwen3-0.6B via transformers
                  │  VLLMGenerator │  or vLLM engine
                  └────────────────┘
                           │
                           ▼
                        Answer
```

**Component responsibilities:**

- **CorpusReader** — crawls the corpus directory, dispatches `.py` files to the Python chunker and `.md` files to the Markdown chunker.
- **BM25Index / SemanticIndex** — build and persist their respective indexes on first run; subsequent runs load from cache (`data/processed/`).
- **Retrievers** — load the index and implement `search(query, k) → list[Chunk]`.
- **HybridRetriever** — calls both sub-retrievers with a deep candidate pool, then fuses rankings with Reciprocal Rank Fusion.
- **Generator / VLLMGenerator** — build a chat prompt from the retrieved chunks and generate an answer. VLLMGenerator falls back to a HuggingFace transformers pipeline when no GPU is available.

---

## Chunking Strategy

Two families of chunkers are available, selectable via `--chunker`:

### Default chunker (`--chunker default`)

| File type | Strategy |
|---|---|
| `.py` | **AST-based**: each top-level function, async function, and class becomes its own chunk. Falls back to the Markdown chunker on syntax errors. |
| `.md` | **Header-based**: sections are split on `#` headers; blank-line paragraph breaks are used when no headers are present. |

Chunks longer than `max_chunk_size` characters are sub-sliced with a 20 % overlap (`overlap = max_chunk_size // 5`) to preserve context across boundaries.

### LangChain chunker (`--chunker lc`)

Uses `RecursiveCharacterTextSplitter.from_language(Language.PYTHON)` for Python files and `MarkdownTextSplitter` for Markdown, both with a fixed overlap of 100 characters. Character offsets are resolved by a forward scan so source spans remain accurate.

### Design choice

The AST-based chunker was preferred as the default because function and class boundaries are the natural unit of code comprehension — retrieving a complete function is more useful than an arbitrary character window. The LangChain chunker is available for comparison and as a fall-back when the AST approach produces chunks that are too large.

---

## Retrieval Method

### BM25 (`bm25`)

Classic probabilistic retrieval. Terms are tokenised, split on snake_case / camelCase boundaries, filtered for stopwords, and stemmed (Snowball). The inverted index stores per-chunk term frequencies. At query time, each query term's BM25 score contribution is:

```
BM25(t, d) = IDF(t) × tf(t,d) × (k+1) / (tf(t,d) + k × (1 - b + b × |d|/avgdl))
```

with `k = 1.5`, `b = 0.5`.

### Semantic (`semantic`)

Dense retrieval using [BAAI/bge-base-en-v1.5](https://huggingface.co/BAAI/bge-base-en-v1.5) (109 M parameters). Chunk embeddings are computed once and cached as a NumPy matrix. At query time:

1. The query is embedded (with the BGE query prefix).
2. Both the stored matrix and the query vector are unit-normalised, so cosine similarity reduces to a single matrix–vector dot product: `scores = embeddings @ query_emb`.
3. `np.argpartition` finds the top-k indices in O(n) time; only those k entries are sorted.

### Hybrid (`hybrid`)

Reciprocal Rank Fusion (RRF) merges BM25 and semantic rankings:

```
RRF_score(d) = Σ  1 / (k_rrf + rank_i(d))
```

with `k_rrf = 60`. Each retriever is queried with `max(k, 50)` candidates so a chunk ranked low by one method but high by the other can still surface after fusion. Chunks are matched by their source span `(file, start, end)` rather than by chunk ID because BM25 and semantic indexes are built independently and assign different UUIDs to the same logical chunk.

---

## Performance Analysis

Evaluation metric: **recall@k** — the proportion of ground-truth source spans that overlap (>5 %) with at least one of the top-k retrieved spans.


### Code dataset (`dataset_code_public.json`)

| k | recall@k bm25 | recall@k semantic | recall@k hybrid |
|---|---------------|-------------------|-----------------|
| 1 |      0.46     |       0.35        |       0.41      |
| 3 |      0.68     |       0.43        |       0.63      |
| 5 |      0.76     |       0.50        |       0.69      |
| 10 |     0.81     |       0.57        |       0.77      |

### Docs dataset (`dataset_docs_public.json`)

| k | recall@k bm25 | recall@k semantic | recall@k hybrid |
|---|---------------|-------------------|-----------------|
| 1 |       0.57    |      0.50         |       0.59      |
| 5 |      0.80     |      0.75         |       0.85      |
| 10 |       0.86   |      0.80         |       0.87      |


### Analysis

**BM25 dominates on code, semantic nearly matches on docs.**
Code questions tend to name specific identifiers: function names, class names, configuration keys. The BM25 tokeniser in `indexer_bm25.py` splits camelCase and snake_case with `_split_identifier`, so a query like *"SamplingParams max_tokens"* maps directly to exact tokens in the inverted index. The semantic model (BGE-base-en-v1.5) is a general-purpose text embedder not fine-tuned on code, so it encodes meaning rather than exact identifiers and loses precision on these queries. On the docs dataset the gap closes because documentation is natural language — semantic search is competitive when questions and passages share conceptual vocabulary rather than exact symbol names.

**Hybrid hurts on code, helps on docs.**
RRF's formula (`1 / (k_rrf + rank_i)`) weights both retrievers equally regardless of their relative accuracy. On the code dataset, BM25 ranks the correct chunk very highly (rank 1–2) while semantic places it much lower (rank 20–40). A chunk that BM25 ranks 1st and semantic ranks 35th scores `1/61 + 1/95 ≈ 0.027`, while a chunk both rank around 4th scores `1/64 + 1/64 ≈ 0.031` — so mutual agreement on a mediocre result can outscore BM25's confident top pick. This is RRF's known failure mode: when one retriever is substantially better than the other, fusing their signals hurts.

On the docs dataset the gain comes from **complementary error profiles**: BM25 and semantic fail on *different* questions, so RRF can rescue each other's misses. BM25's weak spot is paraphrased queries — if a question says "how does vLLM handle memory limits?" but the document says "configuring `gpu_memory_utilization`", term overlap is low and BM25 scores that chunk poorly. Semantic's weak spot is exact proper nouns and config flags — embeddings encode meaning, not exact tokens, so "what does `enforce_eager` do?" maps to a fuzzy neighbourhood rather than the specific chunk defining that flag. When these two failure modes don't overlap (which is more common on natural-language docs than on code), RRF promotes chunks that both methods agree on and covers questions that either alone would miss.

This also explains why the gain peaks at k=5 (+0.05) and shrinks at k=10 (+0.01): at k=1 a rescued chunk rarely surfaces at rank 1; at k=10 both individual methods already cover nearly all answerable questions, leaving little left to rescue.

**Takeaway.**
For a code corpus, BM25 with identifier-aware tokenisation is hard to beat without a code-specific embedding model (e.g. `jinaai/jina-embeddings-v2-base-code`). Hybrid retrieval is most effective when both retrievers are individually competitive — applying RRF when one is clearly dominant tends to dilute the stronger signal.

---

## Design Decisions

**Separate index and retriever classes.** `BM25Index` / `SemanticIndex` own all I/O and data-structure logic; `BM25Retriever` / `SemanticRetriever` own only the query-time ranking loop. This makes each class independently testable and keeps dependency injection straightforward (inject a `FakeEmbeddingModel` in tests without touching the retriever).

**Cache on first run.** Embedding 17 000+ chunks takes several minutes on CPU. The `.npy` matrix and chunk JSON are saved after the first build so every subsequent run loads in milliseconds.

**RRF key on source span.** BM25 and semantic indexes are built in separate runs, so the same chunk gets a different UUID each time. Fusing on `(file_path, first_char, last_char)` guarantees that agreement between the two methods is recognised correctly.

**Generator fallback.** `VLLMGenerator` catches any vLLM initialisation error and transparently falls back to a `transformers` pipeline on CPU. This allows the same code path to work on machines without a compatible GPU.

**Protocol-based chunker interface.** The `Chunker` protocol means any callable object with a `chunk(file_path, content, max_size) -> list[Chunk]` signature is a valid chunker — no inheritance required.

---

## Challenges Faced

**Implementing BM25 from scratch.** Rather than using an off-the-shelf BM25 library, the tokeniser, inverted index, and scoring formula were all written by hand. The main difficulty was code-aware tokenisation: plain word splitting treats `tensor_parallel_size` as a single token, which kills recall for any query that uses only part of the identifier. The solution was `_split_identifier`, which applies snake_case splitting followed by camelCase splitting using two regex substitutions, so `getModelConfig` becomes `['get', 'model', 'config']` and participates in BM25 scoring at the sub-token level. Getting the Okapi BM25 formula right — particularly the length normalisation term in the denominator — required carefully reading the original Robertson & Zaragoza paper rather than relying on intuition.

**Effective chunking with AST and Markdown.** For Python files, using `ast.parse` to extract top-level function and class definitions turned out to be much cleaner than any character-window approach: the chunk boundary is always semantically meaningful and the source span maps directly to `ast.get_source_segment`. The challenge was handling files that fail to parse (auto-generated code, type stubs, encoding issues) — a `SyntaxError` fallback to the Markdown chunker covered this. For Markdown, the priority order mattered: first try to split on `#` headers, then on blank-line paragraph breaks, then treat the whole file as one chunk. Getting character offsets right across all three cases (and then sub-slicing oversized chunks with overlap) required careful tracking of positions through the original string.

**Understanding semantic retrieval and its limits.** Getting embeddings to work was straightforward; understanding *why* they underperform BM25 on this corpus required more thought. BGE-base-en-v1.5 is trained on general text and encodes meaning rather than exact tokens. On a corpus of source code, exact identifier names (`SamplingParams`, `enforce_eager`, `gpu_memory_utilization`) carry most of the retrieval signal — and those are precisely what a general-purpose embedding model blurs together. This realisation led to exploring code-specific models such as `jinaai/jina-embeddings-v2-base-code` and `nomic-ai/nomic-embed-code`, which are already in the `_TRUST_REMOTE_CODE_MODELS` set in `indexer_semantic.py`. In practice, switching models requires a full re-embed of the corpus, which — without a GPU — takes too long to iterate on quickly.

**Understanding when RRF helps and when it hurts.** The initial expectation was that combining two retrieval signals would always improve results. The code dataset proved otherwise: hybrid was consistently *worse* than BM25 alone. Working through the RRF arithmetic made the reason clear — when one retriever is significantly stronger, equal-weight fusion dilutes the good signal rather than amplifying it. The lesson was that RRF is not a free upgrade; it is effective when both retrievers are individually competitive and cover different failure modes, which is only true on the docs dataset in this project.

**No compatible GPU: slow embeddings and no vLLM inference.** The project machine has a Radeon RX 580 (GFX803 / Polaris). PyTorch's ROCm wheels dropped support for this architecture in ROCm 5.7, making GPU-accelerated embedding and vLLM inference both unavailable. Embedding 17 000+ chunks on CPU takes around 6 minutes per run at 21 ms/chunk, which made iterating on chunk size very slow — each configuration change invalidates the cache and requires a full re-embed. The ONNX backend (`sentence-transformers[onnx]`) was explored on a separate branch and reduced per-batch time by 3–4×, but caused OOM at the default batch size of 256 and required dropping to 32. vLLM inference falls back to a HuggingFace transformers pipeline, which works but is noticeably slower than the GPU path the library is designed for.

**Finding an embedding model that improves recall.** Swapping the embedding model is a one-line change in the code but an expensive experiment in practice: every candidate model requires a multi-minute re-embed, and the recall improvement (if any) is only visible after running the full evaluation pipeline. Code-specific models (`jina-embeddings-v2-base-code`, `nomic-embed-code`) are promising but require `trust_remote_code=True` and are larger than BGE-base, making them even slower on CPU. Without a GPU to iterate quickly, it was not practical to perform a thorough model search, so BGE-base-en-v1.5 remained the default as the best-known general-purpose option.

---

## Instructions

### Requirements

- Python 3.10+
- [uv](https://docs.astral.sh/uv/) package manager

### Installation

```bash
git clone <repo-url>
cd milestone4/rag
uv sync
```

To use the vLLM generator (GPU only):

```bash
uv sync --extra vllm
```

### Build and evaluate (BM25)

```bash
make run-bm25
```

### Build and evaluate (semantic)

```bash
make run-semantic
```

### Build and evaluate (hybrid)

```bash
make run-hybrid
```

### Run tests and linting

```bash
make test
make lint
```

### Clean build artefacts

```bash
make clean
```

---

## Example Usage

**Index the corpus and search interactively:**

```bash
# BM25
uv run python3 -m src.student index
uv run python3 -m src.student search "How does vLLM handle KV cache?" --k 5

# Semantic
uv run python3 -m src.student --search_type=semantic index
uv run python3 -m src.student --search_type=semantic search "What is paged attention?" --k 5

# Hybrid
uv run python3 -m src.student --search_type=hybrid index
uv run python3 -m src.student --search_type=hybrid search "How to configure tensor parallelism?" --k 5
```

**Generate an answer for a single question:**

```bash
uv run python3 -m src.student answer "What is PagedAttention?" --k 10

# With vLLM backend (requires GPU)
uv run python3 -m src.student answer "What is PagedAttention?" \
    --k 10 --generator vllm
```

**Batch search and evaluate a dataset:**

```bash
uv run python3 -m src.student search_dataset \
    --dataset_path data/datasets/UnansweredQuestions/dataset_code_public.json \
    --k 10 \
    --save_directory data/output/my_results

uv run python3 -m src.student evaluate \
    --student_answer_path data/output/my_results/dataset_code_public.json \
    --dataset_path data/datasets/AnsweredQuestions/dataset_code_public.json \
    --k 10
```

**Use LangChain chunkers:**

```bash
uv run python3 -m src.student index --chunker lc --max_chunk_size 1500
```

---

## Resources

### RAG and retrieval

- Lewis et al. (2020) — [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) — the original RAG paper.
- Robertson & Zaragoza (2009) — [The Probabilistic Relevance Framework: BM25 and Beyond](https://www.nowpublishers.com/article/Details/INR-019) — definitive BM25 reference.
- Cormack et al. (2009) — [Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods](https://dl.acm.org/doi/10.1145/1571941.1572114) — the RRF paper.

### Embedding models

- [BAAI/bge-base-en-v1.5 on HuggingFace](https://huggingface.co/BAAI/bge-base-en-v1.5) — embedding model used for dense retrieval.
- [Sentence-Transformers documentation](https://www.sbert.net/) — library used for embedding inference.

### Tooling

- [vLLM documentation](https://docs.vllm.ai/) — the corpus and optional generation backend.
- [Qwen3 model card](https://huggingface.co/Qwen/Qwen3-0.6B) — default generation model.
- [LangChain text splitters](https://python.langchain.com/docs/how_to/#text-splitters) — used by the `lc` chunker variant.

### AI usage

Claude (Anthropic) was used throughout this project. Specific tasks included:

- Explaining how cosine similarity reduces to a dot product when vectors are unit-normalised.
- Writing Google-style docstrings for all functions and methods across the codebase.
- Explaining why BM25 and semantic indexes cannot be fused on `chunk.id` and suggesting the source-span key approach for RRF.
- code review throughout development.
