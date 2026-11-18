# 评价、数据与失败分析

**负责人**　AI＋心理同学 · `work/evaluation` · [返回项目总览](../README.md)

研究问题是收益在哪些输入上出现，是否伴随任务质量变化。两条模型路径共用原始消息和输出预算，记录改变长度或计量单位后结论是否稳定。

## 首周任务

1. 固定自然任务小集，从 GSM8K（main）、HumanEval、CNN/DailyMail（3.0.0）及 WMT14（de-en，德译英）的 test split 按原顺序各取前 16 条，登记实际数据 revision 和样本 ID。分别使用数值 exact match、隔离环境中的 pass@1、ROUGE 和 chrF，保留截断和失败结果。
2. 用同一源 tokenizer 将自然任务输出预算设为 512 token，遵守 EOS。若最后一个 hypertoken 跨过边界，将展开后的输出截到 512 个源 token 再评分，实际多算的时间计入请求并另记超出量。给 [runtime](../runtime/README.md) 交付任务输入和评分所需字段。
3. 另准备精确重复、近似重复、低重复与数字／标识符密集四类性能输入，按 512／2048 源 token 构造。固定文本重放用于定位成本，自由生成用 128 源 token 上限。与 [profiling](../profiling/README.md) 核对输出长度、计时和词典状态。

## 交接与验收

交付固定样本选择规则、可复用评分脚本和一张配有样本数的结果表。按任务报告质量、压缩率、时间及代表性失败，至少检查一次吞吐口径是否改变方法排序。若小集中的差异被截断或个别样本主导，先记录该条件并补对照，暂不概括到整个任务。

优先读多分词路径评分及基准负载章节。[Zip2Zip++ §4.1](https://arxiv.org/abs/2610.13605v1) · [SPEED-Bench](https://proceedings.mlr.press/v306/abramovich26a.html)
