### 背景介绍
100M DAU | 约 3–10 亿用户 | 1000 人大组 | 4 分钟视频


### 1. 系统架构图

![系统架构图](imags/architecture.png)


### 2. 可扩展性策略

| 挑战 | 解决方案 | 预期效果 |
| --- | --- | --- |
| 100M DAU → 峰值 17 万 QPS | 64 分库 × 2000 QPS/实例 + Redis 热点缓存 | DB 实际负载 < 2 万 QPS |
| 10 亿总用户 | 64×16 分表，单表 ~1000 万行 | 写入延迟 < 10ms |
| 1000 人大组实时更新 | CRDT 客户端合并 + Kafka 分区 + 一致性哈希 | 更新延迟 < 100ms |
| 热门列表雪崩 | Redis 缓存 + TTL 1h + 写穿 | 缓存命中率 > 95% |
| 访问延迟 | CloudFront 边缘缓存 | 起播 < 1s，更新 < 100ms |

#### 容量规划

| 组件 | 规模配置 |
| --- | --- |
| Spring Boot | 60 Pod |
| MySQL | 64 主 + 128 从 |
| WebSocket Server | 60 Pod（5 万连接/个） |
| Kafka | 1000 分区 |

### 3. 技术选型

| 组件 | 选择 | 理由 | 备选与权衡 |
| --- | --- | --- | --- |
| 前端 | Vue.js 3 | 响应式 + Yjs 集成好 | React：包大 |
| 后端 | Spring Boot 3 | 企业级稳定 | Node.js：并发弱 |
| 实时 | Yjs CRDT + WebSocket | 客户端合并，无锁 | OT：服务端复杂 |
| 广播 | Kafka | 按 list_id 分区 | RocketMQ：吞吐量 |
| 数据库 | MySQL 8.0 + 64×16 分片 | 单表性能最优 | TiDB：学习成本 |
| 分片路由 | ShardingSphere | 零侵入 | 手动 SQL：易错 |
| 搜索 | Elasticsearch | 中文分词 + 拼音 | Redis ZSET：不支持分词 |
| 缓存 | Redis Cluster | 权限 + 会话 | Memcached：无集群 |
| 媒体存储 | Amazon S3 | 无限扩展 | MinIO：运维重 |
| 视频转码 | MediaConvert | Serverless | FFmpeg：负担 |

### 4. 数据模型

— 用户表（64×16 = 1024 张）
```sql
CREATE TABLE db_{db_id}.users_{table_id} (
    user_id BIGINT UNSIGNED PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username)
) ENGINE=InnoDB;
```

— TODO 列表表（64 张）
```sql
CREATE TABLE db_{db_id}.todo_lists (
    list_id BIGINT UNSIGNED PRIMARY KEY,
    owner_id BIGINT UNSIGNED NOT NULL,
    title VARCHAR(100) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_owner (owner_id)
) ENGINE=InnoDB;
```

— 权限表（64 张）
```sql
CREATE TABLE db_{db_id}.permissions (
    perm_id BIGINT UNSIGNED PRIMARY KEY,
    list_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    permission_type ENUM('edit', 'view') NOT NULL,
    UNIQUE KEY uk_list_user (list_id, user_id)
) ENGINE=InnoDB;
```

— TODO 事项表（64 张）
```sql
CREATE TABLE db_{db_id}.todo_items (
    item_id BIGINT UNSIGNED PRIMARY KEY,
    list_id BIGINT UNSIGNED NOT NULL,
    content TEXT,  -- 可为空
    medias JSON,   -- [{id, type, key, status, thumbnail_url, play_url}]
    version BIGINT DEFAULT 0,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_list (list_id),
    INDEX idx_updated (list_id, updated_at DESC),
    INDEX idx_media_status ((CAST(medias->>'$.status' AS CHAR)))
) ENGINE=InnoDB;
```

### 5. 重点功能底层细节

#### 5.1 鉴权流程

![鉴权时序图](imags/check_permission.png)


权限校验采用“三层统一”架构：

- 网关层（Spring Cloud Gateway）拦截所有 REST 请求，解析 JWT 获取 `user_id`。
- 统一调用 `user-server.check_permission(user_id, list_id, action)`，集中校验 `edit/view` 权限。
- 权限缓存于 Redis Hash，支持写穿 + 主动失效；在 QPS < 1000 时延迟 < 5ms。
- 业务服务（如 `media-server` 获取签名 URL）仅接收已授权请求，避免重复校验，保证高性能、一致性与可维护性。

#### 5.2 分享以及授权逻辑

![分享列表](imags/share.png)



#### 5.3 视频上传流程
![视频上传流程时序图](imags/uploadvideo.png)


优点
- 直传 S3 → 零服务器压力
- 实时状态 → uploading → done
- serverless 转码 → Lambda + MediaConvert + SNS通知  （AWS云服务）

详细流程：
1. media-server 返回预签名 URL，客户端直传 OSS/S3。
2. 上传完成后通过 WebSocket 广播 uploading 状态。
3. S3 触发 Lambda，调用 MediaConvert 转码为 HLS 并生成缩略图。（AWS 云服务）
4. SNS 通知，data-server 更新 MySQL，并广播 play_url。
5. 上传链路对业务服务器带宽占用为 0。

#### 5.4 视频播放流程

![视频播放流程时序图](imags/lookvideo1.png)

优点
- 不存签名 URL 
- 动态生成 → 每次播放都校验权限
- 防盗链 → IP 绑定 + 1 小时过期
- CDN 加速 

#### 5.5 CRDT 持久化

![CRDT持久化时序图](imags/crdt-update.png)

![CRDT持久化流程图](imags/crdt-update-flow.png)

详细流程：

1. 客户端 Yjs 每 5 秒或 100 次操作，调用 getState() 生成 snapshot → 推送 WebSocket Server；
2. WebSocket Server 转发 + 推 Kafka（list-snapshots 主题，按 list_id 分区）；
3. data-server Worker 消费 Kafka，直接 REPLACE INTO 覆盖 MySQL；
4. 冷启动：客户端从 MySQL 加载最新 snapshot → Yjs 恢复。 

优点
- 零服务端逻辑：不解析 CRDT
- 低内存：无影子文档
- 高可用：Kafka 持久化
- 冷启动快：直接加载最新 snapshot

## 6. 风险点

1. 客户端推送 snapshot 丢失，增加重试
2. Kafka 分区热点
   热点告警，手动/自动扩容，热点列表拆分(新增用户id/时间戳进行分片)
3. Redis 内存爆炸，热点key。
   实时监控，本地缓存，更精准的数据预估

## 7. 总结
没有完美的技术方案，只有最适合当前阶段的平衡艺术。
本方案以 Yjs 客户端 CRDT + 双网关隔离(业务网关，webSocket网关) + 64×16 分片 + serverless 媒体流 + JWT 短效期 + Redis 缓存控制 为核心，
在 100M DAU、10 亿用户、1000 人大组、≤4 分钟视频、实时编辑立即可见 的严苛要求下，
实现了 低延迟（<100ms）、高可用、可扩展、安全合规（防盗链 + 登出即失效） 的设计闭环。



