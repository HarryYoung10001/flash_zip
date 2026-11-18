# 运行、缓存与请求状态

**负责人**　KV Management 同学 · `work/runtime` · [返回项目总览](../README.md)

研究问题是压缩减少主干位置后，词典、组合向量缓存和 KV 分别新增或节省了什么成本。第一步是让全组在同一个入口上运行两套权重。

## 首周任务

1. 按总览中的固定版本跑通源 Phi 和 Zip2Zip++ 的 `hf` 推理导出。先用 batch 1 检查输入与生成，再交给 profiling 测 512／2048 的配置。记录依赖版本、设备和 attention backend。
2. 追踪在线词典、输入和输出组合向量缓存，以及主干 KV 的创建、更新和释放。用两个请求检查重置和隔离，覆盖相同局部 hypertoken ID 对应不同原 token 序列的情况。
3. 记录有效词条数、压缩位置数和缓存命中。若尝试跨请求复用组合向量，用原 token 序列和模型／编码器版本做键，避免把一次请求的局部 ID 带入另一次请求。

## 交接与验收

提供可独立运行的共同入口、成功运行的最小命令和一张状态图。把阶段事件交给 [profiling](../profiling/README.md)，把展开文本、计数与停止原因交给 [evaluation](../evaluation/README.md)，并与 [decoding](../decoding/README.md) 共用源模型加载和输入模板。

若隔离检查发现串用，先定位并修复状态边界。若命中率高而向量缓存耗时占比很小，停止把这项缓存优化作为首周主攻。

优先读原版 §3.6、Zip2Zip++ §3 和公开代码的词典管理入口。[Zip2Zip](https://arxiv.org/abs/2506.01084) · [Zip2Zip++ v1](https://arxiv.org/abs/2610.13605v1) · [CodebookManager](https://github.com/epfl-dlab/zip2zip/blob/c0184ab4dc3f28b54f49e5cffd5134538a642e18/src/zip2zip/codebook.py)
