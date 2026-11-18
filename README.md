# FlashZip

## 核心研究问题

**在线 token 压缩能否降低 LLM 推理的实际耗时？** 

Zip2Zip 在请求中用 LZW 建立词典，把反复出现的原始 token 序列合成 hypertoken，减少主干处理的位置。词典更新、组合向量生成和动态输出头也会消耗时间。

我们先用开放的权重测量这笔开销，再判断缓存、算子和候选生成各有什么研究空间。

## 一些已有的阅读结果

原版 Zip2Zip 在 H100 80GB 上报告过 Phi-3.5-4B 的实测结果。输入 512、生成 256 的配置下，解码吞吐由 54.4 提高到 79.8 token/s（+46.6%）。该实现已使用**组合向量缓存、Triton kernel 和 Rust tokenizer**。[Zip2Zip §3.6，表 5](https://arxiv.org/pdf/2506.01084#page=9)

Zip2Zip++ v1 报告了压缩率与质量变化。附录 D.7 说明，作者尚未测量wall clock的解码速度，也未对 hyper-encoder 和 codec 的每步成本进行计时。我们要在同一设备和工作负载上实测速度，并报告相应的质量变化。[Zip2Zip++ 附录 D.7](https://arxiv.org/pdf/2610.13605v1#page=23)

1. **原生压缩。** 比较源 Phi 模型与公开 Zip2Zip++ 权重，测质量、压缩率、时间和显存。两者权重有别，这组结果回答部署取舍。后续缓存或算子改动与同一 Zip2Zip++ 权重的原始运行路径比较。
2. **在线词典候选。** 冻结源模型，比较普通自回归（AR）和 prompt lookup。用小例子验证 LZW 候选如何展开回原 token，由目标模型接受匹配前缀、纠正首个不匹配的位置。首周检验 greedy 输出一致性，采样分布和近似接受留待后续单独研究。

## 分工

| 工作空间 | 负责人 | 任务 | 工作分支 |
| --- | --- | --- | --- |
| [`runtime/`](runtime/README.md) | 王宇涛 | **跑通源模型与 Zip2Zip++和Zip2Zip；**<br />理清词典、输入／输出组合向量缓存，以及 KV Cache的更新流程，尤其是相对于普通模型有什么特点以及可优化点；<br />检查共享前缀、请求重置、不同请求相同 hypertoken ID。<br />**其他你自己想到的优化空间** | `work/runtime` |
| [`profiling/`](profiling/README.md) | 张晗 | **跑通源模型与 Zip2Zip++和Zip2Zip；**<br />核查已有 Triton 优化；<br />可以做这样一个实验：测量 LZW 更新、向量生成、动态输出头、mask／拼接、CPU/GPU 同步等等的时间分配问题，会不会有哪个阶段突出；<br />一些怀疑（上面这个时间测量实验也是这样提出来的）：在zip2zip中，作者报道过加速；而zip2zip修改算法之后，却没有报道。会不会后者引入的某种机制使得这个开销过大？而我们可不可以从这里找到加速空间？<br />**其他你自己想到的优化空间** | `work/profiling` |
| [`evaluation/`](evaluation/README.md) | 唐睿琪 | 阅读 Zip2Zip++ 评测与 SPEED-Bench；<br />组织共同数据及评价脚本；<br />分析重复度、数字／标识符、语言与任务要求对收益和错误的影响，尤其在不同domain的任务上，我们这个算法能否加速本身也有一定异质性。<br />**其他你自己想到的优化空间** | `work/evaluation` |
| [`decoding/`](decoding/README.md) | 游皓宇 | **跑通源模型与 Zip2Zip++和Zip2Zip；**<br />阅读“异词表”算法与 SAM；<br />复现现成 prompt lookup 的 greedy 验证；<br />以小例子核对 LZW 候选展开、接受、纠正及回滚。<br />对于原生有损压缩、冻结 target 的无损候选模块和近似接受三种方式，分别考察能否基于此进行加速算法设计。<br />复现小词表字符串概率求和成本。 | `work/decoding` |

## 共同Baseline与不同组员的Handoff

| 项目 | 固定起点 |
| --- | --- |
| 源模型 | [`microsoft/Phi-3.5-mini-instruct`](https://huggingface.co/microsoft/Phi-3.5-mini-instruct/tree/2fe192450127e6a83f7441aef6e3ca586c338b77)，revision `2fe192450127e6a83f7441aef6e3ca586c338b77` |
| 压缩模型 | [`epfl-dlab/zip2zip-pp-Phi-3.5-mini-instruct`](https://huggingface.co/epfl-dlab/zip2zip-pp-Phi-3.5-mini-instruct/tree/6531c3c9490e4510c6fc9101d806e4ffa945f5bb)，`hf` 推理导出 revision `6531c3c9490e4510c6fc9101d806e4ffa945f5bb` |
| 上游代码 | [`epfl-dlab/zip2zip`](https://github.com/epfl-dlab/zip2zip/tree/c0184ab4dc3f28b54f49e5cffd5134538a642e18)，commit `c0184ab4dc3f28b54f49e5cffd5134538a642e18` |
| 运行条件 | 自行适配。完全可以按照单卡规划。以实际拿到的 GPU 锁定环境，固定BF16、attention backend、greedy 和 chat template，先 batch 1，后 batch 4。 |

- **运行入口由 runtime 维护，样本由 evaluation 维护，分项计时由 profiling 接入。** 输入统一为原始文本或消息。输出记录展开文本、用源 tokenizer 计量的输入和输出 token 数、实际处理的压缩单元数、EOS 或截断原因和计时信息。样本 ID、模型版本、batch 和词典冷／热状态随结果记录。
- **回答不同问题的两种负载。** 
  1. 自然任务衡量质量与真实生成时间，统一使用展开后输出预算。
  2. 性能诊断用 512／2048 源 token 输入和固定文本重放定位成本，再用自由生成测量真实延迟。

## 本地模型、数据与实验协作

我们各自在自己的机器上跑实验，目录和版本约定统一。

- `models/` 放本地模型权重。runtime 负责确认模型 ID、revision 和加载方式，大家按分工下载自己需要的模型。源 Phi、Zip2Zip++ 和原版 Zip2Zip 的权重及适配器合计约 17.2 GB，原版可以复用源 Phi 的权重。
- `data/` 放本地评测数据和性能测试输入。evaluation 负责确定数据版本、split、样本 ID 和构造规则，大家按同一份清单准备数据。权重和下载的数据留在本地，GitHub 同步加载与评测代码、版本和样本清单。

这样做的代价是每个人都要在自己的机器上配环境，并跑通自己会用到的基线。好处是基线跑通后，缓存、性能分析、评测和解码可以并行推进。完整评测按分工执行，其他人先用少量固定样本确认自己的入口能跑。

速度对照在同一台机器上成对跑。例如，测 Zip2Zip++ 的加速比时，两种模型都在同一张卡上依次运行，固定输入、精度和生成预算。其他机器的结果按 GPU 分开汇报。
