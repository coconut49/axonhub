# Fork Patch 清单

`main` 上位于上游 base tag 之后的每个 commit 在此登记一个条目。新增、修改、删除 patch 时同步更新本文件。

## fork-infra — fork 基础设施

- 用途:fork 的自我描述与自动化。CLAUDE.md 追加行、`.fork/`、`fork-sync.yml`、`fork-image.yml`、`test.yml` 加 `main` 触发。
- 退役条件:无,fork 存续期间永久保留。

## opus-5 — 支持 claude-opus-5

- 用途:网关支持 Anthropic 2026-07-24 发布的 `claude-opus-5`。上游未实现,本 patch 为自研。
- 涉及:`llm/transformer/anthropic/claudecode/constants.go`(DefaultModels 加 `claude-opus-5`,claude-code OAuth 渠道取不到模型列表时的静态回退);`scripts/sync/models.json` 加 opus-5 条目并经 `node scripts/sync/sync-model-developers.js` 重新生成 `frontend/src/features/models/data/providers.json`(生成物不手改);`frontend/src/features/models/data/providers.ts` 的 `DEVELOPERS_URL` 从上游 unstable raw URL 改指本 fork main——前端「添加模型」的候选列表是浏览器运行时从该 URL 现拉的,打包数据只是回退,不改 URL 则 fork 的模型登记永远不生效。transformer 层 adaptive thinking / effort 按平台开关、不绑定模型 ID,无需改动;渠道模型价格在 DB 由管理界面配置,不属于代码。
- 退役条件:上游 `DefaultModels()` 含 `claude-opus-5` 且 providers.json 数据源(ThinkInAIXYZ/PublicProviderConf)收录后,drop 本 commit。若上游实现方式与本 patch 相同,rebase 时 git 会因 patch-id 相同自动丢弃。
