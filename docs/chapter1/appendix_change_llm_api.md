# 如何在项目中更换Embedding和大语言模型的调用（以OpenAI系的模型为例）
在生产级的项目里，公司可能会希望采用别的大模型来搭建并测试MVP，或者后期希望实现模型热插拔更换。OpenAI系的模型（包括embedding模型和大语言模型）采用率还是比较高的，这里就以 OpenAI API 为例，说明在 RAG 系统中如何更换大语言模型和 Embedding 模型的调用方式。
## 1. 更换大语言模型（使用gpt-4o等）
### 1.1 环境变量更新
首先在.env或者.bashrc里新增下面这几个环境变量：
```bash
OPENAI_API_KEY=[你的OpenAI API Key]
OPENAI_BASE_URL=https://api.openai.com/v1 
EMBEDDING_MODEL=text-embedding-3-large
LLM_MODEL=gpt-4o
```
注意base url是什么要根据你想用的模型去查看openai官网文档，如gpt-4o就是在/v1目录下。

### 1.2 调用代码更新
修改 01_langchain_example.py中的如下部分：  

**1.更换引入的LangChain模块**  
将
```python
from langchain_deepseek import ChatDeepSeek
```
改为
```python
from langchain_openai import ChatOpenAI
```
**2.替换LLM初始化代码**  
将
```python
llm = ChatDeepSeek(
    model="deepseek-chat",
    temperature=0.7,
    max_tokens=4096,
    api_key=os.getenv("DEEPSEEK_API_KEY")
)
```
改为
```python
llm = ChatOpenAI(
    model=os.getenv("LLM_MODEL", "gpt-4o"),
    temperature=0.7,
    max_tokens=4096,
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY")
)
```
## 2. 更换Embedding模型（使用text-embedding-3-large等）
### 2.1 环境变量更新
同上。
### 2.2 调用代码更新
**1.更换引入的模块**  
将
```python
from langchain_huggingface import HuggingFaceEmbeddings
```
改为
```python
from langchain_openai import OpenAIEmbeddings
```

**2.替换Embedding定义**
```python
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",  # 或 text-embedding-3-small
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY")
)
```

完成。
