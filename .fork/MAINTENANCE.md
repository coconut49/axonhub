# Fork 维护手册

## 分支模型

- `upstream` remote → `looplj/axonhub`(只读,开发分支为 `unstable`);`origin` → `coconut49/axonhub`。
- `main` 永远等于「某个上游 release tag + fork patch 栈」。base tag 不单独记录,随时可推导(最高的、是 main 祖先的非 fork tag):

  ```sh
  git -c versionsort.suffix=-alpha -c versionsort.suffix=-beta \
    tag --list 'v*' --sort=-v:refname --merged main | grep -v -- '-fork.' | head -1
  ```

- fork delta 即 `git log <base>..main`,每个 commit 对应 `.fork/PATCHES.md` 中一个条目。
- 每次成功同步或发布后打 annotated tag `v<上游版本>-fork.<N>`(N 从 1 递增,同一上游版本上 fork 自身迭代时 +1)。tag 不可变,是历史追溯与镜像版本的锚点。
- 上游 tag 版本序:`-alpha` < `-beta` < 正式版,比较时必须带上面的 versionsort 配置,否则 `v1.0.0-beta5` 会排在 `v1.0.0` 之后。

## 自动同步(.github/workflows/fork-sync.yml)

每日轮询上游 release tag(上游发版不频繁,无需更密),落后时执行:

1. `git rebase --onto <新tag> <旧base> main`,启用 `rerere`(rr-cache 通过 actions/cache 持久化,同型冲突只需人/agent 解一次)。
2. 冲突时由 Claude agent 按 `.fork/prompts/resolve-rebase.md` 解决并完成 rebase。
3. 跑与上游 CI 相同的测试:`make test-backend-all`(根模块 + `llm/` 两个 Go module)。
4. 通过后 `git push --force-with-lease origin main`,打 `v*-fork.N` tag 并推送,触发镜像构建。
5. 任一步失败:中止的 rebase 直接 abort;已完成 rebase 但测试失败的结果推到 `sync-failed/*` 分支;开 issue 告知,等本地处理。

处理 `sync-failed/*` 分支时:本地 checkout 该分支,修复后按「本地开发流程」推回 main,并删除该分支与关联 issue。

## 本地开发流程(新定制 / 修 patch)

1. 直接在 `main` 上开发(或临时分支上开发后 rebase 回 `main`)。
2. 定制完成后整理成语义完整的 commit(fixup 用 `git rebase -i` squash 掉),更新 `.fork/PATCHES.md`。
3. 跑测试:`make test-backend-all`。改了前端还要 `cd frontend && pnpm build` 验证可构建。
4. `git push --force-with-lease origin main`。
5. 需要出镜像时打 tag 并推送:

   ```sh
   base=$(git -c versionsort.suffix=-alpha -c versionsort.suffix=-beta \
     tag --list 'v*' --sort=-v:refname --merged main | grep -v -- '-fork.' | head -1)
   # N = 该 base 已有 -fork.N 的最大值 + 1
   git tag -a "${base}-fork.<N>" -m "<一句话说明本次变更>"
   git push origin "${base}-fork.<N>"
   ```

## 撤销一个 patch(上游已支持等价功能时)

1. 在 `.fork/PATCHES.md` 找到该 patch 及其 commit。
2. `git rebase -i <base>` 删除该 commit(冲突时对照上游实现取舍)。
3. 更新 `.fork/PATCHES.md`,跑测试,force-push。

## 冲突处理原则

- rebase 冲突的正确解 = 保留上游的新实现 + 保持 fork patch 的语义意图,二者兼得;对照 `.fork/PATCHES.md` 中该 patch 的「用途」判断意图。
- 某个 patch 与上游新代码在功能上重复时,倾向按「撤销一个 patch」处理,采用上游实现。
- 解完后用 `git range-diff <旧base>..<旧main> <新base>..main` 审查每个 patch 的变化是否符合预期。

## 镜像

- `ghcr.io/coconut49/axonhub:<tag>` 与 `:latest`,linux/amd64 + linux/arm64。
- 由 `.github/workflows/fork-image.yml` 在 `v*-fork.*` tag push 时构建;沿用上游机制,把 tag 写入 `internal/build/VERSION` 后 docker build(amd64/arm64 各自原生 runner 构建,再合 manifest)。

## 环境事实(新环境接手时核对)

- GitHub fork 上已禁用继承自上游的 `release.yml`、`docker-publish.yml`、`helm-chart.yml`(会被 `v*` tag 误触发,且 docker-publish 推的是上游 Docker Hub)以及三个定时任务 `docker-unstable.yml`、`stale.yml`、`sync-model-developers.yml`。`test.yml` 保持启用并已加入 `main` 分支触发,作为 push 后的第二道测试关。
- Repo secret `CLAUDE_CODE_OAUTH_TOKEN`:Claude 订阅的长期 OAuth token(`claude setup-token` 生成),供 fork-sync 的冲突解决 agent 使用。
- 本地 clone:`git config rerere.enabled true`、`git config pull.rebase true`;`main` 追踪 `origin/main`(`git branch --set-upstream-to=origin/main main`)。
- fork-sync 会 force-push `main`,本地同步只用 `git pull --rebase`(本地未推送的 commit 会被重放到新 main 上);编辑器分支指示器的 ahead/behind 以 origin/main 为参照。
- 仓库是双 Go module(根 + `llm/`),`llm/` 内的 Go 命令必须在 `llm/` 目录下跑;`make test-backend-all` 已覆盖两者。
