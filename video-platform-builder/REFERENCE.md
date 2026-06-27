# 视频平台构建 — 参考手册

本文件提供 SKILL.md 的详细补充材料。构建时按需查阅。

---

## 1. 完整数据模型清单（82 张表）

### 核心业务表（v1.0 必建）

| 表名 | 说明 | 关键字段 | 索引 |
|------|------|---------|------|
| `users` | 用户 | username(unique), password_hash, avatar_url, status(banned/active) | CakeID, created_at, status |
| `videos` | 视频 | user_id, title, status(5种), video_url, cover_url, play_count | user_id, status, play_count, created_at |
| `danmakus` | 弹幕 | video_id, user_id, content, color, video_time | video_id, user_id |
| `comments` | 评论 | video_id, user_id, parent_id, level, content, pinned, approved | video_id, user_id, parent_id |
| `comment_likes` | 评论点赞 | user_id+comment_id(unique) | user_id, comment_id |
| `notifications` | 通知 | recipient_id, type, related_id, is_read | recipient_id, type |
| `favorite_folders` | 收藏夹 | user_id, title, is_public, is_default | user_id |
| `video_favorites` | 视频收藏 | user_id+video_id+folder_id(unique) | 联合唯一 |
| `user_follows` | 关注 | follower_id+followee_id(unique) | follower_id, followee_id |
| `user_blocks` | 拉黑 | blocker_id+blocked_id(unique) | 联合唯一 |
| `video_likes` | 视频点赞 | user_id+video_id(unique) | 联合唯一 |
| `video_coins` | 投币 | user_id+video_id(unique), amount(1|2) | 联合唯一 |
| `watch_laters` | 稍后再看 | user_id+video_id(unique), watched | 联合唯一 |

### 管理后台表

| 表名 | 说明 | 模块 |
|------|------|------|
| `admin_users` | 管理员 | RBAC |
| `admin_roles` | 角色 | RBAC |
| `role_permissions` | 角色-权限关联 | RBAC |
| `permissions` | 权限定义（resource:action） | RBAC |
| `audit_logs` | 操作审计日志 | RBAC |
| `approval_flows` | 审批流 | RBAC |
| `home_banners` | 首页轮播图 | Banner |
| `hot_search_ops` | 热搜运营记录 | HotSearch |
| `reports` | 用户举报 | Report |
| `report_handling_logs` | 举报处理记录 | Report |
| `tickets` | 工单 | Ticket |
| `ticket_messages` | 工单消息 | Ticket |
| `risk_rules` | 风控规则 | Risk |
| `risk_bw_list` | 风控黑白名单 | Risk |
| `risk_hit_logs` | 风控命中日志 | Risk |
| `copyright_complaints` | 版权投诉 | Copyright |
| `cs_conversations` | 客服会话 | CS |
| `cs_templates` | 客服模板回复 | CS |
| `feature_flags` | 功能开关 | Config |
| `release_records` | 版本发布记录 | Config |
| `task_logs` | 任务日志 | Ops |
| `alert_rules` | 告警规则 | Ops |
| `alert_records` | 告警记录 | Ops |
| `trace_records` | 链路追踪 | Ops |
| `system_health_checks` | 健康检查 | Ops |
| `cdn_refresh_tasks` | CDN 刷新任务 | Ops |
| `oss_lifecycle_rules` | OSS 生命周期规则 | Ops |

### 扩展功能表

| 表名 | 说明 | 模块 |
|------|------|------|
| `articles` | 专栏文章 | Article |
| `article_comments` | 文章评论 | Article |
| `user_dynamics` | 用户动态 | Dynamic |
| `dynamic_comments` | 动态评论 | Dynamic |
| `video_chapters` | 视频章节 | Player |
| `video_bitrates` | 多码率 | Player |
| `subtitles` | 字幕 | Subtitle |
| `live_rooms` | 直播间 | Live |
| `live_chat_messages` | 直播聊天 | Live |
| `live_warn_templates` | 直播警告模板 | Live |
| `comment_images` | 评论图片 | Comment Enhance |
| `danmaku_likes` | 弹幕点赞 | Danmaku |
| `scheduled_publishes` | 定时发布 | Creator Center |
| `follow_groups` | 关注分组 | Social |
| `follow_group_members` | 分组关联 | Social |
| `coin_ledgers` | 硬币流水 | Coin |
| `user_daily_rewards` | 每日奖励 | User |
| `user_daily_tasks` | 每日任务 | User |
| `view_histories` | 观看历史 | User |
| `search_histories` | 搜索历史 | Search |
| `special_pages` | 专题页面 | Special |
| `campaigns` | 营销活动 | Special |
| `agent_profiles` | AI 角色配置 | Agent |
| `agent_settings` | AI 全局设置 | Agent |
| `llm_configs` | LLM 配置 | LLM |
| `dm_conversations` | 私信会话 | DM |
| `dm_messages` | 私信消息 | DM |
| `video_drafts` | 视频草稿 | Video |

---

## 2. RBAC 权限码完整清单（19 种）

| 权限码 | 中文名 | 控制模块 |
|--------|--------|---------|
| `user:ban` | 用户封禁/解封 | 用户管理 |
| `video:approve` | 视频审核 | 视频审核 |
| `video:delete` | 视频删除 | 视频管理 |
| `comment:delete` | 评论删除 | 评论管理 |
| `ticket:handle` | 工单处理 | 工单+举报 |
| `risk:manage` | 风控管理 | 风控 |
| `copyright:handle` | 版权处理 | 版权 |
| `cs:manage` | 客服管理 | 客服 |
| `ops:manage` | 运维管理 | 运维 |
| `config:manage` | 配置管理 | 配置发布 |
| `rbac:manage` | 权限管理 | RBAC审计 |
| `banner:manage` | 轮播管理 | 首页轮播 |
| `hotsearch:manage` | 热搜运营 | 热搜 |
| `agent:manage` | AI角色管理 | AI角色 |
| `setting:manage` | 系统设置 | 设置+LLM |
| `article:approve` | 专栏审核 | 专栏 |
| `dynamic:manage` | 动态管理 | 动态 |
| `live:manage` | 直播管理 | 直播 |
| `special:manage` | 专题活动 | 专题 |
| `subtitle:manage` | 字幕管理 | 字幕 |
| `dashboard:export` | 数据导出 | BI |

---

## 3. 视频状态机

```
upload
  │
  ▼
processing ──(FFmpeg 成功)──▶ pending_review ──(管理员通过)──▶ published
  │                                │                              │
  │ (永久性错误)                   │ (驳回)                       │ (UP主删除)
  ▼                                ▼                              ▼
failed                          rejected                      (软删除或物理删除)
```

状态说明：
- `processing`：转码中，仅上传者可见
- `pending_review`：待审核（需管理员审核）
- `published`：已发布，全站可见
- `rejected`：审核驳回，仅上传者可见，含驳回原因
- `failed`：转码失败，仅上传者可见，含错误详情

---

## 4. 环境变量完整清单

```bash
# 数据库
MYSQL_DSN=root:password@tcp(127.0.0.1:3306)/video_platform?charset=utf8mb4&parseTime=True

# Redis
REDIS_ADDR=127.0.0.1:6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT
JWT_SECRET=your-256-bit-secret-here

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@127.0.0.1:5672/

# 对象存储（阿里云 OSS）
OSS_ENDPOINT=oss-cn-hangzhou.aliyuncs.com
OSS_ACCESS_KEY_ID=your-key
OSS_ACCESS_KEY_SECRET=your-secret
OSS_BUCKET=your-bucket
OSS_PUBLIC_URL_PREFIX=https://your-bucket.oss-cn-hangzhou.aliyuncs.com

# Elasticsearch
ELASTICSEARCH_URL=http://127.0.0.1:9200

# 视频转码
FFPROBE_PATH=/usr/bin/ffprobe  # 或留空从 PATH 查找
FFMPEG_PATH=/usr/bin/ffmpeg    # 或留空从 PATH 查找
TEMP_UPLOAD_DIR=./data/temp

# 敏感词
SENSITIVE_WORDS_FILE=./configs/sensitive_words.txt

# LLM (DeepSeek)
DEEPSEEK_API_KEY=sk-xxx
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-chat

# IP 定位
IP2REGION_V4_XDB=./data/ip2region.xdb

# 应用
APP_ENV=development  # development | production
HTTP_ADDR=:8080

# NocoBase 集成（可选）
NOCOBASE_API_KEY=
```

---

## 5. GORM AutoMigrate 注册模板

```go
// internal/data/db.go
func NewDB(dsn string, log *zap.Logger) (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Warn),
    })
    if err != nil {
        return nil, err
    }
    
    // AutoMigrate all models
    err = db.AutoMigrate(
        &model.User{},
        &model.Video{},
        &model.Danmaku{},
        &model.Comment{},
        &model.CommentLike{},
        &model.Notification{},
        &model.VideoLike{},
        &model.FavoriteFolder{},
        &model.VideoFavorite{},
        &model.VideoCoin{},
        &model.WatchLater{},
        &model.UserFollow{},
        &model.UserBlock{},
        &model.Article{},
        // ... 追加所有其他模型
    )
    return db, err
}
```

---

## 6. WebSocket 实时架构

```
Client A ──WebSocket──▶ ws.Hub (room: video_id)
                           │
                      Redis Pub/Sub
                           │
Client B ◀──WebSocket── ws.Hub (room: video_id)
```

Hub 核心结构：
```go
type Hub struct {
    rooms      map[uint64]map[*Client]bool  // video_id → clients
    broadcast  chan *Message
    register   chan *Client
    unregister chan *Client
}

type Client struct {
    hub    *Hub
    conn   *websocket.Conn
    send   chan []byte
    userID uint64
    roomID uint64
}
```

---

## 7. 前端路由分组

```javascript
// 用户端路由
const routes = [
  { path: '/',              component: () => import('@/pages/home/Index.vue') },
  { path: '/video/:id',    component: () => import('@/pages/video/Play.vue') },
  { path: '/upload',       component: () => import('@/pages/upload/Index.vue'), meta: { auth: true } },
  { path: '/search',       component: () => import('@/pages/search/Index.vue') },
  { path: '/ranking',      component: () => import('@/pages/ranking/Index.vue') },
  { path: '/space/:id',    component: () => import('@/pages/home/Space.vue') },
  { path: '/not-found',    component: () => import('@/pages/notFound/Index.vue') },
]

// 管理端路由（使用 AdminLayout 布局）
const adminRoutes = [
  { path: '/admin/login',  component: () => import('@/pages/admin/AdminLogin.vue') },
  { path: '/admin',        component: () => import('@/pages/admin/AdminLayout.vue'), meta: { admin: true }, children: [
    { path: '', redirect: '/admin/dashboard' },
    // data 分组
    { path: 'dashboard',          component: () => import('@/pages/admin/data/Dashboard.vue') },
    { path: 'bi-report',          component: () => import('@/pages/admin/data/BIReport.vue') },
    // review 分组
    { path: 'video-review',       component: () => import('@/pages/admin/review/VideoReview.vue') },
    { path: 'article-review',     component: () => import('@/pages/admin/review/ArticleReview.vue') },
    { path: 'live-manage',        component: () => import('@/pages/admin/review/LiveManage.vue') },
    // content 分组
    { path: 'banner-manage',      component: () => import('@/pages/admin/content/BannerManage.vue') },
    { path: 'hot-search',         component: () => import('@/pages/admin/content/HotSearchManage.vue') },
    { path: 'subtitle-manage',    component: () => import('@/pages/admin/content/SubtitleManage.vue') },
    { path: 'special-manage',     component: () => import('@/pages/admin/content/SpecialManage.vue') },
    // social 分组
    { path: 'comment-manage',     component: () => import('@/pages/admin/social/CommentManage.vue') },
    { path: 'report-manage',      component: () => import('@/pages/admin/social/ReportManage.vue') },
    { path: 'ticket-manage',      component: () => import('@/pages/admin/social/TicketManage.vue') },
    { path: 'risk-manage',        component: () => import('@/pages/admin/social/RiskManage.vue') },
    { path: 'copyright-manage',   component: () => import('@/pages/admin/social/CopyrightManage.vue') },
    { path: 'cs-manage',          component: () => import('@/pages/admin/social/CSManage.vue') },
    // ops 分组
    { path: 'ops-monitor',        component: () => import('@/pages/admin/ops/OpsMonitor.vue') },
    { path: 'config-manage',      component: () => import('@/pages/admin/ops/ConfigManage.vue') },
    { path: 'rbac-manage',        component: () => import('@/pages/admin/ops/RBACManage.vue') },
    // 其他
    { path: 'user-manage',        component: () => import('@/pages/admin/UserManage.vue') },
    { path: 'agent-manage',       component: () => import('@/pages/admin/AgentManage.vue') },
    { path: 'settings',           component: () => import('@/pages/admin/Settings.vue') },
  ]}
]
```

---

## 8. 常见问题与决策陷阱

### Q: 用单体还是微服务？
**A:** 从模块化单体开始。视频平台前期流量不确定，单体足够支撑到 10 万 DAU。关键是用接口隔离 handler/service/dao 三层，为未来拆分做准备。不要在 Day 1 引入 gRPC 或服务网格。

### Q: 视频存储用 OSS 还是自建？
**A:** 用 OSS。自建对象存储需要运维 CDN/备份/扩容，成本远高于 OSS 按量付费。本地文件系统仅用于开发环境。

### Q: 弹幕存储选 MySQL 还是 MongoDB？
**A:** MySQL。弹幕量虽大但不是非结构化数据，GORM + MySQL 的索引查询对按 video_time 分段取弹幕完全够用。单视频弹幕 > 10 万条时考虑分表或归档。

### Q: 转码要支持哪些码率？
**A:** 至少 360p/720p/1080p 三档。video_bitrates 表允许后期扩展。不要一开始做 4K/8K —— 存储和转码成本过高。

### Q: 需要 CDN 吗？
**A:** OSS 自带 CDN 能力（阿里云 CDN 回源 OSS）。前期直接用 OSS 公网 URL，流量上来了再配置 CDN 域名。只需在 `OSS_PUBLIC_URL_PREFIX` 环境变量中切换即可。

---

## 9. 技能使用工作流

```
用户说："帮我搭建一个视频平台"
    │
    ▼
1. 确认技术栈偏好（有无特殊要求？）
    │
    ▼
2. 按阶段执行（从阶段 1 脚手架开始）
    │
    ├── 每完成一个阶段：go build 验证 + curl 验证端点
    │
    ▼
3. 逐阶段推进，不跳步
    │
    ▼
4. 全部 10 阶段完成后：运行完整功能测试
```

每个阶段内的执行模式：
```
TaskCreate → 拆分子任务 → 逐个实现 → 编译验证 → TaskUpdate 标记完成 → 进入下一阶段
```

**禁止行为：**
- 严禁跳过脚手架阶段直接写业务逻辑
- 严禁在模型未建表时就写 handler
- 严禁 ID 用自增 string 或 UUID 混用（统一 uint64 自增）
- 严禁在多个 handler 中复制粘贴相同逻辑（抽取到 service 层）
