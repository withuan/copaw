# 个人知识库系统设计方案

> 创建日期：2026-03-11

## 一、需求背景

拥有大量本地知识库文档（PDF 等常见格式），希望在 CoPaw 中实现：

1. **知识库统一管理**：上传、分类、删除文档
2. **知识沉淀**：文档解析、分块、向量化，持久存储
3. **日常使用**：Agent 对话时可语义检索知识库内容（RAG）

同时有 3 个文档解密 API 接口需要集成：

| 接口 | 功能 | 输出 |
|------|------|------|
| 接口 1 | 单文件解密 | base64 文件流 |
| 接口 2 | 单文件解密 | 保存至桌面 |
| 接口 3 | 文件夹递归解密 | 批量处理结果 |

---

## 二、现有基础设施分析

### 2.1 可复用的组件

| 组件 | 位置 | 能力 |
|------|------|------|
| MemoryManager | `src/copaw/agents/memory/memory_manager.py` | 基于 ReMeLight，支持向量 + BM25 混合检索 |
| Embedding 配置 | 环境变量 `EMBEDDING_API_KEY` / `EMBEDDING_MODEL_NAME` | 已支持阿里云 DashScope 等 |
| 向量存储 | `MEMORY_STORE_BACKEND` | 支持 ChromaDB / SQLite / local |
| memory_search Tool | `src/copaw/agents/tools/memory_search.py` | Agent 可语义搜索 memory 文件 |

### 2.2 Workspace 页面的局限

当前 Workspace（`/workspace`）**不适合**作为知识库入口：

- 仅支持 Markdown 文件
- 文件直接注入系统提示词（大量文档会撑爆上下文窗口）
- 无文档解析、分块、向量化流程
- 定位是 Agent 配置管理，非知识管理

---

## 三、整体架构设计

```
┌─────────────────────────────────────────────────┐
│                 Web Console                      │
│  Knowledge Base 页面（左侧菜单 Agent 分组下）      │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │ 文档列表     │  │ 文档预览 / 分块详情       │  │
│  │ - 上传按钮   │  │                          │  │
│  │ - 文件夹管理 │  │                          │  │
│  │ - 搜索框     │  │                          │  │
│  └─────────────┘  └──────────────────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │ API
┌──────────────────────▼──────────────────────────┐
│              Backend (FastAPI)                    │
│  新增 routers/knowledge.py                       │
│  - POST /knowledge/upload    上传文档             │
│  - GET  /knowledge/files     列出文档             │
│  - DEL  /knowledge/files/:id 删除文档             │
│  - POST /knowledge/search    语义搜索             │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│           Document Processing Pipeline           │
│  1. 文件解析 (PDF→文本, DOCX→文本, 等)            │
│  2. 文本分块 (按段落/固定长度/语义)               │
│  3. 向量化 (复用已有 Embedding 配置)              │
│  4. 存储 (ChromaDB，复用已有 backend)             │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            Agent Integration                     │
│  新增 Tool: knowledge_search                     │
│  Agent 对话时自动检索知识库相关内容                │
└─────────────────────────────────────────────────┘
```

---

## 四、菜单栏位置

在 Agent 分组中新增 **Knowledge Base** 菜单项：

```
Agent 分组:
  - Workspace        ← 原有，Agent 配置 / 记忆
  - Knowledge Base   ← 新增，个人知识库管理
  - Skills
  - Tools
  - MCP
  - Configuration
```

### Workspace vs Knowledge Base 职责划分

| 维度 | Workspace | Knowledge Base |
|------|-----------|----------------|
| 定位 | Agent 配置 + 短期记忆 | 长期知识沉淀 |
| 文件格式 | 仅 Markdown | PDF / DOCX / TXT / MD 等 |
| 使用方式 | 直接注入系统提示词 | RAG 语义检索 |
| 存储 | 文件系统 | 文件系统 + ChromaDB 向量库 |
| Agent 访问 | 自动加载到上下文 | 通过 `knowledge_search` Tool 按需检索 |
| 适合内容 | 规则、人设、偏好 | 技术文档、笔记、参考资料 |

---

## 五、后端实现方案

### 5.1 知识库管理器

```python
# src/copaw/agents/knowledge/manager.py

class KnowledgeManager:
    """知识库管理器，复用已有 embedding 和向量存储基础设施"""

    def __init__(self, storage_dir: Path, vector_store):
        self.storage_dir = storage_dir  # 原始文档存储
        self.vector_store = vector_store  # ChromaDB

    def upload(self, file_path: str, collection: str = "default"):
        """解析 → 分块 → 向量化 → 存入 ChromaDB"""

    def search(self, query: str, top_k: int = 5) -> list[Chunk]:
        """语义检索"""

    def delete(self, file_id: str):
        """删除文档及其所有分块"""

    def list_files(self, collection: str = "default") -> list[FileInfo]:
        """列出所有已入库文档"""
```

### 5.2 文档处理流水线

```
原始文档 → 解析器 → 纯文本 → 分块器 → 文本块列表 → Embedding → 向量存储
```

推荐依赖：

| 格式 | 解析库 |
|------|--------|
| PDF | pypdf / pdfplumber（项目已有 pdf skill） |
| DOCX | python-docx |
| TXT / MD | 直接读取 |
| HTML | beautifulsoup4 |

分块策略：
- 默认按段落分割，每块 512-1024 tokens
- 保留上下文重叠（overlap 128 tokens）
- 记录原始文档路径 + 页码/行号作为元数据

### 5.3 API 路由

```python
# src/copaw/app/routers/knowledge.py

router = APIRouter(prefix="/knowledge", tags=["knowledge"])

@router.post("/upload")
async def upload_document(file: UploadFile, collection: str = "default"):
    """上传文档，自动解析分块向量化"""

@router.get("/files")
async def list_documents(collection: str = "default"):
    """列出所有文档"""

@router.delete("/files/{file_id}")
async def delete_document(file_id: str):
    """删除文档及其向量数据"""

@router.post("/search")
async def search_knowledge(query: str, top_k: int = 5):
    """语义搜索知识库"""
```

### 5.4 存储目录结构

```
~/.copaw/
├── knowledge/                  ← 新增
│   ├── documents/              ← 原始文档存储
│   │   ├── default/            ← 默认集合
│   │   │   ├── doc1.pdf
│   │   │   └── doc2.docx
│   │   └── work/               ← 自定义集合（按项目/主题分类）
│   │       └── spec.pdf
│   └── index/                  ← 向量索引数据（ChromaDB）
├── memory/
├── MEMORY.md
└── config.json
```

---

## 六、Agent Tool 集成

### 6.1 knowledge_search Tool

```python
# src/copaw/agents/tools/knowledge_search.py

async def knowledge_search(
    query: str,
    top_k: int = 5,
    collection: str = "default",
) -> ToolResponse:
    """从个人知识库中语义检索相关内容。

    Args:
        query: 搜索查询
        top_k: 返回结果数量
        collection: 知识库集合名称
    """
```

在 `react_agent.py` 的 `_create_toolkit` 中注册，与 `memory_search` 并列。

### 6.2 与 memory_search 的分工

| Tool | 搜索范围 | 适用场景 |
|------|---------|---------|
| `memory_search` | MEMORY.md + memory/*.md | 对话历史、日常记忆 |
| `knowledge_search` | knowledge/ 下的所有文档 | 技术文档、参考资料 |

Agent 会根据用户问题自动选择合适的 Tool。

---

## 七、文档解密 API 集成

### 7.1 封装为 Tool（推荐）

```python
# src/copaw/agents/tools/decrypt.py

async def decrypt_file_to_stream(file_path: str) -> ToolResponse:
    """解密单个文件，返回 base64 文件流。"""

async def decrypt_file_to_desktop(
    file_path: str, output_dir: str | None = None
) -> ToolResponse:
    """解密单个文件并保存至桌面。"""

async def decrypt_folder(
    folder_path: str, recursive: bool = True
) -> ToolResponse:
    """递归解密文件夹中的所有文档。"""
```

### 7.2 解密 → 知识库流水线

```
加密文档 → decrypt Tool 解密 → KnowledgeManager.upload() → 分块向量化 → Agent 可检索
```

解密后可自动入库，也可由用户手动通过 Knowledge Base 页面上传。

---

## 八、前端实现要点

### 8.1 路由注册

```typescript
// console/src/layouts/Sidebar.tsx — Agent 分组中新增
{ label: "Knowledge Base", path: "/knowledge", icon: BookIcon }

// console/src/layouts/MainLayout/index.tsx — 新增路由
<Route path="/knowledge" element={<KnowledgePage />} />
```

### 8.2 页面组件结构

```
console/src/pages/Agent/Knowledge/
├── index.tsx                 ← 主页面（双栏布局）
├── components/
│   ├── DocumentList.tsx      ← 左栏：文档列表 + 上传 + 筛选
│   ├── DocumentPreview.tsx   ← 右栏：文档预览 / 分块详情
│   ├── UploadDialog.tsx      ← 上传对话框（支持多格式）
│   └── SearchTest.tsx        ← 搜索测试面板
└── hooks/
    └── useKnowledge.ts       ← 数据管理 hook
```

### 8.3 核心功能

- 多格式文件上传（拖拽 + 点击），支持批量
- 文件夹 / 标签分类（collection）
- 文档列表：名称、格式、大小、分块数、入库时间
- 文档预览：原文 + 分块可视化
- 搜索测试：输入 query，预览检索结果及相似度得分
- 删除文档（同时清理向量数据）

---

## 九、实施路线图

| 阶段 | 内容 | 优先级 |
|------|------|--------|
| P0 | 后端：KnowledgeManager + 文档解析 + ChromaDB 存储 | 高 |
| P0 | 后端：knowledge API 路由 | 高 |
| P0 | Agent：knowledge_search Tool 注册 | 高 |
| P1 | 前端：Knowledge Base 页面（上传 / 列表 / 删除） | 中 |
| P1 | 后端：decrypt Tool 封装 | 中 |
| P2 | 前端：搜索测试面板 | 低 |
| P2 | 解密 → 入库自动化流水线 | 低 |
| P2 | 高级分块策略（语义分块、表格处理） | 低 |

---

## 十、技术选型参考

| 组件 | 推荐方案 | 备注 |
|------|---------|------|
| PDF 解析 | pypdf / pdfplumber | 项目已有 pdf skill |
| DOCX 解析 | python-docx | 轻量 |
| 文本分块 | langchain text_splitter 或自实现 | RecursiveCharacterTextSplitter |
| 向量存储 | ChromaDB | 项目已支持 |
| Embedding | 复用 `EMBEDDING_*` 环境变量配置 | 已有阿里云 DashScope |
| HTTP 客户端 | httpx（调解密 API） | 异步支持 |
