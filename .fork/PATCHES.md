# Fork Patch 清单

`main` 上位于上游 base tag 之后的每个 commit 在此登记一个条目。新增、修改、删除 patch 时同步更新本文件。

## fork-infra — fork 基础设施

- 用途:fork 的自我描述与自动化。CLAUDE.md 追加行、`.fork/`、`fork-sync.yml`、`fork-image.yml`、`test.yml` 加 `main` 触发。
- 退役条件:无,fork 存续期间永久保留。

## opus-5 — 支持 claude-opus-5

- 用途:claude-code OAuth 渠道取不到模型列表时,静态回退列表包含 `claude-opus-5`。上游未实现,本 patch 为自研。
- 涉及:`llm/transformer/anthropic/claudecode/constants.go` 的 `DefaultModels()` 加一行。模型元数据侧(providers.json 及其数据源)上游自 v1.0.0-beta6 起已收录,无需 fork 改动;transformer 层 adaptive thinking / effort 按平台开关、不绑定模型 ID,亦无需改动;渠道模型价格在 DB 由管理界面配置,不属于代码。
- 退役条件:上游 `DefaultModels()` 含 `claude-opus-5` 后 drop 本 commit。若上游实现方式与本 patch 相同,rebase 时 git 会因 patch-id 相同自动丢弃。
