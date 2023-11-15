# A Survey of Techniques for Maximizing LLM Performance

【YouTube】[A Survey of Techniques for Maximizing LLM Performance](https://www.youtube.com/watch?v=ahnGLM-RC1Y)

## 总结

和其他开发问题类似，解决问题的过程是：

1. 从最简单的 prompt 开始，建立评估 baseline
2. 明确瓶颈在哪里，是 context 还是 LLM
    + context 问题，知识库不够，用 RAG
    + LLM 问题，预期行为不对，用 fine-tuning
3. 改进、评估、重复


## 优化 LLM 很困难

1. 明确知道要解决的问题是什么
2. 如何衡量、评估 LLM 的性能，应该用什么指标
3. 用什么方法解决问题

两种方式：
* retrieval-augmented generation RAG
* fine-tuning 微调

解决不同的问题

y轴：context optimization，模型需要知道的上下文信息

x轴：LLM optimization，模型预期的行为,解决问题需要执行的步骤

|||
|---|---|
|RAG（context optimization）|ALL of the above|
|Prompt engineering（起点）|Fine-tuning（LLM optimization）|


一般从提示工程开始，快速测试开发，通过【评估】输出的结果，明确要解决的问题性质，采取相应的方法，继续评估。。。

* context problem 需要更多相关上下文 -> RAG
* 需要一致的行为 -> fine-tuning


### Prompt engineering 提示工程

方法策略，来自[官网最佳实践](https://platform.openai.com/docs/guides/prompt-engineering)

+ Start with
    * Write clear instructions 写下清晰的指示
    * Split complex tasks into simpler subtasks 拆分复杂任务
    * Give GPTs time to think 给GPT时间思考
    * Test changes systematically 系统测试更改, 对结果有标准化的评估，避免陷入打地鼠一样的困境里
+ Extend to
    * Provide reference text 提供参考文本，提供知识库给 GPT 学习，使用文本内容回答
    * Use external tools 使用外部工具

------------------
提示工程优点：
* 快速开始测试，及时获得反馈
* 提供一个 baseline，方便对后续改进做评估

缺点：
* 需要引用大量新的信息，可扩展性差
* 受 context 限制，实际能展示的示例有限
* token 有限，延迟高，成本高

------------------
演讲者做了几个示例

1. 在 system 提示中，明确告诉 GPT，你要面对的是什么问题，会接受什么样的输入，花时间思考，按照以下的步骤来思考问题，不要跳过其中的步骤，1、2、3.。。。分别是什么，最后应该输出什么样的结果
2. 模拟 user 进行一轮问答，作为示例给 GPT 学习


### RAG vs. Fine-tuning

如果比喻成期末考试

* RAG -> 类似于短期记忆，给 GPT 一本书，GPT 看到问题，去书中找对应的答案

* Fine-tuning -> 类似于长期记忆，让 GPT 学习知识理论和方法论，用方法去解决问题

实际上，GPT 即需要方法论指导，也需要知识库（context），才能解决问题。只是不同的问题，不同的情景，GPT 解决问题所需要的东西侧重不同。

### Retrieval-augmented generation RAG 检索增强生成

解决问题的流程为：

* 引入知识库（文档）
* 用户输入问题
* 通过相关性检索，找到文档中与用户问题相关的内容
* 告诉 GPT 【Prompt engineering】在以下内容中找用户问题的答案，用户的问题是 xxxxx，内容是 xxxxx
* GPT 输出答案

------------------
RAG 优点：
* 可以提供给 GPT 新的知识库，可扩展性强
* 仅用文档内容回答用户问题，减少 GPT 胡说八道的概率

缺点：
* 嵌入了对广泛领域的理解，（解决问题的方法论还是 GPT 本身已有的知识）
* 很难教会模型特定领域的知识，比如法律、医学、新的语言、格式、风格等等，（试图教会 GPT 解决问题的方式，是微调可以做的事情）
* token 的使用，还是需要很多提示、示例

------------------
演讲者讲了两个示例，

正面的事，只用 RAG 解决问题，正确率从 45% 提升到 98%，中间没有使用 fine-tuning，所以明确要解决的问题是哪种问题，是非常重要的。

中间解决的问题是：要么没有给 GPT 正确的上下文，要么 GPT 不知道那块上下文是正确的。

另一个反面例子是，如果搜索的效果不好，那么 RAG 几乎不可能有正确的答案。

------------------
[Rages 框架](https://github.com/explodinggradients/ragas)，评估 RAG 效果

+ LLM 回答的怎么样，context 和 answer 的相关性
    * Faithfulnesss 忠诚 - 生成的答案与给定上下文的一致性，question 、context 和 answer
    * Answer relevancy 回答相关性 - 生成的答案与问题的相关程度， question 和 answer
+ 内容与问题的相关程度，context 和 ground truth 的相关性
    * Context recall 情境回忆 - 检索到的上下文与基本事实(真实期望答案)的一致程度，contexts 是否和 ground truth 一致
    * Context precision 上下文精度 - 用于评估 contexts 中与提问的相关程度。使用 question 和 contexts 计算的 （应该提供与问题最相关的 context 给 GPT，否则提供的越多，GPT 会迷失在大量的文档里）


端到端的评价：
+ Answer semantic similarity 回答语义相似度 - 生成的答案与基本事实(真实期望答案)之间语义相似性的评估。基于 ground truth 和 answer 
+ Answer Correctness 答案正确性 - 生成答案与真实答案的准确性。依赖于 ground truth 和 answer
+ Aspect Critique 方面批判 - 根据预定义评估 危害性、恶意性、连贯性、正确性、简洁性


### Fine-tuning 微调

在特定领域的数据集上训练已有的模型，这个数据集比 GPT 原始的数据集更小，得到一个完全不同的模型。新的模型更适合我们特定的任务。

* 做的结果更好，提高模型在特定任务上的表现，比 prompt engineering 效率更高，token 总是有限，对上亿数据微调很容易
* 做的效率更高，微调模型比基础模型在特定任务上的交互更有效，用更少的 token 完成同样的任务，效率高、成本低

------------------
Fine-tuning 适合于：
* 增强现有模型已知的知识（比如强制某种 sql 方言，强调已有知识的子集）
* 自定义模型输出的结构或语气（比如强制输出有效的 json）
* 教会模型复杂的指令（在微调过程中展示更多的示例）

缺点：
* GPT 预训练的过程中，掌握大量知识，微调无法学习到新的知识，这种情况应该用 RAG
* 微调对于需要快速迭代的场景不友好，微调是一个相对较慢的反馈循环，因为创建训练数据集和微调的其他组件需要大量投资，所以不要从这里开始

------------------
演讲者讲了两个例子

正面的是，一个设计师助手，用户输入自然语言描述，输出结构化的内容。使用 GPT3.5 或者 GPT4 ，回答是合理的，但是内容和描述的相关性不高。在 GPT3.5 上使用微调之后，效果比 GPT4 都更好。

得益于：有基础评价的 baseline，高质量的训练数据

另一个反面例子是，用户希望得到一个和自己语气很像的写作助手，用 Slack 数据训练，得到的是一个和 Slack 语气很像的写作助手，简短、口语化、不关注语法。

要提供合适的训练数据

------------------
微调一般过程：

* 构建训练数据集，收集数据标记数据
* 训练模型
    + openai api，相对简单
    + 其他开源模型，自己搭建环境，重点是理解可供调整的超参数，是否过度拟合、欠拟合、灾难性遗忘；理解损失函数。
+ 评估模型
    + 人工评估，找专家
    + 不同的模型，生成不同的输出，相互比较排名
    + 用更强大的模型（GPT4）对开源模型（GPT3.5）的输出进行排名
+ 实际部署运行
    + 对生产中的数据收集采样，继续训练，形成闭环

------------------
微调的最佳实践：

* 从 prompt engineering 开始，少量样本学习尝试，得到一些直觉，理解 LLM 的工作原理
* 建立一个 baseline，尝试了 3.5 Turbo、尝试了 GPT4，对他们试图解决的问题的来龙去脉有了很好的理解，他们了解了这些模型的失败案例，他们了解了模型表现良好的地方，所以他们准确地了解他们想要解决的问题
* 微调，开发小型高质量数据集，进行微调，然后评估，看是否朝着正确的方向前进。可以使用主动学习方法，微调模型，查看其输出，看看它在哪些领域陷入困境，然后使用新数据专门针对这些领域。数据质量比数据量更重要，数据量是在基本模型训练过程中完成的。微调应该专注于小型高质量数据集。


## Spider 1.0

[Spider 1.0](https://yale-lily.github.io/spider) 是一个解析自然语言到 sql 的挑战


给定自然语言问题和数据库模式，生成语法正确的 SQL 查询。

尝试了以下方法

* prompt engineering 69%
* RAG 
    + 对问题进行相似性搜索，效果并不好
    + 使用假设的文档嵌入，效果更好
    + 上下文检索，对所得到的问题的难度进行了排名，然后只带回同等难度的示例
    + 尝试链式思维推理。识别列，然后识别表，最后构建查询
    + 自我一致性检查，构建一个查询，运行该查询，如果错误，我们会给它错误消息，再试一次。
* fine-tuning
    + 首先一定要建立基线，使用 prompt engineering，得到 69% 的正确率
    + 使用简单的示例技术进行微调，得到 82% 的正确率
    + 结合使用 RAG，得到 83.5% 的正确率

