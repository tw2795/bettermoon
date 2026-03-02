# 自定义修改记录

本文件记录为适配 Vercel 部署所做的修改。

## 1. vercel.json — Cron Job 配置

**文件**: `vercel.json`

**修改内容**: 保持原有 cron schedule `0 1 * * *`（每天凌晨 1 点），未做频率变更

**说明**: 原项目在 Docker 部署时通过 `start.js` 中的 `setInterval` 每小时调用 `/api/cron`，执行配置订阅刷新、直播源刷新、播放记录/收藏更新等任务。Vercel Hobby 套餐仅支持每天一次的 cron，无法对齐每小时频率（需 Pro 套餐）。但 cron 的作用是从外部源拉取新数据写入数据库（数据新鲜度），与跨实例内存缓存同步（由第 4 条的 30 秒 TTL 机制保证）是独立的，降低频率不影响多实例数据一致性。

## 2. vercel.json — functions maxDuration 配置

**文件**: `vercel.json`

**修改内容**: 新增 `functions` 配置，为两个路由设置 `maxDuration`：
- `src/app/api/search/ws/route.ts`: 30 秒
- `src/app/api/cron/route.ts`: 60 秒

**原因**:
- **搜索路由**: SSE 流式搜索需要并发请求多个资源站并逐个返回结果，默认的 10 秒超时可能不够。设置 30 秒给予足够的搜索时间。
- **Cron 路由**: 定时任务涉及刷新配置、直播源、以及遍历所有用户的播放记录和收藏，数据量大时耗时较长，设置 60 秒避免中途超时。

**注意**: `maxDuration` 超过 10 秒需要 Pro 套餐。Hobby 套餐下这些值会被限制为 10 秒。

## 3. SSE 搜索单源超时按部署环境自适应

**文件**: `src/app/api/search/ws/route.ts`

**修改内容**: 新增模块级常量 `SEARCH_TIMEOUT_MS`，根据 `SERVER_TYPE` 环境变量动态选择超时值：
- `SERVER_TYPE=serverless` 时：8 秒
- 其他（Docker 等传统部署）：保持原始 20 秒

**原因**: Serverless 平台（如 Vercel）的 Function 有执行时长限制（Hobby 10 秒，Pro 60 秒），原始 20 秒单源超时在该环境下过长。8 秒是 Serverless 场景下的合理平衡点：大多数正常的资源站 API 应在 8 秒内响应，超过此时间的源大概率已不可用。而 Docker 等传统部署无此限制，保持 20 秒以兼容响应较慢的资源站。

## 4. Serverless 配置跨实例同步（`SERVER_TYPE=serverless`）

**文件**: `src/lib/config.ts`、`vercel.json`

**修改内容**:

1. `vercel.json` 新增环境变量 `SERVER_TYPE=serverless`
2. `config.ts` 新增模块级变量 `cachedConfigTime`、`isServerless`、`CONFIG_CACHE_TTL_MS`（30 秒）
3. `getConfig()` 增加 TTL 检查：当 `SERVER_TYPE=serverless` 时，若内存缓存超过 30 秒则从数据库重新加载
4. `resetConfig()` 和 `setCachedConfig()` 同步更新 `cachedConfigTime` 时间戳

**原因**: Vercel 等 Serverless 环境下，每个请求可能由不同的 Lambda 实例处理，模块级变量 `cachedConfig` 不在实例间共享。原始实现中 `getConfig()` 一旦缓存就永不过期，导致：

- 实例 A 上管理员修改了配置（如添加视频源），`cachedConfig` 更新并写入数据库
- 实例 B 的 `cachedConfig` 仍是旧值，且永远不会重新读取数据库
- 用户请求被路由到实例 B 时看到的是过期配置

参考 [grok2api](https://github.com/chenyme/grok2api) 项目的 `reload_if_stale(interval=30)` 方案，为 `getConfig()` 增加 30 秒 TTL 机制。缓存过期时从数据库重新加载；加载失败**或数据库返回空**时，继续使用旧缓存并重置计时器，避免频繁重试，同时防止空配置回写覆盖数据库。非 Serverless 环境（Docker 等单实例部署）行为不变，`cachedConfig` 仍然永久有效。

**注意**: `SERVER_TYPE=serverless` 需要在部署平台的环境变量中手动设置（Vercel 已在 `vercel.json` 中预设）。其他 Serverless 平台（如 Netlify、Cloudflare 等）如需此行为，也应设置该环境变量。
