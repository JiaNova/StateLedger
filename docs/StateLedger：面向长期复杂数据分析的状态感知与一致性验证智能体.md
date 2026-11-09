# StateLedger：面向长期复杂数据分析的状态感知与一致性验证智能体

**英文暂定标题：** StateLedger: Intent-Grounded and Provenance-Aware State Transitions for Reliable Long-Horizon Data Analysis Agents

**研究方向：** Agentic Data Science / Long-Horizon Agents / Data Provenance / Reliable Data Systems

**目标会议：** KDD 2027 Research Track

**文档版本：** V1.0 — Research Design & Development Specification

**研究设计日期：** 2026-11-04

# 0. 研究方案摘要

## 0.1 研究定位

本研究不再试图构建一个功能更加丰富的通用数据分析智能体，而是针对现有智能体在长期、多轮、动态数据分析中存在的状态管理问题，设计一套能够理解用户状态修改意图、维护分析历史、识别计算依赖、正确更新中间结果并验证状态转换的机制。

核心问题：

> 当用户在持续数十轮甚至数百轮的数据分析过程中，不断修改分析对象、指标定义、筛选条件和分析假设时，如何保证智能体始终使用正确的历史分析状态，并且不会将过期的中间结果错误地用于后续分析？

## 0.2 最终拟解决的三个关键问题

| 问题               | 具体表现                                 | 计划解决机制                           |
| ------------------ | ---------------------------------------- | -------------------------------------- |
| 意图与状态映射错误 | 用户说的修改与智能体实际修改的状态不一致 | Intent-grounded State Patch            |
| 依赖传播与结果失效 | 修改上游条件后，下游结果仍然使用旧数据   | Semantic Dependency-aware Invalidation |
| 状态转换不安全     | 回滚、临时假设和多分支分析相互污染       | Transactional State Transition         |

## 0.3 预期研究贡献

本研究暂定三项核心贡献：

**Contribution 1：面向对话式数据分析的语义状态转换建模。**

建立用户自然语言指令与历史分析状态之间的显式映射，解决分析状态锚点识别、状态修改范围识别和歧义处理问题。

**Contribution 2：语义依赖感知的分析状态更新算法。**

将分析状态的语义属性与实际执行产生的数据血缘联系起来，识别需要重新计算的下游节点，避免过期结果继续传播。

**Contribution 3：面向长程 Agent 的事务式状态管理机制。**

将更新、临时反事实分析、回滚和多状态组合转化为具有明确提交、验证和恢复语义的操作。

注意：上述三项是拟议贡献，不代表已经证明它们具有足够的新颖性。必须通过与 Wrattler、noWorkflow、Data Interpreter、LongDS 等工作进行对比，才能最终确定贡献边界。

# 1. 研究背景与问题定义

## 1.1 当前数据分析智能体的典型工作方式

目前大多数数据分析 Agent 采用类似以下的流程：

```
用户提出分析任务
       |
       v
LLM理解需求
       |
       v
制定分析计划
       |
       v
生成Python/SQL代码
       |
       v
执行代码
       |
       v
获取执行结果
       |
       v
总结并回答用户
```

对于单轮任务，这种模式通常能够完成：

- 读取 CSV。
- 数据清洗。
- 描述性统计。
- 相关性分析。
- 机器学习建模。
- 生成图表。
- 输出分析报告。

但是，真实的数据分析通常不是单轮任务。

用户会根据之前的结果持续提出新的要求。

例如：

| 轮次 | 用户指令                                      |
| ---- | --------------------------------------------- |
| T1   | 分析 1995—2023 年所有国家的 GDP 变化          |
| T2   | 只保留亚洲国家                                |
| T3   | 将 GDP 替换成人均 GDP                         |
| T4   | 比较亚洲国家与欧洲国家                        |
| T5   | 回到 T2 的状态，但将时间范围改成 2010—2023 年 |
| T6   | 比较 T3 和 T5 的结果                          |
| T7   | 恢复最初的分析条件                            |
| T8   | 在最初结果基础上增加人均 GDP 指标             |

这些操作不仅需要记住用户说过什么，还需要准确维护：

1. 当前分析数据是什么？
2. 当前筛选条件是什么？
3. 当前指标定义是什么？
4. 当前结果由哪些操作产生？
5. 某个历史结果是否仍然有效？
6. 用户所说的之前那个版本究竟是哪一个？
7. 当前修改应该影响主分析状态还是仅影响一个临时分支？

## 1.2 为什么简单增加上下文记忆无法解决问题？

假设智能体保存了全部历史对话：

```
T1：分析所有国家GDP
T2：只保留亚洲
T3：改成人均GDP
T4：增加欧洲
T5：回到T2
```

即使历史对话完整保留，智能体也不一定知道：

- T2 对应的数据对象是否仍然存在。
- T3 是否修改了原始 DataFrame。
- T4 是否覆盖了 T3 的结果。
- T5 应该恢复哪些参数。
- 之前生成的图表是否依赖已经修改的数据。

因此：

**Conversation Memory（对话记忆）与 Analytical State（分析状态）必须分开管理。**

对话记忆记录交流历史；分析状态记录真实的数据、计算和分析结果。

这是本研究最基本的设计前提。

## 1.3 研究范围

本研究第一阶段明确限制为：

- 结构化表格数据。
- CSV、Parquet 等常见格式。
- Python/Pandas 数据处理。
- DuckDB/SQL 查询。
- 多轮分析任务。
- 多分支状态演化。
- 可追踪的分析操作和结果。

暂时不研究：

- 多模态数据分析。
- 自主训练大型语言模型。
- 完整的统计推断正确性保证。
- 医学诊断和因果发现。
- 任意编程语言的完整动态依赖分析。
- 分布式大规模增量计算。

原因是研究范围必须足够集中，避免把状态管理、模型训练、统计推断和多模态分析混合成一个无法充分验证的系统。

# 2. 相关工作与技术基础

本节是正式研究方案的文献依据。

重点不是简单说明论文提出了什么，而是明确：

- 论文实际解决了什么。
- 可以复用哪些设计。
- 哪些技术不能重复作为创新。
- 与 StateLedger 的真正区别是什么。

## 2.1 LongDS-Bench：长期分析状态的评测依据

论文：

LongDS-Bench: On the Failure of Long-Horizon Agentic Data Analysis

EMNLP 2026 Main Conference。

论文：https://arxiv.org/abs/2605.30434

数据集：https://huggingface.co/datasets/zjunlp/LongDS

### 2.1.1 论文的核心设计

LongDS 不是普通的数据分析问答集合。

它从真实 Kaggle 分析 Notebook 出发，构造长期交互任务。

其数据构建过程包含：

1. 收集 64 个竞赛和数据集来源。
2. 每个来源筛选多个高质量 Notebook。
3. 初始筛选出 77 个可执行 Notebook。
4. 构建包含历史依赖的长程任务。
5. 经过专家审查、参考代码执行和一致性检查。
6. 最终保留 68 个任务。

每个任务都包含：

- 用户请求。
- 可执行参考代码。
- 参考答案。
- 状态演化标签。
- 跨轮次依赖信息。

### 2.1.2 六种状态演化模式

LongDS 定义的状态演化模式应直接作为 StateLedger 的任务类型设计依据。

| 类型                    | 含义               | StateLedger 中的处理方式 |
| ----------------------- | ------------------ | ------------------------ |
| Initialization          | 初始化分析状态     | 创建新状态               |
| Inheritance             | 继承之前的分析状态 | 引用历史状态             |
| Update                  | 修改默认分析条件   | 创建新版本               |
| Counterfactual          | 临时改变假设       | 创建独立分支             |
| Rollback                | 恢复历史分析状态   | 定位并恢复历史版本       |
| Multi-state Composition | 组合多个分析状态   | 创建组合状态             |

这些操作类型不是我们的原创贡献。

它们是已有 Benchmark 已经明确提出的评测分类。

### 2.1.3 论文的关键实验发现

LongDS Full 包含：

- 68 个任务。
- 2,225 轮交互。
- 6 个领域。
- 平均每个任务约 33 轮。
- 平均依赖跨度 11.3 轮。

论文的评价采用任务级宏平均：
$$
\frac{1}{M}
\sum_{i=1}^{M}
\left(
\frac{1}{N_i}
\sum_{j=1}^{N_i}s_{i,j}
\right)
$$
其中：

- (M)：任务数量。
- (N_i)：第 (i) 个任务的轮数。
- (s_{i,j})：第 (i) 个任务第 (j) 轮是否正确。

其重要实验发现：

- 最好的模型平均准确率仍低于 50%。
- 任务进度增加时，模型表现明显下降。
- 依赖跨度和依赖数量增加时，准确率下降。
- 长程错误占模型失败案例的 52%—69%。
- 其中 Cascade Error（错误级联）是重要的失败来源。

论文还进行了环境 Reset 实验，发现重置可能帮助状态已经损坏的任务，却会破坏表现较好的任务中有价值的历史状态。

这意味着：

**简单地清空上下文或者重启 Python Kernel，并不能解决长期状态管理问题。**

### 2.1.4 我们可以直接复用什么？

直接复用：

1. 六种状态演化类型。
2. 长期多轮任务结构。
3. 可执行参考代码。
4. 参考答案。
5. 任务级宏平均指标。
6. 长程错误分类方法。
7. Reset 对照实验设计。

我们需要新增的是：

LongDS 主要评价智能体是否给出了正确的答案。

我们进一步评价智能体是否使用了正确的状态、是否进行了正确的状态转换，以及中间结果是否具有正确的来源和依赖关系。

这就是 LongDS 的评测基础与 StateLedger 的研究差异。

## 2.2 Data Interpreter：任务图和执行验证机制

论文：

Data Interpreter: An LLM Agent for Data Science

Findings of ACL 2025。

论文：https://aclanthology.org/2025.findings-acl.1016/

### 2.2.1 核心框架

Data Interpreter 的重要思想是采用 Hierarchical Graph Modeling。

传统 Agent 通常采用线性任务规划：

```
数据清洗
   |
特征工程
   |
模型训练
   |
模型评估
```

但真实数据分析中存在：

- 任务依赖。
- 分支关系。
- 动态子任务。
- 递归依赖。
- 前后任务相互影响。

因此，Data Interpreter 使用图结构组织分析任务。

其重要机制包括：

1. Hierarchical Task Graph。
2. 动态子节点生成。
3. Programmable Node Generation。
4. Stateful Graph Executor。
5. Progressive Verification。

它的核心思想是：每个子任务可以生成代码、执行代码、检查结果，并根据反馈继续修正。

### 2.2.2 可复用的设计

我们可以直接复用它的任务规划思想：

- 任务节点结构。
- 节点间依赖。
- 失败后的局部重规划。
- 节点级执行反馈。
- 渐进式验证。

尤其是 Stateful Graph Executor，值得在代码设计时参考。

但是必须注意：

**Task Graph 不等于 Analytical State Graph。**

任务图描述接下来需要做什么。

分析状态则描述目前的数据、指标、历史结果和版本到底是什么。

StateLedger 必须把两者分开。

### 2.2.3 论文不足与研究机会

Data Interpreter 的重点是任务分解、节点生成和执行反馈。

我们需要进一步研究：

- 已经执行的节点是否对应不可变的数据版本？
- 用户修改指标定义后，如何识别所有受影响的历史结果？
- 临时分支是否会污染默认分析状态？
- 回滚时能否精确恢复历史分析对象？
- 是否能够证明被复用的结果仍然对应当前状态？

这些是本研究需要进一步验证的内容，不能简单断言原论文的所有实现都没有这些功能。

## 2.3 EvoDS：上下文管理与技能学习

论文：

EvoDS: Self-Evolving Autonomous Data Science Agent with Skill Learning and Context Management

KDD 2026。

DOI：[10.1145/3770855.3818002](https://doi.org/10.1145/3770855.3818002)

论文：https://arxiv.org/abs/2606.03841

### 2.3.1 核心机制

EvoDS 提出了：

**Autonomous Skill Acquisition（ASA）**

让智能体从历史任务中自动提取、验证和复用技能。

**Adaptive Context Compression（ACC）**

通过全局和局部上下文管理，减少长程任务中的上下文冗余。

其多智能体框架采用 Manager 和 Sub-agent 分工。

论文还使用 SFT 与在线强化学习联合训练。

其中理论分析包括：

- 层次化工具选择的错误概率分析。
- 信息瓶颈视角下的上下文压缩目标。

### 2.3.2 可以复用的技术

我们不需要重新发明上下文压缩机制。

建议直接使用：

- 全局任务摘要。
- 当前状态摘要。
- 局部操作上下文。
- 技能检索。

但是需要额外保存完整的结构化分析状态。

即使上下文被压缩，状态图也不能被压缩成一段无法验证的自然语言描述。

### 2.3.3 研究边界

EvoDS 的核心是让 Agent 学习更多技能并有效管理上下文。

StateLedger 的核心不是技能学习，也不是强化学习。

我们研究的是用户分析意图与实际数据执行状态之间的一致性。

两者应该作为不同方向比较，而不是重复开发。

## 2.4 DataPRM：过程级验证机制

论文：

Rewarding the Scientific Process: Process-Level Reward Modeling for Agentic Data Analysis

KDD 2026。

论文：https://arxiv.org/abs/2604.24198

### 2.4.1 关键设计

DataPRM 指出数据分析任务存在 Silent Errors。

代码可能正常运行，但是分析步骤存在逻辑错误。

它采用环境交互式验证：
$$
(\hat z_k,\hat a_k,\hat o_k)
$$
其中：

- ($\hat z_k$)：验证过程中的推理状态。
- ($\hat a_k$)：验证动作。
- ($\hat o_k$)：环境观察结果。

通过多轮交互更新验证上下文：
$$
h^{PRM}*{t,k}\oplus\kappa*{t,k}
$$
另外，DataPRM 使用三分类奖励：
$$
r_t\in{0,0.5,1}
$$
分别对应：

| 奖励 | 含义                 |
| ---- | -------------------- |
| 1    | 正确且推进任务的步骤 |
| 0.5  | 可修复的错误         |
| 0    | 不可恢复的错误       |

这一设计值得借鉴，因为不能将所有执行失败都当作相同性质的错误。

### 2.4.2 对 StateLedger 的启发

我们应该区分：

- 代码执行失败。
- 数据 Schema 不匹配。
- 状态引用错误。
- 依赖关系缺失。
- 过期结果被使用。
- 状态提交失败。

不同错误应该有不同的恢复策略。

例如：

状态版本引用错误可以尝试重新定位历史状态；数据依赖错误需要重新计算；如果语义不明确，则应该要求用户澄清。

但不建议第一版就训练新的 Process Reward Model。

先使用规则验证和执行结果检查，保证核心机制能够被复现。

## 2.5 Wrattler：必须认真对待的先行系统

论文：

Wrattler: Reproducible, Live and Polyglot Notebooks

TaPP 2018。

论文：https://www.usenix.org/system/files/conference/tapp2018/tapp2018-paper-petricek.pdf

这是 StateLedger 研究中最重要的先行系统之一。

Wrattler 已经实现：

1. 状态管理与代码执行分离。
2. 依赖图。
3. 节点哈希。
4. 依赖变化后的失效传播。
5. 计算缓存。
6. 不可变 DataFrame 存储。
7. 历史版本。
8. 回滚。
9. 多语言执行环境。

它的基本 Hash 计算思想可以表示为：
$$
h(v)=H(kind(v)\cup antecedents(v))
$$
当节点及其依赖不变时，可以复用之前计算的结果。

Wrattler 的数据存储还可以保存 DataFrame 的类型、语义信息、数据来源等元数据。

### 对我们的直接影响

以下内容不能作为 StateLedger 的原创贡献：

- 使用 DAG。
- 对节点进行哈希。
- 只重新执行发生变化的下游节点。
- 保存历史结果。
- 支持回滚。
- 分离状态与执行环境。

Wrattler 主要针对可解析的 Notebook 代码及其依赖图。

它的 Python 和 JavaScript 节点主要采用 Cell 级别的粒度，而 TheGamma 可以进行更细粒度的语法分析。其原型当时也没有实现完整的大数据和元数据功能。

我们的差异必须来自 LLM Agent 的对话式状态语义：

用户可能通过自然语言引用一个历史版本、改变某个指标的含义、临时创建一个分析分支，或者组合两个之前的分析结果。

我们要研究如何把这种自然语言意图准确转换成有约束的状态转换，而不是重新实现 Wrattler。

## 2.6 noWorkflow：Python 数据血缘追踪

论文：

noWorkflow: A Tool for Collecting, Analyzing, and Managing Provenance from Python Scripts

PVLDB 2017。

论文：https://www.vldb.org/pvldb/vol10/p1841-pimentel.pdf

noWorkflow 的核心是自动收集 Python 执行来源信息。

它通过：

- AST。
- Python Bytecode。
- Runtime Profiling。
- Execution Tracing。
- 文件内容哈希。
- 环境和依赖记录。

记录脚本执行过程。

它能够记录：

- 脚本和模块。
- 函数调用。
- 参数和返回值。
- 全局变量。
- 文件读写。
- 环境依赖。
- 不同实验版本。

这对我们的执行追踪模块非常有参考价值。

**开发建议：不要从零开始实现一个完整的 Python 动态追踪系统。**

先使用明确的输入输出声明和运行时 API；如果需要更细粒度的 Python 依赖追踪，再评估 noWorkflow 的兼容性或借鉴其 AST/Runtime Tracing 机制。

## 2.7 Loops：Notebook 历史和分析分支的可视化

论文：

Loops: Leveraging Provenance and Visualization to Support Exploratory Data Analysis in Notebooks

IEEE Transactions on Visualization and Computer Graphics，2025 年正式出版。

论文：https://doi.org/10.1109/TVCG.2024.3456186

Loops 记录 Notebook 历史，展示：

- 代码变化。
- 数据表变化。
- 图表变化。
- 分析版本之间的差异。

其一个重要区别是：它的设计目标主要是帮助分析者理解探索过程，而不是提供完整的、能够保证重现性的变量级执行来源追踪。

这提醒我们：

StateLedger 最终不仅应该维护机器可读状态，也应该能够解释一次状态转换究竟修改了什么。

但可视化不是本论文的核心算法。

## 2.8 DBSP：真正的增量视图维护算法

论文：

DBSP: Automatic Incremental View Maintenance for Rich Query Languages

PVLDB 2023。

论文：https://www.vldb.org/pvldb/vol16/p1601-budiu.pdf

这是我们后续可能使用的高级优化技术。

DBSP 通过数学上的积分和微分操作，建立增量视图维护机制。

对于计算函数 (Q)，其增量版本定义为：
$$
Q^\Delta=D\circ Q\circ I
$$
其中：

- (I)：积分，将变化序列还原为完整状态。
- (Q)：原始计算。
- (D)：微分，提取结果变化。

对于多个输入：
$$
T^\Delta(a,b)=D(T(I(a),I(b)))
$$
论文进一步证明了增量操作的组合性质，并给出了关系查询转换为增量电路的算法。

例如，对于连接操作：
$$
\Delta A\Join B
+
A\Join\Delta B
+
\Delta A\Join\Delta B
$$
这说明当输入数据只发生少量变化时，可以只计算受到影响的部分，而不是重新计算整个结果。

### 但这里有一个重要限制

DBSP 研究的是具有明确数学语义的数据库计算和查询语言。

它不能直接应用于任意 LLM 生成的 Python 代码。

所以：

- 第一阶段采用节点级依赖失效和选择性重算。
- 第二阶段仅对可解析的 SQL、过滤、投影、连接、聚合等操作研究细粒度增量计算。
- 不对任意 Python 函数声称具有 DBSP 级别的增量计算保证。

# 3. 最终研究问题与研究假设

结合上述文献，我建议正式定义五个 Research Questions。

## RQ1：自然语言意图能否被可靠地转换为分析状态修改？

用户的自然语言请求可能涉及：

- 当前状态。
- 历史状态。
- 临时状态。
- 多个状态。
- 不明确的状态引用。

研究如何识别用户实际想要修改的状态和属性。

**H1：** 引入结构化状态补丁和历史版本锚点解析，可以提高状态引用与状态转换的准确性。

## RQ2：语义依赖追踪能否减少过期结果传播？

研究用户修改分析条件后，如何自动识别受到影响的下游分析节点。

**H2：** 与仅使用对话历史和普通任务 DAG 相比，显式的语义依赖关系能够降低 Stale Result Rate 和 Cascade Error Rate。

## RQ3：事务式状态转换能否保证不同分析分支相互隔离？

研究：

- 更新。
- 临时假设。
- 回滚。
- 多状态组合。

**H3：** 使用不可变状态快照和分支隔离机制，可以提高 Rollback Accuracy 和 Branch Isolation Accuracy。

## RQ4：选择性重新计算是否能够降低计算成本？

**H4：** 当用户修改只影响分析图中较小部分节点时，依赖感知的选择性重新计算能够降低执行成本；当上游数据发生全局变化时，其优势会减弱甚至消失。

这是一项需要实验检验的条件性假设，而不是无条件成立的结论。

## RQ5：在不确定和不完整的执行环境下，StateLedger 是否仍然可靠？

研究：

- 隐式全局变量。
- 随机数。
- 外部文件变化。
- 环境变化。
- 依赖关系遗漏。
- 运行中断。

**H5：** 引入保守的不可缓存执行模式和提交前验证，可以在一定程度上限制隐藏依赖导致的错误复用。

# 4. 核心数学建模

这是整个方案最重要的理论部分。

## 4.1 首先区分三种图

不要将所有信息全部放进一个 DAG。

建议采用三层结构。

### 第一层：Task Graph

任务图。

表示 Agent 为当前用户请求制定的执行计划。

例如：

```
读取数据
   |
筛选亚洲国家
   |
计算GDP增长率
   |
生成图表
```

它可以随着用户新请求而改变。

### 第二层：Provenance Graph

数据来源与执行依赖图。

记录：

- 哪些数据被读取。
- 哪段代码被执行。
- 使用了哪些参数。
- 生成了哪些中间结果。
- 最终结果依赖哪些上游节点。

这是一个持久化的执行记录。

### 第三层：State Version Graph

状态版本图。

记录：

- 当前分析状态。
- 历史分析状态。
- 状态继承关系。
- 状态更新。
- 临时分支。
- 回滚。
- 多状态组合。

这三种图不能互相替代。

## 4.2 状态形式化定义

定义第 (t) 轮的分析状态为：
$$
S_t=(b_t,v_t,r_t,C_t,E_t)
$$
其中：

| 符号    | 含义                           |
| ------- | ------------------------------ |
| ($b_t$) | 当前分析分支                   |
| ($v_t$) | 当前状态版本                   |
| ($r_t$) | 当前 Provenance Graph 的根引用 |
| ($C_t$) | 当前分析语义契约               |
| ($E_t$) | 当前执行环境配置               |

其中最重要的是 ($C_t$)。

它负责描述当前分析的真实含义。

## 4.3 Analysis Contract：分析语义契约

例如：

```
{
  "dataset": "wdi_snapshot_2026",
  "entity": "country",
  "grain": ["country", "year"],
  "time_range": [1995, 2023],
  "filters": {
    "region": "all"
  },
  "metric": {
    "name": "gdp_per_capita",
    "numerator": "NY.GDP.MKTP.CD",
    "denominator": "SP.POP.TOTL",
    "unit": "USD/person"
  },
  "aggregation": "mean",
  "missing_policy": "pairwise_complete"
}
```

这里：

- entity 表示分析对象。
- grain 表示数据粒度。
- time_range 表示时间范围。
- filters 表示筛选条件。
- metric 表示指标的具体定义。
- aggregation 表示聚合方式。
- missing_policy 表示缺失值处理方式。

注意：这只是示例契约，不代表所有数据分析都可以用相同字段描述。

第一版仅支持能够明确表达的结构化表格分析语义。

## 4.4 State Patch：状态修改补丁

用户请求不应该直接修改 Python 变量。

首先需要生成一个结构化补丁：
$$
P_t=(a_t,o_t,\Delta_t,\sigma_t,\phi_t)
$$
其中：

- ($a_t$)：目标历史状态。
- ($o_t$)：状态操作类型。
- ($\Delta_t$)：需要修改的属性。
- ($\sigma_t$)：修改范围。
- ($\phi_t$)：执行前置条件。

例如：

用户：

> 将之前的分析改成只保留亚洲国家。

生成：

```
{
  "anchor_version": "v12",
  "operation": "UPDATE",
  "scope": "PERSISTENT",
  "changes": [
    {
      "path": "filters.region",
      "old_value": "all",
      "new_value": "Asia"
    }
  ],
  "preconditions": [
    "dataset_has_region_metadata"
  ]
}
```

这里需要强调：

补丁解析的正确性尚不能被理论保证，必须通过实验测量。

# 5. StateLedger 总体技术架构

## 5.1 总体架构

```
                  User Request
                       |
                       v
             +-------------------+
             |  Intent Parser    |
             |  意图解析模块      |
             +-------------------+
                       |
                       v
             +-------------------+
             | Anchor Resolver   |
             | 历史状态定位       |
             +-------------------+
                       |
                       v
             +-------------------+
             | State Patch       |
             | 补丁生成与验证     |
             +-------------------+
                       |
                       v
             +-------------------+
             | State Ledger      |
             | 状态版本管理       |
             +-------------------+
                       |
                       v
             +-------------------+
             | Dependency        |
             | Impact Analyzer   |
             +-------------------+
                       |
                       v
             +-------------------+
             | Incremental       |
             | Execution Engine  |
             +-------------------+
                       |
                       v
             +-------------------+
             | State Verifier    |
             | 一致性验证         |
             +-------------------+
                       |
                 +-----+-----+
                 |           |
               PASS        FAIL
                 |           |
                 v           v
             Commit        Abort
                 |
                 v
             New State
                 |
                 v
             User Response
```

这是一个逻辑架构。

第一版不需要实现多个独立 LLM Agent。

建议使用一个主 LLM 处理自然语言与代码，其他状态管理、依赖追踪、哈希和事务逻辑尽量由确定性程序完成。

## 5.2 模块划分

| 模块                | 核心职责         | 是否需要 LLM   |
| ------------------- | ---------------- | -------------- |
| Intent Parser       | 解析用户修改意图 | 是             |
| Anchor Resolver     | 定位历史状态     | 部分需要       |
| Patch Validator     | 检查补丁是否合法 | 否，必要时辅助 |
| State Manager       | 保存版本与分支   | 否             |
| Dependency Analyzer | 计算受影响节点   | 否             |
| Execution Engine    | 执行分析代码     | 否             |
| Provenance Manager  | 记录数据血缘     | 否             |
| State Verifier      | 检查一致性       | 主要否         |
| Report Generator    | 输出变更说明     | 是             |

# 6. 三项核心算法设计

## 6.1 算法一：Intent-Grounded State Patch

**目标：解决用户说的状态与智能体实际修改的状态不一致的问题。**

### 6.1.1 历史状态锚点解析

假设当前状态版本为 v20。

用户说：

> 回到之前只分析亚洲国家的版本。

系统需要从历史状态中找出候选版本：

| 版本 | 区域 | 时间范围  | 指标     |
| ---- | ---- | --------- | -------- |
| v12  | 亚洲 | 1995—2023 | GDP      |
| v15  | 亚洲 | 2010—2023 | GDP      |
| v17  | 欧洲 | 1995—2023 | GDP      |
| v20  | 全部 | 1995—2023 | 人均 GDP |

不能直接选择最近的 v15。

因为用户可能指的是 v12。

因此需要结合：

- 用户明确提及的语义。
- 历史状态摘要。
- 指标定义。
- 时间范围。
- 分支关系。
- 版本历史。

得到候选锚点集合：
$$
A(u_t)={v_1,v_2,\ldots,v_k}
$$
**如果无法唯一确定目标状态，则应该询问用户，而不是自行猜测。**

### 6.1.2 补丁候选生成

由 LLM 生成若干候选补丁：
$$
{P_1,P_2,\ldots,P_m}
$$
每个补丁必须明确：

- 锚点版本。
- 操作类型。
- 修改字段。
- 修改范围。
- 修改前后的值。

### 6.1.3 确定性验证

检查：

1. 目标版本是否存在。
2. 目标属性是否存在。
3. 修改值是否符合 Schema。
4. 是否存在单位冲突。
5. 是否存在不允许的状态转换。
6. 是否会导致多个历史状态被错误覆盖。

可以定义一个简单的补丁选择问题：
$$
[
P^*=\arg\min_{P\in\mathcal{P}}
Cost(P)
]
$$
约束为：
$$
Valid(P,S_t)=1
$$

$$
Satisfy(P,u_t)=1
$$

其中：

- (Cost(P))：补丁修改范围。
- (Valid)：状态与类型约束是否合法。
- (Satisfy)：是否满足已明确提取的用户要求。

**如果存在多个会产生不同结果的有效补丁，不应该随意选一个，而应该要求用户确认。**

### 6.1.4 必须测试的失败情况

| 用户请求                 | 可能问题               | 期望处理             |
| ------------------------ | ---------------------- | -------------------- |
| 回到之前的版本           | 多个历史版本符合描述   | 询问具体版本         |
| 只保留高收入国家         | 高收入定义不明确       | 查询收入分类标准     |
| 使用 GDP                 | GDP 存在不同价格口径   | 根据数据和上下文澄清 |
| 恢复上一次分析           | 上一次包含多个临时分支 | 识别目标分支         |
| 保留最新数据但恢复旧条件 | 数据版本和语义版本混合 | 生成复合补丁并验证   |

**这里的创新重点不是生成 JSON，而是将自然语言状态引用、语义修改和执行范围绑定起来，并能够识别不确定情况。**

## 6.2 算法二：Semantic Dependency-aware Invalidation

**目标：当分析条件发生变化时，准确识别哪些历史结果需要失效。**

### 6.2.1 建立有类型的依赖图

定义：
$$
G_P=(V,E_D,E_S)
$$
其中：

- (V)：分析操作、数据和结果节点。
- ($E_D$)：数据依赖边。
- ($E_S$)：语义依赖边。

例如：

```
Raw Dataset
     |
     v
Region Metadata
     |
     v
Filter: Asia
     |
     v
GDP Data
     |
     v
GDP Growth
     |
     v
Ranking
     |
     v
Visualization
```

另外，语义属性也需要连接到实际使用它们的节点：

```
filters.region = Asia
          |
          v
     Filter Node
          |
          v
     GDP Growth
          |
          v
       Ranking
```

这样就能够知道某个结果依赖的不只是哪个 DataFrame，还依赖什么筛选定义和指标定义。

### 6.2.2 计算受影响节点

设用户修改的属性集合为：
$$
\Delta={c_1,c_2,\ldots,c_n}
$$
先找到这些属性对应的起始节点：
$$
Seeds(\Delta)
$$
然后计算这些节点的所有下游后代：
$$
Descendants(Seeds(\Delta),G_P)
$$
需要重新执行的节点就是：
$$
Affected(\Delta)
$$
不受影响的节点可以尝试复用，但必须通过指纹检查。

### 6.2.3 内容指纹

每个计算节点建立指纹：
$$
FP(v)=H(
T_v,
Code_v,
Params_v,
Inputs_v,
Semantics_v,
Env_v,
Seed_v
)
$$
其中：

- ($T_v$)：节点类型。
- ($Code_v$)：代码哈希。
- ($Params_v$)：计算参数。
- ($Inputs_v$)：输入数据内容哈希。
- ($Semantics_v$)：该节点实际依赖的语义属性。
- ($Env_v$)：执行环境摘要。
- ($Seed_v$)：随机种子。

只有指纹匹配的缓存结果才允许复用。

注意：

如果只对输入文件计算哈希，而忽略代码版本、环境版本和参数，仍然可能产生错误缓存。

### 6.2.4 缓存策略

需要区分三种计算类型：

| 类型         | 例子                            | 缓存策略           |
| ------------ | ------------------------------- | ------------------ |
| 严格确定性   | 过滤、排序、聚合                | 允许缓存           |
| 可控随机性   | 固定随机种子的采样              | 经过验证后允许缓存 |
| 不可控随机性 | 非确定性 GPU 训练、外部实时 API | 默认不复用计算结果 |

对于无法完整追踪输入输出依赖的 Python 代码，必须将其标记为 Opaque Node。

Opaque Node 不允许进行不安全的局部缓存复用，必要时重新执行它及其下游节点。

这是防止错误缓存的重要设计。

## 6.3 算法三：Transactional State Transition

**目标：保证状态修改、分支、回滚和提交过程不会出现部分成功或分支污染。**

### 6.3.1 事务流程

建议采用：

```
BEGIN
  |
  v
读取当前版本
  |
  v
解析状态补丁
  |
  v
创建候选状态
  |
  v
计算受影响节点
  |
  v
执行计算
  |
  v
验证数据、依赖和状态
  |
  +------ FAIL ------> ABORT
  |
  PASS
  |
  v
原子提交
  |
  v
新版本生效
```

在执行过程中，不直接修改当前有效状态。

所有修改都先在候选状态中完成。

### 6.3.2 状态提交条件

定义：
$$
Commit(S')=
\begin{cases}
1,&\text{所有必要验证通过且版本未冲突}\
0,&\text{否则}
\end{cases}
$$
只有在以下条件同时满足时才允许提交：

1. 所有必要节点执行成功。
2. 数据依赖完整。
3. 输出 Schema 正确。
4. 指纹与实际输入一致。
5. 没有使用失效的历史结果。
6. 当前分支版本没有被其他操作并发修改。

并发修改采用 Compare-and-Swap（CAS）方式：

只有当分支当前版本仍然等于事务开始时的基础版本，才允许更新分支 Head。

否则中止并重新处理冲突。

### 6.3.3 回滚

回滚不应该直接删除历史状态。

假设：

```
v10 -> v11 -> v12 -> v13
```

用户希望恢复 v11。

可以创建新的状态版本：

```
v10 -> v11 -> v12 -> v13 -> v14
                         rollback to v11
```

其中：

- v14 是新的回滚事件版本。
- v14 的状态根引用 v11 的分析快照。
- v12 和 v13 仍然保留。
- 历史执行记录不被删除。

### 6.3.4 临时反事实分支

用户说：

> 如果只保留欧洲国家，结果会怎样？

应该创建一个独立分支：

```
                  Main v10
                  /      \
                 /        \
          Main v11       CF v10-A
           Asia           Europe
```

临时分支执行结束后，可以：

- 查看结果。
- 与主分支比较。
- 保存为正式版本。
- 丢弃分支引用。

但绝不能修改主分支中的数据和结果。

### 6.3.5 多状态组合

假设用户希望比较：

- v11：亚洲国家。
- v12：欧洲国家。

系统需要生成一个组合分析状态。

如果两个分支具有相同的数据源和指标定义，可以直接进行比较。

如果它们使用了不同数据版本、不同指标口径或不同时间范围，则需要先进行兼容性验证。

不能默认两个分析结果可以直接合并。

# 7. 理论分析与可证明性质

这里必须区分：

1. 可以严格证明的性质。
2. 可以通过工程约束保证的性质。
3. 只能通过实验评价的性质。

不要把三者混为一谈。

## 7.1 性质一：增量重新计算的正确性

设分析执行图：
$$
G=(V,E)
$$
满足：

1. (G) 是有向无环图。
2. 每个节点是确定性的纯函数。
3. 所有数据依赖都被完整记录。
4. 代码、参数、数据和环境版本固定。
5. 发生变化的节点集合识别正确。

令 (U) 表示发生变化的节点，令 (Desc(U)) 表示其下游节点。

那么：

只重新计算 (Desc(U))，并复用未受影响节点的结果，其最终输出与完整重新执行相同。

即：
$$
Eval_{\mathrm{full}}(G,U)
$$


### 证明思路

按照 DAG 的拓扑顺序进行归纳。

对于不属于受影响集合的节点，其输入和计算定义保持不变，因此输出不变。

对于受影响节点，其所有必要输入都已经得到正确的更新结果，因此重新执行该节点能够得到与完整重新执行一致的输出。

由此可得到整个图的输出一致性。

### 必须强调的限制

这个证明不保证 LLM 能够正确识别用户意图，也不保证所有 Python 代码的依赖关系都能被识别。

因此，实际系统必须使用保守的依赖追踪和 Opaque Node 处理策略。

## 7.2 性质二：分支隔离

如果：

- 已提交状态不可变。
- 新状态通过新节点和新快照创建。
- 不同分支拥有独立的状态根引用。
- 分支修改不允许覆盖已有内容。

那么一个分支的修改不会改变其他历史状态的内容。

这是基于不可变对象和分支引用的工程性质。

## 7.3 性质三：过期结果阻断

定义结果节点的有效性：
$$
FP_{\mathrm{expected}}(v,S)
$$
只有：
$$
Valid(v,S)=1
$$
的结果才允许在当前状态中被使用。

在所有结果读取都经过该验证器的条件下，可以阻止已知依赖变化导致的过期结果被使用。

这并不意味着能够发现所有未知依赖。

如果某个代码节点读取了未记录的外部文件，依赖图不完整，那么该性质不成立。

## 7.4 复杂度分析

假设：

- (V)：总节点数。
- (E)：总边数。
- (V_a)：受影响节点数。
- (E_a)：受影响边数。

使用图遍历识别受影响节点的时间复杂度：
$$
O(|V_a|+|E_a|)
$$
重新计算的总成本为：
$$
\sum_{v\in V_a}C(v)
+
C_{\mathrm{verify}}
+
C_{\mathrm{metadata}}
$$
完整重算成本：
$$
\sum_{v\in V}C(v)
$$
定义成本节省率：
$$
Saving=
1-\frac{C_{\mathrm{inc}}}{C_{\mathrm{full}}}
$$
注意：如果修改发生在原始数据或全局语义条件上，可能导致绝大多数节点失效。

此时：
$$
Saving\approx 0
$$
甚至可能因为验证和缓存开销导致增量方案更慢。

这不是算法失败，而是需要在实验中如实报告的边界条件。

# 8. 工程实现方案

## 8.1 推荐技术栈

第一版不需要复杂的分布式架构。

| 技术              | 用途               |
| ----------------- | ------------------ |
| Python            | 主体开发           |
| Pydantic          | 状态和补丁 Schema  |
| NetworkX          | 初期 DAG 操作      |
| Pandas            | 表格分析           |
| DuckDB            | SQL 查询           |
| PyArrow / Parquet | 中间数据存储       |
| SQLite            | 元数据和版本信息   |
| SHA-256 / BLAKE3  | 内容哈希           |
| Docker            | 隔离执行环境       |
| pytest            | 单元测试和回归测试 |

后续如需多用户和并发状态管理，可以将 SQLite 元数据存储替换为 PostgreSQL。

## 8.2 项目目录

建议采用以下结构：

```
StateLedger/
│
├── agent/
│   ├── intent_parser.py
│   ├── anchor_resolver.py
│   ├── patch_generator.py
│   └── response_generator.py
│
├── core/
│   ├── state.py
│   ├── state_version.py
│   ├── state_branch.py
│   ├── state_patch.py
│   └── state_transition.py
│
├── graph/
│   ├── provenance_graph.py
│   ├── semantic_dependency.py
│   ├── graph_invalidation.py
│   └── graph_traversal.py
│
├── execution/
│   ├── python_executor.py
│   ├── sql_executor.py
│   ├── sandbox.py
│   └── execution_cache.py
│
├── storage/
│   ├── metadata_store.py
│   ├── artifact_store.py
│   └── hash_manager.py
│
├── verifier/
│   ├── patch_validator.py
│   ├── schema_validator.py
│   ├── provenance_validator.py
│   └── state_consistency.py
│
├── benchmark/
│   ├── longds_adapter.py
│   ├── wdi_tasks.py
│   └── synthetic_tasks.py
│
├── evaluation/
│   ├── accuracy.py
│   ├── state_metrics.py
│   ├── cost_metrics.py
│   └── statistical_tests.py
│
├── tests/
│
├── configs/
│
└── main.py
```

## 8.3 核心数据表

建议至少实现以下表。

### state_versions

| 字段          | 类型     | 说明       |
| ------------- | -------- | ---------- |
| version_id    | TEXT     | 版本编号   |
| branch_id     | TEXT     | 所属分支   |
| parent_ids    | JSON     | 父版本     |
| root_hash     | TEXT     | 状态根哈希 |
| contract_json | JSON     | 分析契约   |
| operation     | TEXT     | 状态操作   |
| created_at    | DATETIME | 创建时间   |

### provenance_nodes

| 字段          | 类型    | 说明         |
| ------------- | ------- | ------------ |
| node_id       | TEXT    | 节点编号     |
| node_type     | TEXT    | 节点类型     |
| code_hash     | TEXT    | 代码哈希     |
| params_hash   | TEXT    | 参数哈希     |
| input_hashes  | JSON    | 输入哈希     |
| semantic_refs | JSON    | 语义依赖     |
| env_hash      | TEXT    | 环境哈希     |
| output_hash   | TEXT    | 输出哈希     |
| cacheable     | BOOLEAN | 是否允许缓存 |

### provenance_edges

| 字段      | 类型 | 说明     |
| --------- | ---- | -------- |
| source_id | TEXT | 起点     |
| target_id | TEXT | 终点     |
| edge_type | TEXT | 依赖类型 |

### transition_log

| 字段           | 类型 | 说明     |
| -------------- | ---- | -------- |
| transition_id  | TEXT | 转换编号 |
| base_version   | TEXT | 基础版本 |
| target_version | TEXT | 目标版本 |
| patch_json     | JSON | 状态补丁 |
| changed_slots  | JSON | 修改属性 |
| affected_nodes | JSON | 失效节点 |
| verification   | JSON | 验证记录 |
| status         | TEXT | 转换状态 |

## 8.4 执行环境设计

建议采用显式输入输出的执行方式。

例如：

```python
def calculate_gdp_growth(context):

    df = context.read("gdp_data")

    result = (
        df.sort_values(["country", "year"])
          .groupby("country")
          .apply(
              lambda x: x["gdp"].pct_change()
          )
    )

    context.write(
        "gdp_growth",
        result
    )

    return result
```

这里的 `context.read()` 和 `context.write()` 用于记录真实输入输出依赖。

不建议第一版允许 Agent 任意修改全局变量、直接读写任意文件或访问未记录的外部资源。

如果必须支持这些行为，就要将对应节点标记为不透明操作，并禁止不安全缓存。

## 8.5 必须实现的核心 API

```text
create_state()
get_state()
get_state_history()
resolve_anchor()
generate_patch()
validate_patch()
create_branch()
apply_patch()
compute_affected_nodes()
execute_affected_nodes()
verify_state()
commit_state()
rollback_state()
compose_states()
```

每个 API 必须有对应单元测试。

# 9. 数据集与 Benchmark 设计

这是论文实验能否成立的关键。

不建议只用一份自己编造的数据集，也不建议只使用现成 Benchmark。

采用：

**现有 Benchmark + 真实公开数据 + 可控合成数据。**

## 9.1 主 Benchmark：LongDS v1.1

地址：

https://huggingface.co/datasets/zjunlp/LongDS

推荐版本：

- 开发：v1.1-Lite，24 个任务、777 轮。
- 最终测试：v1.1-Full，68 个任务、2,225 轮。

建议先使用 Lite 调整算法，再固定系统配置，运行 Full。

必须注意：

LongDS 的数据卡目前标注为 `other` 许可证。

公开可下载并不等于可以不受限制地重新分发其中的原始数据。需要逐项核对来源数据的使用许可，论文代码仓库应优先提供数据准备脚本和原始数据获取说明。

## 9.2 真实数据：World Development Indicators

地址：

https://datacatalog.worldbank.org/search/dataset/0037712/world-development-indicators

许可证：

CC BY 4.0。

官方目录显示：

- 200 多个国家和地区。
- 1,500 多个发展指标。
- 年度数据。
- 时间覆盖 1960—2025。
- CSV 压缩包约 270MB。

建议固定一个数据快照，并在论文中记录：

- 下载时间。
- API 请求参数。
- 原始文件哈希。
- 指标元数据版本。
- 预处理代码版本。

### 建议选取的指标

| 指标           | 含义               |
| -------------- | ------------------ |
| NY.GDP.MKTP.CD | GDP，现价美元      |
| SP.POP.TOTL    | 总人口             |
| NY.GDP.PCAP.CD | 人均 GDP，现价美元 |
| SP.DYN.LE00.IN | 预期寿命           |
| EN.ATM.CO2E.PC | 人均 CO2 排放      |

实际使用时应检查所选指标在目标时间段内的完整性。

建议先选择 1995—2023 年的稳定数据区间，再根据数据审查决定是否扩展。

### 任务设计示例

任务 A：

分析亚洲国家过去 20 年的 GDP 增长。

任务 B：

将指标修改为人均 GDP。

任务 C：

建立欧洲国家分支。

任务 D：

回到亚洲国家分支，将时间范围修改为 2010—2023 年。

任务 E：

比较亚洲和欧洲的人均 GDP 增长。

任务 F：

恢复初始条件，并比较 GDP 和人均 GDP 的分析结果。

这些任务可以构造出真实的多轮状态修改和数据依赖关系。

## 9.3 合成控制数据

建议额外建立一个小规模的可控实验集。

暂定名称：

StateLedger Controlled Transition Suite（SLCTS）。

初始规模：

- 48 条分析轨迹。
- 每条 12—16 轮。
- 共约 576—768 轮。
- 6 种状态演化模式。
- 每种模式 8 条轨迹。

每条任务都具有完整的 Ground Truth：

1. 正确锚点版本。
2. 正确操作类型。
3. 正确修改属性。
4. 受影响节点集合。
5. 期望分支状态。
6. 预期输出结果。
7. 预期是否需要用户澄清。

该数据集不应该只是生成几百条看起来复杂的自然语言问题。

必须由程序自动生成可验证的状态转换，并进行人工检查。

初期它是内部诊断测试集，不应在没有充分规模、专家验证和许可审查的情况下宣称是一个新的通用 Benchmark。

## 9.4 辅助 Benchmark

可选择 InfiAgent-DABench 作为单轮能力回归测试。

它包含：

- 603 个数据分析问题。
- 124 个 CSV 文件。
- 公开验证集 257 个问题。
- 其余测试问题不公开。

它采用格式约束和结构化答案评价。

但它不适合作为 StateLedger 的主要实验，因为它并不专门评价长期状态转换。

# 10. Baseline 设计

这一部分决定实验是否有说服力。

不能只和普通 GPT + Python 比较。

建议设置以下基线。

| Baseline | 方法                             | 主要目的                     |
| -------- | -------------------------------- | ---------------------------- |
| B1       | ReAct + Persistent Kernel        | 传统 Agent 基线              |
| B2       | ReAct + Context Summary          | 仅增强对话记忆               |
| B3       | Task DAG Agent                   | 只进行任务图管理             |
| B4       | Provenance DAG + Cache           | Wrattler 风格的数据依赖管理  |
| B5       | StateLedger w/o Intent Grounding | 移除语义补丁解析             |
| B6       | StateLedger w/o Transaction      | 移除事务式提交               |
| B7       | Full StateLedger                 | 完整方法                     |
| B8       | Oracle StateLedger               | 使用人工正确状态补丁作为上界 |

说明：

B4 是参考 Wrattler 设计重新实现的对照系统，不应冒充原始 Wrattler 的完整复现。

Data Interpreter、EvoDS 可以作为额外的外部系统对照，但要考虑代码可用性、模型规模和执行环境差异。

DataPRM 更适合作为过程验证的补充对照，而不是直接等同于状态管理系统。

## 10.1 公平性要求

所有主要基线尽量保持：

- 相同基础模型。
- 相同数据。
- 相同执行环境。
- 相同工具权限。
- 相同最大交互次数。
- 相同上下文预算。
- 相同温度和随机种子设置。

否则无法判断提升究竟来自 StateLedger，还是来自更强的基础模型。

# 11. 实验设计与评价体系

建议围绕七项实验展开。

## E1：整体任务准确率

在 LongDS Full 上比较所有主要基线。

主指标采用：
$$
\frac{1}{M}
\sum_{i=1}^{M}
\frac{1}{N_i}
\sum_{j=1}^{N_i}s_{i,j}
$$
此外分别报告：

- Initialization。
- Update。
- Counterfactual。
- Rollback。
- Multi-state Composition。

重点看状态操作复杂的任务是否获得更明显提升。

## E2：状态转换准确率

这是 StateLedger 应该新增的核心评价。

定义：
$$
STA=
\frac{
N_{\mathrm{correct\ transitions}}
}{
N_{\mathrm{all\ transitions}}
}
$$
一次转换只有在以下项目都正确时才算成功：

- 目标历史状态正确。
- 操作类型正确。
- 修改字段正确。
- 修改范围正确。
- 最终状态正确。

同时报告：

**Patch Exact Match**

补丁与标准答案是否完全一致。

**Anchor Resolution Accuracy**

是否选中了正确的历史版本。

## E3：过期结果和错误级联

定义：
$$
SR=
\frac{
N_{\mathrm{stale\ artifacts\ consumed}}
}{
N_{\mathrm{artifacts\ consumed}}
}
$$
SR 越低越好。

再对失败任务进行人工分类：

- Context Memory Error。
- State Management Error。
- Cascade Error。
- Coding Error。
- Planning Error。

重点观察 StateLedger 是否能够减少 Cascade Error。

## E4：依赖分析准确率

在可控合成数据上计算：
$$
Precision=
\frac{TP}{TP+FP}
$$

$$
Recall=
\frac{TP}{TP+FN}
$$

$$
F1=
2\cdot
\frac{Precision\cdot Recall}
{Precision+Recall}
$$

其中：

- TP：正确识别为受影响的节点。
- FP：不需要重新计算却被失效的节点。
- FN：实际上受影响但没有被识别的节点。

不能只追求 Recall。

如果所有节点都被重新计算，Recall 可能很高，但完全没有效率优势。

## E5：回滚与分支隔离

评价：

| 指标                      | 含义                         |
| ------------------------- | ---------------------------- |
| Rollback Exactness        | 回滚后状态是否与目标版本一致 |
| Branch Isolation Accuracy | 分支之间是否存在错误污染     |
| Composition Accuracy      | 多状态组合是否正确           |
| Recovery Success          | 执行失败后是否能够恢复       |

建议在 48 条合成轨迹中进行精确验证。

## E6：计算成本

记录：

- 执行时间。
- Token 数量。
- LLM 调用次数。
- Python 执行次数。
- 缓存命中率。
- 重新计算节点比例。

比较：
$$
CostSaving=
1-
\frac{C_{\mathrm{StateLedger}}}
{C_{\mathrm{FullRecompute}}}
$$
这里必须同时报告准确率。

不能通过不执行必要任务来换取成本下降。

## E7：跨数据集泛化

在 WDI 上构造没有参与算法调试的分析任务。

重点检验：

- 新数据源。
- 新指标。
- 新时间范围。
- 新筛选条件。
- 新的历史分支。
- 多表关联。

如果时间允许，可以增加 DSBench 或 ScienceAgentBench 作为补充测试，但不建议为了增加 Benchmark 数量而牺牲主要实验质量。

# 12. 消融实验

至少进行以下八项消融。

| 实验版本 | 移除的机制               | 需要回答的问题             |
| -------- | ------------------------ | -------------------------- |
| A1       | Anchor Resolver          | 历史版本定位是否重要？     |
| A2       | Semantic Patch Validator | 补丁验证是否重要？         |
| A3       | Semantic Dependency      | 语义依赖是否带来提升？     |
| A4       | Fingerprint              | 内容指纹是否必要？         |
| A5       | Selective Re-computation | 局部计算是否降低成本？     |
| A6       | Transactional Commit     | 事务机制是否降低状态损坏？ |
| A7       | Branch Isolation         | 分支隔离是否必要？         |
| A8       | Ambiguity Detection      | 歧义识别是否减少错误修改？ |

另外增加一个重要对照：

**Full Recompute vs Incremental Recompute**

用来检验选择性计算是否真的有收益。

## 12.1 必须设计的负面实验

不能只展示算法表现良好的案例。

至少测试：

1. 隐式全局变量。
2. 外部文件被修改。
3. 代码中存在随机数。
4. 环境依赖发生变化。
5. 数据依赖边被遗漏。
6. 用户请求具有歧义。
7. 多个分支同时修改。
8. 事务执行中途崩溃。

我们希望知道系统在哪些条件下会主动放弃缓存或提交。

# 13. 统计分析与实验可信度

所有主要随机实验至少运行 3 个随机种子。

推荐：

- 报告均值和标准差。
- 报告 95% Bootstrap Confidence Interval。
- 以任务为单位进行配对 Bootstrap。
- 对配对结果使用适当的统计检验。
- 多重比较使用 Holm 校正。
- 报告效应量。

特别注意：

LongDS 中不同任务的轮数不同。

不能简单地把 2,225 轮全部当作互相独立的样本，否则容易低估不确定性。

应以任务为统计抽样单位。

对自动化评价：

- 数值答案优先使用参考结果和容差比较。
- 状态版本使用精确匹配。
- 数据血缘使用结构化匹配。
- 开放式解释才考虑 LLM-as-a-Judge。

建议对一定比例的结果进行盲法人工审查。

LongDS 原论文已经对 450 个判断案例进行人工审计，报告了 93.11% 的一致率和 0.8623 的 Cohen's kappa。我们可以参考其方法，但应针对自己的状态转换评价重新进行审计。

# 14. 开发阶段与验收标准

## Phase 1：研究问题和实验环境

交付：

- 完整文献清单。
- LongDS v1.1 Lite 环境。
- ReAct 基线。
- 30 条初始状态转换测试。
- State Contract 初稿。

验收：

能够复现至少一组普通 Agent 的多轮分析过程。

## Phase 2：状态版本管理

交付：

- State Version。
- Branch。
- State Patch。
- Rollback。
- Composition。
- 元数据存储。

验收：

历史版本不可被意外覆盖；回滚后状态根能够与原始目标版本匹配。

## Phase 3：Provenance Graph

交付：

- 数据节点。
- 操作节点。
- 结果节点。
- 数据依赖边。
- 语义依赖边。
- 哈希和缓存。

验收：

对所有受支持的确定性测试操作，完整重算与选择性重算的结果一致。

## Phase 4：意图解析与事务管理

交付：

- Intent Parser。
- Anchor Resolver。
- Patch Validator。
- Transaction Controller。
- Commit Certificate。

验收：

不明确的修改不会在未确认的情况下静默提交。

## Phase 5：Benchmark 和核心实验

交付：

- LongDS Lite 结果。
- LongDS Full 结果。
- WDI 实验。
- 状态转换指标。
- 依赖分析指标。
- 成本分析。
- 消融实验。

验收：

所有主要实验均有可复现日志和固定版本的执行环境。

# 15. 论文最终实验结构

建议最终论文包含以下实验表格和图。

| 编号    | 内容              | 主要展示                |
| ------- | ----------------- | ----------------------- |
| Table 1 | Baseline 总体性能 | LongDS 准确率           |
| Table 2 | 状态转换性能      | Patch/Anchor/Transition |
| Table 3 | 依赖分析准确率    | Precision/Recall/F1     |
| Table 4 | 计算成本          | 时间、Token、执行量     |
| Table 5 | 消融实验          | 各模块贡献              |
| Table 6 | 负面实验          | 系统适用边界            |

建议准备以下图：

**Figure 1：整体系统架构。**

展示 Intent Parser、State Ledger、Provenance Graph、Execution Engine 和 Verifier 的关系。

**Figure 2：状态版本与分支演化示意图。**

展示 Update、Counterfactual、Rollback 和 Composition。

**Figure 3：错误级联对比。**

比较普通 Agent 与 StateLedger 的错误传播。

**Figure 4：Long-Horizon Accuracy Curve。**

横坐标为任务进度或交互轮次，纵坐标为准确率。

**Figure 5：Accuracy-Cost Pareto Frontier。**

展示不同系统的准确率与执行成本关系。

# 16. 主要研究风险与应对措施

| 风险                       | 严重程度 | 应对措施                                       |
| -------------------------- | -------- | ---------------------------------------------- |
| 与 Wrattler 等系统创新重叠 | 极高     | 聚焦 LLM 意图到状态补丁的映射                  |
| 与 LongDS 研究问题重复     | 高       | 研究解决机制及状态级评价，而非重新定义状态操作 |
| Python 依赖追踪不完整      | 极高     | 采用显式 I/O 和保守 Opaque Node                |
| LLM 锚点解析不准确         | 高       | 候选状态检索、约束验证和歧义澄清               |
| 缓存结果不可靠             | 高       | 完整指纹和确定性分级                           |
| 状态管理增加执行成本       | 中高     | 进行成本消融，限制元数据粒度                   |
| LongDS 规模有限            | 中       | 增加 WDI 和可控合成轨迹                        |
| 研究范围过大               | 高       | 第一版只支持表格数据和受控操作                 |

## 16.1 最重要的 Go/No-Go 判断

建议在正式开发之前完成一个小型可行性实验。

用 30 条多轮任务，比较：

1. 普通 ReAct。
2. ReAct + 对话摘要。
3. Wrattler 风格的依赖图与缓存。
4. 加入 Intent-Grounded State Patch 的 StateLedger。

如果结果表明：

- 仅增加依赖图已经解决了大部分问题；
- Intent Parser 没有显著改善状态转换；
- 分支和回滚没有形成可测量的收益；
- 系统性能提升完全来自缓存；

那么 StateLedger 目前还不足以形成具有说服力的 KDD 算法贡献。

此时应该及时调整研究方向，而不是为了保留原始 Idea 强行增加模块。

反过来，如果语义补丁和状态事务机制能够在状态正确性、错误级联和长期任务表现上产生独立且可复现的提升，才值得投入完整论文开发。

# 17. KDD 2027 时间规划

第二轮官方截止日期尚待确认，因此先按照 2027 年 2 月上旬的内部目标倒排。

| 时间        | 工作                            | 交付物               |
| ----------- | ------------------------------- | -------------------- |
| 11.04—11.14 | 文献复核、创新边界、LongDS 环境 | Research Proposal    |
| 11.15—11.28 | 基线与状态数据结构              | MVP                  |
| 11.29—12.12 | State Patch、版本和分支         | 核心状态管理         |
| 12.13—12.26 | Provenance、依赖追踪、执行引擎  | 核心算法             |
| 12.27—01.09 | LongDS 与 WDI 实验              | 主实验结果           |
| 01.10—01.16 | 消融、负面实验、统计分析        | 完整实验报告         |
| 01.17—01.23 | 论文初稿                        | Paper Draft          |
| 01.24—01.30 | 修改、复核和补充实验            | Submission Candidate |
| 01.31 起    | 根据官方截止时间调整            | 正式投稿             |

建议将 2027 年 1 月 20 日作为内部实验冻结与论文基本定稿的目标。

如果第二轮官方截止时间提前，则优先保留：

1. 核心状态补丁算法。
2. LongDS Full。
3. 关键基线。
4. 主要消融。

不建议为了赶时间而牺牲核心对比实验。

# 18. 论文结构与篇幅规划

KDD 2027 Research Track 首轮正文为 8 页。

建议：

| 章节                         | 预计篇幅 |
| ---------------------------- | -------- |
| 1. Introduction              | 1.0 页   |
| 2. Related Work              | 0.8 页   |
| 3. Problem Formulation       | 1.0 页   |
| 4. Methodology               | 2.4 页   |
| 5. Theoretical Analysis      | 0.7 页   |
| 6. Experiments               | 1.7 页   |
| 7. Discussion and Conclusion | 0.4 页   |
| 合计                         | 8 页     |

详细数学证明、伪代码、补充任务、更多消融实验和完整实现细节放在 Appendix。

# 19. 最终研究决策

本研究暂定采用以下方案：

**研究名称：**

StateLedger

**正式研究定位：**

Intent-Grounded and Provenance-Aware State Transitions for Long-Horizon Data Analysis Agents。

**主研究问题：**

如何将长期对话中的自然语言分析修改意图，转化为正确、可验证、可恢复的分析状态转换，并避免过期数据和中间结果在后续任务中传播？

**主要算法模块：**

1. Intent-Grounded State Patch。
2. Semantic Dependency-aware Invalidation。
3. Transactional State Transition。

**主要数据：**

1. LongDS v1.1。
2. World Development Indicators。
3. StateLedger Controlled Transition Suite。

**主要评价：**

1. Long-horizon Task Accuracy。
2. State Transition Accuracy。
3. Stale Result Rate。
4. Cascade Error Rate。
5. Rollback Accuracy。
6. Branch Isolation Accuracy。
7. Accuracy-Cost Trade-off。

**目标会议：**

KDD 2027 Research Track。

# 20. 参考文献与开发资源

以下是本方案真正需要精读和复用的文献，不是泛泛的推荐阅读列表。

| 编号 | 论文或技术                          | 用途                         |
| ---- | ----------------------------------- | ---------------------------- |
| [1]  | LongDS-Bench, EMNLP 2026            | 长期状态任务及评价           |
| [2]  | EvoDS, KDD 2026                     | 上下文管理与技能学习对照     |
| [3]  | Data Interpreter, ACL Findings 2025 | 任务图和渐进式验证           |
| [4]  | DataPRM, KDD 2026                   | 过程级验证与错误分类         |
| [5]  | DS-Agent, ICML 2024                 | 数据科学 Agent 执行流程      |
| [6]  | InfiAgent-DABench, ICML 2024        | 单轮数据分析能力测试         |
| [7]  | Wrattler, TaPP 2018                 | 状态分离、依赖图、缓存、回滚 |
| [8]  | noWorkflow, PVLDB 2017              | Python 执行来源追踪          |
| [9]  | Loops, IEEE TVCG 2025               | Notebook 历史与分支可视化    |
| [10] | DBSP, PVLDB 2023                    | 增量视图维护                 |
| [11] | W3C PROV-DM, 2013                   | 数据来源描述标准             |

### 论文原文

1. [LongDS-Bench](https://arxiv.org/abs/2605.30434)
2. [EvoDS](https://arxiv.org/abs/2606.03841)
3. [Data Interpreter](https://aclanthology.org/2025.findings-acl.1016.pdf)
4. [DataPRM](https://arxiv.org/abs/2604.24198)
5. [DS-Agent](https://proceedings.mlr.press/v235/guo24b.html)
6. [InfiAgent-DABench](https://proceedings.mlr.press/v235/hu24s.html)
7. [Wrattler](https://www.usenix.org/system/files/conference/tapp2018/tapp2018-paper-petricek.pdf)
8. [noWorkflow](https://www.vldb.org/pvldb/vol10/p1841-pimentel.pdf)
9. [Loops](https://doi.org/10.1109/TVCG.2024.3456186)
10. [DBSP](https://www.vldb.org/pvldb/vol16/p1601-budiu.pdf)
11. [W3C PROV-DM](https://www.w3.org/TR/prov-dm/)

### 数据与官方资源

- [LongDS Dataset](https://huggingface.co/datasets/zjunlp/LongDS)
- [DataMind GitHub](https://github.com/zjunlp/DataMind)
- [World Bank WDI](https://datacatalog.worldbank.org/search/dataset/0037712/world-development-indicators)
- [KDD 2027 Research Track CFP](https://kdd2027.kdd.org/research-track-call-for-papers/)

**最终原则：StateLedger 的研究价值不在于它有多少模块，而在于它能否证明：当用户不断修改长期分析任务时，智能体比现有方法更准确地理解状态、更可靠地更新结果、更有效地避免错误级联，而且这些优势能够在严格控制的公开实验中得到复现。**