# Scalable Resume Screening and ATS Ranking System

A distributed resume screening pipeline that parses resumes with spaCy, embeds them with sentence-transformers, and ranks candidates against job descriptions using a hybrid of semantic similarity and explicit feature matching.

It replaces the conventional regex-plus-TF-IDF approach to applicant tracking with NLP-based structured extraction, dense vector retrieval, and a weighted multi-signal ranker — decomposed into independent services so that parsing, embedding, and ranking can scale separately.

---

## Why this exists

Traditional ATS ranking relies on keyword overlap, which fails in both directions: it misses candidates who describe the same skill with different vocabulary, and it rewards candidates who pad their resume with job-description keywords. Replacing lexical matching with semantic similarity addresses the first problem; combining that similarity with explicit, inspectable features (skill coverage, experience, education) keeps the score explainable rather than an opaque embedding distance.

---

## Architecture

```text
Candidate Upload
  → Ingestion API
  → Object Storage
  → Kafka / Redpanda event
  → Resume Parser        → Structured Candidate Profile
  → Embedding Service    → Vector Index
  → Ranking Service      → Ranked Shortlist
  → Recruiter API / Dashboard
```

Services communicate asynchronously rather than through direct calls, for three reasons:

- **Decoupling** — the parser does not need to know the embedding service exists, so either can be deployed or restarted independently.
- **Buffering** — parsing is fast and embedding is slow. A queue absorbs that mismatch instead of applying backpressure all the way to the uploader.
- **Replay** — because topics retain messages, the corpus can be reprocessed after an embedding model change without re-uploading anything.

PostgreSQL is the system of record. The message broker is transport, not truth.

---

## Implementation status

**Implemented**

| Service | What it does |
|---|---|
| `services/parser` | Extracts name, email, phone, skills, education, experience, certifications, and projects from PDF resumes; writes structured JSON profiles and persists them to `public.candidate_profiles` |
| `services/embedding` | Ingests job descriptions, generates `all-MiniLM-L6-v2` embeddings for both candidates and jobs, stores vectors in PostgreSQL, and builds a local FAISS index for top-k retrieval |
| `services/ranking` | Combines semantic retrieval with parser-derived features into a weighted score; persists ranked candidates to `public.job_candidate_rankings` |

**Not yet implemented**

- Learned ranking model to replace the current weighted heuristic

---

## Results

Load tested with [`hey`](https://github.com/rakyll/hey) against the FastAPI service:

```bash
hey -n 5000 -c 200 http://localhost:8000/<endpoint>
```

| Metric | Value |
|---|---|
| Concurrency (connections in flight) | 200 |
| Total requests | 5,000 |
| Duration | 42.19 s |
| Throughput | 118.5 req/s |
| Response throughput | ~344 MB/s |
| Data transferred | 13.5 GiB |
| Mean response size | 2.76 MiB |
| Latency p50 / p95 / p99 | 1.48 s / 2.89 s / 3.42 s |
| Fastest / slowest | 0.112 s / 3.84 s |
| Status codes | 5,000 × `200` |

**Finding: response serialization was the bottleneck, not request handling.** Every request returned an unbounded result set — 2.76 MiB serialized in full, regardless of what the client needed. At 118 req/s that is ~344 MB/s of response payload, which is where the time was going. Introducing server-side pagination bounds the result set at the query layer, so response size stops scaling with corpus size.

**The service did not fail, but it was already degraded.** All 5,000 requests returned `200`, which is easy to misread as headroom. A p50 of 1.48 s and a p99 of 3.42 s is well past acceptable for an interactive API — the service was returning successful responses too slowly to be usable. Status codes were the wrong success criterion.


**Limitation — no breaking point was established.** `hey` is a closed-loop generator: it holds 200 requests in flight and issues the next only when one returns, so it throttles itself to the service's own speed and cannot produce sustained overload. Little's Law confirms the test ran at full concurrency (118.5 req/s × 1.652 s ≈ 196 ≈ 200 in flight). Finding the actual ceiling requires an open-loop, fixed-arrival-rate generator.

Full methodology and configuration: [`docs/load-testing.md`](docs/load-testing.md) <!-- FILL IN — create this file -->

---

## Ranking model

The final score combines five weighted signals:

| Signal | Source |
|---|---|
| Semantic similarity | Cosine similarity between candidate and job-description embeddings |
| Skill coverage | Overlap between parsed candidate skills and job requirements |
| Experience alignment | Parsed years and role history against the requirement |
| Education match | Parsed degree and field against the requirement |
| Role relevance | Title and domain proximity |

Weights are hand-tuned and validated by inspection against known job descriptions. **This is the pipeline's main methodological limitation:** without ground-truth hiring outcomes there is no way to validate the weighting empirically. Collecting outcome labels and replacing the heuristic with a learned ranker — evaluated with NDCG against held-out data — is the intended next step.

---

## Dataset

- **Resumes:** 10000 PDF/doc/txt resumes in `data/raw/resumes/`
- **Job descriptions:** 3 in `data/raw/job_descriptions/` — software engineer, data analyst, ML engineer
- **Generated synthetic resumes using LLMs to scale the corpus from 97 real resumes to 10,000 for throughput testing**

---

## Quickstart

Requires Docker, Python 3.10+, and Linux (or WSL2).

```bash
cp .env.example .env          # fill in credentials
docker compose -f infra/docker-compose.yml up -d postgres
python3 -m venv .venv && source .venv/bin/activate
python3 -m pip install -e services/parser -e services/embedding -e services/ranking
```

Then run the three pipeline stages — parser, embedding, ranking — as described in [`docs/RUNNING.md`](docs/RUNNING.md).

---

## Repository layout

```text
docs/         Architecture, running instructions, load test methodology
infra/        Docker Compose and environment configuration
services/
  ingestion/  Resume upload API                    (not yet implemented)
  parser/     NLP parsing and profile extraction
  embedding/  Embedding generation and vector indexing
  ranking/    Candidate ranking and shortlist generation
  api/        Recruiter-facing read API            (not yet implemented)
shared/       Shared schemas, utilities, and config contracts
```

---

## Design decisions

**Redpanda over Kafka** — wire-compatible with the Kafka protocol, but a single binary with no ZooKeeper or JVM tuning. Chosen for local development simplicity; the client code is unchanged if swapped for Kafka.

**FAISS over a hosted vector database** — at this corpus size, exhaustive flat search is exact and fast. Approximate indexes (IVF, HNSW) only pay off at a scale this dataset does not reach. Milvus is the intended path for distributed search.

**`all-MiniLM-L6-v2`** — 384-dimensional embeddings with a good speed/quality tradeoff for semantic similarity. Note its 256-token input limit, which constrains how much of a resume can be embedded in a single vector.

**PostgreSQL for both profiles and vectors** — keeping structured profiles and their embeddings in one store avoids a consistency problem between two databases while the corpus is small enough for it not to matter.

**Weighted heuristic before a learned model** — a hand-tuned scorer is inspectable and requires no labels. A learned ranker is strictly better once outcome data exists, but training one on no labels is not possible.

---

## Documentation

- [Architecture](docs/architecture.md)
- [Running the pipeline](docs/running.md)
- [Load testing methodology](docs/load_testing.md)
- [Implementation roadmap](docs/implementation-roadmap.md)

---

## Contributors

-[Takshak P](https://github.com/Takshak-CS)

-[Suhas Kanamadi](https://github.com/suhas-kanamadi)
