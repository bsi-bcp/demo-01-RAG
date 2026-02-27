# RAG 课程问答系统

基于检索增强生成（RAG）技术的课程材料智能问答系统，支持语义搜索与多轮工具调用，帮助用户快速获取课程相关知识。

## 效果演示

![系统界面](https://img.shields.io/badge/Web-UI-blue) 用户在网页输入问题 → 系统语义检索相关课程内容 → AI 综合生成答案并标注来源。

## 技术架构

```
┌─────────────────────────────────────────────────────┐
│                   Frontend (静态页面)                 │
│              HTML / CSS / JavaScript                 │
└───────────────────┬─────────────────────────────────┘
                    │ POST /api/query
┌───────────────────▼─────────────────────────────────┐
│                  FastAPI Backend                     │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ RAGSystem   │  │ AIGenerator  │  │ SearchTools│  │
│  │ (编排器)    │→ │ (DeepSeek)   │→ │ (工具调用) │  │
│  └─────────────┘  └──────────────┘  └─────┬──────┘  │
│  ┌──────────────────────────────┐          │         │
│  │      VectorStore (ChromaDB)  │◄─────────┘         │
│  │  course_catalog / course_content                  │
│  └──────────────────────────────┘                   │
└─────────────────────────────────────────────────────┘
```

### 查询流程

1. 前端 POST `{query, session_id}` → `/api/query`
2. RAGSystem 获取会话历史，构建上下文
3. **第一次 DeepSeek 调用**：模型决定调用 `search_course_content` 工具
4. ChromaDB 语义搜索，返回 Top 5 相关文本块
5. **第二次 DeepSeek 调用**（可选）：支持最多 2 轮顺序工具调用，处理复杂多部分问题
6. 模型综合工具结果，生成含来源引用的最终答案

### 项目结构

```
01.RAG/
├── frontend/
│   ├── index.html        # 主页面
│   ├── script.js         # 前端逻辑
│   └── style.css         # 样式
├── backend/
│   ├── app.py            # FastAPI 入口，挂载前端，定义 /api/query 和 /api/courses
│   ├── config.py         # 配置（API Key、模型、分块参数、DB 路径等）
│   ├── rag_system.py     # 主编排器，组合所有组件
│   ├── document_processor.py  # 解析 .txt 课程文件，按 ~800 字符分块（100 字符重叠）
│   ├── vector_store.py   # ChromaDB 封装，管理两个集合
│   ├── ai_generator.py   # DeepSeek API 客户端（OpenAI 兼容 SDK），支持顺序工具调用
│   ├── search_tools.py   # 工具定义 + ToolManager
│   ├── session_manager.py     # 会话历史管理（内存）
│   ├── models.py         # Pydantic 数据模型
│   └── tests/            # 测试套件
├── docs/                 # 课程 .txt 文件（启动时自动加载）
├── pyproject.toml
├── run.sh                # 一键启动脚本
└── .env                  # 环境变量（需自行创建）
```

## 快速开始

### 环境要求

| 项目 | 要求 |
|------|------|
| Python | 3.12（Intel Mac 必须使用 3.12，详见下方说明） |
| 包管理器 | [uv](https://docs.astral.sh/uv/) |
| AI 服务 | DeepSeek API Key |
| 操作系统 | macOS / Linux / Windows（Windows 需用 Git Bash） |

### 安装步骤

**1. 安装 uv**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**2. 安装依赖**

```bash
uv sync
```

**3. 配置环境变量**

在项目根目录创建 `.env` 文件：

```bash
DEEPSEEK_API_KEY=your_deepseek_api_key_here
```

> 获取 DeepSeek API Key：[platform.deepseek.com](https://platform.deepseek.com)

### 启动应用

```bash
# 方式一：使用脚本（推荐）
chmod +x run.sh
./run.sh

# 方式二：手动启动
cd backend && uv run --python 3.12 uvicorn app:app --reload --port 8000
```

访问地址：
- **Web 界面**：http://localhost:8000
- **API 文档**：http://localhost:8000/docs

## 添加课程内容

在 `docs/` 目录下创建 `.txt` 文件，格式如下：

```
Course Title: 课程名称
Course Link: https://example.com/course
Course Instructor: 讲师姓名

Lesson 0: 课程简介
Lesson Link: https://example.com/lesson0
课程内容...

Lesson 1: 第一章标题
Lesson Link: https://example.com/lesson1
本章内容...
```

系统启动时自动扫描并加载 `docs/` 目录下所有课程文件，已加载的课程会跳过重复处理。

## 兼容性说明（Intel Mac）

由于 Intel Mac (x86_64) 的依赖限制，`pyproject.toml` 中固定了以下版本：

| 依赖 | 版本限制 | 原因 |
|------|---------|------|
| `sentence-transformers` | `==2.7.0` | 5.x 需要 torch 2.7+，Intel Mac 不支持 |
| `onnxruntime` | `<1.20` | 1.20+ 已移除 Intel Mac x86_64 wheel |
| `torch` | `<2.3` | 2.3+ 已移除 Intel Mac x86_64 wheel |
| Python | `>=3.12` | torch/onnxruntime 旧版需要 3.12 |

## 运行测试

```bash
cd backend && uv run --python 3.12 pytest tests/ -v
```

## 核心特性

- **语义搜索**：使用 sentence-transformers 生成嵌入向量，ChromaDB 存储与检索
- **多轮工具调用**：支持最多 2 轮顺序工具调用，处理跨课程比较等复杂查询
- **会话记忆**：基于 session_id 维护多轮对话上下文
- **来源引用**：每条回答附带课程和课时来源，便于追溯
- **OpenAI 兼容**：工具调用使用 OpenAI 格式，便于切换模型
