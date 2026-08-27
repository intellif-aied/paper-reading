---
layout: default
title: "XSkill 分享：三种叙事方案"
permalink: /2026-08-28/skill/narratives.html
---

# XSkill 分享：三种叙事方案

这份文件只决定讲述顺序，不改动事实口径。每套方案都使用同一批论文、源码、Wiki 与实验材料。

| 方案 | 开场问题 | 核心内容 | 适合听众 | 建议时长 |
|---|---|---|---|---:|
| A · 一次更新的生命史 | 一条工作经验怎样成为可回滚的 Skill 版本？ | 蒸馏、版本、分发、Canary | 综合听众、Agent 工程 | 20–25 分钟 |
| B · 四层证据审计 | 论文中的系统设计，当前实现和实验分别走到了哪里？ | Paper / Wiki / Source / Run | Paper reading、系统研究 | 20–25 分钟 |
| C · Canary 在线评估 | 一个 Skill 更新凭什么获得全部流量？ | 曝光、评分、分组、晋升 | Eval、实验平台、ML Infra | 15–20 分钟 |

当前建议：以 **C** 为主轴；用 **A 的第 2–6 页**补足蒸馏与版本背景；把 **B 的实现差异和实验审计**放入附录。

---

## 方案 A：一条 Skill 更新的生命史

**中心论点**

XSkill 用轨迹证据、独立 Git 版本和真实流量评分，把 `SKILL.md` 变成可维护、可试用、可回滚的团队资产。当前实现完成了版本骨架，评分归因仍有缺口。

**适合听众**

产品、Agent 工程师，以及首次接触 XSkill 的听众。

**控制点**

全程只跟踪“一个已有 Skill 的一次更新”，避免讲成功能清单。

### A1 · 139 个 Skill 以后，管理成本出现了

- 核心句：作者报告的单机库存有 139 个 Skill，目录说明占 26,448 字符，是 8,000 字符预算的 3.3 倍。
- 证据：139 / 26,448 / 3.3× 三个大数字。
- 口径：作者报告的单用户快照，不外推总体分布。

### A2 · 先定义六个对象

- `Skill`：一个可安装目录。
- `Version`：该目录的一次 Git commit。
- `Main`：当前稳定版本。
- `Staging`：等待真实任务评估的更新。
- `Baby`：新 Skill 的首版草稿。
- `Atom`：表达单一用户任务的会话片段。
- 画面：六张名词卡，沿用版本状态图的颜色。

### A3 · 会话先被切成可归属的证据

- 核心句：TaskAgent 把长会话切成 Atom，并保留任务、结果、来源行号和上下文。
- 画面：[轨迹标准化图](./diagrams/trajectory-normalization.html)。
- 实现边界：Team 上传的是经过筛选、截断和脱敏的 Markdown。

### A4 · Weightscore 决定何时改写

- 核心句：TaskClusterAgent 给 Atom 与 Skill 的相关性打 1–10 分；累计达到 `θ=10` 后触发 SkillEdit。
- 画面：[蒸馏流水线](./diagrams/distillation-pipeline.html)。
- 必讲细节：一次权重 10 和十次权重 1 都能达到阈值；当前源码没有独立轨迹数量门槛。

### A5 · 一次编辑产生一个可定位的版本

- 核心句：已有 Skill 的自动更新进入 Staging；新 Skill 的首版从 Baby 毕业到 Main。
- 画面：[版本状态图](./diagrams/skill-version-state.html)。
- 脚注：`generate` 和 jam breaker 存在直接更新 Main 的路径。

### A6 · Standalone 与 Team 只改变分发边界

- Standalone：同一台机器本地采集、生成和安装。
- Team：服务端保存版本清单，客户端同步指定 commit SHA。
- 画面：[系统边界图](./diagrams/xskill-system-map.html)。

### A7 · Canary 给候选版本一小部分真实任务

- 核心句：多数任务继续使用 Main，默认约 20% 分配给 Staging，再比较两组任务结果。
- `side`：一次分流选择的 Main 或 Staging。
- 画面：[Canary 反馈闭环](./diagrams/canary-feedback-loop.html)。

### A8 · 用户提供任务结果，系统生成 UX 分

- 论文：根据 completion、corrections、attribution 按 5/3/2 公式计算。
- 当前源码：TaskAgent 阅读轨迹后直接给 1–10 整数。
- 交代清楚：当前产品没有人工星级评分按钮。

### A9 · 论文协议与当前实现采用不同晋升规则

- 论文：每侧至少 20 条；单侧 Welch 检验；`α=.05`。
- 当前源码：按均值比较；未分桶每侧 5 条；平分也晋升；14 天不足则丢弃。
- 画面：[论文与源码对照](./diagrams/canary-paper-vs-code.html)。

### A10 · 每个分数必须知道实际使用的 commit

- 核心句：一条有效评分需要对应 `(skill, side, sha)` 曝光记录。
- 当前缺口：Team 上传协议没有携带该记录，评分阶段重新计算 side。
- 画面：[Canary 归因缺口](./diagrams/canary-attribution-gap.html)。

### A11 · 旧实验走到了采集层

- 47 个唯一模型响应。
- Native 与统一会话副本均有 44 次工具调用，工具名称序列逐项一致。
- 完整流水线最终产生 0 个 Atom，编辑、安装和 Canary 未被观察。
- 画面：[实验 trace](./diagrams/experiment-cohort-trace.html)。

### A12 · 收束

- 源码已确认：per-skill Git、Main / Staging 生命周期、拒绝版本留档。
- 待验证：UX 分是否对应用户价值、曝光归因能否逐条对齐、晋升规则能否控制误晋升率。

---

## 方案 B：把一篇系统论文拆成四层证据

**中心论点**

XSkill 的论文协议、Wiki 产品口径、当前源码和本地运行结果处在不同阶段。按“提出、实现、观察”分层后，每项贡献的证据边界都能明确定位。

**适合听众**

Paper reading、系统研究者，以及重视可复现性与源码审计的听众。

**控制点**

主讲只保留四个关键差异；其余漂移放入附录，防止审计细节盖过系统思想。

### B1 · 同一个 XSkill，有四层事实

- Paper：目标协议与研究主张。
- Wiki：面向使用者的产品口径。
- Source：当前 commit 的实际执行路径。
- Run：本地实验真正走过的路径。
- 画面：Paper → Wiki → Source → Run 四层阶梯。

### B2 · 固定证据快照

- 论文标注 WIP，生成日期为 2026-05-30。
- 源码基线：`xskill@bc9bf94`。
- Wiki 基线：`xskill-wiki@bc7ac83`。
- 定义三种标签：设计主张、实现事实、运行观察。

### B3 · 用一条生命周期串起所有主张

- 主链：采集 → Atom → 路由 → 编辑 → 版本分发 → UX 评分 → 晋升或拒绝。
- 画面：[系统全图](./diagrams/xskill-system-map.html)。
- 后续每页只高亮一个控制点。

### B4 · 数据契约发生了变化

- 论文：统一 `TrajectoryEvent`，保留 native payload。
- 当前 Team 路径：服务端接收筛选、截断、脱敏后的 Markdown。
- 画面：[轨迹标准化图](./diagrams/trajectory-normalization.html)。

### B5 · 路由与阈值发生了变化

- 论文：embedding top-k 路由。
- 当前源码：扫描完整 catalog，并要求每个 Atom 以 1–10 落入至少一个 Skill。
- `θ=10` 只检查权重总和。
- 画面：候选漏斗；并列展示一次权重 10 与十次权重 1。

### B6 · UX 分的生成方式发生了变化

- 论文：可复算的 5/3/2 行为公式。
- 当前源码：LLM 根据定性 rubric 直接给 1–10 整数。
- 画面：公式与 TaskAgent prompt 两个控制点。

### B7 · Canary 的决策协议发生了变化

- 论文：`n≥20`、单侧 Welch、`α=.05`。
- 当前源码：`staging_mean ≥ main_mean`；同时存在 5/20 样本与 14 天规则。
- 画面：[论文与源码对照](./diagrams/canary-paper-vs-code.html)。

### B8 · 分流与记分缺少共同主键

- Manifest 决定实际 side。
- 评分器重新计算 side。
- 上传数据缺少 `(skill, side, sha)`。
- 画面：[归因缺口](./diagrams/canary-attribution-gap.html)。

### B9 · “支持生态”要拆成四项能力

- 论文列 5 个生态、Wiki 列 6 个、源码可探测 9 个。
- 分别检查采集、安装、分流、归因；“能发现”不能代表四项都完整。
- 画面：生态 × 四能力矩阵，只标完整、部分、缺失。

### B10 · 本地实验实际观察了哪一段

- 47 个唯一响应产生 44 / 44 工具序列。
- 完整流水线经过 39 次成功 LLM 调用、628,205 tokens，最终 0 Atom。
- Case 2-only 提交 3 次，任务级预期为 1 次。
- 画面：[实验 trace](./diagrams/experiment-cohort-trace.html)。

### B11 · 三类数字分别能支持什么

- 139 / 3.3×：单个重度用户存在 catalog 压力。
- README `+5.58 pp`：缺少对应 run artifact 的第一方报告。
- 论文：明确没有受控实证评估。
- 画面：库存快照、第一方 benchmark、受控实验三栏证据账本。

### B12 · 收束

- 源码可确认：Atom、per-skill Git、生命周期分层。
- 实验仍需回答：Canary 效果、评分标定、跨生态闭环。

---

## 方案 C：Canary 如何把真实任务变成 Skill 评估

**中心论点**

Canary 的核心数据链是“版本曝光 → 用户任务 → UX 分 → 分组比较 → 版本决策”。任何一环无法审计，晋升结果就缺少可靠归因。

**适合听众**

LLM eval、实验平台、ML Infra，以及熟悉 A/B test 的听众。

**控制点**

Canary 占主讲篇幅；轨迹蒸馏的代理细节放入附录。

### C1 · 一个 Skill 更新，凭什么获得全部流量

- 核心句：新版本先处理一部分日常任务，再用两组结果决定是否替换当前版本。
- 画面：Main 80% / Staging 20% 的单屏分流图。

### C2 · 定义一次在线评估的五个字段

- `Version`：一个 Skill 目录的 Git commit。
- `Main / Staging`：两个被比较的 side。
- `Exposure`：实际下发的 `(skill, side, sha)`。
- `Outcome`：任务轨迹中的完成、修正和撤销等行为。
- `UX score`：系统生成的 1–10 分。

### C3 · Git 状态机提供可回滚的实验对象

- 核心句：每个 Skill 独立成库，使 Main、Staging、被拒 commit 和回滚点都有稳定标识。
- 画面：[版本状态图](./diagrams/skill-version-state.html)。

### C4 · 分流发生在任务开始之前

- Team manifest 为用户选择 side 并下发 commit。
- 默认约 20% 用户获得 Staging，路由器尝试保持 sticky。
- 画面：manifest → install SHA → task 的短 sequence。

### C5 · 日常任务怎样形成反馈

- 用户执行真实任务。
- 轨迹进入 TaskAgent。
- 系统生成 UX 分。
- 分数写入 Main 或 Staging 样本组。
- 画面：[Canary 反馈闭环](./diagrams/canary-feedback-loop.html)。

### C6 · “用户打分”具体指什么

- 用户贡献完成、修正、撤销等任务结果。
- 论文按 5/3/2 公式计算。
- 当前源码让 LLM 读取轨迹并给等级。
- 当前没有人工评分界面。

### C7 · 何时宣布 Staging 获胜

- 论文规则控制统计显著性。
- 当前规则以样本均值决定；未分桶每侧 5 条即可比较；平分会晋升。
- 画面：[决策门对照](./diagrams/canary-paper-vs-code.html)。

### C8 · Canary 最关键的主键当前缺失

- Team 上传没有记录实际 exposure。
- 评分时重新计算 side。
- 管理员 override、小团队配额和服务重启都可能造成错记。
- 画面：[归因缺口](./diagrams/canary-attribution-gap.html)。

### C9 · 现有数字没有验证这条在线链路

- README 报告三项离线 benchmark 平均 `+5.58 pp`。
- 旧实验只观察到采集与切分尝试。
- 两组材料都没有产生 Main / Staging 的真实对照数据。
- 画面：离线 benchmark 与在线 Canary 两条分离轨道。

### C10 · 下一轮实验先校验数据，再比较算法

1. 固定客户端记录实际 `(skill, side, sha)`，与评分日志逐条 join。
2. 仿真当前均值规则、Welch 检验和按用户聚类检验的误晋升率。

画面：Attribution integrity → Decision quality 两阶段实验。

### C11 · Canary 主张成立需要三个条件

- 曝光可追溯：当前 Team 路径缺少记录。
- 评分可标定：当前为 LLM 直接评分。
- 决策可控：当前为均值门槛。
- 画面：三项检查表，逐项标当前状态。

### C12 · 收束

XSkill 提供了在线 Skill 演化的工程框架。它的下一条关键证据是一次可审计的 Main / Staging 因果比较。

---

## 合并建议

采用 **C1–C8** 建立主叙事，在 C3 前插入 **A2–A5** 解释 Atom、Candidate 与 Git 版本；用 **B10–B12** 收束证据边界。完整演讲控制在 20 页左右：

1. 问题与五个字段：2 页。
2. 轨迹到版本：4 页。
3. Canary 数据链：6 页。
4. 论文与源码差异：2 页。
5. 真实实验与下一轮：3 页。
6. 结论：1 页。
7. 附录：实现漂移、生态矩阵、审计修正。

返回当前 [slide deck](./)；查看 [重新分析笔记](./reanalysis.html)。
