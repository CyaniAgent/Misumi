# Misumi 中文说明

Misumi 是一个用 Rust 编写的嵌入式、轻量向量与混合检索引擎，专为去中心化节点、中继和隐私优先的消息系统提供搜索功能。

> 搜索优先，而非数仓优先。Misumi 定位是嵌入到去中心化社交 / 消息平台内部的本地搜索模块，而不是独立部署的十亿级集群。

英文版：[README.md](../README.md)

## 为什么做 Misumi

去中心化实例（ActivityPub / Fediverse 风格、P2P 中继、廉价 VPS）用不起 GPU 检索集群，它们需要：

1. **混合检索：** 短文本噪声大， hashtag、@mention、URL 需要精确匹配，同时需要语义相似。
2. **廉价 CPU 亚毫秒：** 1-2 vCPU、2-4GB 内存、无 GPU，单节点 1 万 ~ 500 万条。
3. **可嵌入：** 单库 / 单二进制，零 C 依赖，增量更新，mmap 持久化。

十亿级 + 廉价 CPU + 亚毫秒不可能。单实例百万级 + 量化 + 剪枝可行。Misumi 选择后者。

## 特性（设计目标）

- **混合数据模型：** `id + dense[384] + sparse{term: impact} + filter{author, instance, lang, time, visibility}`
- **单路径混合执行：** 量化向量粗排 + 剪枝，全精度精排，RRF（k=60）融合
- **稠密端：** HNSW / IVF 图索引，优先 384 维（MiniLM 级），支持 cosine / L2 / IP
- **稀疏端：** BM25 系 + WAND / MaxScore / Block-Max 剪枝，兼容 SPLADE 权重
- **量化：** RaBitQ 1-bit（D 维 -> D-bit，相对 FP32 压缩 32x）做粗排，AVX2 / NEON / VPOPCNT 运行时检测加速距离计算
- **过滤 + 可见性：** 融合前做 `author, instance, lang, to/cc` 过滤，兼容 ActivityPub 可见性语义
- **联邦就绪：** 本地优先检索；远端只交换描述子（centroid + 词频直方图）做资源选择，结果用 RRF 合并
- **存储：** 内存图 + 压缩向量，WAL + mmap 快照，支持增量插入 / 删除

## 架构

```text
                +-------------------+
Query ------->  | Query Planner     |
                +--------+----------+
                         |
        +----------------+----------------+
        |                                 |
+-------v--------+               +--------v-------+
| Sparse branch  |               | Dense branch   |
| 倒排索引       |               | HNSW / IVF     |
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

单 vCPU 延迟预算（100 万条，384 维，目标 p99 <1ms）：

| 阶段 | 预算 |
| ---- | ---- |
| 稀疏 WAND | 0.15 ms |
| 稠密粗排（量化） | 0.40 ms |
| 精排（全精度，top-200） | 0.20 ms |
| RRF 融合 + 过滤 | 0.05 ms |
| 合计 | 0.80 ms |

内存目标：`每 100 万帖子 <0.5GB`（RaBitQ 48B + 图度数 16 + 裁剪后 posting）。

## 使用方式

```rust
// 愿景 API（尚未实现）
use misumi::{Misumi, Document, Query};

let db = Misumi::open("./data")?;
db.insert(Document {
    id: "post:123",
    dense: vec![0.12, -0.33 /* ... 384维 */],
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

以 `lib` 形式嵌入节点二进制，无需独立服务。每个实例可选薄 HTTP/gRPC 封装。

## 路线图

- [ ] Phase 0：`Flat + BM25 + RRF` 基线，验证延迟模型
- [ ] Phase 1：`HNSW + RaBitQ + Block-Max WAND`，SIMD 距离计算
- [ ] Phase 2：WAL + mmap 持久化，增量删除
- [ ] Phase 3：联邦描述子 + 选择 / 合并协议

当前状态：设计阶段，API 未稳定。见 `Dev` 分支。

## 学术基础

- Malkov & Yashunin 2016/2020, HNSW
- Subramanya et al. NeurIPS19, DiskANN / Vamana
- Chen et al. 2021, SPANN
- Gao & Long SIGMOD24, RaBitQ
- He et al. 2026, Ascend-RaBitQ (IVF-RaBitQ)
- Li et al. ASP-DAC26, pHNSW
- Formal et al. SIGIR21, SPLADE；Lassance et al. 2024, SPLADE-v3
- Lu 2024, BM25S eager 稀疏打分
- Qiao et al. SIGIR23, Guided Traversal / MaxScore 剪枝
- Cormack et al. 2009, RRF；Bruch et al. 2024 单索引混合；EAHR 2026 自适应混合
- W3C ActivityPub；Callan 联邦检索（表示 / 选择 / 合并）；FedVS KDD25；Semantica 2025

## 许可证

MIT，见 [LICENSE](../LICENSE)。Copyright (c) 2026 CyaniAgent。
