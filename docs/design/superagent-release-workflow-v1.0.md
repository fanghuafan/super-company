# Superagent 研发交付流水线设计 v1.0

> 状态：设计稿 v1.0 ｜ 2026-11-16 ｜ 需求来源：钉钉「superagent 工作流程」九步草案
> 范围：需求表 → 版本 → 分支 → 十步开发 → 集成测试 → 合并 → TOP 测试发布 → 验证验收 → BUG 回环
> 落地对象：`运营IM平台/message-nexus`（主）、`message-nexus-dingtalk-adapter`（按需）
> 参考实现：`功能开发区/dsh-superagent-prototype`（release lane，P0/P1 已提交，46 项测试通过）

---

## 0. 一页结论

### 0.1 五个判断

1. **九步主干成立，但不能照抄顺序。** 真实仓库的发布规约是「提测分支 → TOP 测试 → 验收 → 合 main → 生产」，而草案写的是「先合 develop → 再发测试」。两者顺序相反（见 3.5-D2）。
2. **`develop` 不存在。** `message-nexus` 只有 `main`，且仓库明确规定所有新开发只走 `main → TOP 测试 → TOP 生产`；凭空建 `develop` 会直接违反仓库治理（见 3.5-D1）。
3. **三个必须由人放行的闸门不能省**：G0 开工确认、G1 设计签字、G3 验收；G2 是 TOP 平台审批（樊芳华 + 运维），agent 只能提交审核，不能自批。
4. **agent 的终点是「待发布」**：生产发布、生产审批、生产回滚仍由人执行。这与 11-12 已确认的边界一致，也满足「循环到上线」的业务语义——agent 负责把版本推到「可上线」。
5. **不是从零开始**：原型已有表解析、就绪门、G0 确认、CAS 加锁、版本取号、建分支、状态机的骨架；缺的是 PingCode 适配器、十步接线、合并、TOP 发布、验证与 BUG 回环。

### 0.2 一张图

```
  钉钉需求表 02-需求池（ALIDOC，17 列，44 条）
        │  S0 轮询 5min / 解析 / S1 就绪门
        ▼
  候选清单 ──G0 @孙俊林(10min)──► S3 锁定：状态=开发中，开发人员=Superagent
        │                                   │
        │                                   ▼
        │                        S4 PingCode 建版本【v4.1217.xxx】
        │                                   │
        │                                   ▼
        │                        S5 分支 feature/v4.1217.xxx + worktree
        │                                   │
        │                                   ▼
        │                        S6 十步 ①-⑥（②③ 需 G1 签字）
        │                                   │
        │                                   ▼
        │                        S7 ⑦-⑨ + 本地集成测试 / 修复（≤3 轮）
        │                                   │
        │                                   ▼
        │                        S8 推送分支 + 建 PR（draft）
        │                                   │
        │                                   ▼
        │                        S9 TOP 测试环境部署提测分支
        │                           └──G2 审批：樊芳华 + 运维（人）
        │                                   │
        │                                   ▼
        │                        S10 验证：SHA/库表/日志/页面/证据包
        │                                   │
        │                                   ▼
        │                        S11 状态=待验收 ──G3 孙俊林──► 通过
        │                                   │
        │                                   ▼
        │                        S12 合并 main（PR + CI）→ 状态=待发布
        │                                   │
        │                                   ▼
        │                        人：TOP 生产发布 / 观察 / 写「已上线」
        │
        └── 生产 BUG（群）→ S13 新 BUGFIX 版本 → fix/v4.1217.xxx → 回到 S6
```

### 0.3 需要人拍板的 6 件事

| # | 决策 | 建议 | 不决策的后果 |
| --- | --- | --- | --- |
| D1 | 合并目标：`main` 还是新建 `develop` | **`main`**（仓库硬规约，且无 `develop`） | 违反 `AGENTS.md`/`.ai/RELEASE.md`，PR 无法按规约进入生产 |
| D2 | 测试与合并顺序 | **先提测分支 → 验收 → 合 main** | 未验收代码进 `main`，生产基线被污染 |
| D3 | 需求状态字典 | 新增「待验收/待发布」，`开发中` 由 agent 写 | 状态回写落到不存在的枚举，看板统计失真 |
| D4 | 版本双轨：表内「版本」列写什么 | **PingCode 号 v4.1217.xxx**，产品版本进备注 | 分支、PingCode、表三处版本对不上，追溯断链 |
| D5 | 生产边界与 TOP 审批 | **agent 到「待发布」；审批由樊芳华 + 运维** | 与既有安全边界冲突，生产审批无法绕过 |
| D6 | 控制群/通知群 | 新开一个专用群，或沿用 `AI-运营IM` 群并加 `[RELEASE]` 前缀 | 控制消息与业务讨论混流，误触发 |

---

## 1. 目标、边界与既有约束

### 1.1 目标

| 编号 | 目标 | 可衡量结果 |
| --- | --- | --- |
| G1 | 需求表驱动 | 表里一条「就绪」需求可自动走到「待发布」，中间不靠人手工搬数据 |
| G2 | 流程标准化 | 每个版本走同一套 S0–S13、同一套状态、同一套证据 |
| G3 | 人控节奏与优先级 | 开工、设计、发布审批、验收四处由人决定，agent 不自行扩范围 |
| G4 | 全链路可追溯 | 需求 ↔ PingCode 版本 ↔ 分支/提交 ↔ 部署 SHA ↔ 验收记录可查 |
| G5 | 不破坏现有能力 | 群消息 → 评估 → 工单 的老链路保持不变；新流程开关关闭时行为不变 |

### 1.2 非目标

- 不接管生产发布、生产审批、生产回滚、生产故障处置；
- 不替代产品决策、架构评审、验收判断；
- 不做多租户，先服务运营 IM 平台这一条线；
- 不重写十步工作法，复用 `~/.dsh/ten-step/spec-repo` 与仓库 `.agent/workflows`；
- 不改需求表的结构（新增状态值除外），不做表内公式修复。

### 1.3 既有硬约束（设计必须遵守）

| 来源 | 约束 | 对设计的影响 |
| --- | --- | --- |
| `message-nexus/AGENTS.md` | 主检出目录不得实现业务功能；未经明确要求不建分支/Worktree | 每个版本一个 worktree，由授权动作创建 |
| `message-nexus/AGENTS.md` | 发布只走 TOP：`feature/fix → PR → main → TOP 测试 → 验收 → TOP 生产`；测试与生产同一完整 commit SHA | 版本流程必须围绕 SHA 做证据 |
| `message-nexus/.ai/RELEASE.md` | 测试从提测分支部署，验收后 PR 合 main，生产固定 main SHA；禁 DeployHub | 决定了 S8/S9/S11 的顺序 |
| `.workbench/` 治理 | `preflight-project.sh . start/implement/verify/deliver/release`；功能状态只能 `transition-feature.sh` | 十步每个阶段前后要过预检 |
| 十步工作法 | ①需求→②产品→③架构→④计划→⑤任务→⑥开发→⑦单元→⑧用例→⑨测试→⑩报告 | 版本执行期的文档骨架 |
| 11-12 已确认边界 | 生产不由 agent 触碰，流程收敛到测试环境验收 | 状态机终点=待发布 |

---

## 2. 现状基线（2026-11-16 实测）

### 2.1 需求表

- 工作簿：钉钉 ALIDOC「运营IM平台需求管理」，node `YQBnd5ExVEwmGPlPU6GOEA7X8yeZqMmz`，共 10 个 sheet。
- 目标表：`02-需求池`（sheetId `s3`），数据区 `A1:Q45`，44 条需求、17 列，首行表头。
- 实测列：`提出日期, 提出人, 需求ID, 需求标题, 详细需求+业务价值, 优先级, 类型, 截图, 状态, 模块, 来源, 预计上线时间, 实际上线时间, 版本, 备注, 开发人员, 开发分析说明`
- 状态分布：`已上线 29 / 待评估 9 / 开发中 4 / 已搁置 1 / 已拒绝 1`。
- 字段缺失：需求ID 空 15、模块 空 34、优先级 空 22、开发人员 空 21、版本 空 15。
- 就绪度：9 条「待评估」里只有 3 条同时具备 ID/标题/描述/模块/优先级/提出人（REQ-202609-005、REQ-202610-005、REQ-202609-002）。
- 其它 sheet：`01-需求看板` 公式大量 `#VALUE!`；`05-版本规划`、`06-评审记录` 仍是模板样例（V2.3/李然/张敏），与本项目无关，**不要**把它们当数据源。

结论：需求池可用，但「自动开工」在当前数据质量下只能命中 3 条；缺口必须先播报、由产品补齐。

### 2.2 PingCode

- 项目：AI创新工作组项目（瀑布），`projectId=696e10c094ed0e00a4a430c6`。
- 版本命名：`【v4.{季度号}.{序号}】`，2026Q4 主力号 `v4.1217`；号段按业务线划分，挂在阶段节点下。
- 通道：`ad-deploy` MCP（`https://ads-deploy.tyhdev.com/mcp/stream`），工具 `pingcode_list_work_items / pingcode_create_work_item / pingcode_update_work_item`；成员 ID、状态 ID、号段见技能 `ai-weekly-pingcode-sync/references/pingcode-context.md`。
- 已知问题：`pingcode-context.md` 记录的是 Q3 号段（v4.958），表中与分支已在用 `v4.1217.43x`；Q4 阶段节点 ID 有两个历史值冲突，**建版本前必须实时查一次当季号段与节点**。

### 2.3 代码与发布

| 项 | 实测 |
| --- | --- |
| 主仓 | `~/Documents/code/功能开发区/运营IM平台/message-nexus`，基线分支 `main` |
| 适配器仓 | `message-nexus-dingtalk-adapter`，基线分支 `master` |
| 已有版本分支 | `feature/v4.1217.431`（两仓都有），说明 `feature/v4*` 命名已有先例 |
| Worktree | `var/worktrees/`，创建脚本 `scripts/create-feature-worktree.sh`，关闭脚本 `scripts/close-integrated-worktree.sh` |
| 测试环境 | `ark-test`（`message-nexus-test.tyhark.com`），服务 `message-nexus-app / -telegram / -whatsapp / -wecom-gateway` |
| 生产环境 | `ark-prod`（`message-nexus-new.tyhark.com`），固定 `main` 的完整 SHA |
| 发布工具 | TOP 平台（`top.tyhdev.com`）+ 本地 `npm run release:top -- prepare/record-test/authorize-production/record-production/close-observation` |
| 审批链 | 负责人审批（樊芳华）+ 运维审批（冯雍/周科/蓝俊森/邬昊霖/陈创杰） |
| 日志 | TOP MCP `query_logs`（`service` + `environment` + `keyword`）；ops-mcp 只读观测 |
| 证据 | `var/top-releases/<release-id>/`（被 gitignore），`/build-info.json` 暴露运行提交 |

### 2.4 十步工作法

- 清单：`~/.dsh/ten-step/spec-repo/.agent/workflows/speckit.*.md`，与技能 `speckit-*` 一一对应。
- ① 的固定契约：`NNN-short-name` 分支、`.specify/specs/<NNN-name>/spec.md`、归档 `docs/demand/`。
- `message-nexus` 已有 `.specify/specs/`、`.agent/workflows/`、`docs/specs/active|completed/`，两套目录并存；版本执行期要指定唯一落盘位置（见 4.4）。

### 2.5 DSH 原型现状

`dsh-superagent-prototype/src/release/` 已实现（纯函数 + 可注入端口）：

| 模块 | 能力 | 状态 |
| --- | --- | --- |
| `requirements.ts` | CSV 解析、列名匹配、就绪门、优先级排序、变更 diff | ✅ |
| `digest.ts` | 表快照、群播报文案、播报去重 ID | ✅ |
| `confirm.ts` / `go.ts` | G0 提问文案、只认审批人、开始/先做/暂缓/超时 | ✅ |
| `lock.ts` / `sheet-write.ts` | CAS 加锁、只写状态/开发人员/备注三列 | ✅ |
| `branch.ts` / `worktree.ts` | 从远端基线拉分支、读 head SHA | ✅ |
| `machine.ts` | 状态机、表状态映射、`feature/` 与 `fix/` 命名 | ✅ |
| `start.ts` | 加锁→取号→建版→建分支的顺序编排与失败即 blocked | ✅ |
| PingCode 适配器 | `PingCode` 端口只有接口，无 MCP 实现 | ❌ |
| 持久化 / 面板 / 合并 / TOP / 验证 / BUG 回环 | — | ❌ |

验证：`npx vitest run tests/release.spec.ts tests/release-lane.spec.ts` → 2 文件 46 项全绿（2026-11-16）。

### 2.6 工具通道可用性

| 通道 | 端点/命令 | 本会话实测 | 风险 |
| --- | --- | --- | --- |
| 钉钉表格 | `dws sheet csv-get/list-sheets` | ✅ 读表成功 | 写回命令 `sheet range update` 未在本轮联调 |
| 钉钉群 | `dws` 群消息/发送 | ✅ 可用 | 控制消息识别、@人 ID 需配置 |
| PingCode | `ad-deploy` MCP | ⚠️ 未联通（本机网络/会话未挂载），工具名来自技能记录 | 建版必须真实联调一次 |
| TOP | `https://top.tyhdev.com/token/mcp/` | ❌ 本机 12s 超时 | 发布工具名/参数待平台确认 |
| ops-mcp | `https://ops-mcp.tyhsys.com/mcp` | 未测 | 只读观测，38 个工具 |
| GitHub | `gh` | ✅ 已登录 `fanghuafan` | — |
| Git | 本地仓库 | ✅ | 两仓基线分支不同 |

> 结论：设计可以定稿，但 P1/P3 必须先做「通道联调」这一独立前置，不能假设 MCP 一定可用。

---

## 3. 流程总体设计

### 3.1 角色与职责

| 角色 | 谁 | 职责 | 不可越界 |
| --- | --- | --- | --- |
| 需求方/验收人 | 孙俊林 | 确认开工顺序、澄清产品问题、验收 | 不审批 TOP、不改代码 |
| 技术负责人 | 李兴根 | 架构/技术澄清、设计签字 | 不验收产品 |
| 发布负责人 | 樊芳华 | TOP 负责人审批、发布窗口 | 不替代运维审批 |
| 运维 | 冯雍/周科/蓝俊森/邬昊霖/陈创杰 | TOP 运维审批、生产操作 | — |
| Superagent | 本系统 | S0–S13 中除人闸外的全部执行 | 不生产发布、不自批、不扩范围 |

### 3.2 状态机

```
candidate ──就绪──► awaiting_go ──G0 开始──► developing
     ▲                   │ 暂缓/超时              │
     └───────────────────┘                       │ 版本+分支完成
                                                 ▼
                                            designing（十步②③）
                                                 │ G1 签字
                                                 ▼
                                            integrating（⑦⑧⑨+本地集成）
                                                 │ 通过
                                                 ▼
                                            deploying（提测分支 → TOP 测试，G2 审批）
                                                 │
                                                 ▼
                                            verifying（SHA/日志/页面）
                                                 │
                                                 ▼
                                       awaiting_acceptance（状态=待验收）
                                          │ 通过              │ 不通过
                                          ▼                   ▼
                                   merging（PR + CI + 合 main） fixing
                                          │                   │
                                          ▼                   └──► integrating
                                  awaiting_release（状态=待发布）
                                          │
                                          ▼
                                      released（人写，agent 不写）

任一阶段 ──无法继续──► blocked（带原因，群里报，等人）
任一阶段 ──需求取消──► cancelled（关 PingCode 版本、释放锁）
已验收后 ──线上故障──► rolled_back（人决定，agent 只记录）
```

### 3.3 阶段表

| 阶段 | 输入 | 动作 | 产物/证据 | 闸门 | 失败即 |
| --- | --- | --- | --- | --- | --- |
| S0 发现 | 表 revision | 轮询、解析、diff | 快照 + 变更播报 | — | 只告警 |
| S1 就绪 | 候选行 | 缺字段检查、映射仓库、查活跃版本 | 就绪/不就绪原因 | 就绪门 | 播报缺口 |
| S2 确认 | 就绪候选 | 群内 @孙俊林，列前 10 条 | 确认/改派/暂缓 | **G0**（10min） | 顺延 |
| S3 锁定 | 确认结果 | CAS 写 `开发中` + `开发人员` | 行 revision、锁记录 | — | 行被改→回队列 |
| S4 版本 | 锁定 | PingCode 查重、取号、建版 | workItemId、版本号 | — | blocked |
| S5 分支 | 版本 | worktree + `feature/vX` | 分支、head SHA | — | blocked |
| S6 十步 | 分支 | ①–⑥，②③ 人签字 | spec/design/plan/tasks、代码 | **G1** | 回 designing |
| S7 集成 | 代码 | ⑦⑧⑨ + 本地启动、e2e、修复 | 测试日志、截图、报告 | — | 修 3 轮后 blocked |
| S8 提测 | 分支 | 推送、建 PR（draft） | 分支 tip、PR 号 | — | blocked |
| S9 发布 | 分支 tip | TOP 测试环境部署、提交审核、等审批 | releaseId、buildId、deployed SHA | **G2** | blocked |
| S10 验证 | deployed SHA | build-info、库表、日志、页面 | 证据包 | — | 回 fixing |
| S11 验收 | 证据包 | 状态=待验收，@孙俊林 | 验收结论 | **G3** | 回 fixing |
| S12 收口 | 通过 | PR 合 main、CI、状态=待发布 | merge SHA、通知 | CI | blocked |
| S13 回环 | 生产 BUG | 新 BUGFIX 版本、`fix/` 分支 | 关联原版本 | **G0** | — |

### 3.4 九步草案 → 本设计映射

| 草案步骤 | 本设计阶段 | 可行性 | 说明 |
| --- | --- | --- | --- |
| 1 拉需求/锁定/改状态 | S0–S3 | ✅ 可行 | 表可读、CAS 可写；建议状态值见 D3 |
| 2 PingCode 建版本 | S4 | ⚠️ 需前置 | ad-deploy 通道需联调；Q4 号段/节点需实时确认 |
| 3 建 `feature/v*` 分支 | S5 | ✅ 可行 | 已有 `feature/v4.1217.431` 先例；需走 worktree 脚本 |
| 4 十步 + 澄清 @人 | S6 | ✅ 可行 | 澄清路由需固定「产品 @孙俊林 / 技术 @李兴根」 |
| 5 本地集成测试 | S7 | ✅ 可行 | 仓库命令齐全；需 PG + 本地端口 |
| 6 合并到 develop | S8/S12 | ❌ 不成立 | 无 `develop`，规约要求合 `main`（D1） |
| 7 TOP 测试发布+审批 | S9 | ⚠️ 顺序相反 | 仓库规约先提测分支后合 main（D2）；审批链是樊芳华+运维 |
| 8 验证+通知验收 | S10–S11 | ✅ 可行 | `build-info.json` + `query_logs` + 页面截图 |
| 9 BUG 循环到上线 | S13 | ⚠️ 边界收敛 | agent 到「待发布」，生产由人（D5） |

### 3.5 六处必须对齐的地方

**D1 `develop` 不存在。** `message-nexus` 只有 `main`/`master` 两条基线，`.ai/RELEASE.md` 明确「测试通过后必须通过 PR 合并入本地与远端 `main`，生产固定选择 `main` 的完整 Commit SHA」。新建 `develop` 会同时破坏 CI、Worktree 治理和生产基线。→ 本设计把「合并目标」做成配置，`message-nexus=main`、`adapter=master`；若业务坚持要 `develop`，需要先改仓库发布规约，那是另一个变更。

**D2 顺序：先提测，后合并。** 仓库规约是「版本提测分支 → 测试环境验收 → PR 合并 main → 生产」。草案是「先合并 → 再发测试」。若按草案执行，未验收代码会先进入 `main`，生产基线被污染，且 TOP 测试环境按分支部署的能力被浪费。→ 本设计采用仓库顺序；S8 只建 PR（draft），S12 验收通过后才合。

**D3 状态字典要扩。** 现状只有 `已上线/待评估/开发中/已搁置/已拒绝`，没有 `待验收/待发布`。agent 没有合法的回写值。→ 建议新增 `待验收`、`待发布` 两个值（以及可选的 `Agent开发中`），并同步修 `01-需求看板` 的统计公式；在未扩之前，agent 只能写 `开发中` + 备注。

**D4 版本双轨。** 表里「版本」列是产品版本（`v0.7.26`/`v0.8.0`），PingCode/分支是季度版本（`v4.1217.xxx`）。→ 约定：分支名与 PingCode 用 `v4.1217.xxx`；表「版本」列写 `v4.1217.xxx`，产品版本写入「备注」；「实际上线时间/版本」由人填。

**D5 生产边界与审批链。** 生产不由 agent 触碰（11-12 已确认），agent 到「待发布」为止。TOP 发布审核 = 负责人（樊芳华）+ 运维（冯雍/周科/蓝俊森/邬昊霖/陈创杰），不是孙俊林；agent 提交审核后只能轮询，不能自批。→ 两个闸门分开：G2 平台审批、G3 业务验收。

**D6 控制群与通知群。** 控制消息（开始/先做/暂缓/验收/停）与业务闲聊同群会误触发。→ 建议专用群；若沿用现有群，消息统一加 `[RELEASE]` 前缀，且控制指令只认授权人。

### 3.6 人闸与澄清路由

| 闸门 | 谁 | 触发 | 时限 | 不回复 |
| --- | --- | --- | --- | --- |
| G0 开工 | 孙俊林 | 有就绪候选 | 10 分钟 | 顺延下一条，候选保留 |
| G1 设计 | 孙俊林（产品）/ 李兴根（技术） | ②③④ 完成后 | 工作时间内 4 小时 | 版本停 `designing`，群里提醒一次 |
| G2 发布审批 | 樊芳华 + 运维 | TOP 提交审核后 | 平台 SLA | 停 `deploying`，每 30 分钟查一次 |
| G3 验收 | 孙俊林 | 测试环境验证完成 | 24 小时 | 提醒一次，不自动通过 |

**澄清路由**（草案第 4 步）：

```
产品问题（做什么/边界/优先级/验收口径） → @孙俊林
技术问题（架构/接口/数据/环境/风险）     → @李兴根
两者都涉及                              → 一条消息列出，分别 @
回复的采纳规则                          → 只认被 @ 的人的回复；写入 spec 的「澄清记录」并注明来源
超时                                    → 版本停在该阶段并 blocked，不猜
```

一条消息只问一组问题，最多 3 个；回复落盘到 `.specify/specs/<id>/clarifications.md`。

### 3.7 实施前置：分支治理

`AGENTS.md` 要求用 `scripts/create-feature-worktree.sh <主项目> <功能ID> codex/feature/<功能ID>` 创建 worktree 并登记 Workbench，而版本分支命名是 `feature/v4.1217.xxx`。两者是否兼容，P1 必须先做一次真实演练：脚本是否接受版本号作为功能 ID、登记表能否容纳、关闭时能否正确清理。若不兼容，用「功能 ID = 版本后缀、分支名 = feature/v4.1217.xxx」的参数组合，或给脚本增加合法分支名参数（需仓库 owner 同意），**不得**绕过登记直接 `git worktree add`。

---

## 4. 分步设计

### 4.1 S0–S1 需求拉取与就绪门

- 轮询：默认 5 分钟；表 revision 未变则跳过；写回后 1 个周期内忽略自己造成的变更。
- 解析：`dws sheet csv-get --node <node> --sheet-id 02-需求池 --range A1:Q<last>`，按表头名匹配列，不按列序。
- 身份：优先 `需求ID`；无 ID 时用 `date+requester+title` 的短哈希（与原型一致），补号后自动迁移。
- 就绪门（全部满足才可开工）：

| 条件 | 判据 | 当前缺口 |
| --- | --- | --- |
| 有 ID | 匹配 `^[A-Za-z]+-\d+` | 15 条缺 |
| 有标题 | 非空 | — |
| 有描述 | `详细需求+业务价值` 非空 | 少量缺 |
| 有模块 | `模块` 非空且能映射仓库 | 34 条缺 |
| 有优先级 | `超高/高/中/低` | 22 条缺 |
| 有提出人 | `提出人` 非空 | — |
| 无活跃版本 | 同一需求没有未终结 Release | — |

- 不就绪：只播报「缺什么」，不锁、不建版本；产品补齐后下一轮自动进入候选。
- 验收标准的缺口：表中没有独立「验收标准」列，S1 不强制；验收清单在 S6 由 ②/⑧ 产出，G3 时使用。

### 4.2 S2–S3 开工确认与锁定

- G0 消息：前 10 条就绪候选 + 建议第 1 条 + 三种回复（`开始` / `先做 REQ-xxx` / `暂缓`）。
- 只认孙俊林；他人回复忽略；窗口 10 分钟；超时顺延但候选不丢。
- 锁定 CAS：写 `状态=开发中`、`开发人员=Superagent`、备注追加 `agent 锁定 <ISO 时间>`；写前重读该行，行被移动/改 ID 则放弃并回队列。
- 锁定后播报：需求 ID、版本号（下一步回填）、分支名、预计路径。
- 幂等：同一需求同一时刻只允许一个活跃 Release；锁记录带 `sheetRevision`，写回时校验。

### 4.3 S4 PingCode 建版本

- 查重：`pingcode_list_work_items(projectId, assigneeId="all", keywords=版本前缀)`，同时按裸号段查（`1217.4xx`）兼容 `v.4.` 变体。
- 取号：`prefix=v4.1217`，段位按当季约定（Q4 需重新确认；草案记录 400–449 为企微/运营 IM），段内取下一个空号。
- 建版：`pingcode_create_work_item(typeName="版本", projectId, phaseId, assigneeId, startDate=今天, endDate=预计上线, name="【v4.1217.xxx】REQ-xxx 标题", description=需求链接+分支+agent 标记)`。
- 回填：版本号写表「版本」列（只写本次新版本行，历史行不动）；`pingcodeId` 入 Release 记录。
- 幂等键：`pingcode:create:<requirementId>:<version>`；成功即入 effect 台账，重试前先查台账，禁止盲目重建。
- 失败：`blocked`，不自动重试；群里报「建版失败 + 原因」，人工确认后由指令重试。

### 4.4 S5 分支

- 基线：`origin/main`（adapter 为 `origin/master`），先 `fetch` 再校验远端引用存在。
- 命令序列：`fetch origin <base>` → `rev-parse --verify origin/<base>` → `switch --detach origin/<base>` → `switch -c feature/v4.1217.xxx` → `rev-parse HEAD`。
- worktree：`var/worktrees/<version>`，由 `create-feature-worktree.sh` 创建并登记；主检出不动。
- 已存在：同名分支存在 → `blocked`，不删不覆盖；若上次失败在「建了分支但没记录」，且 head == base，可按记录复用。
- 多仓：一个版本可含多个仓库，各自同名分支；任一失败 → 记录已建成的仓库并 `blocked`。

### 4.5 S6 十步开发

| 十步 | 落盘位置（建议） | 产出 | 闸门 |
| --- | --- | --- | --- |
| ① 需求分析 | `.specify/specs/<version>/spec.md` + `docs/demand/` | 需求规格 | 就绪门 |
| ② 产品设计 | `docs/specs/active/<feature>/product.md` | 流程/原型/页面状态 | **G1 孙俊林** |
| ③ 架构设计 | `docs/specs/active/<feature>/architecture.md` | 接口/数据/风险 | **G1 李兴根** |
| ④ 开发计划 | `docs/specs/active/<feature>/plan.md` | 里程碑/回滚/验证 | G1 一并确认 |
| ⑤ 任务分配 | `tasks.md` | 任务清单 | — |
| ⑥ 执行开发 | worktree 内代码 | 代码 + 提交 | 预检 `implement` |
| ⑦ 单元测试 | `tests/` | 单测 | 覆盖率门槛按仓库 |
| ⑧ 测试用例 | `docs/test/<feature>-testcase.md` | 验收场景 | — |
| ⑨ 功能测试 | `docs/test/<feature>-testreport.md` | 报告 + 证据 | — |
| ⑩ 开发报告 | `docs/report/<version>.md` | 总结 | 进验收前产出 |

- 与十步默认契约的差异：① 不再新建 `NNN-short-name` 分支（分支已由 S5 建好），spec 目录改用版本号；其余文档模板照用。
- 每阶段前后跑 `.workbench/bin/preflight-project.sh . <mode>`；功能状态用 `transition-feature.sh` 前进，不手改。
- ②③ 未签字不得进 ⑥；签字用群内明确回复，记录 messageId。
- 需求变更：走 `speckit.update` 增量更新，不重跑十步。

### 4.6 S7 本地集成测试

- 命令（以仓库为准，顺序固定）：
  `npm run format:check` → `npm run lint` → `npm run typecheck` → `npm test` → `npm run build`。
- 本地启动：`dev:core`（3001）+ `dev:web`（5173）+ PostgreSQL；按需 `dev:wecom-gateway`/`dev:tg`/`dev:wa`。
- 集成场景：从 ⑧ 的验收场景里取，覆盖「发送链路不受影响」这一硬约束的回归。
- 修复循环：≤3 轮；每轮记录失败用例、修复点、重跑结果；仍失败 → `blocked` + 证据。
- 证据：命令输出摘要、失败截图、复现步骤，存入 `var/evidence/<version>/`（gitignore）。

### 4.7 S8 提测与 PR

- 推送分支到远端；建 PR（`gh pr create --draft --base main`），标题带版本号，正文先放「待验收」。
- PR 只作为合并载体，不在验收前合入。
- CI 必须绿；CI 失败按 4.6 修，不在 `main` 上修。

### 4.8 S9 TOP 测试发布与审批

- 前置：本地 `npm run build`；`npm run release:top -- prepare` 冻结候选 SHA 与回滚 SHA。
- TOP 发布单：步骤① SQL（本次有 DDL 时，先执行）→ 步骤② 部署服务 `message-nexus-app`，环境 `ark-test`，分支 = 提测分支（不是 main）。
- 提交审核 → 等负责人 + 运维审批（**人**）。
- 轮询发布状态；记录 `releaseId`、`buildId`、配置版本、实际运行 SHA，执行 `npm run release:top -- record-test`。
- 幂等：一个版本一个发布单；发布失败不自动重发，先看日志再决定。
- 失败：`blocked`，群里报「部署失败 + task/日志线索」。

### 4.9 S10 验证

| 检查 | 方法 | 通过标准 |
| --- | --- | --- |
| 运行提交 | `GET /build-info.json` | 等于候选 SHA（前缀≥7 位） |
| 库表 | `schema_migrations` 最大版本、目标表存在 | 与发布单一致 |
| 启动健康 | Pod/实例 Ready、重启 0 | 无 CrashLoop |
| 日志 | TOP `query_logs` 过滤 `error/fatal/42P01` | 无新增错误族 |
| 页面/功能 | 浏览器逐条走 ⑧ 的场景，截图 | 与验收清单一致 |
| 发送链路 | 发送成功率/时延对比 | 不劣于发布前 |

- 证据包：截图、日志片段、build-info、发布单号、SHA，写入 `var/top-releases/<release-id>/`，并生成一页摘要。
- 任一不通过：回 `fixing`，修复后重新提测（不沿用旧证据）。

### 4.10 S11 验收与 S12 收口

- 状态写 `待验收`；群里 @孙俊林，带 deployed SHA + 证据摘要 + 验收清单。
- 回复「验收通过」→ 合 main（PR ready → CI → `--no-ff` merge），记录 merge SHA，状态写 `待发布`，通知樊芳华走生产。
- 回复「有问题：…」→ 状态回 `开发中`，问题入 ⑧ 用例与修复清单，回 S7。
- 验收超时 24h：提醒一次；不自动通过。
- **生产发布由人执行**：TOP 生产（main SHA）→ 观察 30/60 分钟 → 人写 `已上线`、填实际上线时间/版本。

### 4.11 S13 BUG 回环

```
孙俊林群里报 BUG（带版本/现象/复现/期望）
   ▼
判定：BUG 还是新需求？（人判断，agent 不扩范围）
   ├─ 新需求 → 回需求表，走 S0
   └─ BUG → 新建 BUGFIX Release
             · 新 PingCode 版本（新号）
             · 分支 fix/v4.1217.xxx（从 main 起）
             · parentReleaseId 指向原版本
             · 表备注追加「修复版本 vX」，原版本记录保留
             ▼
         G0（同一流程）→ 十步 → 集成 → 提测 → 验证 → 验收 → 合 main → 待发布
```

- 同一 BUG 反复出现：升级为「质量问题」，在 ⑩ 报告里做根因，不重复开版本。
- 生产回滚：人执行；agent 只记录 `rolled_back` 与证据。

---

## 5. 数据、幂等与恢复

### 5.1 表结构（在原型独立表上扩展，不动现有 Ticket 表）

```sql
release(
  id TEXT PRIMARY KEY, type TEXT, requirement_id TEXT, title TEXT,
  version TEXT UNIQUE, parent_release_id TEXT, state TEXT,
  sheet_row INTEGER, sheet_revision TEXT, pingcode_id TEXT,
  reason TEXT, created_at TEXT, updated_at TEXT
);
release_repo(
  release_id TEXT, name TEXT, path TEXT, branch TEXT, base TEXT,
  head_sha TEXT, merge_sha TEXT, PRIMARY KEY (release_id, name)
);
release_event(
  id INTEGER PRIMARY KEY AUTOINCREMENT, release_id TEXT,
  stage TEXT, at TEXT, actor TEXT, detail_json TEXT
);
effect(                              -- 外部副作用台账
  key TEXT PRIMARY KEY,              -- 幂等键
  release_id TEXT, action TEXT,      -- pingcode.create / git.branch / top.release / sheet.write
  target TEXT, external_id TEXT,
  status TEXT,                       -- pending | done | failed
  at TEXT, detail_json TEXT
);
bug(
  id TEXT PRIMARY KEY, release_id TEXT, parent_release_id TEXT,
  message_id TEXT, title TEXT, severity TEXT, state TEXT, created_at TEXT
);
```

### 5.2 幂等键

| 动作 | 键 | 冲突处理 |
| --- | --- | --- |
| 建 PingCode 版本 | `pingcode:create:<requirementId>:<version>` | 先查台账，再查 PingCode |
| 建分支 | `git:branch:<repo>:<branch>` | 已存在→blocked（除 4.4 的例外） |
| TOP 发布 | `top:release:<version>:<env>` | 一个版本一个发布单 |
| 表写回 | `sheet:write:<releaseId>:<state>:<revision>` | 同 revision 同状态只写一次 |
| 群播报 | `announce:<changeKey>` | 已发过跳过 |

### 5.3 恢复

- 每个动作执行前写 `effect(status=pending)`，成功写 `done` + external id；重启后先扫 pending，查外部状态再决定完成或失败。
- 断点续跑：状态机 + 事件表可重放；禁止从「当前聊天上下文」推断进度。
- 表 revision 变化：以锁定时的 revision 为基准，写前重读校验。

---

## 6. 并发与资源

| 资源 | 限制 | 机制 |
| --- | --- | --- |
| 活跃版本 | 2（可配） | 队列 + 优先级 |
| 同仓开发 | 1 | 仓库锁 |
| 合并 main | 串行 | merge queue |
| 测试环境部署/验收 | 1 | deploy lock（同一环境不并发） |
| G0 提问 | 一次一条 | 串行，避免刷屏 |
| Worker 槽 | 复用现有 `maxWorkers` | 阶段间释放 |

---

## 7. 权限、安全与审计

| 能力 | agent | 人 |
| --- | --- | --- |
| 读需求表 | ✅ | ✅ |
| 写 状态/开发人员/备注 | ✅ | ✅ |
| 建 PingCode 版本 | ✅ | ✅ |
| 建分支/推送/建 PR | ✅ | ✅ |
| 合并 main | ✅（仅 G3 通过后） | ✅ |
| TOP 提交审核 | ✅ | ✅ |
| TOP 审批 | ❌ | ✅ |
| 生产发布/回滚 | ❌ | ✅ |
| 写「已上线」 | ❌ | ✅ |
| 改需求范围/优先级 | ❌（只能建议） | ✅ |

- 凭据：token 只从环境变量/本地配置读取，不写进 prompt、日志、提交、群消息。
- 审计：每次写操作记录 actor、时间、目标行/版本、幂等键。
- 群：禁止 @all；控制消息只认授权人；生产故障信息不转贴到外部。

---

## 8. 失败处理矩阵

| 失败 | 处理 | 是否自动重试 |
| --- | --- | --- |
| 表读取失败/格式变化 | 停轮询，告警，保留上一快照 | ✅ 下轮 |
| 行被他人修改 | 放弃锁定，回队列 | ✅ 下轮 |
| 就绪字段缺失 | 播报缺口 | — |
| PingCode 不可达 | blocked，群报 | ❌ |
| 版本号段满 | blocked，请人确认扩段 | ❌ |
| 分支已存在 | blocked | ❌ |
| 十步设计不通过 | 回 designing，记录意见 | — |
| 集成测试失败 | 自动修 ≤3 轮 | ✅ 限次 |
| CI 失败 | 修完重推 | ✅ 限次 |
| TOP 审批拒绝 | blocked，记录拒绝意见 | ❌ |
| 部署失败 | blocked，拉日志 | ❌ |
| 验证不通过 | 回 fixing | — |
| 验收不通过 | 回 fixing | — |
| 验收超时 | 提醒一次 | — |
| 生产 BUG | 新 BUGFIX 版本 | — |
| 需求取消 | cancelled，关 PingCode 版本，释放锁 | — |

---

## 9. 可观测性与度量

| 指标 | 定义 | 目标（M0） |
| --- | --- | --- |
| Lead time | 锁定 → 待发布 | 首版不设硬指标，先测量 |
| Gate wait | G0/G1/G2/G3 等待时长 | 可视化，不考核 |
| 一次验收通过率 | 首次 G3 通过 / 版本数 | ≥70% |
| 返工轮次 | 每版本修复轮次 | ≤3 |
| 自动化率 | 无人工介入的阶段占比 | 先测基线 |
| BUG 逃逸 | 生产 BUG / 版本数 | 记录并归因 |

面板：版本看板（状态分列）+ 详情（需求/版本/分支/SHA/PingCode/部署/证据/时间线）+ 操作（开始/改派/暂缓/重试/取消）。

---

## 10. 原型实现映射

| 设计模块 | 原型位置 | 现状 | 缺口 |
| --- | --- | --- | --- |
| 表源 | `src/sources/sheet.ts` + `src/release/requirements.ts` | ✅ | 真实轮询接线、写回联调 |
| 播报 | `src/release/digest.ts` | ✅ | 控制群配置 |
| G0 | `confirm.ts`/`go.ts` | ✅ | 与入站消息分流接线 |
| 锁定 | `lock.ts`/`sheet-write.ts` | ✅ | `sheet range update` 实测 |
| 版本 | `start.ts` + `PingCode` 端口 | 接口✅ 实现❌ | ad-deploy 适配器 |
| 分支 | `branch.ts`/`worktree.ts` | ✅ | worktree 脚本参数验证 |
| 状态机 | `machine.ts` | ✅ | 持久化、面板 |
| 十步 | — | ❌ | release worker 预设 + 文档落盘 |
| 合并 | — | ❌ | PR/CI/merge queue |
| TOP | — | ❌ | 适配器 + 审批轮询 |
| 验证 | — | ❌ | build-info/日志/页面证据 |
| BUG 回环 | — | ❌ | 触发/判定/新版本 |

复用原则：新增模块放进 `src/release/`，`release.enabled` 默认关闭；关闭时老链路行为不变。

---

## 11. 实施计划

| 阶段 | 内容 | 工期 | 验收 |
| --- | --- | --- | --- |
| P0（已完成） | 表解析/就绪门/G0/锁定/取号/分支/状态机 + 46 项测试 | — | 测试全绿 |
| P1 通道与创建 | ad-deploy 适配器（查重/建版/改状态）、持久化、真实写回、worktree 脚本验证、只读面板 | 3–5 天 | 一条真实需求走到「分支已建 + PingCode 有版本」 |
| P2 开发与集成 | 十步 worker 预设、文档落盘、集成测试编排、PR 创建、CI 门禁 | 4–6 天 | 一条真实需求走到「PR 待合并 + 证据包」 |
| P3 发布与验收 | TOP 测试发布适配器、审批轮询、验证脚本、验收消息、合 main | 3–5 天 | 一条真实需求走到「待发布」 |
| P4 回环与加固 | BUGFIX 版本、指标、面板、失败恢复、并发限制 | 2–3 天 | 模拟生产 BUG 走完修复→待发布 |

先做「只读演练」一周：只播报候选与缺口，不写表、不建版本，校准需求质量与误报。

---

## 12. 测试策略

| 层 | 内容 |
| --- | --- |
| 单元 | 状态机、排序、就绪门、消息解析、幂等键、取号 |
| 集成 | 表读写（CAS）、PingCode 适配器（mock + 一次真实）、git 临时仓库、群路由 |
| 端到端 | 一条真实需求：锁定→版本→分支→十步→提测→验证→验收→合 main |
| 演练 | 只读一周、失败注入（通道超时/审批拒绝/部署失败） |
| 回归 | 老链路（群→评估→工单）行为不变；现有测试全绿 |

---

## 13. 风险与开放决策

### 13.1 风险

| 风险 | 影响 | 对策 |
| --- | --- | --- |
| 需求就绪度低（当前仅 3/9 可开工） | agent 空转 | 缺口播报 + 产品补字段 |
| 看板公式大量报错 | 统计不可信 | 不依赖看板，用表数据自算 |
| PingCode 号段/节点过期 | 建版错位 | 每次建版前实时查当季规则 |
| TOP MCP 本机不可达 | 发布自动化中断 | 先修通道；失败即 blocked |
| 测试环境单槽 | 版本排队 | deploy lock + 批次窗口 |
| 验收等待长 | Lead time 失控 | 时限 + 提醒 + 面板暴露 |
| 分支命名与治理脚本冲突 | 建分支失败 | P1 先验证，必要时改脚本参数 |
| 状态值扩展影响看板 | 统计失真 | 同步改公式，灰度观察 |

### 13.2 开放决策（对应 0.3）

1. **D1 合并目标**：建议 `main`；若坚持 `develop`，需先改仓库发布规约。
2. **D2 顺序**：建议「提测分支 → 验收 → 合 main」。
3. **D3 状态字典**：建议新增 `待验收`/`待发布`，并同步修看板。
4. **D4 版本双轨**：建议表「版本」列写 PingCode 号，产品版本进备注。
5. **D5 生产边界与审批**：建议 agent 到「待发布」；TOP 审批由樊芳华 + 运维。
6. **D6 控制群**：建议独立群或用 `[RELEASE]` 前缀。

---

## 14. 附录

### 14.1 状态字典

- 表中：`待评估`、`开发中`、`待验收`（新增）、`待发布`（新增）、`已上线`、`已搁置`、`已拒绝`
- agent 可写：`开发中`、`待验收`、`待发布`
- 人可写：`已上线`、`已搁置`、`已拒绝`、以及所有字段的人工修正

### 14.2 群消息模板

```
【G0】@孙俊林 这批需求里，建议先做：
1. REQ-202609-005 聊天界面没有头像展示  低 IM聊天
...
回复「开始」做第 1 条，或「先做 REQ-xxx」指定，或「暂缓」。
10 分钟未回复，我顺延下一条。

【提测】v4.1217.xxx 已部署测试环境 ark-test
  deployed SHA: <sha>  发布单: R-xxxx  build: <id>
  变更: <摘要>
  证据: <截图/日志/用例>
  验收清单: 1) ... 2) ...
请回复「验收通过」或「有问题：...」

【BUG】@Superagent BUG v4.1217.xxx
  现象: ... 复现: ... 期望: ... 截图: ...
```

### 14.3 命令速查

```bash
# 读表
dws sheet csv-get --node YQBnd5ExVEwmGPlPU6GOEA7X8yeZqMmz --sheet-id 02-需求池 --range A1:Q45 -f json
# 写表（只写状态/开发人员/备注）
dws sheet range update --node <node> --sheet-id 02-需求池 --range A24:Q24 --values '[[...]]'
# 本地门禁
npm run format:check && npm run lint && npm run typecheck && npm test && npm run build
# 发布证据
npm run release:top -- prepare ... / record-test ... / authorize-production ... / record-production ...
# 原型测试
npx vitest run tests/release.spec.ts tests/release-lane.spec.ts
```

### 14.4 参考

- 需求表：`https://alidocs.dingtalk.com/i/nodes/YQBnd5ExVEwmGPlPU6GOEA7X8yeZqMmz`
- 原型设计稿：`dsh-superagent-prototype/docs/design/release-workflow.md`（v0.1，本文在其基础上对齐真实仓库）
- 发布规约：`message-nexus/.ai/RELEASE.md`、`message-nexus/AGENTS.md`、`message-nexus/docs/operations/087-test-release.md`
- 十步：`~/.dsh/ten-step/spec-repo/.agent/workflows/speckit.*.md`
- PingCode：`ai-weekly-pingcode-sync/references/pingcode-context.md`

### 14.5 本轮验证记录（2026-11-16）

| 项 | 方式 | 结果 |
| --- | --- | --- |
| 需求表结构 | `dws sheet +list-sheets` / `csv-get A1:Q45` | ✅ 10 个 sheet、17 列、44 条，分布与缺口为实测值 |
| 发布规约 | 读 `AGENTS.md`/`.ai/RELEASE.md`/`.ai/WORKFLOW.md`/087 交接单 | ✅ 顺序与审批链来自仓库文档 |
| 十步清单 | 读 `~/.dsh/ten-step/spec-repo/.agent/workflows` | ✅ ①–⑩ 与技能名一致 |
| 原型能力 | `npx vitest run tests/release.spec.ts tests/release-lane.spec.ts` | ✅ 2 文件 46 项通过 |
| ad-deploy MCP | 未联通（本会话未挂载/网络） | ⚠️ 工具名来自技能记录，P1 需真实联调 |
| TOP MCP | `top.tyhdev.com` 12s 超时 | ❌ 本机不可达，工具名与时序待平台确认 |
| 表写回 | `sheet range update` 未执行 | ⚠️ P1 需在一次真实锁定中验证 |

> 本文只做设计，不含代码改动；实现按第 11 章 P1–P4 推进。

---

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
