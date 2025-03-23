# 📓 Demo Notebook

👉 [点击查看 demo.ipynb](./demo.ipynb)

或者你也可以直接浏览下面的 notebook 👇


```python
# 安装依赖（如未安装）
!pip install sentence-transformers qdrant-client -q

```


```python
from pathlib import Path

# 读取 Markdown 内容
def read_markdown_file(filepath: str) -> str:
    return Path(filepath).read_text(encoding='utf-8')

# 示例文件路径
file_path = "BuildBlogSiteFromScratch.md"
raw_text = read_markdown_file(file_path)
print(raw_text[:100])  # 打印前 500 个字符，确认读取正常

```

    ---
    date: '2025-03-04T22:42:08-08:00'
    title: 'Deploy Hugo Blog From Scratch'
    ---
    
    
    ## Getting Starte



```python
import re

def split_markdown_into_chunks(text: str, max_length: int = 300) -> list:
    lines = text.split('\n')
    chunks = []
    current_chunk = ''

    for line in lines:
        line = line.strip()
        if not line:
            continue

        # 特殊处理配置行 / 代码块 / markdown 标记
        if re.match(r'^```', line) or (':' in line and re.search(r'[a-zA-Z_]+:', line)):
            if current_chunk:
                chunks.append(current_chunk.strip())
                current_chunk = ''
            chunks.append(line)
            continue

        current_chunk += line + ' '
        if len(current_chunk) > max_length:
            chunks.append(current_chunk.strip())
            current_chunk = ''

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks

chunks = split_markdown_into_chunks(raw_text)
print(f"共切分为 {len(chunks)} 个 chunks")
print(chunks[:3])  # 预览前几个块

```

    共切分为 31 个 chunks
    ['---', "date: '2025-03-04T22:42:08-08:00'", "title: 'Deploy Hugo Blog From Scratch'"]



```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")  # 第一次会自动下载

# 向量化每个 chunk
vectors = model.encode(chunks, convert_to_numpy=True)
print(f"每个向量维度：{vectors.shape[1]}")

```

    每个向量维度：1024



```python
import uuid
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct

# 初始化 Qdrant 客户端
client = QdrantClient(host="localhost", port=6333)
collection_name = "blog_site_knowledge"

# 如果 collection 已存在，则删除后创建（模拟 recreate_collection）
if client.collection_exists(collection_name):
    client.delete_collection(collection_name=collection_name)

client.create_collection(
    collection_name=collection_name,
    vectors_config=VectorParams(
        size=vectors.shape[1],
        distance=Distance.COSINE
    )
)

# 插入数据
points = [
    PointStruct(id=str(uuid.uuid4()), vector=vec.tolist(), payload={"text": chunk})
    for vec, chunk in zip(vectors, chunks)
]

client.upsert(collection_name=collection_name, points=points)
print(f"✅ 成功插入 {len(points)} 条向量")

```

    ✅ 成功插入 31 条向量



```python
def search_top1(question: str):
    query_vector = model.encode([question])[0]

    results = client.query_points(
        collection_name=collection_name,
        query=query_vector,  # ✅ 不用再 QueryVector(...)，直接传列表
        limit=1,
        with_payload=True
    )

    if results and results.points:
        point = results.points[0]
        print(f"\n📌 问题: {question}")
        print(f"🔹 匹配段落:\n{point.payload['text']}")
        print(f"🥇 相似度分数: {point.score:.4f}")
    else:
        print("❌ 没找到相关内容")
```

    
    📌 问题: 如何从零开始构建博客网站？
    🔹 匹配段落:
    title: 'Deploy Hugo Blog From Scratch'
    🥇 相似度分数: 0.6703



```python
def search_top3(question: str):
    query_vector = model.encode([question])[0]

    results = client.query_points(
        collection_name=collection_name,
        query=query_vector,           # ✅ 不用 QueryVector
        limit=3,
        with_payload=True             # ✅ 返回原始文本内容
    )

    if results and results.points:
        print(f"\n📌 问题: {question}")
        for i, point in enumerate(results.points, 1):
            print(f"\n--- Top {i} ---")
            print(f"🔹 匹配段落:\n{point.payload['text']}")
            print(f"🥇 相似度分数: {point.score:.4f}")
    else:
        print("❌ 没找到相关内容")

```


```python
test_questions = [
    "默认的测试数量是多少？",
    "default_test_num 是多少？",
    "yaml 文件设置了什么默认值？",
    "How many tests are set by default?",
    "hugo的版本要求是什么"
]

for q in test_questions:
    search_top1(q)
    print("-" * 40)

```

    
    📌 问题: 默认的测试数量是多少？
    🔹 匹配段落:
    default_test_num: 539
    🥇 相似度分数: 0.7077
    ----------------------------------------
    
    📌 问题: default_test_num 是多少？
    🔹 匹配段落:
    default_test_num: 539
    🥇 相似度分数: 0.8141
    ----------------------------------------
    
    📌 问题: yaml 文件设置了什么默认值？
    🔹 匹配段落:
    - Set up default yaml config:
    🥇 相似度分数: 0.7421
    ----------------------------------------
    
    📌 问题: How many tests are set by default?
    🔹 匹配段落:
    default_test_num: 539
    🥇 相似度分数: 0.7162
    ----------------------------------------
    
    📌 问题: hugo的版本要求是什么
    🔹 匹配段落:
    hugo version
    🥇 相似度分数: 0.7891
    ----------------------------------------



```python
search_top3("hugo的版本要求是什么")
```

    
    📌 问题: hugo的版本要求是什么
    
    --- Top 1 ---
    🔹 匹配段落:
    hugo version
    🥇 相似度分数: 0.7891
    
    --- Top 2 ---
    🔹 匹配段落:
    - Verify that you have installed Hugo **v0.128.0** or later.
    🥇 相似度分数: 0.6959
    
    --- Top 3 ---
    🔹 匹配段落:
    - Install [Hugo](https://gohugo.io/installation/)
    🥇 相似度分数: 0.6474

