# Aliyun Critique: Strategic Misjudgment in the Cloud-Native Era

> **Languages:** [English](aliyun-critique-en.md) (primary) · [中文](aliyun-critique.md)

## 1. Paradigm Shift: Storage as Database

### Past: Object Storage as a "Network Drive"
- **Core scenarios**: Image hosting, backup, archiving.
- **Core APIs**: `GET`, `PUT`, `LIST`.
- **"Edge" APIs**: `If-None-Match` (conditional write), atomic rename, strong consistency.
- **Result**: 80% compatibility delivered 99% user satisfaction. The missing edge scenarios did not matter.

### Future: Object Storage as a "Database Foundation"
- **Core scenarios**: Neon (Serverless Postgres), Delta/Iceberg (lakehouse), Lance (vector index).
- **Core APIs**: **precisely those "edge" APIs**.
  - **Conditional write (`If-None-Match`)**: The foundation of MVCC and distributed transactions.
  - **Strong consistency**: A prerequisite for ACID.
  - **Atomic rename**: Crucial to commit protocols.
- **Crisis**: The 20% of APIs that Aliyun regards as "edge scenarios" are now the **lifeline of 100%** of modern databases. Aliyun tries to solve the problem of "filling the gaps," but is in fact tearing down the foundation.

## 2. Strategic Misjudgment: The "Walled Garden" Trap

### Goal: Vendor Lock-in Through a "Full Suite"
- **Strategy**: Use OSS as a lock-in tool to push sales of DLF (Catalog), EMR (compute), MaxCompute, and Hologres.
- **Fear**: **Commoditization**. If OSS becomes a perfect, boring S3-compatible layer, it becomes a low-margin commodity. Users could run high-value open-source tools (Databricks, Snowflake, Neon, DuckDB) directly on OSS, completely bypassing Aliyun's ecosystem.

### Execution: Deliberate Friction
- **Result**: OSS is "good enough for backup" but "not good enough for databases."
- **Consequence**: It forces users into Aliyun's proprietary Java/Scala ecosystem, where proprietary patches (such as `hadoop-aliyun`) can mask incompatibilities. It shuts out modern open-source stacks (Rust/Python) that require standard APIs.

## 3. Ecosystem Disconnect: Rust/Python vs. Java

| Era | Stack | Aliyun OSS Experience |
|------|--------|----------------|
| **Past (Java/Spark)** | Hadoop ecosystem | **"Hidden"**: The `FileSystem` abstraction lets patches smooth over incompatibilities. Configuration is painful, but it works.|
| **Future (Rust/Python)** | `object_store`, Polars, DuckDB | **"Crash"**: Modern stacks assume standard S3 semantics. There is no "Hadoop middleware." `If-None-Match` failure = hard crash.|

**Conclusion**: Aliyun's compatibility was built for the fading **Java big-data era**. It completely ignores the **Rust/Python lightweight data stack** that is the future.

## 4. Qwen vs. Other Products: Two Paths

### Qwen (Success)
- **Strategy**: **Globalization + open standards + performance**.
- **Actions**:
  - Open-source the weights on HuggingFace.
  - Full compatibility with the OpenAI API, vLLM, llama.cpp.
  - Compete head-on with Llama, Mistral, GPT-4 on global benchmarks (MMLU, LiveCodeBench).
- **Result**: The only Aliyun product to break into the **global first tier**. It proves that following open standards and a performance orientation brings real influence.

### Other Products ("Self-Congratulation")
- **Products**: OSS, OceanBase, MaxCompute, Dubbo, Sentinel.
- **Strategy**: **Domestic enterprises + walled garden + ecosystem lock-in**.
- **Problems**:
  - **Closed ecosystem**: Heavy Java/Scala lock-in, proprietary protocols.
  - **Follower rather than leader**: PolarDB is a very good Aurora clone, OceanBase is a very good TPC-C score-chaser, but neither defines a new paradigm.
  - **Globally irrelevant**: Ignored by the global cloud-native community (Envoy, Kafka, Kubernetes, S3).
- **Result**: "Self-congratulation." A huge domestic installed base, but zero global momentum.

## 5. Summary

The future belongs to **"boring infrastructure"** — standardized, open, boringly reliable layers (such as S3, Linux, or Kubernetes).

By refusing to standardize OSS for the lakehouse era, Aliyun is betting on **"vendor lock-in > technical excellence."**

Yet in the era of Serverless and shared storage, **lock-in is a liability**. The winners will be those who provide the most compatible, most composable, most open infrastructure, allowing users to bring their own compute.

---

## Cross-References

This document forms a complete decision loop with the following architecture analyses:

- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: Redis as the "network RAM trap," contrasting with the "storage as database" paradigm of Aliyun OSS.
- **[Nginx Critique](nginx-critique-en.md)**: Nginx's legacy design shares the same origin as the closedness of Aliyun's Java ecosystem.
- **[MySQL Critique](mysql-critique-en.md)**: MySQL's SQL anti-patterns align with the closed ecosystem of Aliyun MaxCompute.
- **[Unified Data Layer](unified-data-layer-en.md)**: A modern multi-model database, aligned with the "storage as database" future direction of Aliyun OSS.

**Unified First Principles**: No technology worship, no swallowing the pie in the sky drawn by open source — look only at the real physical limits of hardware and team productivity.