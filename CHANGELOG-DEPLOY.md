# 部署与变更摘要（轻量说明）

日期：2025-10-07
目的：在保证功能不变的前提下，提升稳定性、减少瞬时资源消耗，并增加可恢复的缓存/刷新机制。

主要改动（已提交）
- 后端并发限制
  - 将并发抓取限制从 8 调整为 6（src/app/actions.ts: CONCURRENCY = 6）。
  - 说明：减少瞬时网络连接与 CPU 峰值，降低 ECONNRESET 风险。

- 缓存与持久化
  - 内存缓存 `cachedArticles` 优先返回，持久化到 `data/rss-cache.json`（服务启动时异步加载）。
  - CACHE_TTL 默认 6 小时；每日强刷记录 `lastDailyUpdate`。

- 抓取与更新策略
  - fetchArticles 支持 `force` 参数；默认情况下页面请求不会阻塞抓取（若缓存可用立即返回）。
  - triggerBackgroundUpdate 实现后台静默更新并防止并发运行。
  - 新增 API：
    - POST `/api/refresh` — 触发后台静默更新（已创建： `src/app/api/refresh/route.ts`）。
    - GET `/api/articles?force=true` — 强制获取最新文章并返回（已创建： `src/app/api/articles/route.ts`）。

- 前端体验（news page）
  - 首屏优先返回缓存，sentinel（IntersectionObserver）在用户下拉时触发静默后台更新（非阻塞）。
  - 切源 / 推荐：优先使用内存缓存；若无缓存则发起带 5s 超时的强制请求，超时则降级并后台静默刷新。
  - 添加“刷新”按钮（后台触发）与右上角订阅图标（点击打开模态订阅）。

- 日志与类型
  - 统一使用 `devLog` / `devError`（从 `src/app/actions.ts` 导出）控制 server-side 日志（由 ENV 控制）。
  - 已修复若干 TypeScript 报错（toast variant 等），并运行 `tsc --noEmit` 检查通过。

- 自动化
  - 添加 GitHub Actions 模板 `.github/workflows/refresh.yml`（每日 06:00 Asia/Shanghai 调用 `/api/refresh`）。需要在仓库 Secrets 中添加 `SITE_URL`（你的部署地址）以启用。

验证步骤（在本地或服务器）
1. 重启开发服务：
   - npm run dev
2. 验证首屏：访问 http://localhost:9002 ，若存在 `data/rss-cache.json` 则首屏应立即显示内容（秒开）。
3. 手动测试：
   - 静默刷新（下拉触发）：向下滚动到列表底部，观察终端是否打印后台刷新日志（不会阻塞页面）。
   - 强制获取：curl -X GET "http://localhost:9002/api/articles?force=true"
   - 手动触发后台更新：curl -X POST "http://localhost:9002/api/refresh"
4. 检查持久化文件：data/rss-cache.json 在首次持久化后会生成并更新。

回滚与注意事项
- 若需回退并发改动：在 `src/app/actions.ts` 恢复 `CONCURRENCY` 值为 8 并重启服务。
- 若部署在 serverless 环境（Vercel 等）且文件写入权限受限：
  - data/ 持久化可能失败，请改为仅内存缓存并依赖 GitHub Actions 定时调用 `/api/refresh`。
- 安全性：`/api/refresh` 当前无认证；建议在公开部署时增加简单秘钥校验或仅由 CI 调用。

建议的后续优化（可选）
- 增加简单失败退避（记录 failCount，跳过高失败率源）。
- 将持久化改为轻量文件或外部存储（如 Redis）以适应不同部署。
- 添加简单监控脚本收集抓取失败率与缓存命中率（日志或文件）。

若需要，我可以：
- 把 README 中加入本文件内容（或合并到 README.md）；
- 帮你在仓库 Secrets 中写入 `SITE_URL` 使用说明文档；
- 添加 failCount 退避逻辑或监控脚本（分别需要你的确认）。