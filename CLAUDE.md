# rag_api

## 项目概述

基于 FastAPI + LangChain 的异步 RAG（检索增强生成）服务，以 `file_id` 为粒度对文档进行向量化索引和检索。主要用于与 [LibreChat](https://librechat.ai) 集成，也可作为通用的 ID-based RAG 后端。

- 默认向量数据库：PostgreSQL + pgvector
- 可选向量数据库：Atlas MongoDB
- 默认嵌入模型提供商：OpenAI (`text-embedding-3-small`)

---

## 目录结构

```
rag_api/
├── main.py                  # FastAPI 应用入口，lifespan 管理线程池与 DB 连接
├── app/
│   ├── config.py            # 所有环境变量读取与全局配置
│   ├── constants.py         # 常量定义
│   ├── middleware.py        # JWT 安全中间件
│   ├── models.py            # Pydantic 请求/响应模型
│   ├── routes/
│   │   ├── document_routes.py   # 文档上传、查询、删除接口
│   │   └── pgvector_routes.py   # pgvector 调试路由（DEBUG 模式下启用）
│   ├── services/
│   │   ├── database.py          # asyncpg 连接池、pgvector 索引初始化
│   │   ├── mongo_client.py      # Atlas MongoDB 客户端
│   │   └── vector_store/        # 向量存储工厂及各 provider 实现
│   └── utils/               # 工具函数
├── tests/                   # pytest 测试套件
├── requirements.txt         # 完整依赖（含 sentence_transformers）
├── requirements.lite.txt    # 精简依赖（不含 HuggingFace 本地模型）
├── Dockerfile               # 完整镜像
├── Dockerfile.lite          # 精简镜像
├── docker-compose.yaml      # API + pgvector 一体化启动
├── api-compose.yaml         # 仅启动 RAG API
└── db-compose.yaml          # 仅启动 pgvector 数据库
```

---

## 开发环境搭建

### 本地运行

```bash
# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 复制并配置环境变量
cp .env.example .env   # 按需修改

# 启动（需要先启动 pgvector 数据库）
uvicorn main:app --reload
```

### Docker 运行

```bash
# API + pgvector 一体启动
docker compose up

# 仅启动数据库
docker compose -f db-compose.yaml up

# 仅启动 API（需已有数据库）
docker compose -f api-compose.yaml up

# 无缓存重建镜像
docker compose build --no-cache
```

---

## 关键环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `RAG_OPENAI_API_KEY` | — | OpenAI 嵌入 API Key（优先于 `OPENAI_API_KEY`）|
| `VECTOR_DB_TYPE` | `pgvector` | 向量数据库类型，可选 `atlas-mongo` |
| `POSTGRES_DB` | `mydatabase` | PostgreSQL 数据库名 |
| `POSTGRES_USER` | `myuser` | PostgreSQL 用户名 |
| `POSTGRES_PASSWORD` | `mypassword` | PostgreSQL 密码 |
| `DB_HOST` | `db` | 数据库主机 |
| `DB_PORT` | `5432` | 数据库端口 |
| `JWT_SECRET` | — | JWT 验证密钥，不设则无需认证 |
| `EMBEDDINGS_PROVIDER` | `openai` | 嵌入提供商：`openai`/`azure`/`huggingface`/`ollama`/`bedrock`/`google_genai`/`vertexai` |
| `EMBEDDINGS_MODEL` | provider 默认值 | 嵌入模型名称 |
| `CHUNK_SIZE` | `1500` | 文本分块大小 |
| `CHUNK_OVERLAP` | `100` | 分块重叠字符数 |
| `EMBEDDING_BATCH_SIZE` | `0` | 批量嵌入每批大小，`0` 禁用批处理 |
| `RAG_UPLOAD_DIR` | `./uploads/` | 上传文件存储目录 |
| `RAG_HOST` | `0.0.0.0` | API 监听地址 |
| `RAG_PORT` | `8000` | API 监听端口 |
| `DEBUG_RAG_API` | `False` | 开启详细日志及 pgvector 调试路由 |
| `COLLECTION_NAME` | `testcollection` | 向量集合名称 |
| `PGVECTOR_CREATE_EXTENSION` | `True` | 是否自动创建 pgvector 扩展 |

完整环境变量列表见 [README.md](README.md)。

---

## 测试

框架：**pytest + pytest-asyncio**

```bash
# 运行所有单元测试（排除需要 Docker 的集成测试）
pytest

# 运行特定测试文件
pytest tests/test_models.py

# 运行集成测试（需要 Docker 环境）
pytest -m integration

# 生成覆盖率报告
pytest --cov=app
```

- 集成测试标记为 `@pytest.mark.integration`，默认不运行
- `tests/conftest.py` 中通过 monkey-patch 绕过真实数据库连接，单元测试无需 DB

---

## 代码规范

- 格式化工具：**Black**（通过 pre-commit 钩子自动触发）
- 启用 pre-commit：`pre-commit install`
- 手动格式化：`black .`
- Python 版本：3.10+
- 全面使用类型注解，禁止裸 `Any`
- 异步优先（`async/await`），所有 I/O 操作使用异步接口

---

## API 接口概览

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/documents` | 上传并向量化文档 |
| `GET` | `/documents` | 按 `file_id` 检索相关文档块 |
| `DELETE` | `/documents` | 按 `file_id` 删除文档向量 |
| `GET` | `/health` | 健康检查 |

调试路由（`DEBUG_RAG_API=True` 时启用）通过 `/pgvector/*` 暴露原始 pgvector 操作。
