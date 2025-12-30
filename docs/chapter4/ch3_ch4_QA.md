# Chapter 3 - Chapter 4 笔记和QA汇总
## Chapter 3
生成词向量的Embedding模型为什么叫做“嵌入”？

- 在数学里，**embedding** 有一个非常明确的定义：<u>Embedding是一种把一个空间映射到另一个空间的函数，并且尽可能保留原空间的结构关系。</u>
  - **关键不是“变成向量”，而是“关系被保留”**： 例如：原空间中“相似”的对象，在新空间中“距离也相近”，可以用几何操作表达语义 ——  这才是embedding 的本质
  - 例如：king - man + woman ≈ queen
- **普通的“向量化” $\neq$ “嵌入”**：早年**one-hot**等编码方式生成的词向量，只能单纯代表词表中的不同词，而无法体现每个词<u>相互之间的语义关系</u> ；从**Word2Vec**模型开始，词向量才真正有了“嵌入”的含义

## Chapter 4 第二节 思考
- 为什么本节的代码中查询“时间最短的视频”时，得到的结果是错误的？

回答：因为这次 **SelfQueryRetriever 生成的结构化查询里根本没有把“最短”翻译成元数据排序/聚合**，它只做了 limit=1，**没有 filter、也没有 sort**：

```
Generated Query: query=' ' filter=None limit=1
```

在这种情况下，Chroma 只能按“向量相似度检索 + 取前 1 条”返回结果；但你的 query=' '（几乎空查询）对向量检索没有任何语义约束，最后就等价于“随便给我一条/默认顺序的一条”，所以拿到的并不是 **length 最小** 的那条。

**更具体地说，错在这几件事叠加:**

1. **SelfQueryRetriever 的能力边界**

   - 它擅长把“时长 > 600”这种条件翻译成 filter=GT(length,600) 

   - 但“时间最短”属于 **排序/最小值（argmin）** 需求，SelfQueryRetriever 默认不一定会生成 **ORDER BY length ASC** 这种东西（很多向量库接口也不支持排序）。

   - 结果它只能退化成 limit=1。

2. **生成的** **query=' '** **是空语义检索**

   - 它没有用文本相似度检索“最短”的语义（因为“最短”不是内容语义）。

   - query 为空，向量检索无法起到筛选作用。

3. **Chroma 默认不会按 length 排序**

   - Chroma 的检索核心是向量近邻，不会自动“按某个 metadata 字段排序”。

   - 所以你拿到的第一条只是“top-1 近邻/默认返回顺序”的结果。

**解决方案：**

最优工程解决方案参见[第四节 查询重构与分发]

## Chapter 4 第五节 思考
- 本节“自定义重排器与压缩管道”部分的代码运行后的输出会出现重复的情况，思考为什么会出现这个问题并尝试修改代码解决。（[参考代码](https://github.com/datawhalechina/all-in-rag/blob/main/code/C4/work_rerank_and_refine.py)）

### 问题原因：

**1) 源码里做了 chunk_overlap=100，导致相邻 chunk 本来就高度重复**

源码的切分参数是：chunk_size=500, chunk_overlap=100

相邻 chunk 会共享 100 个字符。

当问到“AI还有哪些缺陷需要克服？”这种宽泛问题时，**Top20 很可能召回一堆相邻 chunk**。

接下来 LLMChainExtractor 会从这些 chunk 里抽取“相关句子”，而这些相关句子很可能正好落在 overlap 区域，于是：

​	chunk A 抽到了那句话

​	chunk B（相邻且重叠）也抽到了同一句话

​	→ 输出重复

**2) LLMChainExtractor 可能对不同 chunk 抽出相同摘要片段**

就算 chunk 不重叠，只要多个 chunk 都包含类似表述，Extractor 也可能抽出相同/近似文本。

而 DocumentCompressorPipeline 默认**不会做去重**。
