# Feature Design

status: draft   # 仅表示这份设计文档的状态，不是 task.json 状态
execution_lane: standard

## 目标与非目标

### 目标

- 保留现有 Trellis 任务核心与任务数据，DSH 与 Codex 共用同一份。
- 两个宿主各有工具入口和会话提示入口，行为语义一致。
- 维护分支可持续合并上游，定制不被覆盖，升级可回退。

### 非目标

- 本轮不改任何业务代码、源码、技能、模板、`package.json`、安装配置。
- 不宣称 MCP 或自动钩子已接入，也不宣称当前已有公开的任务核心 API。
- 不新增第三方依赖（是否采用 MCP 相关依赖待用户决定）。

## 共用核心：新增稳定导出入口（当前尚不存在）

- 现状：`lib/task.js`、`lib/artifact.js` 是包内内部模块，直接依赖 DSH 的文件系统对象与工具注册库；`package.json:32-43` 没有任务核心的公开子路径。已能在副本通过绝对路径导入任务模块；但没有稳定公开导出，正式接入仍需明确接口。
- 实施阶段新增：`lib/core/index.js`，并在 `package.json` 的 `exports` 增加 `./core` 子路径。
- 稳定接口（宿主无关，输入输出为纯 JSON）：`getState`、`createTask`、`continueBind`、`updateStep`、`updateSteps`、`updateTask`、`writeArtifact`、`archiveTask`、`approveTask`。
- 依赖注入边界：
  - `FileSystemAdapter`：读、写、列目录、stat、mkdir、rename。
  - `SessionAdapter`：读取真实会话标识、读取宿主用户确认事件。
  - `LockAdapter`、`Clock`：可替换，便于测试。
- 错误类型：`EAUTH`、`ESESSION_MISSING`、`ECONFLICT`、`ELOCKED`、`EFMT`。
- 约束：`lib/core/` 不直接依赖 `@deepseek-ai/dsh-fs` 或 `@deepseek-ai/dsh-tools`。
- 数据位置不变：`.trellis/tasks/<slug>/task.json`，会话指针 `.trellis/.runtime/sessions/<session>.json`；不引入第二套 `task.json`。

### 入口 A：DSH（原运行宿主）

- 把 `ctx.fs` 与会话对象封装成上面的适配器，保留原工具名 `trellis_state` / `trellis_task_create` / `trellis_task_update` / `trellis_task_archive` 与产物工具（`lib/index.js:424-982`）。
- 自动提示沿用 `agent/pre-step`（`lib/index.js` 头部说明）。
- 现状：已有实现，本轮不改，接入稳定边界属于实施阶段。

### 入口 B：Codex（当前编程助手）

- MCP（通用工具连接协议）工具与命令行都调用 `lib/core/` 的同一接口。
- 文件系统、会话标识、用户确认事件用 Codex 侧适配器封装。
- 提示入口：Codex 会话启动、用户提交前、工具调用前等钩子读取同一阶段与规则。
- 现状：**未接入、未验收**；需真实宿主验证，模拟输出不算通过。

## 版本、维护构建标识与升级

- `package.json` 的 `version` 保留上游版本，只用于溯源；与上游同版本时不能代表本地维护构建。
- 维护构建标识：`<上游版本>+local.<构建号>`（或独立构建号），在受控 manifest（清单）中记录：上游 commit、维护 commit 或不可变发布标记、构建标识、技能文件指纹、任务格式版本。
- `.agents/skills/.trellis-version` 保存该维护构建标识；升级判断比较构建标识，不只比较上游 `version`。
- 安装源与升级判断以受控 manifest 为准，不以 `package.version` 或旧标记单独判断。

### 同步命令（完整顺序）

```powershell
git fetch upstream
git switch main
git merge --ff-only upstream/main
git switch codex/dsh-codex-maintenance
git merge upstream/main   # 有冲突解决后提交，记录合并提交
```

- `main` 只收上游快进，不接收本地定制；定制只在维护分支。

### 回退步骤

1. 先暂停 DSH 与 Codex 两端的写入。
2. 首选：**不改共享分支历史**，按受控 manifest 重新部署上一个已验证的维护版提交或不可变标记。
3. 必须撤销代码时：合并提交用 `git revert -m 1 <merge-commit>`；普通提交逐个 `git revert`。不用分离头指针（detached HEAD，脱离分支的临时状态）作为发布回退，不强推、不 reset 共享分支。
4. 不把 `revert` 上游基线当回退手段，也不退回不含本项目定制的纯上游版本。
5. 项目数据单独回退：先备份，再复制或用回收目录恢复；历史任务文件不重写，并核对任务格式兼容。

## 必须保留的规则与落点

| 规则 | 现状（证据） | 方案落点 |
| --- | --- | --- |
| 定制技能不被升级覆盖 | 版本变化会整份重写插件管理的技能（`lib/skills.js:168`、`186`） | 按逐文件指纹比较，项目已改的文件不覆盖，只补缺失与上游变更 |
| 旧模板不物理删除 | `unlinkSync` 直接删除废弃模板（`lib/skills.js:89`） | 改为移动到回收目录，失败不中断 |
| 继续只绑定 | `bindTaskPointer` 只写会话指针（`lib/task.js:651`） | 保持；不因继续改写状态或阶段 |
| 批准与普通更新隔离 | `verifiedBy: 'human'` 只是可填写字段（`lib/task.js:217-225`） | 批准只能由宿主用户事件产生，写独立批准记录（方案版本、任务内容指纹、事件标识、时间）；普通更新不能创建或修改。可信宿主批准事件待验证，未预设两端已有此 API；若无法提供，交用户选择手动确认通道，不声称加适配器即解决 |
| 任务状态 | `status` 只允许 `planning` / `in_progress` / `completed`（`lib/task.js:35`） | 设计获批先记录批准，不改 `task.json`；用户批准后才由受控操作置 `in_progress` 与实施阶段（如 feat 的 `impl`） |
| 全量 steps 更新同样验收 | `validateSteps` 只保留已知字段（`lib/task.js:48`），与增量 `applyStepUpdate`（190）不一致 | 全量替换与增量走同一验证门禁，未知字段不得丢失 |
| 真实会话标识缺失拒绝 | 缺失时回退固定名 `dsh-session`（`lib/task.js:547`） | 写操作要求宿主真实会话标识，带平台前缀；缺失直接拒绝 |
| 所有写入走同一接口 | 当前 `writeText` 直接覆盖（`lib/task.js:591`、`915`） | 任务、指针、产物、归档、批准都经 `lib/core` 写入接口 |

### 跨进程锁与版本核对

- 获取锁：对 `.trellis/.runtime/locks/<name>.lock` 做原子 `mkdir`；目录内写持锁记录（进程、会话、租约超时、目标文件、锁前读取的内容指纹）。
- 释放锁：用原子 `rename` 移入 `.trellis/.runtime/locks/recycle/`，**不物理删除**。
- 冲突与超时：有限等待，超时返回 `ELOCKED`；确认失效前必须核对原持有进程或 lease（租约），不盲抢。
- 版本核对：锁前读取目标内容并计算指纹（如 SHA-256），拿到锁后重读再比较；不一致返回 `ECONFLICT`，不覆盖。
- 多锁顺序固定为：任务锁 → 会话锁 → 归档锁；释放按相反顺序 rename。

## 已知边界

- **公开任务核心 API 尚不存在**：`lib/core/` 与 `./core` 导出是实施阶段新增项。
- **真人工批准未实现**：现有 `verifiedBy: 'human'` 只是字段校验，模型可自行填写；真批准必须来自宿主用户确认事件。
- **未接入**：MCP 工具与 Codex 自动钩子都没有真实宿主验收；DSH 端合成指针也不等于已验收。
- **依赖待定**：是否引入 MCP 相关依赖由用户决定，本轮不装、不锁。

## 验证计划

- 本轮只验证文档与仓库准备状态：文档齐全、分支与远端一致、无源码改动。
- 实施阶段的验证命令在用户批准后按 `.trellis/spec/` 质量门补充，并且只跑审查过的测试。

## 风险与回滚

- 风险：上游接口变动导致合并冲突；技能覆盖造成定制丢失；两端并发写入互相覆盖；锁判断错误导致误抢。
- 回滚：按上面「回退步骤」执行；共享分支历史保留，文件不物理删除。

## 人审检查点

- [ ] 共用核心的导出入口与适配边界已确认（并承认当前尚不存在）
- [ ] 批准必须来自宿主用户事件，任务状态转换规则已确认
- [ ] 可信宿主批准事件可获取性已验证；不可获取时已由用户选择手动确认通道（实施前置条件）
- [ ] 维护构建标识与 `.agents/skills/.trellis-version` 路径已确认
- [ ] 同步命令与回退命令可直接执行
- [ ] 已知边界与待确认依赖已列出
- [ ] 用户批准设计后，才由受控操作把 `task.json` 从 `planning` 转为 `in_progress`
