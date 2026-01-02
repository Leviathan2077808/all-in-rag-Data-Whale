# JSONL数据处理和索引构建与Markdown数据处理和索引构建的区别

本文参考 8.2 的数据准备流程与 8.3 的索引构建流程，聚焦**JSONL源数据的处理与索引构建**，并在末尾给出与Markdown的关键差异点。

> [!NOTE]
>
> 一般什么场景下源数据会是JSONL？
>
> - **海量/增量数据落盘**：数据不断追加写入（append-only），按行写更稳、更容易做断点续跑。
> - **日志与事件流**：埋点、服务日志、消息队列落地（每条事件=一行记录）。
> - **离线ETL与分布式处理**：Spark/Hadoop/MapReduce 常用“记录流”思维处理，一行一条便于并行切分与流式解析。
> - **数据集发布与交换**：跨团队/跨系统导出“文档集合”（每条文档一行），比一个巨大 JSON 数组更好处理。
> - **LLM 训练/微调语料**：指令数据、对话数据、QA、偏好数据等通常按样本存一行（便于采样、过滤、打乱）。
> - **爬虫/信息抽取结果**：每个页面/实体/段落抽取结果天然是一条记录，逐条写入便于容错（坏行可跳过）。
> - **检索/索引源文档**：要把“很多条文档+元数据”送去建索引时，JSONL 很适合做中间交换格式。



## 一、JSONL源数据处理（对标 8.2）

JSONL是一行一个JSON对象的结构化数据。处理要点是**逐行解析、字段校验、元数据补全、分块与父子关系建立**。

### 1.1 JSONL读取与校验

```python
import json
import uuid
from pathlib import Path
from typing import List, Dict, Any
from langchain.schema import Document

REQUIRED_FIELDS = ("id", "title", "content")

def load_jsonl_documents(jsonl_path: str) -> List[Document]:
    """逐行读取JSONL，构建父文档列表"""
    documents = []
    with open(jsonl_path, "r", encoding="utf-8") as f:
        for line_no, line in enumerate(f, start=1):
            line = line.strip()
            if not line:
                continue
            try:
                record = json.loads(line)
            except json.JSONDecodeError:
                # 行级容错：坏行跳过，保留可继续处理
                continue

            # 字段校验
            if not all(field in record for field in REQUIRED_FIELDS):
                continue

            parent_id = record.get("id") or str(uuid.uuid4())
            doc = Document(
                page_content=record.get("content", ""),
                metadata={
                    "parent_id": parent_id,
                    "title": record.get("title", ""),
                    "tags": record.get("tags", []),
                    "category": record.get("category", "其他"),
                    "source_line": line_no,
                    "doc_type": "parent"
                }
            )
            documents.append(doc)

    return documents
```

### 1.2 元数据增强

JSONL通常自带标签、分类、时间等字段，可以直接落到`metadata`中；也可补充**行号、来源文件、缺省字段**等信息。

```python
def enhance_jsonl_metadata(doc: Document, source_path: str):
    """补充来源信息与默认字段"""
    doc.metadata.setdefault("source", str(Path(source_path)))
    doc.metadata.setdefault("category", "其他")
    doc.metadata.setdefault("tags", [])
```

### 1.3 分块与父子关系

JSONL常见的做法是**按字段或长度切分**。如果记录只有一个`content`字段，则按长度切分；若存在`sections`字段，可按字段分块。

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

def chunk_jsonl_documents(documents: List[Document]) -> List[Document]:
    """按长度切分，生成子块并建立父子关系"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=400,
        chunk_overlap=80
    )

    chunks = []
    for doc in documents:
        parent_id = doc.metadata["parent_id"]
        for i, text in enumerate(splitter.split_text(doc.page_content)):
            child_id = str(uuid.uuid4())
            chunk = Document(
                page_content=text,
                metadata={
                    **doc.metadata,
                    "chunk_id": child_id,
                    "parent_id": parent_id,
                    "doc_type": "child",
                    "chunk_index": i,
                    "chunk_size": len(text)
                }
            )
            chunks.append(chunk)

    return chunks
```

## 二、JSONL索引构建（对标 8.3）

索引构建流程与Markdown一致：**向量化 -> FAISS索引 -> 持久化缓存**。

### 2.1 构建向量索引

```python
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import FAISS

def build_jsonl_vector_index(chunks: List[Document]) -> FAISS:
    embeddings = HuggingFaceEmbeddings(
        model_name="BAAI/bge-small-zh-v1.5",
        model_kwargs={"device": "cpu"},
        encode_kwargs={"normalize_embeddings": True}
    )

    texts = [c.page_content for c in chunks]
    metadatas = [c.metadata for c in chunks]

    vectorstore = FAISS.from_texts(
        texts=texts,
        embedding=embeddings,
        metadatas=metadatas
    )
    return vectorstore
```

### 2.2 索引缓存（加速启动）

```python
from pathlib import Path

def save_index(vectorstore: FAISS, index_path: str):
    Path(index_path).mkdir(parents=True, exist_ok=True)
    vectorstore.save_local(index_path)

def load_index(index_path: str, embeddings: HuggingFaceEmbeddings):
    if not Path(index_path).exists():
        return None
    return FAISS.load_local(
        index_path,
        embeddings,
        allow_dangerous_deserialization=True
    )
```


> 结论：JSONL强调**结构化字段与流式处理**，Markdown强调**文档结构与语义层级**。索引构建阶段可复用同一套向量化与FAISS流程。
