# 仓库准备检查记录

status: 记录（非独立审查）

## Diff 范围

- 源码改动：无（`lib/`、`skills/`、`scripts/`、`package.json`、安装配置均未改）。
- 本轮改动：仅本任务目录下 `prd.md`、`design.md`、`review.md` 三份文档（未跟踪）。
- Git 元数据：fetch 两个远端、main 快进、新建 `codex/dsh-codex-maintenance`。

## 仓库准备结果

| 项目 | 结果 |
| --- | --- |
| origin（用户派生仓库） | `https://github.com/MrUncleXiang/dsh-trellis.git` |
| upstream（原作者仓库） | `https://github.com/1264459640/dsh-trellis.git` |
| 同步前本地 main | `d6325d8`，领先提交 0、落后 25 |
| 同步方式 | `git merge --ff-only upstream/main`，无冲突 |
| 同步后 main / origin/main / upstream/main / HEAD | 均为 `d30f232` |
| 当前分支 | `codex/dsh-codex-maintenance`，基于 `d30f232` |
| 本地定制提交 | 无；未把定制放进 main |

### 未提交、保留的内容

- `.trellis/tasks/feat-11-10-dsh-codex-maintenance/`（主代理创建的任务文件，必须保留）。
- `.trellis/.runtime/sessions/codex-*.json`（会话指针，不提交）。
- `backups/2026-11-10-maintenance-task-before-progress.json`（本轮保护备份，不提交、不清理）。

## 执行过的检查（全部只读）

- `git remote -v`：确认 origin / upstream 地址。
- `git ls-remote --heads upstream`：确认上游 main 为 `d30f232`。
- `git rev-list --left-right --count main...upstream/main`：`0 25`，本地无独有提交。
- `git merge-base --is-ancestor main upstream/main`：通过，可快进。
- `git status --short --branch`：仅上述未跟踪内容。
- `git log --oneline -5`：HEAD 为 `d30f232 chore(release): bump version to 0.5.1 and update changelog`。

## 发现

| 级别 | 问题 | 文件 | 建议 |
| --- | --- | --- | --- |
| 提示 | 现有升级会整份重写插件管理的技能 | `lib/skills.js:168`、`186` | 实现阶段改为逐文件比较，项目已改的不覆盖 |
| 提示 | 废弃模板用 `unlinkSync` 物理删除 | `lib/skills.js:89` | 实现阶段改为移动到回收目录 |
| 提示 | 会话标识缺失时回退固定名 | `lib/task.js:547` | 写操作要求真实宿主标识，缺失拒绝 |
| 提示 | 全量 steps 替换丢未知字段 | `lib/task.js:48` | 与增量更新共用同一验证门禁 |
| 风险 | 现有全量测试可能包含物理删除清理 | 测试目录 | 未审查前不运行 |

## 验证证据

- 本轮未运行任何测试（按指令）。
- 文档与仓库状态由上述只读命令核对。

## 对独立审查阻断项的文档修正（待复审）

- 新增 `lib/core/index.js` 与 `./core` 导出、文件系统/会话适配边界，并注明当前尚不存在。
- 真批准只来自宿主用户事件（可信事件待验证，不可获取时由用户选手动确认通道）；删除被误用的 `approved` 任务状态写法，`task.json` 仍为 `planning`，批准后转 `in_progress`。
- 锁改为原子 `mkdir` 获取、`rename` 移入回收目录释放；补失效核对、多锁顺序、内容指纹比对。
- 补 `main` 快进与维护分支合并完整命令；回退改为重新部署已验证维护版，不改共享分支历史。
- 维护构建标识与 `.agents/skills/.trellis-version` 准确路径、受控 manifest 已写明。

## 结论

本文件只记录本轮仓库准备与只读自检结果，不代表独立审查结论。上面的修正仍待 `design-review.md` 复审；本代理未修改该审查文件。
