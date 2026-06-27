---
name: video-platform-builder
description: |
  快速构建类 B 站视频平台（Go Gin 后端 + Vue 3 前端）。从零开始，按模块递增式交付——
  用户认证 → 视频上传转码 → 弹幕实时 → 评论系统 → 社交体系 → 运营后台 23 模块。
  包含完整的架构决策记录(ADR)、Go/Vue 代码模板、基础设施配置、NFR 覆盖矩阵。
  触发词：「视频平台」「视频网站」「弹幕网站」「类B站」「视频社区」「搭建视频站」「造一个视频平台」
version: 1.0.0
agent_created: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - TaskCreate
  - TaskUpdate
  - TaskList
  - WebSearch
  - WebFetch
---

# 视频平台快速构建技能（Video Platform Builder）

**角色：** 视频平台架构师。**阶段：** 架构设计 + 模块化实现。

**功能：** 从零构建一个完整的类 B 站视频平台——Go (Gin) 后端 + Vue 3 (Vite) 前端，按 10 个阶段的模块递增式交付。每一步产出可运行的代码，而非纯文档。

## 为什么这个技能存在

视频平台是互联网最复杂的应用类型之一，涉及：
- **实时通信**（WebSocket 弹幕/聊天/直播）
- **媒体处理**（FFmpeg 转码、多码率、封面提取）
- **异步任务**（RabbitMQ 消费转码）
- **全文搜索**（Elasticsearch）
- **审核工作流**（视频/专栏/评论审核）
- **运营后台**（23 个模块：BI、风控、工单、版权、运维...）
- **社交体系**（关注/拉黑/收藏/投币/私信/动态）
- **权限体系**（RBAC resource:action 细粒度，双 JWT 体系）

如果没有统一的架构约束，并行开发会出现 API 风格不一致、数据模型冲突、命名混乱等灾难性后果。本技能将 cakecake-project (mini-bili) 的生产验证架构提炼为可复用的模板。

## 技术栈（锁定决策）

| 层 | 选型 | 替代方案 | 选择理由 |
|----|------|---------|---------|
| **后端框架** | Go + Gin | Echo / Fiber | 生态最大，中间件丰富，中文社区活跃 |
| **ORM** | GORM v2 | sqlx / ent | AutoMigrate 自动建表，适合快速迭代 |
| **数据库** | MySQL 8 | PostgreSQL | GORM MySQL 支持最成熟 |
| **缓存** | Redis 7 | - | 弹幕冷却、播放计数、热点数据 |
| **消息队列** | RabbitMQ | Kafka / Redis List | 转码任务天然适合工作队列模式 |
| **搜索引擎** | ES 8 | Meilisearch | 中文分词成熟，聚合查询强大 |
| **对象存储** | 阿里云 OSS | MinIO / S3 | 国内访问快，SDK 简洁 |
| **前端** | Vue 3 + Vite | React / Nuxt | 国内生态最成熟，Element Plus 开箱即用 |
| **UI 库** | Element Plus | Ant Design Vue | Admin 表格/表单组件丰富 |
| **WebSocket** | gorilla/websocket | nhooyr.io/websocket | 社区首选，久经考验 |
| **日志** | zap | logrus | 零内存分配，高性能 |
| **视频处理** | FFmpeg CLI | GStreamer | 通用性最强，命令行即可 |

## 架构模式

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Vue 3 SPA   │────▶│  Gin HTTP    │────▶│   Service    │
│  (前端用户)  │     │  (API 网关)  │     │   (业务层)   │
└──────────────┘     └──────┬───────┘     └──────┬───────┘
                            │                    │
              ┌─────────────┼────────────────────┼─────────────┐
              │             │                    │             │
              ▼             ▼                    ▼             ▼
        ┌─────────┐  ┌──────────┐        ┌──────────┐  ┌──────────┐
        │  MySQL  │  │  Redis   │        │ RabbitMQ │  │   OSS    │
        │  (主库) │  │ (热数据) │        │ (转码队列)│  │ (视频文件)│
        └─────────┘  └──────────┘        └────┬─────┘  └──────────┘
                                              │
                                        ┌─────▼─────┐
                                        │  FFmpeg   │
                                        │  (转码)   │
                                        └───────────┘
```

模块化单体 —— 按功能拆分 handler/service/model，通过 DI 注入依赖。为未来微服务拆分预留接口边界。

**目录结构：**
```
project/
├── cmd/project/main.go          # 入口：初始化所有依赖，启动 HTTP
├── internal/
│   ├── config/                  # 配置加载（.env → struct）
│   ├── data/                    # DB/Redis 连接 + AutoMigrate + Seed
│   ├── model/                   # GORM 模型（全部实体）
│   ├── handler/                 # HTTP handler（一文件一模块）
│   │   ├── deps.go              # Dependencies 容器
│   │   └── router.go            # 路由注册（所有端点）
│   ├── service/                 # 业务逻辑层（handler → service → gorm）
│   ├── middleware/               # JWT 鉴权 / RBAC / 链路追踪 / CORS
│   ├── errcode/                 # 统一错误码定义
│   ├── logger/                  # zap 日志初始化
│   ├── pkg/                     # 通用工具包（jwt / sensitive / validator）
│   ├── queue/                   # RabbitMQ 生产者/消费者
│   ├── search/                  # ES 客户端 + 索引管理
│   ├── storage/                 # 文件存储抽象（OSS/Local）
│   ├── worker/                  # 后台任务（转码消费/SLA/ASR）
│   └── ws/                      # WebSocket Hub（弹幕/聊天/直播）
├── frontend/                    # Vue 3 + Vite 前端
│   └── src/
│       ├── pages/               # 页面组件
│       │   ├── home/            # 首页
│       │   ├── video/           # 播放页
│       │   ├── upload/          # 上传页
│       │   ├── search/          # 搜索页
│       │   └── admin/           # 运营后台
│       │       ├── AdminLayout.vue
│       │       ├── data/        # 数据概览 / BI
│       │       ├── review/      # 审核
│       │       ├── social/      # 社交管理
│       │       ├── ops/         # 运维
│       │       └── content/     # 内容
│       ├── components/admin/    # 共享管理组件
│       │   ├── AdminDataTable.vue
│       │   └── AdminFormDialog.vue
│       ├── api/admin/           # 管理端 API 模块（按功能分文件）
│       ├── router/              # 路由配置
│       └── utils/               # 工具函数
├── configs/                     # 配置文件
├── deploy/                      # systemd / nginx / docker-compose
└── scripts/                     # 运维脚本
```

## 构建阶段（严格按序推进）

### 阶段 1：项目脚手架 —— 地基

**目标：** 可编译运行的空壳项目。

1. 初始化 Go module + Vue 项目
2. 配置 `.env` 文件（数据库/Redis/RabbitMQ/OSS/JWT 等）
3. 实现 `config.Load()` 从环境变量读取配置
4. 实现 `data.NewDB()` + `data.NewRedis()` 连接初始化
5. 实现 `logger.Init()` zap 日志初始化
6. 实现统一错误码 `internal/errcode/`
7. 实现 `{code, msg, data}` JSON 响应信封
8. Gin 引擎启动，`/api/v1/health` 端点

**验证：** `go build` 通过，`/api/v1/health` 返回 `{"code":0,"msg":"ok","data":null}`

### 阶段 2：用户认证 —— 安全基石

**目标：** 用户注册/登录/个人信息/JWT 双 Token。

1. `model.User` 模型（username, password_hash, avatar, 等）
2. 注册接口：唯一性校验 + bcrypt 密码哈希
3. 登录接口：返回 Access Token (2h) + Refresh Token (7d)
4. JWT 中间件：提取 user_id，校验签名和过期
5. 个人信息：GET/PUT `/users/me`
6. 密码修改（需验证原密码）
7. 头像上传（格式校验 + OSS/本地存储）

**关键决策：** Access Token 用于业务 API，Refresh Token 仅用于刷新端点。刷新时旧 Refresh Token 标记失效（Redis 黑名单防重放）。

### 阶段 3：视频上传与转码 —— 核心闭环

**目标：** 用户上传视频 → 异步转码 → H.264 MP4 发布。

1. `model.Video` 模型（5 种状态：processing/published/failed/pending_review/rejected）
2. multipart 上传接口（校验格式/大小 ≤500MB，时长 ≤30min 用 ffprobe）
3. 上传后写入 RabbitMQ 转码队列
4. `worker.StartTranscodeConsumer`：消费 → FFmpeg 转码 → 上传 OSS → 更新状态
5. 转码失败重试 3 次（指数退避 30s/60s/90s），超过标记 failed
6. 播放量计数：Redis INCR → 10s 定时落库
7. 视频列表：published 状态，按播放量降序排列
8. 视频详情：单视频 GET（含封面/播放量/弹幕数/评论数）

**关键决策：** 
- 转码目标：H.264 编码 + AAC 音频 + MP4 容器
- 封面：默认提取视频第 1 帧 / 用户可自定义上传
- 文件存储路径：`videos/{video_id}.mp4`

### 阶段 4：弹幕系统 —— 实时体验

**目标：** WebSocket 弹幕实时收发 + 历史回放。

1. `model.Danmaku` 模型（content, color, video_time, font_size）
2. WebSocket Hub 架构（rooms 按 video_id 分组）
3. 发送弹幕：5s 冷却（Redis SETNX）+ 敏感词过滤 + 颜色校验
4. 实时广播：同一视频的所有 WebSocket 连接实时收到
5. 历史弹幕：按 video_time 分段返回，每段最多 200 条
6. 前端 Canvas 多轨道弹幕渲染
7. UP 主弹幕管理：删除单条弹幕

**关键决策：**
- WebSocket 鉴权：连接 URL 携带 `?token=access_token`，服务端握手时校验
- 弹幕颜色格式：`^#[0-9A-Fa-f]{6}$`，默认 `#FFFFFF`
- 敏感词库：外部文件加载，词库为空时拒绝所有弹幕

### 阶段 5：评论系统 —— 社区互动

**目标：** 3 级嵌套评论 + 点赞通知聚合。

1. `model.Comment` 模型（video_id, parent_id, level, content, like_count, pinned, approved）
2. 发表评论：一级/二级/三级（最大深度 3）
3. 评论列表：按时间倒序，置顶评论排最前
4. 点赞 toggle：Redis 去重 + 通知聚合
5. 通知聚合：同一条评论多人点赞合并为一条通知（"A、B、C 等X人赞了你的评论"）
6. UP 主权限：置顶/精选/关闭评论区/级联删除
7. 级联删除：父评论 + 所有递归子评论，事务保护

**关键决策：**
- 评论最大深度 3 级（一级评论 + 二级回复 + 三级回复）
- 软删除仅用于审计保留，前端不可见

### 阶段 6：管理后台基础 —— 审核与运营

**目标：** 管理员独立 JWT + RBAC + 核心审核功能。

1. `model.AdminUser` 模型 + 独立 Admin JWT 体系
2. Admin 登录/刷新/个人信息
3. RBAC：19 种权限码（resource:action 格式，如 `video:approve`）
4. 用户管理：列表/详情/封禁/解封/删除
5. 视频审核：pending_review → approve/reject
6. 评论管理：跨表联合查询 + 删除
7. 数据仪表盘：总用户/总视频/今日播放/在线人数等统计卡片
8. 系统设置：基本配置保存（同步更新 .env + 内存 Cfg）
9. 举报处理：用户举报列表 → 管理员处理
10. AdminLayout + AdminDataTable 共享组件

**关键决策：**
- Admin JWT 与 User JWT 完全隔离，使用不同的 secret 或在 payload 中区分 type
- 所有写操作自动记录 AuditLog（操作人/目标/时间/IP）
- 前端 API 调用统一通过 `@/api/admin` barrel 文件

### 阶段 7：运营后台 23 模块全补全

**目标：** 生产级运营工具链。

| 模块 | 核心功能 | 后端文件 |
|------|---------|---------|
| 首页轮播 | CRUD + 图片上传 | `admin_banner.go` |
| 热搜运营 | Redis 热词 + 人工干预置顶/降权 | `admin_hot_search.go` |
| AI 角色 | Agent 配置 + LLM 模型选择 | `admin_agent.go` |
| LLM 配置 | API Key / BaseURL / Model 管理 | `admin_llm_config.go` |
| 专栏审核 | 文章审核通过/驳回 | `admin_article.go` |
| 动态管理 | 用户动态列表/删除 | `admin_dynamic.go` |
| 工单系统 | 提交→分配→回复→关闭→申诉 | `admin_ticket.go` |
| 风控管理 | 风险规则 + 黑白名单 + 命中日志 | `admin_risk.go` |
| 版权管理 | 版权投诉→受理→驳回→下架→恢复 | `admin_copyright.go` |
| BI 报表 | 概览/文章统计/互动统计/分区/创作者 | `admin_bi.go` |
| 客服后台 | 会话分配+模板回复+自动关闭 | `admin_cs.go` |
| 运维监控 | 任务队列/告警/链路追踪/健康检查/CDN/存储 | `admin_ops.go` |
| 配置发布 | Feature Flag 灰度 + 版本发布 + 快照回滚 | `admin_config.go` |
| 权限审计 | 角色/权限/管理员/审批流/审计日志 | `admin_rbac.go` |
| 字幕管理 | 多语言字幕上传/ASR 自动生成 | `subtitle.go` |
| 专题活动 | 专题页面 + 营销活动 | `admin_special.go` |

### 阶段 8：直播系统

**目标：** SRS 推流 + flv.js 播放 + WebSocket 聊天礼物。

1. 创建直播间（标题/封面/分类）
2. SRS 推流回调（on_publish/on_done）自动更新直播状态
3. flv.js 播放（HTTP-FLV 低延迟）
4. WebSocket 实时聊天 + 礼物系统
5. 观众计数追踪
6. 直播历史记录
7. 管理后台：警告/封禁直播间

### 阶段 9：社交体系

**目标：** 完整的社区交互。

1. 关注/取关 + 关注分组
2. 拉黑（双向互阻）
3. 收藏夹（多收藏夹 + 默认收藏夹）
4. 投币（每日限额 + 账本）
5. 图文动态发布
6. 私信 WebSocket 实时推送

### 阶段 10：搜索与发现

**目标：** ES 全文搜索 + 热搜 + 推荐 Feed。

1. ES 索引设计（video/user 索引）
2. 全文搜索 API（标题/描述/标签）
3. 搜索建议（自动补全）
4. 搜索历史（用户维度）
5. 热搜榜（Redis 热词 + 管理干预）
6. Feed 推荐（规则/热度排序，预留 ML 接口）
7. 排行榜（播放量/弹幕数/点赞数）

## NFR 覆盖矩阵（视频平台专项）

| 编号 | NFR | 实现 | 相关 ADR |
|------|-----|------|----------|
| NF-1 | 弹幕 100 人并发 ≤200ms | WebSocket Hub + Redis Pub/Sub 广播 | ADR-004 |
| NF-2 | 存储分级 | MySQL(主) + Redis(热) + RabbitMQ(异步) + OSS(文件) | ADR-001 |
| NF-3 | 双鉴权体系 | User JWT + Admin JWT 隔离，RBAC 细粒度 | ADR-003 |
| NF-4 | API 统一规范 | RESTful + {code, msg, data} 信封 | ADR-002 |
| NF-5 | 前端模块化 | Vue 3 SPA + AdminLayout + 分组子目录 | ADR-005 |
| NF-6 | 配置管理 | .env + Feature Flag FNV-1a hash 灰度 | ADR-007 |
| NF-7 | 全链路审计 | 所有写操作自动 AuditLog | ADR-003 |
| NF-8 | 转码容错 | 3 次重试 + 永久性错误不重试 | ADR-006 |
| NF-9 | 敏感词安全 | 外部词库 + 空词库拒绝所有 | ADR-002 |
| NF-10 | 日志规范 | zap 结构化 + Info/Error 分级 | ADR-001 |

## ADR 决策记录（项目启动时必须锁定）

### ADR-001: 模块化单体 + Go Gin + MySQL + Redis + RabbitMQ + OSS

**决策：** 使用 Go Gin 模块化单体架构，MySQL 主库，Redis 热数据缓存，RabbitMQ 异步任务，OSS 文件存储。

**Context:** 视频平台需要处理 HTTP API + WebSocket + 媒体转码 + 全文搜索。中等规模（< 100 万 DAU），团队 1-3 人。

**Consequences:** 
- 易：单体调试简单，Go 编译快
- 难：后续拆微服务需要改内部调用链
- 缓解：handler → service → gorm 三层分离，service 通过接口暴露

### ADR-002: RESTful API + JSON 信封格式

**决策：** 所有 API 使用 `{code: int, msg: string, data: any}` 格式，HTTP 200 返回，业务错误通过 code 区分。

**错误码分段：** 40001-40099(参数) / 40100-40199(认证) / 40300-40399(权限) / 40400-40499(不存在) / 50000-50099(服务器)。

**决策理由：** 简化客户端错误处理，避免 HTTP 状态码与业务状态混淆。

### ADR-003: JWT 双体系 + RBAC 资源级权限

**决策：** 用户 JWT (2h Access + 7d Refresh) 和 管理员 JWT (2h Access + 24h Refresh) 完全隔离。管理员权限使用 `resource:action` 格式（如 `video:approve`, `user:ban`）。

### ADR-004: WebSocket Hub 广播模式

**决策：** 弹幕/直播聊天使用单机 Hub + Redis Pub/Sub 跨进程广播。gorilla/websocket 作为底层库。

### ADR-005: Vue 3 + Element Plus + 按分组子目录管理后台

**决策：** 前端使用 Vue 3 + Vite + Element Plus。管理后台页面按功能分组（data/review/social/ops/content），共享 AdminDataTable 和 AdminFormDialog 组件。

### ADR-006: FFmpeg 转码 + 3 次重试 + 永久性错误识别

**决策：** 目标格式 H.264 + AAC + MP4。重试 3 次（30s/60s/90s 退避）。永久性错误（源文件损坏/编码不支持）立即标记 failed。

### ADR-007: Feature Flag FNV-1a 灰度

**决策：** 配置发布使用 FNV-1a hash 分桶 + whitelist + rollout_pct 实现灰度。版本发布流程：draft → deployed → rolled_back。

## 代码模板

使用本技能时，从以下模板快速生成代码文件。

### Go Handler 模板

```go
package handler

import (
    "net/http"
    "strconv"
    "github.com/gin-gonic/gin"
)

// CreateXxx handles POST /xxx
func (a *API) CreateXxx(c *gin.Context) {
    // 1. 参数绑定
    var req struct {
        Title string `json:"title" binding:"required,max=80"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusOK, gin.H{"code": 40001, "msg": "参数错误", "data": nil})
        return
    }
    // 2. 业务逻辑
    // ...
    // 3. 返回结果
    c.JSON(http.StatusOK, gin.H{"code": 0, "msg": "ok", "data": result})
}

// ListXxx handles GET /xxx
func (a *API) ListXxx(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    size, _ := strconv.Atoi(c.DefaultQuery("size", "20"))
    // 调用 service 层
    // ...
}
```

### GORM 模型模板

```go
package model

import "time"

type Xxx struct {
    ID        uint64    `gorm:"primaryKey"`
    UserID    uint64    `gorm:"index;not null"`
    Status    string    `gorm:"size:32;index;not null;default:active"`
    CreatedAt time.Time `gorm:"index"`
    UpdatedAt time.Time
}
```

### Vue Admin 页面模板

```vue
<template>
  <AdminDataTable :data="items" :loading="loading" :page="page" :total="total"
    @update:page="page = $event; fetch()">
    <template #search-bar>
      <el-input v-model="keyword" placeholder="搜索..." @input="fetch" />
    </template>
    <el-table-column prop="id" label="ID" width="80" />
    <el-table-column prop="title" label="标题" />
    <el-table-column label="操作" width="180">
      <template #default="{ row }">
        <el-button size="small" @click="handleAction(row)">操作</el-button>
      </template>
    </el-table-column>
  </AdminDataTable>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import AdminDataTable from '@/components/admin/AdminDataTable.vue'
import { listXxx } from '@/api/admin/xxx'

const items = ref([])
const loading = ref(false)
const page = ref(1)
const total = ref(0)
const keyword = ref('')

async function fetch() {
  loading.value = true
  try {
    const res = await listXxx({ page: page.value, keyword: keyword.value })
    items.value = res.data.list
    total.value = res.data.total
  } finally {
    loading.value = false
  }
}

onMounted(fetch)
</script>
```

## 基础设施 Docker Compose

```yaml
version: "3.8"
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: video_platform
    ports: ["3306:3306"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports: ["9200:9200"]
```

## 验证清单

每个阶段完成后执行：

1. `go build -o ./bin/app ./cmd/` 通过
2. 新增 API 端点可 curl 验证
3. 前端页面可正常渲染
4. 数据库表通过 GORM AutoMigrate 自动创建
5. 错误场景返回正确的错误码和消息

---

> 基于 cakecake-project (mini-bili v2.0) 生产验证的架构提炼。所有模式均经过 23 模块运营后台 + 直播 + 社交体系的实战检验。
