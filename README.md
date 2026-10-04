# Misumi

Misumi is an embedded, lightweight vector & hybrid search engine written in Rust, purpose-built for decentralized nodes, relays, and privacy-first message systems.

> Search-first, not warehouse-first. Misumi is designed to be embedded into a decentralized social / messaging platform as its local search function, not operated as a standalone billion-scale cluster.

Chinese version: [docs/README.zh.md](docs/README.zh.md)

## Why Misumi

Decentralized instances (ActivityPub / Fediverse-style, P2P relays, single-board servers) cannot afford GPU search clusters. They need:

1. **Hybrid retrieval:** short noisy posts need exact lexical match (hashtag, mention, URL) + semantic similarity.
2. **Sub-millisecond on cheap CPU:** 1-2 vCPU, 2-4 GB RAM, no GPU, single node with 10K-5M items.
3. **Embeddable:** single library / single binary, zero C dependencies, incremental updates, mmap persistence.

Billion-scale + cheap CPU + sub-ms is impossible. Single-instance million-scale + quantization + pruning is feasible. Misumi chooses the latter.

## Features (Design Target)

- **Hybrid data model:** `id + dense[384] + sparse{term: impact} + filter{author, instance, lang, time, visibility}`
- **Single-path hybrid execution:** coarse rank on quantized vectors with pruning, fine re-rank on full precision, fused by Reciprocal Rank Fusion (RRF, k=60)
- **Dense:** HNSW / IVF graph, 384-dim first (MiniLM-class), cosine / L2 / IP
- **Sparse:** BM25-family with WAND / MaxScore / Block-Max pruning, SPLADE-compatible impact scores
- **Quantization:** RaBitQ 1-bit (D-dim -> D-bit, 32x vs FP32) for coarse rank, SIMD distance via AVX2 / NEON / VPOPCNT with runtime detection
- **Filter + visibility:** `author, instance, lang, to/cc` aware filtering before fusion, ActivityPub-compatible
- **Federated-ready:** local-first search; remote descriptors (centroids + term histograms) for resource selection, RRF for result merging
- **Storage:** in-memory graph + compressed vectors, WAL + mmap snapshot, incremental insert / delete

## Architecture

```text
                +-------------------+
Query ------->  | Query Planner     |
                +--------+----------+
                         |
        +----------------+----------------+
        |                                 |
+-------v--------+               +--------v-------+
| Sparse branch  |               | Dense branch   |
| inverted index |               | HNSW / IVF     |
| BM25 + WAND    |               | RaBitQ 1-bit   |
| Top-200        |               | Top-200        |
+-------+--------+               +--------+-------+
        |                                 |
        +----------------+----------------+
                         |
                +--------v----------+
                | Fusion (RRF)      |
                | + filter + rerank |
                | Top-K             |
                +-------------------+
```

Latency budget on 1 vCPU (1M items, dim=384, target p99 <1ms):

| Stage | Budget |
| ----- | ------ |
| Sparse WAND | 0.15 ms |
| Dense coarse (quantized) | 0.40 ms |
| Fine re-rank (full precision, top-200) | 0.20 ms |
| RRF fusion + filter | 0.05 ms |
| Total | 0.80 ms |

Memory target: `<0.5 GB per 1M posts` (48 B RaBitQ code + graph degree 16 + clipped postings).

## Intended Use

```rust
// Vision API (not yet implemented)
use misumi::{Misumi, Document, Query};

let db = Misumi::open("./data")?;
db.insert(Document {
    id: "post:123",
    dense: vec![0.12, -0.33 /* ... 384d */],
    sparse: vec![("rust".into(), 12), ("fediverse".into(), 8)],
    lang: "en",
    visibility: "public",
})?;

let hits = db.search(Query {
    dense: query_embedding,
    sparse: query_terms,
    top_k: 10,
    filter_lang: Some("en"),
})?;
```

Embed as `lib` in your node binary. No server required. Optional thin HTTP/gRPC wrapper per instance.

## Roadmap

- [ ] Phase 0: `Flat + BM25 + RRF` baseline, latency model validation
- [ ] Phase 1: `HNSW + RaBitQ + Block-Max WAND`, SIMD distance
- [ ] Phase 2: WAL + mmap persistence, incremental delete
- [ ] Phase 3: Federated descriptors + selection / merging protocol

Current status: design phase, no stable API yet. See branch `Dev`.

## Research Basis

- Malkov & Yashunin 2016/2020, HNSW
- Subramanya et al. NeurIPS19, DiskANN / Vamana
- Chen et al. 2021, SPANN
- Gao & Long SIGMOD24, RaBitQ
- He et al. 2026, Ascend-RaBitQ (IVF-RaBitQ)
- Li et al. ASP-DAC26, pHNSW
- Formal et al. SIGIR21, SPLADE; Lassance et al. 2024, SPLADE-v3
- Lu 2024, BM25S eager sparse scoring
- Qiao et al. SIGIR23, Guided Traversal / MaxScore pruning
- Cormack et al. 2009, RRF; Bruch et al. 2024 single-index hybrid; EAHR 2026 adaptive hybrid
- W3C ActivityPub; Callan federated search (representation / selection / merging); FedVS KDD25; Semantica 2025

## License

MIT, see [LICENSE](LICENSE). Copyright (c) 2026 CyaniAgent.
