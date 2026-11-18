# 在线开销与性能分析

**负责人**　算子优化／SNN 同学 · `work/profiling` · [返回项目总览](../README.md)

研究问题是哪些在线成本抵消了主干少算的收益。先测整条请求，再判断改算子、访存或调度能省多少。

## 首周任务

1. 接入 [runtime 的共同入口](../runtime/README.md)，分解 tokenization／LZW 更新、组合向量生成、动态输出头、主干计算和 CPU/GPU 同步。核对公开 HF 路径与原版论文所述 Triton 优化在哪些地方一致。把 `bmm`、mask 和 logits 拼接作为待测项。
2. 用 512／2048 源 token 输入，先跑 batch 1，再跑 batch 4。记录有效词条数、冷／热状态和峰值显存。每组先热身 3 次，然后交错运行两臂，各测 5 次。端到端用同步后的墙钟时间，分项 profiler 单独跑，重叠的 CPU 和 GPU 时间按时间线解释。
3. 与 [decoding](../decoding/README.md) 核对概率处理的必要条件。用小词表枚举检查字符串概率求和的成本，再分别记录候选展开与目标验证的成本。对齐概率的计算是否必要，取决于实际使用的验证算法。

## 交接与验收

给出一张耗时分解图、一张随有效词条数变化的成本图，以及一个按可消除耗时估计收益上界的优化候选。将中位数、范围、样本数和 GPU 设置交给 [evaluation](../evaluation/README.md)。若即使完全消除某个算子也只能节省很少的请求时间，就把原型工作转向占比更大的环节。

优先读原版 §3.6 和异词表推测解码 §3.4。[Zip2Zip](https://arxiv.org/abs/2506.01084) · [HyperLinear](https://github.com/epfl-dlab/zip2zip/blob/c0184ab4dc3f28b54f49e5cffd5134538a642e18/src/zip2zip/nn/linear.py) · [异词表推测解码](https://proceedings.mlr.press/v267/timor25a.html)
