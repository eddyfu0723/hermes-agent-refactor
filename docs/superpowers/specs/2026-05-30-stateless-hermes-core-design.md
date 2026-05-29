# 无状态 Hermes 核心改造设计

> 子系统 #1 — 存储适配器模式
> 日期：2026-05-30
> 状态：已确认

## Context

基于 Hermes (v0.15.2) 开源项目构建银行信用卡中心 AI 岗位助手。银行有 10,000+ 员工、多种岗位类型，不可能每人一个 Hermes 实例。需要将 Hermes 改造为无状态计算节点：所有数据外化到外部存储（MySQL/Redis/S3/ES），实例仅承担计算职责，任何请求可落到任意实例。

## 需求约束

| 维度 | 决定 |
|---|---|
| LLM | 本地开源模型（Qwen/DeepSeek/ChatGLM），Ollama/vLLM 部署，OpenAI 兼容协议 |
| 存储中间件 | MySQL + Redis + S3(MinIO) + Elasticsearch |
| 部署 | 本地开发（SQLite/文件） → K8s 生产（外部中间件） |
| 认证 | 第一阶段暂不涉及 |
| 并发规模 | 200-1000 同时在线 |
| 实时能力 | SSE（流式文本）+ WebSocket（双向实时），HTTP 预留 |
| Provider | 仅保留 OpenAI 兼容协议（覆盖 Ollama、vLLM 等所有本地部署） |

## 架构设计

### 整体架构：三层分离

```
┌─────────────────────────────────────────────────┐
│  接入层 (API Server — FastAPI)                    │
│  SSE / WebSocket / HTTP(预留)                     │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  计算层 (无状态 Hermes 实例 × N)                  │
│  AIAgent Core · Tool Registry · LLM Provider     │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  适配器层 (Abstract Interfaces)                   │
│  SessionStore · MemoryStore · ConfigStore         │
│  SkillStore · CronStore · FileStore · SearchStore │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────┬───────────┼───────────┬──────────────┐
│  MySQL   │  Redis    │  S3/MinIO │Elasticsearch │
└──────────┴───────────┴───────────┴──────────────┘
```

### 设计原则

1. **接口只暴露业务语义** — SessionStore.create_session() 不暴露 SQL/Redis 命令
2. **工厂模式切换实现** — `storage.adapter: "local" | "external"` 控制使用哪套实现
3. **渐进式迁移** — 优先迁移 SessionStore，其余逐个跟进
4. **所有接口 async** — 保持接口统一，LocalAdapter 内部同步操作包装为 async

## 存储关注点映射

### ① SessionStore — 会话与消息

**数据特征**：高频读写、需持久化

**接口定义**：
```python
class SessionStore(ABC):
    # 会话生命周期
    async def create_session(self, user_id: str, model: str, source: str) -> str
    async def end_session(self, session_id: str) -> None
    async def get_session(self, session_id: str) -> Optional[Session]
    async def list_sessions(self, user_id: str, limit: int, offset: int) -> List[Session]
    # 消息读写
    async def add_message(self, session_id: str, role: str, content: str, metadata: dict) -> None
    async def get_messages(self, session_id: str, limit: int) -> List[Message]
    async def get_recent_context(self, session_id: str, max_tokens: int) -> List[Message]
    # 压缩与标题
    async def set_session_title(self, session_id: str, title: str) -> None
    async def compress_session(self, old_id: str, new_id: str, summary: str) -> None
```

**双实现**：
- LocalAdapter: SQLite (封装当前 `hermes_state.py` 的 SessionDB)
- ExternalAdapter: MySQL 写入 + Redis 缓存最近 N 条消息 + 压缩时双写

### ② MemoryStore — Agent 记忆

**数据特征**：低频写、每次会话读一次

**接口定义**：
```python
class MemoryStore(ABC):
    async def read_memory(self, user_id: str, memory_type: str) -> str  # "memory" | "user"
    async def write_memory(self, user_id: str, memory_type: str, content: str) -> None
    async def list_sections(self, user_id: str, memory_type: str) -> List[str]
    async def replace_section(self, user_id: str, memory_type: str, section: str, content: str) -> None
```

**双实现**：
- LocalAdapter: `~/.hermes/memories/MEMORY.md` / `USER.md` 文件
- ExternalAdapter: S3 对象存储 + Redis 缓存（会话开始时读一次，写入后更新缓存）

### ③ ConfigStore — 用户配置

**数据特征**：极低频写、启动时读

**接口定义**：
```python
class ConfigStore(ABC):
    async def load_config(self, user_id: str) -> dict
    async def save_config(self, user_id: str, config: dict) -> None
    async def get_value(self, user_id: str, key_path: str) -> Any  # dot-notation: "model.name"
    async def set_value(self, user_id: str, key_path: str, value: Any) -> None
    async def load_env(self, user_id: str) -> dict  # 敏感配置（API keys）
```

**双实现**：
- LocalAdapter: `~/.hermes/config.yaml` + `.env`
- ExternalAdapter: MySQL user_configs 表（YAML JSON 化存储）+ 敏感字段加密

### ④ SkillStore — 技能文档

**数据特征**：低频写、需全文检索

**接口定义**：
```python
class SkillStore(ABC):
    async def list_skills(self, user_id: str) -> List[SkillMeta]
    async def read_skill(self, user_id: str, skill_id: str) -> str
    async def write_skill(self, user_id: str, skill_id: str, content: str) -> None
    async def delete_skill(self, user_id: str, skill_id: str) -> None
    async def search_skills(self, user_id: str, query: str) -> List[SkillMeta]
```

**双实现**：
- LocalAdapter: `~/.hermes/skills/` 目录扫描
- ExternalAdapter: S3 存文档 + ES 索引元数据 + ik_max_word 中文分词

### ⑤ CronStore — 定时任务

**数据特征**：低频写、需互斥执行

**接口定义**：
```python
class CronStore(ABC):
    async def load_jobs(self, user_id: str) -> List[CronJob]
    async def save_job(self, user_id: str, job: CronJob) -> None
    async def delete_job(self, user_id: str, job_id: str) -> None
    async def acquire_lock(self, job_id: str, ttl: int) -> bool
    async def release_lock(self, job_id: str) -> None
```

**双实现**：
- LocalAdapter: YAML + file lock
- ExternalAdapter: MySQL + Redis 分布式锁

### ⑥ FileStore — 通用文件

**数据特征**：二进制文件、附件

**接口定义**：
```python
class FileStore(ABC):
    async def read(self, path: str) -> bytes
    async def write(self, path: str, data: bytes) -> None
    async def delete(self, path: str) -> None
    async def list(self, prefix: str) -> List[str]
    async def get_presigned_url(self, path: str, expires: int) -> str  # ExternalAdapter: S3 presigned URL; LocalAdapter: file:// 本地路径
```

**双实现**：
- LocalAdapter: 本地文件系统（get_presigned_url 返回 `file://` 本地路径）
- ExternalAdapter: S3 (MinIO)（get_presigned_url 返回 S3 presigned URL）

### ⑦ SearchStore — 全文检索

**数据特征**：读多写少、需中文分词

**接口定义**：
```python
class SearchStore(ABC):
    async def search(self, query: str, filters: dict) -> SearchResult
    async def index(self, doc_id: str, content: str, metadata: dict) -> None
    async def remove(self, doc_id: str) -> None
```

**双实现**：
- LocalAdapter: SQLite FTS5 + trigram（当前实现）
- ExternalAdapter: Elasticsearch + ik 分词

## API 接入层设计

### 协议与场景

| 协议 | 场景 | 方向 | 端点 |
|---|---|---|---|
| SSE | 超级工作台 — 流式对话 | 单向推送 Server→Client | `POST /api/v1/chat/stream` |
| WebSocket | 智能副驾 — 实时坐席辅助 | 双向实时 | `WS /api/v1/chat/ws` |
| HTTP (预留) | SDK 调用、简单问答 | 请求-响应 | `POST /api/v1/chat` |

### SSE 数据流

```
Client → POST /api/v1/chat/stream
  Body: {user_id, session_id, message, tools: [...]}
  → API Server 从 Redis/MySQL 恢复 session 上下文
  → AIAgent.run_conversation()
    → LLM 调用
  ← SSE events:
    event: token     data: {"delta": "...", "type": "text"}
    event: tool_call data: {"name": "...", "args": {...}, "status": "running"}
    event: tool_result data: {"name": "...", "result": "...", "status": "done"}
    event: done      data: {"session_id": "...", "tokens_used": N}
  → 异步持久化 session → MySQL + 更新 Redis 缓存
```

### WebSocket 消息类型

**Client → Server**：
- `user_message` — 用户/客户输入
- `asr_result` — ASR 转写结果（来自银行独立 ASR 服务）
- `interrupt` — 中断当前生成

**Server → Client**：
- `token` — 流式文本片段
- `tool_call` — 工具调用开始
- `tool_result` — 工具调用结果
- `suggestion` — 话术建议 / 知识推荐
- `alert` — 合规提醒 / 异常告警
- `done` — 回复完成

### API 路由

```
POST  /api/v1/chat/stream       # SSE 流式对话
WS    /api/v1/chat/ws           # WebSocket 双向实时
POST  /api/v1/chat              # HTTP 请求-响应 (预留)

GET   /api/v1/sessions          # 会话列表
GET   /api/v1/sessions/{id}     # 会话详情
DELETE /api/v1/sessions/{id}    # 删除会话
POST  /api/v1/sessions/search   # 搜索会话 (ES)

GET   /api/v1/skills            # 技能列表
GET   /api/v1/config            # 当前配置
GET   /api/v1/health            # 健康检查
```

## 工具过滤 & Provider 精简

### 过滤策略

配置级屏蔽，不删代码。在 `config.yaml` 中新增 `disabled_tools` 和 `disabled_providers`，支持 glob 通配符。工具注册时检查屏蔽列表，被屏蔽的工具不注册、不暴露给 LLM。Provider 加载时跳过被屏蔽的 provider 目录。

### 保留的工具

- **知识 & 记忆**: memory, skills_list, skill_view, skill_manage, session_search, todo, clarify
- **Agent 协作**: delegate_task, kanban_*, cronjob
- **代码执行**: execute_code, read_file, write_file, patch, search_files, terminal, process
- **MCP 集成**: mcp_tool (对接银行内部系统)
- **数据分析**: vision_analyze (分析报表/截图)
- **消息**: send_message (适配内部 IM)

### 屏蔽的工具

web_search, web_extract, x_search, browser_*(全套), ha_*(智能家居), spotify_*, computer_use, image_generate, video_*, text_to_speech, 以及所有外部通讯平台网关 (Telegram/Discord/Slack/WhatsApp/Signal 等)。

### Provider

**仅保留**: OpenAI 兼容协议 (custom provider)，覆盖 Ollama、vLLM 等所有本地部署场景。

**屏蔽**: 其余 28 个 provider (alibaba, anthropic, azure-foundry, bedrock, copilot*, deepseek, gemini, openrouter 等)。

### 配置示例

```yaml
disabled_tools:
  - web_search
  - web_extract
  - x_search
  - browser_*
  - ha_*
  - spotify_*
  - computer_use
  - image_generate
  - video_*
  - text_to_speech

disabled_providers:
  - alibaba
  - anthropic
  - azure-foundry
  - bedrock
  - copilot*
  - deepseek
  - gemini
  - openrouter
  # ... 其余外部 provider
```

## 开发模式 vs 生产模式

### 模式切换

通过 `storage.adapter` 配置项控制：

| 关注点 | 开发模式 (local) | 生产模式 (external) |
|---|---|---|
| Session | SQLite (~/.hermes/state.db) | MySQL + Redis 缓存 |
| Memory | 文件 (~/.hermes/memories/) | S3 + Redis 缓存 |
| Config | YAML (~/.hermes/config.yaml) | MySQL (加密) |
| Skills | 目录 (~/.hermes/skills/) | S3 + ES 索引 |
| Cron | YAML + 文件锁 | MySQL + Redis 分布式锁 |
| Files | 本地文件系统 | S3 (MinIO) |
| Search | SQLite FTS5 | Elasticsearch |

### 生产模式配置

```yaml
storage:
  adapter: "external"
  mysql:
    host: "${MYSQL_HOST}"
    port: 3306
    database: "hermes"
    pool_size: 20
  redis:
    url: "${REDIS_URL}"
    session_ttl: 3600
  s3:
    endpoint: "${MINIO_ENDPOINT}"
    bucket: "hermes-data"
  elasticsearch:
    hosts: ["${ES_HOST}:9200"]
    index_prefix: "hermes"
```

## 关键改造文件

### 新增文件

| 文件 | 内容 | 优先级 |
|---|---|---|
| `storage/__init__.py` | 7 个抽象接口定义 + 工厂函数 | P0 |
| `storage/local/*.py` | Local 适配器实现 (~7 文件) | P0 |
| `storage/external/*.py` | External 适配器实现 (~7 文件) | P0 |
| `api/server.py` | FastAPI 应用 + SSE/WS/HTTP 端点 | P0 |
| `api/routes/*.py` | chat, sessions, skills, config 路由 | P0 |

### 改造文件

| 文件 | 改造内容 | 优先级 |
|---|---|---|
| `run_agent.py` | AIAgent 构造时注入 Store 实例 | P0 |
| `agent/agent_init.py` | 初始化适配器工厂，注入 Store | P0 |
| `agent/conversation_loop.py` | 流式输出接入 SSE/WS 推送 | P0 |
| `hermes_state.py` | SessionDB → SessionStore 适配器调用 | P0 |
| `tools/memory_tool.py` | 文件读写 → MemoryStore 适配器 | P1 |
| `hermes_cli/config.py` | YAML 读写 → ConfigStore 适配器 | P1 |
| `tools/registry.py` | 注册时检查 disabled_tools | P1 |
| `providers/__init__.py` | 加载时跳过 disabled_providers | P1 |

## 迁移阶段

| 阶段 | 内容 |
|---|---|
| Phase 1 | 适配器接口定义 + SessionStore 双实现 + API Server 骨架 |
| Phase 2 | MemoryStore + ConfigStore + 工具/Provider 过滤 |
| Phase 3 | SkillStore + CronStore + FileStore + SearchStore |
| Phase 4 | K8s 部署配置 + 压力测试 + 生产验证 |

## 后续子系统（本设计不覆盖）

- #2 多租户层（用户隔离、认证、资源配额）
- #3 超级工作台 UI
- #4 智能副驾 API
- #5 集成 SDK
