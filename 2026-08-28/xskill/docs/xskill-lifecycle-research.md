# xskill 生命周期源码调查：从 harness 注入到经验回流

> 研究问题：xskill 如何进入 Codex、Claude Code 等 coding-agent harness，何时采集会话，如何把会话提取成 Skill，又如何把 Skill 注入回 harness。
>
> 证据边界：本文只引用本仓库当前源码和配置，不把 README、设计文档或 harness 外部文档当作实现证据。现有静态分析位于仓库根目录 `walkthrough.md`；其标题和范围见 `walkthrough.md:1-8`。本文没有修改该文件。

## 结论先行

xskill 的主链不是“向 Codex/Claude 进程打 hook”，而是两组文件协议组成的闭环：

1. **向 harness 写 Skill 目录**：把每个 Skill 的完整目录安装到 harness 已知的 Skill discovery root。Claude Code 的目标是 `~/.claude/skills/<name>`，Codex 的目标是 `~/.agents/skills/<name>`（`src/xskill/ecosystems/claude_code.py:60-74,82-124`；`src/xskill/ecosystems/codex.py:39-50,58-93`）。安装优先使用符号链接，其次 Windows junction，最后才复制；链接模式让源 Skill 更新后目标立即可见（`src/xskill/ecosystems/_fallback.py:1-54,222-316`）。
2. **从 harness 读原生会话文件**：Claude Code 的源目录是 `~/.claude/projects`，Codex 是 `~/.codex/sessions`；xskill 轮询这些目录，把原生 JSONL 适配成统一的 Markdown 轨迹和 JSON sidecar（`src/xskill/ecosystems/_shared.py:145-212,230-267`）。
3. **把轨迹逐级提炼为 Skill**：统一轨迹先按用户意图切成 AtomTask，再聚类进各 Skill 的候选缓冲区；候选权重达到阈值后，SkillEditAgent 读取原始证据并更新 `SKILL.md`，产物保存在每个 Skill 自己的 Git 仓库中（`src/xskill/agents/task_agent.py:76-164,351-435`；`src/xskill/agents/task_cluster_agent.py:141-249`；`src/xskill/skill/candidates.py:287-345`；`src/xskill/agents/skill_edit_agent.py:187-286`）。
4. **再安装回 harness**：standalone 在 SkillEdit 成功后立即安装到当前检测到的生态；team 模式则由服务端给每个客户端下发 side/SHA manifest 和 Git bundle，客户端 checkout 本地 working copy 后再安装进本机 harness（`src/xskill/pipeline/runner.py:811-849,1010-1054`；`src/xskill/team/server/api.py:777-899`；`src/xskill/team/client/daemon.py:194-243`）。

因此，xskill 这一侧与 harness 的集成边界止于“把目录放到 discovery root、从会话目录读文件”。仓库源码没有实现 Codex/Claude 内部何时扫描或如何把 `SKILL.md` 拼进模型上下文；那是 harness 自身的加载行为，不应由本仓库源码外推。

## 生命周期总览

```text
xskill init / connect
        │
        ├─ 检测本机 harness ──> 安装 Skill 目录到 discovery root
        │                         Claude: ~/.claude/skills
        │                         Codex:  ~/.agents/skills
        │                                      │
        │                               harness 执行用户任务
        │                                      │
        └─ watcher/collector <── 原生 JSONL 会话目录
                                  │
                       settle / quiet / hash-stable
                                  │
                     适配、脱敏、统一 Markdown + JSON
                                  │
                 TaskAgent 切 Atom ──> 聚类/候选权重
                                  │
                       SkillEditAgent 更新 Git Skill
                                  │
                   standalone 即装 / team manifest+bundle
                                  │
                         再进入 discovery root
                                  │
                      下一轮会话和效果归因/评分
```

## 0. 先建立 xskill 的领域词汇

这套系统最容易被误读的地方，是把“文件处理阶段”当成同义词。完整规范词汇表见 [`CONTEXT.md`](../CONTEXT.md)；源码中的最小领域模型是：

```text
Ecosystem
  └─ Native Session → Bridge → Trajectory → Atom
                                     └─ Candidate + Weightscore
                                                └─ Skill Edit
                                                     ├─ Baby → Main
                                                     └─ Main → Staging
                                                          └─ Side/Session Assignment
                                                               └─ UX Score → Canary
                                                                    └─ Manifest/Reconcile → harness
```

| 概念 | 定义与生命周期位置 | 必须避免的混淆 |
|---|---|---|
| **Ecosystem** | 一个 harness 的源会话、adapter 和安装规则集合 | 不是 LLM provider |
| **Native Session** | harness 自己拥有的原始交互记录 | 不是已标准化 Trajectory |
| **Bridge** | Native Session 到统一 Trajectory 的标准化边界 | 不是简单 copy/upload |
| **Watch Directory** | registry 注册的 Trajectory 流，带来源与处理策略 | 不是任意文件夹 |
| **Trajectory** | 一次 agent session 的完整、标准化学习证据 | 不是单一用户任务 |
| **Atom / AtomTask** | Trajectory 中按用户意图切出的最小连续学习单元，可跨多轮 | 不是 token、固定 chunk 或单条 turn |
| **Candidate** | Atom 支持但尚未写入 Skill 正文的模式证据 | 不是 staging 版本 |
| **Weightscore** | Atom→Skill 路由的证据权重，累计后触发 Skill Edit | 不是 UX Score |
| **Atom Adoption** | Atom 已向某个 Skill 贡献证据的耐久事实 | 不代表 Skill 已生成或安装 |
| **Baby / Main / Staging** | 首个版本组装态 / 稳定分发态 / 既有 Skill 更新的灰度态 | Baby 不是 Staging；Candidate 也不是 Staging |
| **Side + Session Assignment** | session 实际使用的 main/staging 侧与精确 SHA | 不能用事后当前 checkout 反推 |
| **UX Score** | 精确 Skill 版本在一个 Atom 上的真实交互质量分 | 不决定 Atom 属于哪个 Skill |
| **Canary** | 真实流量下比较 main/staging、晋升或拒绝候选版本的闭环 | 不是离线单次 LLM 评价 |
| **Jam** | staging 过老、反馈平台期、候选积压三条件同时成立时的强制收敛 | 不是普通 promotion 快捷方式 |
| **User Staging** | team client 的专家手改隔离分支 | 不是 Canary Staging，也不能直写 Main |
| **Manifest / Reconcile** | team server 声明个性化 Skill side/SHA 目标，client 对齐本地并安装 | 不是 server 直接修改远端 harness |
| **Native Skill / SkillHub Skill** | 前者进入 Git/Candidate/Canary 演进；后者是可检索三方包 | 两者不共享完整版本生命周期 |

这组区分也给出了故障定位语法。例如“Trajectory 已入库但 Atom 为 0”表示失败在 TaskAgent split；这时 embedding、Candidate、SkillEdit、回注和 Canary 都尚未发生，不能用“模型接口可连通”替代下游成功证据。

## 1. 启动：先把 xskill 的使用入口放进 harness

`xskill init` 会找到随包分发的 `xskill/data/skill/xskill`，实时检测本机生态，并对每个生态调用对应 installer；也可以用 `--no-skill` 跳过。完成后 CLI 提示用户在 agent 中输入 `/xskill`（`src/xskill/cli.py:179-235`）。这个内置 Skill 自己说明 xskill 是“thin client + daemon”，负责挂载共享 Skill、采集和同步轨迹（`src/xskill/data/skill/xskill/SKILL.md:1-17`）。

`xskill connect` 是 team 生命周期的另一入口：首次执行握手并持久化 `ClientState`，随后默认安装并启动后台服务；foreground 模式则直接运行循环（`src/xskill/cli.py:295-344,347-424`）。它建立的是客户端与团队服务的同步关系，不能与 `init` 安装内置 `/xskill` Skill 混为一谈。

## 2. 注入：目录安装，而不是进程 hook

生态检测器以用户 home 下的原生会话路径作为存在性探针，并同时确定 xskill bridge：Claude Code 对应 `.claude/projects` / `.xskill/cc_sessions`，Codex 对应 `.codex/sessions` / `.xskill/codex_sessions`。检测在安装时实时执行，因此用户中途安装一个新 harness 也能被发现（`src/xskill/ecosystems/_shared.py:145-212,230-267`；`src/xskill/pipeline/runner.py:1015-1054`）。

| harness | 原生轨迹源 | Skill 安装目标 | xskill 侧的集成方式 |
|---|---|---|---|
| Claude Code | `~/.claude/projects` | `~/.claude/skills/<skill>` | 完整目录 symlink/junction/copy |
| Codex | `~/.codex/sessions` | `~/.agents/skills/<skill>` | 完整目录 symlink/junction/copy；源码特别注明不是 `.codex` |

目标 Skill 可以选择稳定的 `SKILL.md`（main）或 `.canary/SKILL.md`（staging）；共享 installer 会校验源、处理既有目标并执行安装（`src/xskill/ecosystems/_shared.py:275-382`）。优先链接意味着 harness 看到的是 xskill working copy 的实时视图；退化到 copy 时只是安装时快照（`src/xskill/ecosystems/_fallback.py:1-54,222-316`）。

## 3. 采集：何时认为一份会话可以进入流水线

### 3.1 Standalone

API 启动时会拉起持久 `agent-worker`；standalone 还单独拉起 Claude Code ecosystem-ingest worker，关闭时统一停止这些 worker（`src/xskill/api/app.py:1208-1233,1248-1307`）。agent worker 构建 DirectoryWatcher，并在每次 poll 里运行生态发现/摄取；Claude Code 由专门的高频 ingester 处理，其他生态由 watcher poll 的 one-shot ingestion 处理（`src/xskill/_workers.py:48-127,176-301`；`src/xskill/pipeline/watcher_factory.py:1-6,97-214`）。

默认 watcher poll 是 5 秒；原生 JSONL 的默认 `settle_seconds` 是 120 秒（`src/xskill/config.py:202-239,593-637`）。`JsonlIngester` 只有在源文件 mtime 至少静默这么久后才 bridge；若已处理文件后来继续增长，会重建 bridge 并重置下游状态重新处理（`src/xskill/ecosystems/_shared.py:891-939,951-1056`）。所以默认语义是**最后一次文件修改后约 120 秒，再叠加轮询延迟**，而不是收到某个“会话结束”回调。

Claude 适配器定义了 `last-prompt` 完成谓词，但通用 ingester 仅在 `settle_seconds <= 0` 时强制使用完成谓词；默认 120 秒配置依赖静默窗口（`src/xskill/ecosystems/claude_code.py:567-596`；`src/xskill/ecosystems/_shared.py:1002-1009`）。Codex spec 没有完成谓词，同样依赖 settle（`src/xskill/ecosystems/codex.py:160-171`）。

### 3.2 Team client

Team 客户端的 native ingester 先把各生态原始文件镜像进标准 bridge；随后 collector 还施加两道上传门槛：bridge 文件 mtime 静默至少 180 秒，并且内容 hash 稳定至少 600 秒（`src/xskill/team/client/collector.py:1-16,83-123,130-184,210-268,270-341`）。native ingester 未覆盖 settle 参数，因而还继承默认 120 秒。按默认值，一份不再变化的原生轨迹最早大约经过 `120 + 180 + 600 = 900` 秒，再叠加 ingester/client poll 延迟，才会上传；这是三段顺序门槛，不是“会话一结束立刻上传”。

上传前，collector 会把 PEM、内置 secret detector 命中、`sk-`、敏感关键字和环境变量赋值替换为 `[REDACTED]`（`src/xskill/team/client/redact.py:98-122`）。协议明确 content 已经在客户端脱敏，sidecar 只携带 model/harness（`src/xskill/team/shared/protocol.py:52-65`）。服务端验证 ID 与 SHA 后先原子写 sidecar，再写 Markdown，随后把客户端目录注册给 watcher（`src/xskill/team/server/api.py:325-360,684-726`；`src/xskill/api/app.py:1099-1138`）。

TeamClient 默认每 30 秒执行一次 tick，顺序是上传轨迹、同步 manifest/bundle、对齐 side、处理显式下载、推送本地编辑及清理（`src/xskill/team/client/daemon.py:100-136,612-655`）。

## 4. 适配：从 harness 方言到统一轨迹

通用 `adapt_trajectory` 按 `TrajectorySpec.format` 分派到 Claude/Codex adapter；`submit_trajectory` 随后执行 sanitizer 和配置中的 mask patterns，并写出 `.md` 与 `.json`（`src/xskill/ecosystems/_shared.py:529-601,674-730`）。

- **Claude Code**：只从 user/assistant 事件提取用户文本、tool result、assistant 文本和 tool use；thinking 被跳过。同时保留 session、cwd、git branch、model 以及 tool 调用元数据，最后组织成标准 `## User` / `## Assistant` 结构（`src/xskill/ecosystems/claude_code.py:604-623,636-816`）。
- **Codex**：解析 `session_meta`、`event_msg` 和 `response_item`；为了不把运行时注入内容当成人类意图，它会跳过以 `<` 开头的注入型 user/developer 内容，并输出相同的标准 Markdown/metadata 形状（`src/xskill/ecosystems/codex.py:197-214,225-355`）。

DirectoryWatcher 只接收能通过统一轨迹校验、且至少有一个非空用户段的文档（`src/xskill/pipeline/trajectory.py:171-226`）。

## 5. 提取：轨迹如何变成可复用 Skill

提取不是一次摘要，而是有持久状态的四段流水线：

1. **切 Atom**：TaskAgent 读取整条标准轨迹，用“用户意图是否切换”决定边界，产出 intent、summary、tags、used_skills 和 UX score；它记录读取 offset，确保处理到 EOF（`src/xskill/agents/task_agent.py:76-164,351-435`）。Atom 通常覆盖 1–10 个 turns，以 `<traj_id>/tasks/atom_*.json` 持久化并进入向量索引（`src/xskill/pipeline/atom.py:1-15,33-113`）。
2. **聚类/路由**：TaskClusterAgent 同时查看 atom 与现有 Skill catalog，优先复用现有 Skill；只有新 Skill 价值足够高时才创建，并保证每个 atom 落入一个或多个 `.candidates.yml`，每次贡献附带 1–10 的 weightscore（`src/xskill/agents/task_cluster_agent.py:141-249,270-371`；`src/xskill/agents/agent_tools.py:1270-1299,1353-1500`）。
3. **达到编辑阈值**：同一 Skill 的候选累计权重达到 10 就进入 ready 状态（`src/xskill/skill/candidates.py:37-49,287-345`）。
4. **蒸馏/提交**：SkillEditAgent 回读 atom/trajectory 证据，把通用知识写入 `SKILL.md` 及必要辅助资源，并通过受限写工具校验 frontmatter；新生 Skill 先 baby checkpoint/graduate 到 main，已有 main 的更新则提交到 staging（`src/xskill/agents/skill_edit_agent.py:187-286,425-608,803-865,1050-1251`；`src/xskill/agents/agent_tools.py:968-1016,1886-2060`；`src/xskill/skill/git.py:1817-1922,1958-2017`）。

DirectoryWatcher 本身维护 `discovered → splitting → split_done → indexed → done` 状态，并负责发现轨迹、调度 split/embed/cluster/SkillEdit 等阶段（`src/xskill/pipeline/runner.py:84-97,325-418,2334-2437`）。

## 6. 回注：生成结果何时重新进入系统

### 6.1 Standalone

SkillEdit future 成功收割后，runner 立即实时检测所有 harness，并逐个调用 installer；一个生态安装失败不会阻止其他生态（`src/xskill/pipeline/runner.py:811-849,1010-1054`）。新 Skill 从 baby 毕业后，main 目录因此立刻出现在 Claude/Codex 的 discovery root。已有 main 的更新先落到 staging；Claude Code 的 canary 轮换再按分配选择 main/staging 并调用 installer（`src/xskill/pipeline/runner.py:1907-2058`；`src/xskill/canary.py:469-482`）。

### 6.2 Team

服务端只把已有 main 的 Skill 放入 catalog；若存在 staging，则按客户端持久分配 main/staging side，并把 side 与 commit SHA 放入 manifest（`src/xskill/team/server/skill_manifest.py:155-179,243-348,390-444`）。客户端根据 manifest 拉取包含各分支的 Git bundle、更新本地 refs，再把 `_active` working copy checkout 到目标 SHA；有未推送用户编辑时会跳过覆盖（`src/xskill/team/shared/git_bundle.py:29-96`；`src/xskill/team/shared/reconcile.py:27-82`）。working copy 对齐后，`install_skill_to_ecosystems` 重新检测本机 harness 并执行同样的目录安装（`src/xskill/team/client/daemon.py:194-243,894-969`）。

这里的“回注系统”有两层：服务端把知识版本同步回客户端本地 Git working copy；客户端再把 working copy 挂载进 Codex/Claude 的 discovery root。服务端本身不会把 Skill 装进客户端 harness。

## 7. Claude 与 Codex 的归因能力差异

两者都能采集并标准化轨迹，但当前源码中的**精确 Skill 效果归因并不对称**：

| 能力 | Claude Code | Codex |
|---|---|---|
| 标准化原生会话 | 有 | 有 |
| 识别会话实际调用了哪个 Skill | 有：解析真实 `Skill` tool use | 当前 Codex adapter 未实现等价识别 |
| 关联 main/staging 与具体 Skill SHA | 有：结合 install history，并给已使用 Skill 的 bridge 轨迹添加 `xskill:skill/side/sha` header | 当前 adapter 只产出普通轨迹 metadata，没有等价 header/side/SHA 归因 |
| 用于 canary 灰度额度/翻面 | 只有真正触发 Skill tool 的 Claude 会话才消费灰度额度并触发 side flip | 当前源码没有等价的 Codex 精确反馈链 |

Claude 的 header 生成逻辑在 `src/xskill/ecosystems/claude_code.py:473-509`；CCSessionIngester 会利用安装历史判断 side，并且仅在识别到真实 Skill tool 调用时记录 assignment、加 header 和评分（`src/xskill/ecosystems/claude_code.py:866-893,1299-1317,1459-1544`），随后可触发相反 side 的安装（`src/xskill/ecosystems/claude_code.py:2186-2309`）。

Codex adapter 的事件分支只覆盖 session metadata、用户/助手内容及通用 response items，输出的 metadata 也没有 xskill skill/side/SHA 字段（`src/xskill/ecosystems/codex.py:225-355`）。因此可证明的是：**Codex 会话能贡献经验内容，但当前源码不能像 Claude 那样把一次真实 Skill 调用精确归因到具体版本**。不能仅因两者都支持安装和采集，就宣称两者已有同等 A/B 效果闭环。

## 8. 两种部署模式的关键差异

| 阶段 | Standalone | Team |
|---|---|---|
| 会话来源 | 本机 harness 原生目录 | 每个客户端本机原生目录 |
| 默认入流水线时机 | 原生文件静默 120 秒 + poll | 120 秒 settle + 180 秒 bridge quiet + 600 秒 hash stable + poll |
| 隐私边界 | 本机 sanitize/mask 后写 bridge | 客户端先 redact，再向服务端上传；服务端再校验/sanitize |
| Skill 生成位置 | 本机 watcher/agents | 团队服务端 watcher/agents |
| Skill 回流 | SkillEdit 成功后直接安装本机 harness | 服务端 manifest+bundle → 客户端 checkout → 安装本机 harness |
| side 分配 | 本机流程；Claude 有 canary rotation | 服务端为每个 client 持久决定 side/SHA |

## 源码边界与容易误读之处

- “注入”是目录级安装，不等于修改 harness 可执行文件、系统 prompt 或 agent loop。仓库中能证明的是安装目标与 fallback 语义。
- 默认采集依据静默/稳定窗口，不是统一的显式 session-ended 事件；尤其 team 的三个默认时间门槛应分开计算。
- “轨迹标准化支持 Codex”不等于“Codex 已有 Claude 等价的 Skill 版本归因”。当前精确 header、tool-use 和 canary 反馈实现是 Claude Code 专属路径。
- 链接安装提供实时视图；copy fallback 不具备同样的自动更新语义，需要后续重新安装才能刷新目标。
