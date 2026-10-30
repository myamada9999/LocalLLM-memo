# GPU × SSD 関連文献一覧

最終更新: 2026-10-30

> 公開日は、arXiv初回投稿日または公式公開日を記載。日付を一次情報で日単位まで確認できなかったものは年月表記。

## S

### Tutti: Making SSD-Backed KV Cache Practical for Long-Context LLM Serving
- 公開日: 2026-05-05
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2605.03375)
- 概要: SSD上のKV Cacheを長文LLM推論で実用的に利用するため、GPU側I/O実行、SGL、I/O scheduling、Green Contextなどを組み合わせた研究。
- 評価: S

### FlashANNS: Breaking the Storage-Compute Bottleneck in Billion-Scale ANNS
- 公開日: 2025-07
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2507.10070)
- 概要: SSD上の大規模ANNグラフをGPUで探索し、query-grained asynchronous I/O、stale-heap pipeline、graph degree最適化によってI/O待ちを隠蔽する研究。
- 評価: S

### GORIO: GPU-Centered Remote I/O for Graph ANNS over NVMe-oF
- 公開日: 2026-07-05
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2607.04415)
- 概要: NVMe-oF上のgraph ANNSで、GPUがpage miss、pending request、resumeを管理し、CPUを主にremote I/O proxyとして利用するGPU中心型アーキテクチャ。
- 評価: S

### GNStor: Design of GPU-Native High-Performance Remote All-Flash Array
- 公開日: 2026-06-03
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2606.04908)
- 概要: GPUからNVMe-oF commandを生成し、RDMA/NICを介してremote SSDへアクセスするGPU-native all-flash arrayアーキテクチャ。
- 評価: S

### Janus: Efficient Agentic LLM Serving over SSD-based Sparse KV Storage
- 公開日: 2026-09-29
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.36938)
- 概要: Agentic LLM向けにSparse KV CacheをSSDへ保存し、GPU側からNVMe I/Oを発行して必要部分のみを読み出すシステム。予測readとcorrective readを組み合わせる。
- 評価: S

### NVIDIA SCADA Server SDK / Storage Lender Architecture
- 公開日: 2026-09-30
- リンク: [一次資料 / 公式資料](https://developer.nvidia.com/blog/expanding-ai-storage-access-with-nvidia-cuobject-and-the-nvidia-scada-server-sdk/)
- 概要: GPU側へNVMe queueとdoorbell操作を公開し、CPUを通常I/Oのcontrol pathから外すためのNVIDIA公式GPU-initiated storageアーキテクチャ。
- 評価: S

### HBF-Sim: An Extensible HBF Simulator for Large-scale GPU Memory Systems
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.29246)
- 概要: High-Bandwidth FlashをGPU memory hierarchyに組み込んだ際の性能を評価するシミュレータ。GPUの細粒度アクセスとNAND page粒度の差もモデル化する。
- 評価: S

### Beyond Capacity: Scalable MoE LLM Inference via High-Bandwidth Flash with Direct GPU and HBM Paths
- 公開日: 2026-08-14
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2608.14333)
- 概要: MoE LLMの巨大weightやKV CacheをHBFへ配置し、GPU direct pathとHBM relay pathを使い分けるメモリ階層アーキテクチャ。
- 評価: S

### Bridging LLM Serving and CXL-SSDs with Chunk-Aware KV Cache Management
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.26828)
- 概要: CXL接続SSDのdevice DRAMとNANDを利用し、LLM KV Cacheをchunk単位で管理するmemory-semantic storage方式。
- 評価: S

### MemLLM: End-to-end In-Storage Acceleration of On-device Retrieval-Augmented Generation using Precomputed KV Caches
- 公開日: 2026-10-15
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.22579)
- 概要: SSD内部のNAND channel近傍でKV Cache処理やAttention計算を実行し、GPUとのデータ転送量を削減するcomputational storage方式。
- 評価: S

## A

### GIDS: Accelerating Sampling and Aggregation Operations in GNN Frameworks with GPU Initiated Direct Storage Accesses
- 公開日: 2024
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2306.16384)
- 概要: GNNのsamplingとaggregationで、GPUからSSD上のfeatureを直接取得するGPU-initiated storage方式。BaMをベースにした代表的なGPU×SSD研究。
- 評価: A

### uGDS: A User-Space Development Kit for High-Performance GPU Direct Storage
- 公開日: 2026-10-22
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.30868)
- 概要: NVMe queueをuser spaceから直接操作し、Linux kernel overheadを削減するGDS高速化方式。特に4KB I/Oで大きな性能改善を示す。
- 評価: A

### FlashAccel
- 公開日: 2026-08-22
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2607.10186)
- 概要: HBFをGPU仮想アドレス空間へ統合し、SRAM prefetchやHBMとの階層配置によってFlashをGPU memory hierarchyとして利用する研究。
- 評価: A

### A Full-Stack Characterization of High-Bandwidth Flash for KV-Centric LLM Serving
- 公開日: 2026-08-12
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2608.11668)
- 概要: HBFをLLM KV Cacheへ利用した際の性能をfull-stackで評価し、帯域向上だけではend-to-end性能が改善しない条件を分析した研究。
- 評価: A

### HBFSim: Fast and Faithful Simulation of High-Bandwidth Flash Under Real GPU Execution
- 公開日: 2026-09-09
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.09800)
- 概要: 実GPU上のPTX executionと組み合わせてHBF latencyを注入し、HBF導入時のGPU workload性能を評価するシミュレーション基盤。
- 評価: A

### Hot-Cold Tiering of HBM and High Bandwidth Flash for Agentic LLM Serving
- 公開日: 2026-08-31
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.25782)
- 概要: Agentic LLM servingで、activeなKV CacheをHBM、pausedまたはcoldなKV CacheをHBFへ配置するhot-cold tiering方式。
- 評価: A

### Characterizing High Bandwidth Flash for LLM Serving
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.39131)
- 概要: HBM、HBF、host DRAM、SSDを含む多階層memory system上でLLM servingを評価し、capacity、latency、enduranceを分析する研究。
- 評価: A

### Lachesis: Lifetime-Aware KV Cache Placement for Agent Serving across HBM and HBF
- 公開日: 2026-10-06
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.08378)
- 概要: KV Cacheの予測寿命に基づいてHBMとHBFの配置を決定し、HBFへの不要なwriteを削減するagent serving向けplacement方式。
- 評価: A

### TempoKV: Timely Staging of LLM KV Caches for Memory-Semantic Flash
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.35065)
- 概要: CXL memory-semantic flashにおいて、KV Cacheが必要になる時刻を予測し、NANDからdevice DRAMへのstaging開始時刻を制御する方式。
- 評価: A

### CXL-GPU: Pushing GPU Memory Boundaries with the Integration of CXL Technologies
- 公開日: 2025-06
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2506.15601)
- 概要: GPU側からCXL memoryやstorageを利用し、GPU memory capacityを拡張するためのCXL統合アーキテクチャ。
- 評価: A

### CoinRAG: Contextualized Information Nugget KV Cache Reuse for Long-Context RAG
- 公開日: 2026-08-07
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2608.07458)
- 概要: RAGで必要なcontext部分だけを選択し、そのKV CacheのみをSSDから再利用することで長文推論の再計算とI/Oを削減する研究。
- 評価: A

### Predictive Multi-Tier Memory Management for KV Cache in Large-Scale GPU Inference
- 公開日: 2026-08
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2604.26968)
- 概要: HBM、CPU DRAM、CXL memory、NVMe、RDMAなどを含む多階層環境で、KV Cacheの将来アクセスを予測して配置と移動を最適化する方式。
- 評価: A

### KVMem: Virtualizing Million-Token Agent Workspaces on a Consumer GPU
- 公開日: 2026-09-04
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.04852)
- 概要: GPU memory、host DRAM、NVMe SSDを階層利用し、consumer GPU上でmillion-token級のagent workspaceを仮想化する研究。
- 評価: A

### Building py-kvcache: A Performance Characterization of External KV Caching for vLLM with NVMe SSDs
- 公開日: 2026-09-10
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.11744)
- 概要: vLLMのKV CacheをNVMe SSDへ保存し、io_uring、O_DIRECT、preloadingなどの実装方式を比較評価した研究。
- 評価: A

### Range: Decoupling KV-Cache Reuse Granularity from Storage Work
- 公開日: 2026-10-15
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.21241)
- 概要: KV Cacheの論理reuse単位と物理storage I/O単位を分離し、多数の小さなKV objectによるI/O overheadを削減する方式。
- 評価: A

### Characterizing Predictability-Latency Trade-offs of KV-Cache SSD Offloading in LMCache
- 公開日: 2026-10-15
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.21249)
- 概要: LMCacheにおけるSSD offloadについて、buffered I/O、O_DIRECT、read amplification、latencyのトレードオフを評価した研究。
- 評価: A

### SARIA: Persistence-Aware KV Caching with Coordinated DRAM–SSD Reload for LLM Agents
- 公開日: 2026-10-22
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.31169)
- 概要: LLM agentのKV CacheをDRAMとSSDへ配置し、再利用可能性とSSD write量を考慮しながらreloadを協調実行する方式。
- 評価: A

### KVStream: Adaptive Parallel Storage Management for KV Caching in Large-Scale LLM Serving
- 公開日: 2026-10-22
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.31827)
- 概要: local NVMe SSDとparallel/shared filesystemからKV Cacheを並列読み出しし、storage pathを動的に使い分けるLLM serving方式。
- 評価: A

### Gandalf: Flash Shall Not Die! Increasing Flash Storage Lifetime for KV Cache Offloading in LLM Serving
- 公開日: 2026-10-22
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.30791)
- 概要: LLM KV Cacheの再利用特性とAttention typeを利用してSSDへの保存対象を選別し、Flash書き込み量と寿命を改善する研究。
- 評価: A

### NeuroPrefetcher: Storage-Aware Sparse LLM Inference via Delta Prefetching
- 公開日: 2026-08-23
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2608.22643)
- 概要: Sparse LLM inferenceで将来必要になるweightを予測し、SSDから必要部分だけをprefetchすることでI/O latencyを隠蔽する研究。
- 評価: A

### SSD-LLaMA: SSD-Native Inference for Trillion-Parameter MoE
- 公開日: 2026-09-16
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.18110)
- 概要: MoE expert weightをSSDへ保存し、連続配置、io_uring、GPU decompressionなどを利用して巨大モデルをGPU memory容量以上で実行する方式。
- 評価: A

### SpAx: Activation Sparsity with Weight Approximation for Faster LLM Decoding on Offloaded Weights
- 公開日: 2026-10-01
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.02598)
- 概要: Activation sparsityを利用して必要weightのみをSSDから読み出し、weight offloading時のI/O量を削減するLLM decoding方式。
- 評価: A

### DeepInfer-Mac: Running a 750B-Class MoE on a 24 GB Laptop
- 公開日: 2026-10-27
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.36564)
- 概要: 巨大MoEモデルのexpertをSSDへ格納し、必要なexpertだけをprefetchして限られたGPU/Unified Memory上で推論する実装。
- 評価: A

### PrairieFire: Igniting High-Throughput Updates in Large-Scale Approximate Nearest Neighbor Search
- 公開日: 2026-10-16
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.23232)
- 概要: GPUとNVMe SSDを組み合わせ、ANNS index updateとsearchを高速化する研究。CPU処理削減とSSDアクセスの最適化を重視する。
- 評価: A

### WorldPager: Zero-Stall Tiered Memory for Long-Horizon World Models on a Single Consumer GPU
- 公開日: 2026-10-26
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.35529)
- 概要: GPU、DRAM、NVMe SSDの3階層を利用し、将来必要なKV Cacheをprefetchしてworld modelのI/O stallを隠蔽する方式。
- 評価: A

### Cascade: GPU-Accelerated Out-of-Core Suffix Array Construction for LLM Corpora
- 公開日: 2026-10-11
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.15254)
- 概要: GPU、CPU DRAM、NVMe SSDを利用し、大規模LLM corpusのsuffix array構築をout-of-coreで高速化する研究。
- 評価: A

### Visual Cache Reuse for Efficient Multimodal Inference
- 公開日: 2026-10-18
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.25136)
- 概要: VLMのvisual KV CacheをSSDへ保持し、完全一致や近似一致に応じて必要なpatchのKVのみを再利用する方式。
- 評価: A

### NVIDIA SCADA Metrics Profiling in Nsight Systems
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://docs.nvidia.com/nsight-systems/UserGuide/#scada-metrics-profiling)
- 概要: NVIDIA SCADAによるGPU-initiated storage I/Oのlatency、command数、実行状況をNsight Systems上で可視化する公式profiling機能。
- 評価: A

## B

### Real Long-Term Memory for AI: A 50-Million-Token Window That Is Faster and Cheaper Than Recompute
- 公開日: 2026-10-07
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.10845)
- 概要: 長期agent memoryとして大量のKV CacheをNVMe SSDへ保存し、必要時にrestoreすることで再計算を回避する方式。
- 評価: B

### Where Should the KV Cache Live? Placement Policies Across GPU, CPU, and SSD for Long-Lived Sessions
- 公開日: 2026-09-14
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2609.16215)
- 概要: 長時間LLM sessionにおいてKV CacheをGPU、CPU DRAM、SSDのどこへ配置するかを比較評価する研究。
- 評価: B

### SARP: An On-SSD Index for KVCache with Fast Prefix Matching and Recovery
- 公開日: 2026-10-19
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.26849)
- 概要: KV Cacheのprefix matching用metadata indexをSSD上に永続化し、高速lookupとrecoveryを両立する方式。
- 評価: B

### Metrum AI: Agentic Shared KV Cache on NVMe
- 公開日: 2026
- リンク: [一次資料 / 公式資料](https://www.metrum.ai/whitepaper/agentic-shared-kv-cache-nvme)
- 概要: Agentic workloadでKV CacheをNVMe SSDへ共有保存し、LMCacheなどを利用してGPU間やsession間でreuseする実機評価。
- 評価: B

### Cross-Modal Data Co-Locality for Heterogeneous Storage Systems
- 公開日: 2026-10-25
- リンク: [一次資料 / 公式資料](https://arxiv.org/abs/2610.34595)
- 概要: Multi-modal vector dataのアクセス相関を利用し、GPU、DRAM、NVMe、remote storage間の配置を最適化する研究。
- 評価: B

### Compression-Aware GPU Buffer Management
- 公開日: 2026-09-12
- リンク: [一次資料 / 公式資料](https://doi.org/10.1007/s13222-026-00560-w)
- 概要: SSDからGPUへ転送する圧縮データを考慮し、GPU bufferとstorage I/Oを協調管理するtiering方式。
- 評価: B

### NVIDIA GPUDirect Storage 1.19 Scatter-Gather I/O
- 公開日: 2026-09
- リンク: [一次資料 / 公式資料](https://docs.nvidia.com/gpudirect-storage/release-notes/index.html)
- 概要: Fragmented GPU bufferをscatter-gather形式で扱えるようGDS APIを拡張し、GPU memoryへの直接I/Oを改善した公式機能。
- 評価: B

### MLPerf Storage v3.0
- 公開日: 2026-09-01
- リンク: [一次資料 / 公式資料](https://mlcommons.org/2026/09/mlperf-storage-v3-0-results/)
- 概要: Storage benchmarkにKV CacheやVector Database系workloadを追加し、AI storage workloadの標準評価範囲を拡張したbenchmark suite。
- 評価: B
