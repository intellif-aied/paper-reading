---
layout: default
title: "XSkill 重新分析：论文、源码与官方文档三方核对"
permalink: /2026-08-28/skill/reanalysis.html
---

# XSkill 重新分析：论文、当前源码、官方文档三方核对

> 核心判断：XSkill 把技能定义成有状态、可分发、可回滚的团队资产，LLM 改写 `SKILL.md` 位于生命周期中的编辑阶段。论文给出完整的在线实验协议；当前代码实现了生命周期骨架，UX 评分和晋升规则采用另一套协议。下文分别标注论文设计、当前实现和后来新增的 README 评测。

## 0. 研究范围与证据快照

- 论文：完整阅读 `xskill/paper/xskill_v4.pdf`，19 页。PDF 元数据显示生成于 2026-05-30；论文首页明确标注 WIP，并声明尚无受控实证评估（论文 p. 1，Review flag）。
- 源码：`xskill` commit `bc9bf941662467ac711523e450968f2677cd230e`，2026-08-20。
- 官方文档：`xskill-wiki` commit `bc7ac834dd7186f4f3913bc573cc92c3fa141a40`，2026-08-25。该仓库 README 明确称它是产品站和文档站源码，`main` 会同步到线上（`xskill-wiki/README.md:1-7`）。对应页面是 [xskill.wiki/wiki.html](https://xskill.wiki/wiki.html)。当前环境无法直接抓取线上站点，因此本文以用户提供的官方文档仓库为准。
- 评测数字的仓库核查：对当前跟踪文件执行精确字符串检索。README 的 `88.57 / 84.33 / 60.47 / 77.79 / 72.21` 只出现在 `xskill/README.md`；仓库的 `scripts/bench/` 是轨迹切分 benchmark，不是 Spreadsheet、ALFWorld、OfficeQA 评测（`xskill/scripts/bench/README.md:1-7,66-106`）。
- 本文不把旧分享或旧实验当成事实来源；它们只适合与这份结论交叉检查。

## 1. XSkill 到底解决什么问题

论文的问题定义是清楚的：团队成员的 agent 会话里已经产生了大量可复用的程序性知识，但这些知识困在个人 session 中；即使有人手写了 skill，也缺少自动蒸馏、跨人分发、版本评估和退化处理机制（论文 p. 2，§1）。

XSkill 位于 Claude Code、Codex 等 runtime 的 skill loader 上层，管理“它们从磁盘读到什么”，并保留原生 loader。论文称之为 skill lifecycle management layer：创建、评估、演化、冻结/淘汰与运行时注入相互正交（论文 p. 3，§1；`xskill/README.md:30-42,72-80`）。

本文按“skill lifecycle manager”理解它，系统边界包括：

1. 从多个 agent 生态收集工作轨迹；
2. 把长会话切成单一意图的 AtomTask；
3. 把 Atom 路由到已有或新 skill，积累证据；
4. 用 LLM 生成/更新 `SKILL.md`；
5. 用 git 表示 `baby → main → staging`；
6. 把选定版本安装回各 runtime；
7. 用后续使用轨迹给版本打分并决定是否晋升。

论文将 2–4 称为三代理管线：TaskAgent、TaskClusterAgent、SkillEditAgent（论文 p. 1 Abstract，p. 6–8 §3.2）。当前源码仍以这三者为主干，但已经增加直接生成、描述触发优化、推荐、按模型分桶、jam breaker 等旁路；不能把论文图当作当前产品的完整调用图（`xskill/src/xskill/agents/generate_agent.py:1-20,45-67`；`xskill/src/xskill/config.py:145-186`）。

论文另一个重要观点来自相关工作调查：11 个被调查系统中没有 live-traffic skill-version A/B；并行工作 SkillClaw 有候选/当前版本比较，但发生在固定验证场景的离线环境；同时，各系统的 skill 正文最终都受 authoring LLM 能力上限约束（论文 p. 3–5，§2；p. 10–12，§4）。当前仓库提供了一份 2026-05-09 的 10 项目横向矩阵，支持“LLM-as-author 普遍存在”和“当时没有真正灰度”两点（`xskill/docs/research/related-work-survey.md:1-5,100-122,257-264`）；论文后来把 Trace2Skill 加为第 11 项，并把 SkillClaw 单列。该矩阵声称依赖的逐项目 `reports/01_hermes.md … reports/10_gepa.md` 没有跟踪在当前 checkout 中，本文也没有重新 clone 并审计全部上游仓库，因此这部分应表述为“论文作者的源码调查结论”，不能当作本次独立复核结果。

## 2. 当前源码实际执行的流程

### 2.1 轨迹收集：当前实现与论文的统一事件流不同

论文描述的是统一 `TrajectoryEvent`：`source/session_id/ts/kind/data`，并称 `data` 原样保留、可供 replay；五个生态是 Claude Code、Codex、OpenClaw、OpenCode、Hermes，watcher 每 30 秒扫描（论文 p. 5，§3.1；p. 17–18，Appendix C/E）。

当前源码的公共边界是标准化 Markdown 加 JSON sidecar，没有生成一组 `TrajectoryEvent`。`submit_trajectory` 把输入适配为 `traj_*.md` 和可选元数据 JSON（`xskill/src/xskill/ecosystems/_shared.py:660-730`）。转换是有损的：

- Claude Code 的 user、assistant、tool output 各自截到 2,000 字符，Markdown 中 tool input 截到 1,000 字符，thinking 被跳过（`xskill/src/xskill/ecosystems/claude_code.py:604-622,658-748,769-793`）。
- Codex 对 tool call/function output 不做深度解析，只留下 response-item 占位；user/assistant 文本截到 2,000 字符（`xskill/src/xskill/ecosystems/codex.py:197-214,247-297,325-355`）。
- team 上传协议只携带脱敏后的 Markdown、内容哈希、模型名和 harness，没有原生事件或 replay payload（`xskill/src/xskill/team/shared/protocol.py:52-65`）。

“原生 payload 原样保留用于 replay”属于论文设计。当前 team server 接收截断、筛选后的 Markdown；原始 session 仍可能留在客户端磁盘。

收集延迟也已改变：通用 watcher 的扫描周期是 5 秒，但 JSONL 默认要在源文件停止写入 120 秒后才 bridge（`xskill/src/xskill/config.py:202-205,225-240,593-637`；`xskill/src/xskill/ecosystems/_shared.py:928-975`）。team client 还要求 bridge 文件静默 180 秒，再要求脱敏内容哈希稳定 600 秒（`xskill/src/xskill/team/client/collector.py:83-105,210-217,319-338`）。对 team JSONL 会话，默认最早上传大约是最后一次源写入后的 15 分钟；论文写的是 30 秒级扫描。

### 2.2 TaskAgent：切分和 UX 打分是同一次 LLM 判断

论文的 UX 分数是可解释的行为公式：

`s(a) = 5 × completion + 3 × (1 - min(corrections / 3, 1)) + 2 × attribution`

其中 completion、corrections、attribution 都声称来自可观察行为，得分范围 0–10；论文还特意将它与 LLM-as-judge 区分（论文 p. 6，Eq. 1；p. 8，§3.3）。

当前实现没有计算这个公式。TaskAgent 的 system prompt 让 LLM 在切 Atom 时直接提交 `used_skills` 和 1–10 整数 `ux_score`，依据一张定性分档表打分（`xskill/src/xskill/agents/task_agent.py:76-164`）。工具只校验它是不是 1–10 的整数，然后原样存入 Atom（`xskill/src/xskill/agents/agent_tools.py:1568-1613`；`xskill/src/xskill/agents/task_agent.py:381-417`）。后续 canary 直接使用这个值，并把原因写成 `TaskAgent split ux_score`（`xskill/src/xskill/pipeline/runner.py:2780-2847`）。

当前 UX 信号是“LLM 阅读轨迹后给出的整体等级”。论文 Eq. 1 的确定性行为特征组合没有执行。当前评分仍参考用户修正、撤销和结果，但其可复算性、标定方式和偏差来源已经不同。

TaskAgent 先看每个 `## User` 回合的行号与首句摘要，需要时再用 `look` 工具读取局部 assistant 原文（`xskill/src/xskill/agents/task_agent.py:81-126`），以此避免一次读入整条轨迹。这个上下文节流设计比论文“处理 normalized trajectory”的描述更具体。

### 2.3 TaskClusterAgent：全目录路由，而且弱 Atom 不能丢

论文称 TCA 通过 embedding similarity 取得 top-k 现有 skills，然后在 ROUTE、CREATE、RECLASSIFY 三种动作中选择，weightscore 范围是 0–10（论文 p. 7，§3.2.2）。

当前源码不做这一步 top-k：它扫描 `skill_dir` 下所有 skill，把所有名字以及在 25,000 字符预算内能容纳的描述一起放进 prompt；预算不够时只保留名字（`xskill/src/xskill/agents/task_cluster_agent.py:36-138,253-262`）。

当前 prompt 还强制每个 Atom 至少写入一个 skill：完全不相关也要挑“最不远”的 skill，以 weightscore=1 落盘；只有分数至少 7 才允许新建 skill（`xskill/src/xskill/agents/task_cluster_agent.py:141-160,180-235`）。工具实际只接受 1–10，不接受 0 或跳过（`xskill/src/xskill/agents/agent_tools.py:1353-1399`）。

官方文档却写成“通常 0–10，弱材料可以跳过”（[生命周期文档](https://xskill.wiki/wiki.html#lifecycle)；`xskill-wiki/i18n.js:340-347,749-756`）。这里是明确的文档/源码冲突。当前强制兜底会留下 provenance，但也会把无关 Atom 以低分污染某个候选池。

### 2.4 累计阈值：10 分不保证“至少 2–3 条轨迹”

论文称阈值 `θ=10` 是经验选择，通常需要 2–3 条不同轨迹的中等相关证据，因此不会由单次交互产生 skill（论文 p. 8，§3.2.3）。

源码只对当前 `.candidates.yml` 中的 weightscore 求和；总和达到 10 就返回全部候选，没有“不同 trajectory 数量”门槛（`xskill/src/xskill/skill/candidates.py:40-49,287-345`）。同一 atom 对同一 skill 重复写入是覆盖而非累加，但一个 atom 可以直接得 10 并立即触发（`xskill/src/xskill/agents/agent_tools.py:1359-1396`；`xskill/src/xskill/skill/candidates.py:246-284`）。

因此下面两种情况都能过门：

- 一个高分 Atom：`10`；
- 十个被强制兜底的无关 Atom：`1 × 10`。

阈值的实际语义是“LLM 给出的相关性权重和达到 10”；源码没有独立证据条数约束。

### 2.5 SkillEdit 与 git 生命周期：工程骨架成立，但有绕过灰度的入口

当前源码确实把每个 skill 做成独立 git 仓库，并以分支作为状态单一事实源：新目录从 `baby` 开始，首版毕业到 `main`，更新写入 `staging`（`xskill/src/xskill/skill/git.py:1-24`）。SkillEditAgent 读取 Atom、轨迹和现有文件，要求提炼通用步骤而非复述对话；baby 做 checkpoint 并在候选清空后毕业，main 更新进入 staging（`xskill/src/xskill/agents/skill_edit_agent.py:187-205,274-305`）。这是论文设计中实现最扎实的一部分。

但“所有更新都必须经过 canary”已经不成立：

- `xskill generate` 是用户显式触发的快路径，可新建或改写并直接提交 `main`，永不打开 staging（`xskill/src/xskill/agents/generate_agent.py:1-20,45-67`；`xskill/src/xskill/agents/agent_tools.py:1082-1108`；[generate 文档](https://xskill.wiki/wiki.html#features)，`xskill-wiki/docs/features/generate.md:31-40`）。
- staging 长期未收敛且候选堆积时，jam breaker 会把 main、staging 和新候选直接合并成新 main，明确“不走灰度”（`xskill/src/xskill/agents/skill_edit_agent.py:166-184`；默认 jam threshold 50，`xskill/src/xskill/config.py:156-164`）。

这两个旁路有明确的产品理由，但演讲时应说“默认自动更新走灰度”，不能说“任何新版本都必须在真实流量上证明自己”。

### 2.6 Canary：论文的统计检验没有落地

论文 Algorithm 2 规定：每侧至少 20 个样本，单侧 Welch t-test，`α=0.05`，staging 只有显著更优才晋升；正文还提出 30 天 timeout 和低流量时 `Δmin=0.5` 的均值 fallback（论文 p. 8–9，§3.3/Algorithm 2；p. 14，§5.4）。论文自己也承认 per-atom 非独立、optional stopping、多重比较、离散分数和低流量 fallback 都不够严谨（论文 p. 9，Threats to statistical validity）。

当前代码没有 t-test、p-value 或 effect-size gate。规则是：

- 流量：默认 20% 去 staging（`xskill/src/xskill/config.py:145-164`；`xskill/src/xskill/canary.py:489-517`）。
- 未分桶路径：每侧最近 5 个有效样本；按模型分桶路径：top-2 模型中每侧总计 20 个样本，按模型占比加权均值（`xskill/src/xskill/canary.py:48-85,925-1008`）。
- 决策：`staging_avg >= main_avg` 直接晋升，否则拒绝；平分也晋升（`xskill/src/xskill/canary.py:1010-1024`）。
- 超时：当前源码在 14 天样本不足时直接丢弃 staging；论文写 30 天，并在部分段落提出 `Δmin=0.5` fallback（`xskill/src/xskill/canary.py:13-20,996-1002`）。
- 被拒版本的 commit 会先挂到 `refs/rejected/...`，再删除 staging 分支，仍可审计（`xskill/src/xskill/canary.py:416-459`）。

官方文档当前描述的正是源码规则：20% 流量、未分桶每侧 5 条、均值不低于 main 即晋升、14 天不足则丢弃（[生命周期文档](https://xskill.wiki/wiki.html#lifecycle)；`xskill-wiki/i18n.js:356-367,765-776`）。Wiki 与源码一致；论文协议与产品协议已经分叉。

论文内部也有一处未统一：Algorithm 2 的 timeout 是 freeze/retain，§3.3 和 §5.4 又说低流量时做 `Δmin=0.5` 的均值 fallback。当前源码/文档选择了第三种行为：14 天直接 reject/discard，但用 git ref 保留 commit。

### 2.7 最严重的实现风险：实际使用版本与记分版本可能不一致

team server 在分发 manifest 时，对普通用户用有状态 `CanaryRouter.assign` 保持 sticky、修正小团队配额；管理员还可覆盖某用户的 side（`xskill/src/xskill/team/server/skill_manifest.py:1-16,243-307,390-426`）。

但是上传协议没有携带“这条轨迹实际装了哪些 skill、每个 skill 的 side 和 commit SHA”，只上传 Markdown、hash、model、harness（`xskill/src/xskill/team/shared/protocol.py:52-65`）。服务端打分时又根据 TaskAgent 推断出的 `atom.used_skills`，用另一个无状态 `pick_side_scoped`/补漏逻辑重新计算 side（`xskill/src/xskill/pipeline/runner.py:2875-2998`）。这条重算路径既不查询有状态 router 的既有 assignment，也不读取 manifest 的管理员 side override。

所以在以下情形中，UX 分可能被记到与用户实际拿到的版本不同的一侧：

- 小基数团队由有状态 router 做过配额平衡；
- 管理员手工锁定了某用户的 side；
- server 重启后 router 内存账本清空（源码明确认为可接受，`xskill/src/xskill/team/server/skill_manifest.py:43-48`）；
- TaskAgent 漏报或误报 `used_skills`。

这是源码静态审计得到的正确性风险，论文尚未验证。最小修复方向是让 manifest assignment 形成持久、可回传的 exposure record，并以实际 exposure 的 `(skill, side, sha)` 做归因；评分时不再重算。

standalone 还有一个覆盖面问题：本地 scorer 要求轨迹头包含 `<!-- xskill:skill=X side=Y sha=Z -->`，否则直接不记分（`xskill/src/xskill/pipeline/runner.py:2780-2824`）。当前源码中只有 Claude Code adapter 明确注入这个 header（`xskill/src/xskill/ecosystems/claude_code.py:473-500,866-892`）；OpenClaw 的 flip hook 只切换安装和写 install history，没有给轨迹注入 header（`xskill/src/xskill/ecosystems/openclaw.py:184-240`）。因此“跨 runtime 的 live canary 反馈闭环”在当前实现里不能仅凭支持列表视为已经打通。

## 3. 五生态、六生态还是九生态

三份一手材料对应三个不同阶段：

| 材料 | 声称/实现的生态 | 证据 |
|---|---|---|
| 论文 | Claude Code、Codex、OpenClaw、OpenCode、Hermes | 论文 p. 5 Fig. 3；p. 18 Appendix E/F |
| 官方 wiki | Claude Code、Codex、OpenCode、OpenClaw、Cursor、Trae，共六个 | `xskill-wiki/index.html:182-209`；`xskill-wiki/i18n.js:470-495` |
| 当前自动探测源码 | Claude、Codex、OpenCode、ngagent、nga3、OpenClaw、Cursor、DSH，再加 Trae，共九个 | `xskill/src/xskill/ecosystems/_shared.py:145-212,261-267`；`xskill/src/xskill/ecosystems/__init__.py:1-17,153-178` |

论文中的 Hermes 路径 `<cwd>/trajectory_samples.jsonl` 也已被源码仓自己的调研文档纠正：那是 RL batch runner 的行为，普通 Hermes session 实际双写 `~/.hermes/state.db` 和 `~/.hermes/sessions/*.jsonl`（`xskill/docs/ecosystem/hermes.md:7-15`）。当前 `xskill.ecosystems` 没有 Hermes adapter；源码目录的 catalog 也标为未支持（`xskill/docs/ecosystem/CATALOG.md:92`）。

另一个容易被支持矩阵掩盖的问题是 standalone 与 team collector 的覆盖面不同。standalone watcher factory 有 OpenClaw 和 Cursor 分支（`xskill/src/xskill/pipeline/watcher_factory.py:249-312`），但 team `TeamCollector.start_ingesters` 只分派 Claude、Codex、nga3、OpenCode、ngagent、Trae、DSH；OpenClaw 和 Cursor 会落入 `else: continue`（`xskill/src/xskill/team/client/collector.py:130-183`）。所以 wiki 所说六个生态都可“加入同一份团队库”（`xskill-wiki/i18n.js:510-523`）对当前自动 team 收集并不完全成立。

## 4. 隐私边界：论文比产品文案更诚实

team client 的确会在上传前脱敏：复用 `detect-secrets` 的凭证正则，再补 PEM、`sk-` token、关键词和大写环境变量赋值规则（`xskill/src/xskill/team/client/redact.py:1-38,48-122`）。但它有意排除公网 IP 和高熵检测，以减少误报（同文件 `:19-25,72-95`）。服务端只做 hash 完整性校验和控制字符清洗，没有第二层 PII/源码内容过滤（`xskill/src/xskill/team/server/api.py:684-726`）。

因此下面两种表述不等价：

- README：“密钥密码和相关隐私不会被别人看到”（`xskill/README.md:36-42`）；
- 论文：正则不会移除 PII、内部 hostname/path 或 proprietary code，server operator 是信任边界（论文 p. 14–15，§6.2）。

论文的限制说明与源码更一致。官方 generate 文档也要求用户不要放入 token、私有内网地址、证件号码等敏感信息（`xskill-wiki/docs/features/generate.md:59-61`）；当前脱敏不构成完整的数据防泄漏方案。

## 5. 论文与后来数据：哪些数字能讲，应该怎么讲

| 数字/结论 | 证据性质 | 分享时应如何表述 |
|---|---|---|
| 139 个 skill；87 personal、52 plugin；listing 26,448 chars；body 699,291 chars；8,000-char listing budget；3.3× overflow；19 个禁用模型调用 | 论文称来自一台真实 Claude Code power-user 安装；当前仓库没有原始 inventory 文件 | “作者报告的单用户真实库存快照”；它不属于可复现实验，也不代表用户总体分布。论文 p. 12，Table 5/§5.1 |
| FastAPI 生命周期中的 6.8 vs 7.4、`p=0.038` | 明确构造的 worked example | 只能解释机制，不能证明效果。论文 p. 13，§5.2 |
| 30 tok/s、TaskAgent 1–3 分钟、4 人团队 1–2 周收敛 | 明确是估算 | 用于容量推演，不当作 benchmark。论文 p. 13–14，§5.3–5.4 |
| 论文整体效果 | 首页明确“no controlled empirical evaluation yet” | 论文属于系统设计/WIP，尚未提供受控效果评估。论文 p. 1 Review flag；p. 12 §5 |
| XSkill vs SkillOpt：Spreadsheet 88.57 vs 87.86；ALFWorld 84.33 vs 77.61；OfficeQA 60.47 vs 51.16；均值 77.79 vs 72.21 | 论文之后新增的 README 第一方结果；当前跟踪文件没有 run ID、逐题结果或这三项评测脚本 | 可以讲“当前 README 报告 +5.58pp”，但必须标注“仓库快照内不可独立复算”。`xskill/README.md:82-143` |

两处数字错误值得在讲稿里直接修正：

1. README 的 OfficeQA delta 写 `+9.30`，但 `60.47 - 51.16 = 9.31`（`xskill/README.md:103-123`）。总均值差 `+5.58` 是对的。
2. 论文结论把相同库存写成“139 skills, 13× budget overflow”，与摘要、Table 5 和 §5.1 的 `3.3×` 冲突；`26,448 / 8,000 ≈ 3.31`，所以 `13×` 是错误（论文 p. 1 Abstract；p. 12 Table 5；p. 15 Conclusion）。

139-skill 数据只证明“catalog budget 压力存在于一个重度用户样本”。它没有证明 XSkill 的推荐、冻结或 canary 实际提高了任务成功率；论文自己也没有测这个因果链。

## 6. 论文、wiki、源码差异总表

| 议题 | 论文 v4 | 官方 wiki 当前口径 | 当前源码 |
|---|---|---|---|
| UX score | Eq. 1，5/3/2 行为公式，0–10 | 交互本身得到 1–10 分 | TaskAgent LLM 直接按分档表给 1–10；无 Eq. 1 |
| TCA 候选检索 | embedding top-k | “与已有 Skills 比较” | 扫全部 skill；25k 字符预算，不做 top-k |
| weightscore | 0–10 | 通常 0–10；弱材料可跳过 | 只收 1–10；每个 Atom 必须落至少一个 skill |
| `θ=10` 含义 | 经验上要求 2–3 条不同轨迹 | 有用材料累计 10 | 纯求和；单 Atom=10 或十个弱 Atom=1 都能过 |
| Canary 样本 | 每侧 20 | 未分桶每侧 5；team 可不同 | 未分桶 5；模型分桶每侧总计 20 |
| Canary 决策 | 单侧 Welch t-test，`α=.05`，显著更优才晋升 | staging 均值不低于 main 即晋升 | 简单/加权均值；`>=` 晋升；无显著性/效应量门槛 |
| Timeout | 30 天；论文不同段落在 freeze 与 `Δ=.5` fallback 间不一致 | 14 天不足即丢弃 | 14 天 discard；被拒 commit 另存 git ref |
| 轨迹标准化 | 统一 `TrajectoryEvent`，native data 原样保留 | 脱敏 session 记录 | 截断、筛选后的 Markdown + sidecar；team 不上传 native payload |
| 收集速度 | 每 30 秒扫描 | 未给明确端到端时间 | watcher 5 秒；JSONL 120 秒 settle；team 再加 180+600 秒门槛 |
| 生态 | 五个，含 Hermes | 六个，不含 Hermes | 九个可探测；无 Hermes；team 自动 ingester 又少 OpenClaw/Cursor |
| 更新是否都灰度 | main 更新进入 staging | generate 明确直接 main | generate 和 jam breaker 都可绕过 staging |
| 隐私 | 明确承认 PII/IP/代码未清理 | 营销页称 privacy built-in；feature 文档另有警告 | 凭证正则脱敏；公网 IP/高熵内容故意不扫，服务端不做内容级过滤 |
| 版本状态 | 2026-05-30 WIP preprint | 首页仍写“截至 v0.6.2” | README news 已到 v0.6.31（`xskill/README.md:305-322`） |

官方文档还有两个直接可验证的维护问题：

- wiki quickstart 写“LLM 和 embedding 可共用 DeepSeek endpoint”，示例模型 `deepseek-embedding`（`xskill-wiki/wiki.html:140-152`）；当前源码配置模板明确写 DeepSeek 不提供 embeddings API，并推荐 DashScope/OpenAI/Ollama/Jina（`xskill/src/xskill/config.py:112-124`）。
- 生命周期页面链接到 `docs/features/lifecycle.md`（`xskill-wiki/i18n.js:368,777`），但当前 `xskill-wiki/docs/features/` 下没有该文件。

## 7. 对论文贡献的判断

### 成立且值得讲的部分

1. **问题定义成立。** skill 数量增加后，供给、版本、分发和淘汰都会成为约束；写作只是第一步。
2. **“管理层而非新 runtime”是合理系统边界。** 复用各 agent 的原生 `SKILL.md` loader，降低接入成本（论文 p. 3 §1；p. 9 §3.4）。
3. **Atom 是有用的中间表示。** 它把一条多主题 session 变成可独立路由、溯源和打分的单位；当前 Atom 仍保存行号范围、上下文、原片段和邻接关系（`xskill/src/xskill/pipeline/atom.py:33-66`）。
4. **per-skill git 状态机是实用工程选择。** diff、rollback、拒绝版本留档都能用现成 git 语义实现。
5. **论文主动公开局限。** 首页 review flag 与 §6 没有把 hypothetical example、估算吞吐或统计缺陷包装成实验结论。

### 尚未被证据支持的部分

1. XSkill 生成的 skill 是否稳定优于人工 skill、SkillOpt 或不使用 skill；
2. 1–10 UX 分是否与真实用户满意度或任务正确率校准；
3. 当前均值 canary 是否能控制误晋升率；
4. team 模式的实际 exposure 是否被准确归因到 main/staging；
5. 多生态“支持”是否意味着收集、安装、归因、反馈四段都端到端可用；
6. 脱敏后轨迹是否满足企业的 PII、源码和内网信息合规要求。

## 8. 如果重新做实验，优先验证这四件事

1. **Exposure attribution 一致性。** 固定一批 client，记录 manifest 实际下发的 `(skill, side, sha)`；完成轨迹后比较 `.ux_scores.jsonl` 的 side/SHA。覆盖 stateful router、side override、server restart。这个实验能直接验证当前 canary 数据是否可信。
2. **Canary 误晋升仿真。** 在 main/staging 同分布、staging 略差、staging 略优三种情形下，模拟 1–10 离散且按 user/session 聚类的 UX 分。比较当前 `5/20 样本 + 均值 >=`、论文 Welch test、按用户 permutation/sequential test 的误报率与时延。
3. **UX score 标定。** 随机抽取 Atom，由两名人工独立标 completion、correction、满意度与 skill 是否实际使用；比较人工、论文公式和当前 TaskAgent 评分的一致性。不要只测相关系数，还要测 main/staging 排序是否翻转。
4. **候选池污染。** 构造一组与现有 catalog 无关的 Atom，观察强制 weight=1 路由后要多少条会误触发 SkillEdit；再与允许 `SKIP/0`、加 distinct-trajectory 门槛的版本比较。

## 9. 可直接用于演讲的 11 个结论

1. XSkill 给 `SKILL.md` 补上团队级生命周期：采集、证据累积、版本、分发、反馈和回滚；自动写作位于其中一环。
2. 论文是 WIP 系统设计，尚无受控效果评估；FastAPI 数字、吞吐和收敛时间都明确是构造或估算。
3. 论文把 live-traffic behavioral-UX canary 列为核心创新主张；当前源码没有实现论文的行为公式和 Welch 检验。
4. 当前 UX 分由 TaskAgent LLM 在切分时直接给出，属于 LLM-mediated judgment；Eq. 1 的可复算 5/3/2 公式没有执行。
5. 当前 canary 的晋升条件是“均值不低于 main”，平分也晋升；未分桶只需每侧 5 条，无法支撑“统计显著更优”的表述。
6. exposure attribution 是当前最直接的正确性风险：分发时的 side 与评分时重算的 side 可能不一致。
7. `θ=10` 没有独立证据条数约束；一个 10 分 Atom 或十个被迫归类的 1 分 Atom 都能触发 SkillEdit。
8. 论文的统一原生事件流没有落到当前 team 协议；服务端拿到的是截断、筛选、脱敏后的 Markdown。
9. “支持某生态”必须拆成采集、安装、灰度分流、结果归因四项。论文五个、wiki 六个、源码探测九个，且 team collector 仍漏 OpenClaw/Cursor。
10. 139 skills 与 3.3× listing overflow 是一个真实单机库存快照，能说明问题存在，不能说明 XSkill 已经解决了问题；论文结论的 `13×` 是笔误。
11. README 后来报告三项 benchmark 平均领先 SkillOpt 5.58pp，但当前仓库没有对应 run artifact；演讲中应标成“项目 README 报告值”，不要包装成已经复现的结果。

## 10. 一手来源索引

- 论文：`xskill/paper/xskill_v4.pdf`，重点为 p. 1 Review flag，p. 5–9 §3，p. 12–15 §5–7，p. 17–19 Appendices。
- 当前源码主干：
  - 轨迹生态与转换：`xskill/src/xskill/ecosystems/`
  - TaskAgent/TCA/SEA：`xskill/src/xskill/agents/task_agent.py`、`task_cluster_agent.py`、`skill_edit_agent.py`
  - 候选池：`xskill/src/xskill/skill/candidates.py`
  - Canary：`xskill/src/xskill/canary.py`
  - team 分发与上传：`xskill/src/xskill/team/server/skill_manifest.py`、`team/client/collector.py`、`team/shared/protocol.py`
- 官方文档：
  - [主文档](https://xskill.wiki/wiki.html)
  - 本地页面源：`xskill-wiki/wiki.html`、`xskill-wiki/i18n.js`
  - 命令细节：`xskill-wiki/docs/features/`
- 后来新增的项目评测口径：`xskill/README.md:82-143`。
