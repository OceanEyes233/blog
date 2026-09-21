---
title: RAG：把文档向量化，基于向量实现真正的语义搜索
tags:
  - AI
  - RAG
categories:
  - AI
---

# RAG 是什么

大模型所知道的知识，取决于在训练的时候给它的数据集。

如果你问它最近发生的事情，或者你企业内部私有文档的一些事情，它是不知道的。

但它很可能不会说自己不知道，而是会胡乱回答，也就是所谓的幻觉（以为自己知道）。

如何解决大模型的幻觉呢？

用户要查询的内容，我们先去内部知识库里查一下，把它放到 prompt 里再给大模型。

这样大模型通过这些文档知道了背景知识，就可以回答响应的问题了。

这就是 RAG：

**Retrieval 检索 - Augmented 增强 - Generation 生成**

去知识库里检索用户问的知识的相关文档片段，作为背景知识加到 prompt 里增强它，让大模型根据这些来生成回答。

在原始 prompt 给到大模型之前，查询下知识库，把相关的文档作为背景知识加入到 Prompt 里，再让大模型回答，这就是 RAG。

RAG 要实现语义查询，需要基于向量来做，把文档向量化存储到向量数据库，查询的时候也把 Prompt 向量化，去数据库中做相似度检索，这样就可以找到语义相近的文档块。

# Loader和Splitter

实际上知识的来源可能有很多：

一个 word 文档、一个 pdf 文件、一个 youtube 视频、一个 url、一个 x 的推文等。

这种显然就不是直接创建 Document 对象了，而是要用各种 loader 来转换，经过对应的 loader 处理后，变成 Document，之后再由嵌入模型向量化后存入知识库。

知识有各种来源，所以对应的各种 loader 也很多：

现在 langchain 文档里有 180+ loader：

https://docs.langchain.com/oss/python/integrations/document_loaders

你可以把各种知识来源通过 loader 转化为文档存入知识库。

当然，有的文档可能会很大，比如一个 pdf 文件可能是一本书的大小。

这种很明显不能直接把转化后的 Document 向量化，需要先拆分文档，也就是需要 Splitter。

大的文档经过 TextSplitter 分割后，变成一个个小文档，再给到嵌入模型做向量化。

分割最简单的就是按照字符，比如换行符 \n，但并不是每一行一个 Document，而是要设置一个 chunk size，按照换行符分割好的内容加入到这个 Chunk，当达到 chunk size 后，再继续生成下个 Chunk。这个 Chunk 也是 Document 对象，只是文档内容是分割好的一个个大小合适的块

```js
const textSplitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,  // 每个分块的字符数 
  chunkOverlap: 50,  // 分块之间的重叠字符数
  separators: ["。", "！", "？"],  // 分割符，优先使用 。段落分隔 先按照separators来分割字符串，然后按照chunkSize来放入一个个的Document 如果分割后还是大于Chunk Size 就需要按照后面的separator继续分割, 然后加上overlap 只有被打断，超过了chunkSize 才会有overlap
});
```

# langchain 都有哪些 splitter 呢？
所有的 Splitter 都继承自 TextSplitter，包括 RecursiveCharacterTextSplitter 等。

而 MarkdownTextSplitter、LatexTextSplitter 又继承自 RecursiveCharacterTextSplitter。

CharacterTextSplitter 是按照某个字符来分割，比如按照句号RecursiveCharacterTextSplitter 是递归分割，比如“ 。 ？ ！”就是先尝试按照 。 分割，如果分割后大于 chunk 剩余空间再按照 ？ 分割，是一个递归过程。而 MarkdownTextSplitter 自然就是按照 #、##、### 等一级级标题来递归分割，所以是RecursiveCharacterTextSplitter 的子类。

CharacterTextSplitter-test.ts
```js
const logTextSplitter = new CharacterTextSplitter({
    separator: '\n',
    chunkSize: 200,
    chunkOverlap: 20
});
```
CharacterTextSplitter 非常死板，你告诉它按照换行符分割，它就会严格按照这个，就算超过了 chunk size 也不拆分。

所以一般还是用 RecursiveCharacterTextSplitter

```js
const logTextSplitter = new RecursiveCharacterTextSplitter({
    chunkSize: 150,
    chunkOverlap: 20,
    separators: ['\n', '。', '，'] // 当 “\n” 分割后还是大，就会用 “。” 还是不行再尝试用 “，”
});
```

那能不能用 RecursiveCharacterTextSplitter 的分割方式，然后按照 token 长度来设置 chunk
size 呢？

可以的。重写一下它的长度计算函数就可以了：

```js
const enc = getEncoding("cl100k_base");
const logTextSplitter = new RecursiveCharacterTextSplitter({
    chunkSize: 150,
    chunkOverlap: 20,
    separators: ['\n', '。', '，'],
    lengthFunction: (text) => enc.encode(text).length,
});
```

现在就是按照现在的 token 数量作为分割依据了。

用 RecursiveCharacterTextSplitter.fromLanguage 这个方法，指定语言，就会按照对应的语
法来分割。

```js
const codeSplitter = RecursiveCharacterTextSplitter.fromLanguage('js', {
    chunkSize: 300,
    chunkOverlap: 60,
})
```

TokenTextSplitter 严格按照 token，会破坏文档语义，不如 RecursiveCharacterTextSplitter
重写 lengthFunction

这节我们把所有 splitter 过了一遍。

结论是直接用 RecursiveCharacterTextSplitter 就行。

splitter 是先按照 sperator 来分割，然后按照 chunk size 放到一个个 chunk 里。

chunk 的实际大小可能小于 chunk size 也可以大于。

如果分割后文本长度大于 chunk size，会继续按照后面的 sperator 拆分，然后放到两个chunk 里，加上 overlap 来保证语义连贯。

如果从前到后尝试 sperator，尝试到最后一个，拆分完还是大于 chunk size 就不会再拆分
了。

chunkSize 更像是“尽量遵守的目标上限”，而不是绝对上限；它优先保证自然的语义边界。