# AI 快速入门实操路线

这是一份可以照着执行的 14 天入门路线。目标不是把所有 AI 理论学完，而是让你能：

- 知道基本 AI 概念是什么意思。
- 理解一次大模型调用的大致运行机制。
- 写出可运行 demo。
- 做简单自定义。
- 会检查基础问题。

建议每天投入 1 到 2 小时。代码语言默认使用 Python，因为入门资料多、生态简单。如果你更熟 JavaScript，也可以把同样任务改成 Node.js。

## 目录建议

在 `AI-cookbook` 文件夹下建立这些目录：

```text
AI-cookbook/
  00-notes/
  01-hello-ai/
  02-prompt-lab/
  03-json-extraction/
  04-rag-mini/
  05-tool-calling/
  06-final-project/
```

如果你想快速创建目录，可以在终端执行：

```bash
cd ~/Desktop/AI-cookbook
mkdir -p 00-notes 01-hello-ai 02-prompt-lab 03-json-extraction 04-rag-mini 05-tool-calling 06-final-project
```

## 准备工作

### 需要安装

- Python 3.10 或更新版本。
- VS Code 或其他代码编辑器。
- Git，可选但建议安装。
- 一个可用的大模型 API key，例如 OpenAI API key 或其他兼容 OpenAI SDK 的服务。

### 推荐资料入口

优先看官方文档和短示例，不要一开始看长课。

- OpenAI API 文档：https://platform.openai.com/docs
- OpenAI Python SDK：https://github.com/openai/openai-python
- OpenAI Cookbook：https://cookbook.openai.com
- Hugging Face NLP Course：https://huggingface.co/learn/nlp-course
- LangChain RAG 教程：https://python.langchain.com/docs/tutorials/rag/

学习方法：

- 先跑官方最小示例。
- 再改成自己的小任务。
- 最后记录你改了什么、为什么有效。

## API Key 配置

不要把 key 写进代码。建议用环境变量。

macOS / Linux：

```bash
export OPENAI_API_KEY="你的 key"
```

检查是否生效：

```bash
echo $OPENAI_API_KEY
```

如果你使用 `.env` 文件，可以在项目目录创建：

```text
OPENAI_API_KEY=你的 key
```

然后代码里用 `python-dotenv` 读取。

## 第 1 天：跑通第一次模型调用

目标：能从代码里调用模型并看到回答。

资料：

- OpenAI API 文档里的 Quickstart。
- OpenAI Python SDK README。

目录：

```bash
cd ~/Desktop/AI-cookbook/01-hello-ai
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv
```

创建 `.env`：

```text
OPENAI_API_KEY=你的 key
```

创建 `hello_ai.py`：

```python
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

client = OpenAI()

response = client.responses.create(
    model="gpt-4.1-mini",
    input="用三句话解释什么是大语言模型，适合初学者。"
)

print(response.output_text)
```

运行：

```bash
python hello_ai.py
```

验收标准：

- 终端能看到模型回答。
- 你能说清楚 `client.responses.create` 是在发起一次模型请求。
- API key 没有写在 `hello_ai.py` 里。

常见问题：

- `OPENAI_API_KEY` 找不到：检查 `.env` 是否在当前目录，是否调用了 `load_dotenv()`。
- 模型名错误：换成你账号可用的模型。
- 网络或额度错误：看错误码，不要只看最后一行。

记录到 `00-notes/day01.md`：

```markdown
# Day 01

今天跑通了什么：

遇到的问题：

怎么解决的：

我理解的一次 API 调用流程：
```

## 第 2 天：理解 messages、system prompt 和上下文

目标：知道系统提示词、用户输入、历史上下文分别起什么作用。

资料：

- OpenAI 文档中关于 text generation / conversations 的部分。

目录：

```bash
cd ~/Desktop/AI-cookbook/02-prompt-lab
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv
```

创建 `chat_with_role.py`：

```python
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

system_prompt = """
你是一个严谨但容易理解的 AI 入门老师。
回答要求：
1. 先给结论。
2. 再给一个生活类比。
3. 最后给一个可以动手验证的小实验。
"""

user_question = "什么是 token？"

response = client.responses.create(
    model="gpt-4.1-mini",
    input=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_question},
    ],
)

print(response.output_text)
```

运行：

```bash
python chat_with_role.py
```

改 3 次：

1. 把老师改成“面试官”。
2. 把回答要求改成“只用表格回答”。
3. 把问题改成“RAG 和微调有什么区别？”。

验收标准：

- 你能看出 system prompt 改变了回答风格。
- 你能解释“模型不是记住你的程序状态，而是根据本次传入的上下文回答”。
- 你能写出一个自己常用场景的 system prompt。

记录：

```markdown
# Day 02

我写的 system prompt：

它改变了模型哪些行为：

上下文是什么意思：
```

## 第 3 天：做 Prompt 对比实验

目标：学会用实验方式改 prompt，而不是凭感觉乱调。

资料：

- OpenAI Cookbook 中 prompt engineering 相关示例。

创建 `prompt_compare.py`：

```python
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

task = """
请把下面的客服反馈分类为：价格问题、质量问题、物流问题、其他。
反馈：这个耳机音质还可以，但是等了 10 天才收到，太慢了。
"""

prompts = [
    "请完成分类。",
    "你是客服质检助手。只输出一个分类名称，不要解释。",
    "你是客服质检助手。先判断用户主要抱怨点，再只输出一个分类名称。分类只能是：价格问题、质量问题、物流问题、其他。",
]

for index, prompt in enumerate(prompts, start=1):
    response = client.responses.create(
        model="gpt-4.1-mini",
        input=f"{prompt}\n\n{task}",
        temperature=0,
    )
    print(f"\n--- Prompt {index} ---")
    print(response.output_text)
```

运行：

```bash
python prompt_compare.py
```

你要观察：

- 哪个 prompt 输出最稳定。
- 哪个 prompt 最容易多说废话。
- 分类边界是否清楚。

验收标准：

- 你能写出“任务 + 角色 + 约束 + 输出格式 + 示例”结构的 prompt。
- 你能说明为什么 `temperature=0` 更适合分类、抽取、格式化任务。

练习：

把客服分类改成“会议纪要行动项提取”，要求输出：

```text
负责人：
事项：
截止时间：
```

## 第 4 天：结构化输出 JSON

目标：能让模型输出可被程序读取的数据。

资料：

- OpenAI 文档中 Structured Outputs / JSON schema 部分。

目录：

```bash
cd ~/Desktop/AI-cookbook/03-json-extraction
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv
```

创建 `extract_task.py`：

```python
import json
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

text = """
张三负责整理 Q2 销售数据，下周三前发给李经理。
王五负责检查客户名单，明天下午 5 点前完成。
"""

schema = {
    "type": "object",
    "properties": {
        "tasks": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "owner": {"type": "string"},
                    "task": {"type": "string"},
                    "deadline": {"type": "string"},
                },
                "required": ["owner", "task", "deadline"],
                "additionalProperties": False,
            },
        }
    },
    "required": ["tasks"],
    "additionalProperties": False,
}

response = client.responses.create(
    model="gpt-4.1-mini",
    input=f"从文本中抽取任务：\n{text}",
    text={
        "format": {
            "type": "json_schema",
            "name": "task_extraction",
            "schema": schema,
            "strict": True,
        }
    },
)

data = json.loads(response.output_text)
print(json.dumps(data, ensure_ascii=False, indent=2))
```

运行：

```bash
python extract_task.py
```

验收标准：

- 输出是合法 JSON。
- `json.loads` 不报错。
- 每条任务都有 `owner`、`task`、`deadline`。

扩展练习：

- 加一个字段 `priority`，取值只能是 `high`、`medium`、`low`。
- 输入一段没有明确截止时间的文本，看模型如何处理。
- 如果 deadline 不明确，要求输出 `"未提及"`。

排错：

- JSON 解析失败：优先使用结构化输出，不要只靠“请输出 JSON”。
- 字段缺失：检查 schema 的 required。
- 模型加解释文字：检查是否使用了 JSON schema strict 模式。

## 第 5 天：理解 Embedding 和相似度搜索

目标：知道 RAG 的检索部分是怎么工作的。

资料：

- OpenAI Cookbook 中 embedding search 示例。
- Hugging Face NLP Course 中 embedding 相关章节。

目录：

```bash
cd ~/Desktop/AI-cookbook/04-rag-mini
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv numpy
```

创建 `search_demo.py`：

```python
import numpy as np
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

docs = [
    "RAG 是检索增强生成，先查资料，再让模型基于资料回答。",
    "微调是用训练样本调整模型行为，适合固定风格或固定任务。",
    "Temperature 越低，输出越稳定；越高，输出越有随机性。",
    "Embedding 会把文本转换成向量，用于相似度搜索。",
]

def embed(text):
    result = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return np.array(result.data[0].embedding)

doc_vectors = [embed(doc) for doc in docs]

query = "怎么让模型基于我的资料回答？"
query_vector = embed(query)

scores = []
for doc, vector in zip(docs, doc_vectors):
    score = np.dot(query_vector, vector) / (
        np.linalg.norm(query_vector) * np.linalg.norm(vector)
    )
    scores.append((score, doc))

scores.sort(reverse=True)

print("问题：", query)
print("\n最相关资料：")
for score, doc in scores[:2]:
    print(round(float(score), 4), doc)
```

运行：

```bash
python search_demo.py
```

验收标准：

- 输入“怎么让模型基于我的资料回答？”时，RAG 相关句子排在前面。
- 你能解释 embedding 不是直接回答问题，而是帮助找到相关资料。

练习：

- 增加 10 条自己的 AI 笔记。
- 改 3 个不同 query，观察命中的资料是否合理。
- 把 top 2 改成 top 3，观察召回变多是否引入噪声。

## 第 6 天：最小 RAG 文档问答

目标：做一个“先检索，再回答”的最小 demo。

创建 `rag_answer.py`：

```python
import numpy as np
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

docs = [
    "RAG 是检索增强生成，适合让模型基于外部资料回答。",
    "RAG 的关键步骤包括：文档切块、向量化、相似度检索、把检索结果放进 prompt。",
    "微调适合让模型学习固定输出风格、专业术语或稳定任务模式。",
    "当回答不准确时，应该先检查检索结果是否相关，再检查 prompt。",
]

def embed(text):
    result = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return np.array(result.data[0].embedding)

def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

doc_vectors = [embed(doc) for doc in docs]

def retrieve(query, top_k=2):
    query_vector = embed(query)
    scored = [
        (cosine(query_vector, vector), doc)
        for doc, vector in zip(docs, doc_vectors)
    ]
    scored.sort(reverse=True)
    return [doc for _, doc in scored[:top_k]]

question = "RAG 回答不准确时，我应该先检查什么？"
contexts = retrieve(question)

prompt = f"""
你只能基于下面资料回答问题。
如果资料中没有答案，就回答：资料不足，无法回答。

资料：
{chr(10).join(f"- {item}" for item in contexts)}

问题：
{question}
"""

response = client.responses.create(
    model="gpt-4.1-mini",
    input=prompt,
    temperature=0,
)

print("检索到的资料：")
for item in contexts:
    print("-", item)

print("\n回答：")
print(response.output_text)
```

运行：

```bash
python rag_answer.py
```

验收标准：

- 程序先打印检索到的资料，再打印回答。
- 回答内容能从资料中找到依据。
- 你能判断“回答错了”时应该先看检索结果。

练习：

- 问一个资料里没有的问题，确认模型会说资料不足。
- 故意把 top_k 改成 1，观察是否漏掉信息。
- 故意加入一条干扰资料，观察是否影响回答。

## 第 7 天：文档切块

目标：知道真实文档不能整篇乱塞给模型，需要切块。

创建 `chunk_demo.py`：

```python
text = """
RAG 是检索增强生成。它的基本思想是先从知识库里检索相关资料，
再把资料交给大模型生成回答。这样可以减少模型编造，也可以让模型使用私有知识。

文档切块是 RAG 的关键步骤。切块太大，会带入太多无关信息。
切块太小，又可能丢失上下文。常见做法是设置 chunk size 和 overlap。

当 RAG 回答不准确时，不要马上改 prompt。应该先打印检索结果，
确认模型看到的资料是否正确。如果检索结果错了，生成阶段很难答对。
"""

def chunk_text(text, max_chars=80, overlap=20):
    chunks = []
    start = 0
    text = text.strip()
    while start < len(text):
        end = start + max_chars
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

chunks = chunk_text(text)

for index, chunk in enumerate(chunks, start=1):
    print(f"\n--- chunk {index} ---")
    print(chunk)
```

运行：

```bash
python chunk_demo.py
```

验收标准：

- 你能看到一篇文本被切成多个片段。
- 你能解释 overlap 是为了保留上下文连续性。

练习：

- 把 `max_chars` 改成 40、120，对比效果。
- 把 overlap 改成 0、30，对比是否断句严重。
- 写一段你自己的笔记文本来切块。

## 第 8 天：做一个本地资料问答

目标：把 Markdown 文件作为知识库。

创建 `notes.md`：

```markdown
# AI 学习笔记

RAG 是检索增强生成，适合连接私有资料。

Embedding 把文本转成向量，用来做相似度搜索。

Temperature 控制输出随机性，分类和抽取任务建议用较低值。

结构化输出适合把模型结果交给程序继续处理。
```

创建 `ask_notes.py`，把第 6 天和第 7 天合并：

```python
import numpy as np
from pathlib import Path
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

def chunk_text(text, max_chars=80, overlap=20):
    chunks = []
    start = 0
    text = text.strip()
    while start < len(text):
        end = start + max_chars
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

def embed(text):
    result = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return np.array(result.data[0].embedding)

def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

text = Path("notes.md").read_text(encoding="utf-8")
chunks = chunk_text(text)
vectors = [embed(chunk) for chunk in chunks]

question = input("请输入问题：")
question_vector = embed(question)

scored = [
    (cosine(question_vector, vector), chunk)
    for chunk, vector in zip(chunks, vectors)
]
scored.sort(reverse=True)
contexts = [chunk for _, chunk in scored[:2]]

prompt = f"""
你是一个知识库问答助手。
只能基于资料回答。
回答后列出你参考的资料片段。

资料：
{chr(10).join(contexts)}

问题：
{question}
"""

response = client.responses.create(
    model="gpt-4.1-mini",
    input=prompt,
    temperature=0,
)

print("\n回答：")
print(response.output_text)
```

运行：

```bash
python ask_notes.py
```

测试问题：

```text
RAG 适合做什么？
Temperature 应该怎么设置？
结构化输出有什么用？
```

验收标准：

- 能从 `notes.md` 中回答问题。
- 回答会显示参考片段。
- 你能通过修改 `notes.md` 改变回答内容。

## 第 9 天：工具调用的基本思想

目标：理解模型不能真的执行动作，必须由程序执行工具。

资料：

- OpenAI 文档中 function calling / tool calling 部分。

先做不用模型的工具：

创建 `tools.py`：

```python
def calculate_total(price, count):
    return price * count

def get_order_status(order_id):
    fake_orders = {
        "A001": "已发货",
        "A002": "待付款",
        "A003": "已取消",
    }
    return fake_orders.get(order_id, "订单不存在")

print(calculate_total(99, 3))
print(get_order_status("A001"))
```

运行：

```bash
python tools.py
```

验收标准：

- 你能说清楚：工具结果来自你的代码，不是模型编造。
- 你能写出 2 个简单工具函数。

## 第 10 天：让模型选择工具

目标：做一个最小工具调用 demo。

目录：

```bash
cd ~/Desktop/AI-cookbook/05-tool-calling
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv
```

创建 `tool_calling_demo.py`：

```python
import json
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI()

def get_order_status(order_id):
    fake_orders = {
        "A001": "已发货",
        "A002": "待付款",
        "A003": "已取消",
    }
    return fake_orders.get(order_id, "订单不存在")

tools = [
    {
        "type": "function",
        "name": "get_order_status",
        "description": "查询订单状态",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "订单号，例如 A001",
                }
            },
            "required": ["order_id"],
            "additionalProperties": False,
        },
    }
]

response = client.responses.create(
    model="gpt-4.1-mini",
    input="帮我查一下订单 A001 现在是什么状态。",
    tools=tools,
)

for item in response.output:
    if item.type == "function_call":
        args = json.loads(item.arguments)
        result = get_order_status(args["order_id"])
        print("模型决定调用工具：", item.name)
        print("工具参数：", args)
        print("工具结果：", result)
```

运行：

```bash
python tool_calling_demo.py
```

验收标准：

- 你能看到模型选择了 `get_order_status`。
- 你能看到工具参数和工具结果。
- 你能解释模型只是决定调用哪个工具，真正查询由 Python 函数完成。

扩展练习：

- 增加 `calculate_total(price, count)` 工具。
- 输入“99 元的商品买 3 件多少钱”，让模型选择计算工具。
- 对不存在的订单号返回清晰提示。

## 第 11 天：给 demo 加日志和错误处理

目标：学会检查问题，而不是只看模型回答。

给任意一个 demo 增加这些日志：

```python
print("用户问题：", question)
print("检索片段数量：", len(contexts))
print("检索片段：")
for item in contexts:
    print("-", item)
```

给 API 调用加 try/except：

```python
try:
    response = client.responses.create(
        model="gpt-4.1-mini",
        input="你好",
    )
    print(response.output_text)
except Exception as error:
    print("调用失败：", error)
```

验收标准：

- 程序出错时不会只有一大串看不懂的报错。
- RAG 回答错时，你能先看到检索片段。
- 你能记录问题发生在哪一步：输入、检索、模型调用、解析输出、工具执行。

排错模板，保存为 `00-notes/troubleshooting.md`：

```markdown
# AI Demo 排错记录

## 问题现象

## 复现步骤

## 输入是什么

## 模型看到的上下文是什么

## 检索结果是什么

## 模型原始输出是什么

## 程序报错是什么

## 最终原因

## 解决办法
```

## 第 12 天：做最终项目的最小版本

目标：开始做一个完整小应用。

推荐项目：个人 AI 知识库助手。

功能范围只做 4 件事：

1. 读取 `notes.md`。
2. 切块。
3. 根据问题检索 top 2。
4. 基于检索片段回答并显示引用。

目录：

```bash
cd ~/Desktop/AI-cookbook/06-final-project
python3 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv numpy
```

文件：

```text
06-final-project/
  .env
  notes.md
  app.py
  README.md
```

`README.md` 必须写清楚：

```markdown
# 个人 AI 知识库助手

## 功能

## 如何运行

## 如何添加资料

## 如何排错

## 当前限制
```

验收标准：

- 另一个人按 README 能跑起来。
- 换一份 `notes.md` 后，回答会跟着变化。
- 问资料外的问题时，不会乱编。

## 第 13 天：自定义你的助手

目标：让 demo 有一点自己的特点。

从下面选 2 个做：

### 自定义 1：回答风格

增加 system prompt：

```text
你是我的 AI 学习教练。
回答要短、直接、可执行。
如果资料不足，明确说资料不足。
每次回答最后给一个下一步练习。
```

验收：

- 回答明显变得更像学习教练。
- 资料不足时不会硬答。

### 自定义 2：问题类型

支持用户输入模式：

```text
summary: 总结资料
qa: 根据资料回答
action: 提取行动项
```

验收：

- 同一份资料在不同模式下输出不同结果。

### 自定义 3：结构化输出

让 `action` 模式输出 JSON：

```json
{
  "actions": [
    {
      "task": "string",
      "owner": "string",
      "deadline": "string"
    }
  ]
}
```

验收：

- Python 可以 `json.loads` 成功。

### 自定义 4：配置化

把这些写到 `config.json`：

```json
{
  "model": "gpt-4.1-mini",
  "top_k": 2,
  "temperature": 0
}
```

验收：

- 改 `top_k` 不需要改 Python 代码。

## 第 14 天：学习成果验收

目标：证明你已经不是只会看概念，而是能做出东西。

### 必做验收

1. 从零运行 `01-hello-ai/hello_ai.py`。
2. 展示一个 prompt 改动前后的输出对比。
3. 运行 JSON 抽取 demo，并证明输出能被 `json.loads` 解析。
4. 运行 RAG demo，并展示检索片段。
5. 问一个资料外问题，模型回答“资料不足”。
6. 运行工具调用 demo，展示工具参数和工具结果。
7. 打开最终项目 README，按步骤重新运行。

### 你需要能口头解释

- Prompt 是什么。
- Token 和上下文窗口大概是什么。
- Temperature 影响什么。
- Embedding 是做什么的。
- RAG 的 4 个步骤是什么。
- 为什么回答不准要先看检索结果。
- 工具调用里模型和代码分别负责什么。
- 为什么 API key 不能写进代码。

### 最终交付物

你的 `AI-cookbook` 里应该至少有：

```text
00-notes/day01.md
00-notes/troubleshooting.md
01-hello-ai/hello_ai.py
02-prompt-lab/chat_with_role.py
02-prompt-lab/prompt_compare.py
03-json-extraction/extract_task.py
04-rag-mini/search_demo.py
04-rag-mini/rag_answer.py
04-rag-mini/chunk_demo.py
04-rag-mini/ask_notes.py
05-tool-calling/tool_calling_demo.py
06-final-project/app.py
06-final-project/README.md
```

## 常见问题检查表

### API 调不通

检查顺序：

1. `.env` 是否在运行脚本的当前目录。
2. 是否调用 `load_dotenv()`。
3. API key 是否正确。
4. 模型名是否可用。
5. 是否有额度。
6. 错误码是什么。

### 回答不准

检查顺序：

1. 用户问题是否清楚。
2. prompt 是否要求基于资料回答。
3. RAG 检索片段是否相关。
4. chunk 是否太小或太大。
5. temperature 是否太高。
6. 是否缺少“资料不足就不要编”的约束。

### JSON 不稳定

检查顺序：

1. 是否使用 JSON schema。
2. 是否要求 strict。
3. schema 里的 required 是否正确。
4. 是否允许 additionalProperties。
5. 是否用 `json.loads` 做校验。

### 工具调用不符合预期

检查顺序：

1. 工具 description 是否清楚。
2. 参数 schema 是否清楚。
3. 用户问题是否真的需要工具。
4. 程序是否执行了工具函数。
5. 是否把工具结果返回给模型继续生成最终回答。

## 推荐继续学习方向

完成 14 天后，再选一个方向深入：

- 应用开发：继续做知识库、客服助手、报告生成器。
- RAG 深入：学习向量数据库、重排序、混合检索、引用溯源。
- Agent 深入：学习规划、工具选择、状态管理、失败恢复。
- 模型理解：学习 Transformer、attention、训练数据、评估方法。
- 工程化：学习日志、监控、缓存、成本控制、安全策略。

最推荐的下一步：把最终项目改成一个真实可用的小工具，用你自己的资料、你自己的问题、你自己的工作流来验证。

