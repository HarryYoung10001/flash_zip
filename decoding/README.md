# 在线词典与推测解码

**负责人**　组长 · `work/decoding` · [返回项目总览](../README.md)

研究问题是 LZW 在线词典能否为冻结的目标模型提供比 prompt lookup 或 n-gram 更划算的候选。候选被展开后，目标模型仍在原 token 位置上验证并更新 KV。潜在收益来自减少串行调用，取决于命中、接受长度和总验证成本。

## 首周任务

1. 使用 [runtime](../runtime/README.md) 的源模型设置比较普通 AR 和 prompt lookup。prompt lookup 用当前输出的短后缀在输入中找匹配，把匹配片段之后的 token 作为候选。逐 token 比对两路 greedy 输出。
2. 用短例子说明 LZW 词条展开、匹配前缀接受、首个不匹配位置纠正和未接受 KV 状态的处理。若做候选原型，与普通 n-gram 在相同候选数和验证 token 预算下比较，记录接受长度、目标调用次数与端到端时间。
3. 比较 SAM 和 Suffix 的索引、更新范围与候选生成。与 [profiling](../profiling/README.md) 核算候选检索、展开和验证成本。首周核查 greedy 一致性，另行用小词表检查随机采样所需的概率与拒绝修正。

## 交接与验收

交付 AR／prompt lookup 对照、带失败情况的 LZW 验证示例，以及与直接近邻的比较。若 greedy 出现不一致，先定位接受、纠正或缓存回滚逻辑。若相同验证预算下，LZW 的候选覆盖或接受长度与既有检索方法相近且总成本更高，就停止推进这版词典候选。近似接受可以单独比较质量和时间。

优先读异词表推测解码的验证算法与 SAM 的候选检索部分。[异词表推测解码](https://proceedings.mlr.press/v267/timor25a.html) · [SAM-Decoding](https://aclanthology.org/2025.acl-long.595/) · [SuffixDecoding](https://proceedings.neurips.cc/paper_files/paper/2025/hash/b7aea253ab34a773967f1e4cdea9e4fb-Abstract-Conference.html)
