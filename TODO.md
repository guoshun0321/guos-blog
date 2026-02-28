# TODO (Guo's blog)

## 0. 最短上线（今天）

- [ ] 在 GitHub 创建仓库：`guoshun0321/guos-blog`
- [ ] 添加 remote 并 push：
  - `git remote add origin git@github.com:guoshun0321/guos-blog.git`
  - `git push -u origin main`
- [ ] Vercel 导入该仓库并部署
- [ ] 验证：
  - `/` 自动跳转到 `/zh`
  - `/zh/blog/rendering-test` Mermaid 与 LaTeX 渲染正常

## 1. 站点能力（本周）

- [ ] 用 content collections 做中文文章列表（/zh 首页）
- [ ] 建英文占位文章策略：中文发布后自动生成 `en` stub + translation 链接
- [ ] RSS：拆分成 `/zh/rss.xml` 与 `/en/rss.xml`（当前模板是单 RSS）
- [ ] 评论：接入 Giscus
- [ ] 搜索：接入 Pagefind 或 Algolia（可选）

## 2. 写作路线图（前 5 篇）

- [ ] 01：一个可上线的 LLM 应用最小架构（Mermaid 架构图）
- [ ] 02：Prompt 回归测试怎么做（样例集 + 评分准则）
- [ ] 03：RAG 失败的 5 种原因 + 指标定位
- [ ] 04：工具调用的安全设计（权限/沙箱/审批）
- [ ] 05：Agent 可观测性（事件/trace/runId/重放）
