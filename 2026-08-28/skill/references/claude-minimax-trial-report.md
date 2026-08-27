# Claude Code + MiniMax-M3：xskill 真实试用报告

## 实测进度（2026-08-25—2026-08-27）

本节只记录已经发生且可复核的结果；尚未运行的项目继续保留在后文“实验设计与证据口径”中。

脱敏机器可读证据见 [`docs/claude-minimax-trial-evidence.json`](./claude-minimax-trial-evidence.json)。其中只保存计数、测试结果和 SHA-256，不复制 prompt、API key、模型端点或会话正文。

### 环境与安装结果

| 项目 | 实测值 | 证据 |
|---|---:|---|
| 试验起点 T0 | `2026-08-25T08:27:25Z` | Herdr pane 命令输出 |
| 仓库 commit | `bc9bf941662467ac711523e450968f2677cd230e` | `git rev-parse HEAD` |
| xskill 版本 | `0.0.1.dev1+unknown.gbc9bf9416` | 隔离 venv 中 `xskill --version` |
| Python | `3.13.14` | `uv venv` 输出 |
| 安装依赖 | 53 个 | `uv pip install -e` 输出；39 个包需准备，准备耗时 2m06s，安装耗时 1.25s |
| Claude Code | `2.1.204` | 本机 `claude --version` |
| Claude 请求模型 | `MiniMax-M3` | 本机配置；认证值未读取或记录 |
| 服务端观测模型 ID | `MiniMax-M3-MXFP8` | 本次 native JSONL 的 91/91 条 assistant 事件 |
| Herdr 试验 pane | `w2:p3` / `minimax_xskill_trial` | agent 启动时为 `idle`、`interactive_ready=true`；试验后正常退出并回到 shell |
| 试验工作区 | `/tmp/xskill-minimax-trial-20260825/workload` | 从当前 commit 创建的隔离 clone |

安装使用当前源码执行，不改仓库依赖文件：虚拟环境位于 `/tmp/xskill-minimax-trial-20260825/venv`。`xskill init --skills-only --yes` 实测检测到 `claude_code/codex/cursor`，并把内置 xskill 使用指南安装到对应 Skill 目录。Claude Code 目标存在，`readlink -f ~/.claude/skills/xskill` 解析到：

```text
/home/aied/liujingyi/skills_report/xskill/src/xskill/data/skill/xskill
```

`test -L` 同样成功，证明这次是 live symlink。随后在 Claude Code 中实际输入 `/xskill`，11 秒后得到 xskill 命令帮助，证明 slash-skill discovery 可用。该 slash 展开没有在原生 JSONL 中形成 `Skill(xskill)` tool call；后续 `diagnosing-bugs` 和 `tdd` 则各形成一次可核验的真实 Skill tool call。

### T0 基线

| 计数 | T0 值 |
|---|---:|
| `~/.claude/projects/**/*.jsonl` | 3 |
| `~/.xskill/cc_sessions/traj_cc_*.json` | 0 |
| `~/.xskill/skill/*/SKILL.md` | 0 |
| `~/.xskill/config.yaml` | 不存在 |

因此本次 cohort 可以用唯一新 session ID 与存量 3 条原生会话区分。首次 `xskill serve` 后生成了 14,766 字节的配置模板，但其中 LLM/embedding 仍是默认占位配置。

### Claude/MiniMax 真实任务

用户明确授权数据出站与本机轨迹扫描后，在隔离 clone 中先执行了一个开放式 pilot，再执行一个有明确 oracle 的 primary case；所有源码改动都留在 `/tmp/xskill-minimax-trial-20260825/workload`，没有进入本报告所在工作树。

这两个场景都是直接基于 xskill 当前源码临时设计的实验任务，**不是 SWE-bench 实例，也不是仓库已有 issue**。它们用于验证真实 coding-agent 轨迹能否被采集和提取；其中只有第二个场景具有预先确定的验收条件，不能把两者当作客观的修复能力基准。

| 场景 | 实际行为 | 可复核结果 |
|---|---|---|
| Pilot / Debug | 调用 `diagnosing-bugs`，运行 Claude ecosystem 与 CLI init 测试，然后开放式寻找未覆盖边界 | 指定测试 `30 passed in 17.78s`；没有建立新的失败复现。开放式 bug hunting 数分钟仍在低影响候选间反复推演，人工终止 |
| Primary / TDD | 调用 `tdd`，为 registry list 增加 `--json`；先写 red tests，再改 CLI | Red：2 failed / 1 passed；Green：registry 相关 `41 passed in 13.38s`，原指定测试 warm run `30 passed in 3.91s` |
| 独立复核 | 在 Claude 退出后由实验控制端运行全部四个相关测试文件 | `71 passed in 14.29s`，`git diff --check` 通过 |

#### Pilot（淘汰）：开放式寻找 Claude Code ingestion 边界缺陷

任务先要求运行 `tests/test_claude_code_ecosystem.py` 与 `tests/test_cli_init.py`。若基线通过，则检查 `src/xskill/ecosystems/claude_code.py`、共享 ingestion 逻辑及 trajectory 调用方，找到一个真实边界缺陷；必须先得到失败回归测试，再做最小修复。

基线得到 `30 passed in 17.78s`。模型随后考察了空 user content、非字典 tool input、未闭合 tool call、tool result 中 Markdown fence 等候选，但没有证明任何一个候选能真实复现，也没有产生 failing test 或源码修改，最终由实验控制端终止。因此这个 case 实际测到的是开放式诊断能否收敛；由于没有已知 issue、失败测试或标准 oracle，它不是一次成功的 bug repair，也是本轮实验中设计较弱的 case。

#### Primary Case（Case 2）：为 `xskill registry list` 增加 `--json`

任务要求保留原文本输出，并增加顶层 JSON 数组输出；每个 watch directory 固定包含 `id`、`label`、`ecosystem`、`path`、`trajectories`、`indexed` 六个字段，空 registry 输出 `[]`，且只使用 Python 标准库。模型先新增非空 JSON、空 JSON、文本兼容性三个测试，red 结果为 `2 failed / 1 passed`；随后在 `src/xskill/cli.py` 增加参数与 `json.dumps()` 分支，green 结果为 registry 测试 `41 passed`，独立复核合计 `71 passed`。

这个 case 有明确输入、输出、兼容性要求和测试 oracle，能具体观察 TDD 顺序、修改范围以及轨迹中的 Read/Bash/Edit 行为。不过首条 prompt 因格式问题把命令名丢成了“add machine-readable output for .”；模型根据上下文推断为 registry list，实验控制端随后明确补充了命令和六个字段。因此它是一次成功但带人工澄清的功能实现，不是完全无干预运行。

TDD 产物为 2 个文件、119 行新增：`src/xskill/cli.py` 17 行，`tests/test_team_cli_registry_list_client.py` 102 行，未 commit。实现功能正确，但测试代码约为实现代码的 6 倍，且模型在已有 red 结果后仍尝试重复运行单测，需要人工拒绝并重新收束任务。

因此，对本次“采集 → 提取”生命周期烟测，直接使用第二个实验就够：它具有连续的 red → green → 独立复核证据。Pilot 没有失败复现和修复产物，不应进入有效学习 cohort。2026-08-27 的修复实验只重放 Primary Case；但单个临时设计 case 仍不能替代 SWE-bench 这类固定基准。

两个任务之所以进入同一条 Trajectory，不是模型或 xskill 配置写错，而是实验控制端在同一个 Claude 顶层 session 中连续执行了 Pilot 和 Primary Case。源码中的默认采集范围是 `<home>/.claude/projects/*/*.jsonl`；每个 JSONL 的 `session_id` 单独生成一个 `traj_cc_<project>_<sid>`，因此 xskill 会发现 HOME 下所有新 session，**但不会把不同 session 互相拼接**。同一 session 内有多少任务，Bridge 都会原样保留，之后才由 TaskAgent 按“用户意图切换”拆成 Atom。

所以这里有两个不同结论：作为生产形态压力测试，多意图长 session 是合法输入，TaskAgent 本应处理；作为 Case 2 的干净可归因实验，复用 session 造成了混杂，应该一开始就新开 Claude session，或像修复实验一样只给 TaskAgent 受控的 Case 2 Trajectory。隔离 HOME 只能限制“扫描哪些 session”，不能自动拆开一个 session 内的两个任务。

为下一轮更客观的实验，已另行核验三个带固定 `base_commit`、真实 issue、FAIL_TO_PASS 与 PASS_TO_PASS oracle 的 [SWE-bench Lite 候选](./swe-bench-case-candidates.md)。这些候选尚未运行，不能计入本报告的实测结果。

### 原生 Claude 会话数据

本次只产生一个新的顶层 Claude session：

```text
session_id: 1bc5c4df-268e-4e31-a02b-47d759294a0c
started:    2026-08-25T08:31:27.000Z
finished:   2026-08-25T08:48:30.734Z
duration:   1023.734s（17分03.734秒）
source:     ~/.claude/projects/-tmp-xskill-minimax-trial-20260825-workload/<session-id>.jsonl
```

| 原生指标 | 实测值 |
|---|---:|
| JSONL 大小 / 行数 | 792,658 bytes / 218 行 |
| assistant / user 事件 | 91 / 56 |
| 工具调用 | 44 |
| 工具分布 | Bash 25、Read 14、Edit 3、Skill 2 |
| 真实 Skill 调用 | `diagnosing-bugs` 1、`tdd` 1 |
| Claude 请求模型 | `MiniMax-M3` |
| 服务端观测模型 ID | `MiniMax-M3-MXFP8`，91/91 assistant 消息一致 |
| input / output tokens | 7,223,652 / 22,137 |
| cache creation / cache read tokens | 0 / 0 |
| permission-mode 事件 | 12 |

这组数据表明 MiniMax 对边界明确的 TDD 任务能交付正确结果，但本次长会话的上下文效率很差：91 次 assistant 响应累计读取 722 万 input tokens，且缓存命中为 0。开放式缺陷搜索也明显比有验收条件的任务更容易发散。

### xskill 采集与标准化结果

Claude 退出后按默认 120 秒 settle barrier 等待，再使用当前安装包的生产 `ingest_claude_code_sessions` 路径桥接该唯一 session。会话在退出前曾被桥接一次；退出追加 2,670 bytes 后又按最终源文件重建，验证了 resumed-session/rebridge 路径。

最终 bridge：

```text
~/.xskill/cc_sessions/traj_cc_workload_1bc5c4df.md
~/.xskill/cc_sessions/traj_cc_workload_1bc5c4df.json
```

| Bridge 指标 | 实测值 |
|---|---:|
| 状态 | `stored` |
| 最终 bridge 时延 | 135.131s（源文件最终 mtime → 最终 Markdown mtime） |
| Markdown / sidecar 大小 | 88,672 / 181,903 bytes |
| sidecar model | `MiniMax-M3-MXFP8` |
| 标准化 turns / tool calls | 146 / 44 |
| tool names | Skill、Bash、Read、Edit |
| Markdown tool call / output 段 | 44 / 45 |
| `validate_trajectory_source` | `valid=true`，14 个非空 user intent |
| xskill canary attribution header | 无 |

由此可确认：安装、Claude skill discovery、原生 JSONL 采集、模型归因、工具调用提取、标准 Markdown/JSON 生成均在真实数据上通过。

### xskill 的领域概念：不能混用的几组词

完整、可复用的领域词汇表已整理到 [`CONTEXT.md`](./CONTEXT.md)。理解 xskill 最短的心智模型是：

```text
Native Session
  → Bridge → Trajectory → Atom
  → Candidate + Weightscore → Skill Edit
  → Baby → Main → Staging
  → Session Assignment + UX Score → Canary
  → Manifest/Reconcile → harness 安装 → 下一条 Native Session
```

这些概念不是流水线阶段的随意别名，尤其要区分以下边界：

| 概念 | xskill 中的准确含义 | 最容易误认为 |
|---|---|---|
| **Ecosystem** | 一类 harness 的会话源、适配格式和 Skill 安装约定 | 模型 provider |
| **Native Session** | Claude/Codex 自己保存的原生交互记录 | xskill Trajectory |
| **Bridge** | 把 harness 方言标准化为 xskill 轨迹并保留归因元数据的边界 | 普通文件复制或上传 |
| **Watch Directory** | registry 中有来源与处理策略的 Trajectory 流 | 任意目录 |
| **Trajectory** | 一次 session 的完整、标准化学习证据 | 单一任务或 Atom |
| **Atom / AtomTask** | 从 Trajectory 中按用户意图切出的最小连续学习单元，可跨多轮 | token、固定 chunk、单条消息 |
| **Candidate** | Atom 支撑、但尚未写入 `SKILL.md` 的可复用模式 | staging Skill 版本 |
| **Weightscore** | Atom→Skill 归类的证据权重；累计到阈值触发 Skill Edit | UX Score |
| **Atom Adoption** | 某 Atom 已向某 Skill 贡献候选证据的耐久事实 | Skill 已安装或已毕业 |
| **Baby / Main / Staging** | 新生不可分发版本 / 稳定可分发版本 / 已有 Main 的待灰度更新 | 三个普通开发分支 |
| **Side + Session Assignment** | 某 session 实际使用的 Main/Staging 侧及精确 SHA | 当前磁盘恰好 checkout 的版本 |
| **UX Score** | 精确 Skill 版本在一个 Atom 上服务用户的 1–10 分 | Candidate Weightscore |
| **Canary** | 用真实 session 在 Main/Staging 间分流并依据版本绑定 UX 证据选赢家 | 离线 LLM 打分 |
| **Jam** | staging 过老、反馈停滞且候选积压同时成立时的受控强制收敛 | 普通 promotion |
| **Manifest / Reconcile** | team server 下发的个性化 Skill side/SHA 目标，以及 client 对齐本地并安装的动作 | 把服务端目录直接挂到 harness |
| **User Staging** | 隔离保存专家本地修改的 client 专属分支，不能直写 Main | 普通 Canary Staging |
| **Native Skill / SkillHub Skill** | 前者进入 Git/Candidate/Canary 演进；后者是可搜索三方包，不自动拥有完整演进链 | 同一种 Skill 来源 |

原始完整会话的失败位置可以精确表述为：**Bridge 和 Trajectory 已成功，但 TaskAgent 没有提交任何 Atom；所以 Candidate、Weightscore、Adoption、Baby/Main/Staging 和 Canary 都还不存在。** 后续 Case 2-only 重放跨过了工具提交关口，但暴露了过切和噪声 Atom，详见下文。Qwen embedding 接口可用仍只是旁路证据，因为本轮自动流水线没有运行到 embedding。

### 原始完整会话：真实自动蒸馏重跑与失败定位

初始阶段完整 standalone daemon 没有成功常驻。第二次 `xskill --debug serve --port 8877` 能启动 FastAPI，并记录模板默认的 `deepseek-v4-flash` 和 `text-embedding-v4`，但随后进入 embedding 维度探测并退出。

用户随后授权复用 Claude settings 中的 MiniMax API，并提供独立 Qwen embedding 服务。固定、无仓库内容的兼容性探测结果如下：

| 探测 | HTTP | 时延 | 结果 |
|---|---:|---:|---|
| MiniMax `GET /v1/models` | 200 | 331.0ms | 返回 11 个路由 ID |
| MiniMax Anthropic `/v1/messages` | 200 | 485.8ms | 可用 |
| MiniMax OpenAI `/v1/chat/completions` | 200 | 479.5ms | 可用 |
| MiniMax OpenAI function calling，model=`MiniMax-M3` | 200 | 1616.6ms | 返回 1 个预期 `ping` tool call |
| MiniMax embedding | 400 | 77.2ms | 网关明确不支持该 provider/model 的 embeddings |
| Qwen `qwen3-vl-embedding` | 200 | 177.5ms | 返回 1 个 4096 维向量，usage=6 tokens |

显式 model 路由 `openai/MiniMax-M3-MXFP8` 在 chat-completions 上返回 404，反而是 Claude settings 中的 `MiniMax-M3` 别名可正常完成 OpenAI tool call，因此 xskill 使用后者。最终配置为 MiniMax chat + Qwen embedding，`settle_seconds=10`；配置文件权限为 `0600`，原模板备份为 `~/.xskill/config.yaml.before-minimax-trial`。

用户随后明确授权真实轨迹出站，因此补跑了两次 daemon 启动。

#### 第一次：真实 HOME 暴露了自动发现的范围风险

`09:02:54Z` 在真实 HOME 启动后，LLM、embedding、agent worker 和 ecosystem ingester 均正常就绪；但 standalone 会自动发现机器上的**所有**已知 ecosystem，而不是只处理本次 Claude cohort。到 `09:04:42Z` 人工停止时：

- 自动生成了 624 个 Codex bridge 文件，共 137,981,171 bytes；
- 另生成 6 个不属于本次 cohort 的 Claude bridge 文件；
- registry 自动注册了 `codex` 与 `claude_code` 两个 watch directory；
- watcher 状态仍为 `polls=0`，目标 Claude 轨迹尚未进入 split，说明启动钩子仍忙于历史 bridge。

这批无关文件没有删除，已移到可恢复隔离区 `/tmp/xskill-minimax-trial-20260825/auto-discovery-quarantine-090254/`；主 HOME 的 bridge 恢复为只保留目标 Claude 文件，两个空 watch-directory 注册项也已移除。该结果是重要的部署结论：**在存量 HOME 上首次启动 `serve` 会吞入全机历史，试验和生产上线都应先评估数据范围、积压量和模型预算。**

#### 第二次：隔离 HOME，只放一条目标 Claude session

新建 `/tmp/xskill-minimax-pipeline-20260825-cAGeaT`，其中只复制目标原生 JSONL、同一份模型配置和内置 `/xskill` Skill，再启动同一生产 `serve` 路径。结果如下：

| 时刻/指标 | 实测结果 |
|---|---|
| daemon ready | `09:07:03Z`，MiniMax/Qwen 均为 `ok` |
| bridge | `09:07:04Z`，1 条目标 session |
| registry | 1 个 `claude_code` watch directory，1 条 trajectory |
| discover/split start | `09:07:10Z`，目标 Markdown 2,220 行 |
| 首次 split | 约 165 秒后失败；14 个待拆 User 回合，0 次 `submit_atom`，自动标记重拆 |
| 自动重试 | 开头 4 次 `500 No healthy backend`；恢复后继续运行 |
| 第二次 trace | 启动 29 个 round、36 次 `look`、0 次 `submit_atom`；为阻止无效消耗，在 `09:12:25Z` 人工停止 |
| watcher 终值 | polls 64、new trajectories 1、errors 1、retries 1、atoms/indexed/clustered/skills/scores 均为 0 |
| trajectory DB | `status=splitting`（人工停止时仍在跑），`tasks_extracted=0`、`retry_count=1`，保留首次 0-submit 错误 |

两次 split 共留下 39 条成功 LLM usage 记录；另有 4 次 500 重试未计入成功 ledger：

| MiniMax split 资源 | 实测值 |
|---|---:|
| prompt tokens | 620,521 |
| completion tokens | 7,684 |
| total tokens | 628,205 |
| xskill 默认价格表估算 | USD 0.643573；`estimated=true`，不是供应商账单 |

这不是“端点完全不可用”：固定 function-call 探针能正确返回工具调用，真实 split 也能持续调用 `look`。在这条混合了 Pilot、Primary Case、初始化、权限中断与退出命令的完整 Trajectory 上，MiniMax 没有调用必须的 `submit_atom`；trace 还反复出现空内容被协议层净化后继续 `look` 的行为。该结果说明完整长会话不适合作为单一学习 cohort，但还不足以断言模型在边界清楚时完全不会提交 Atom。

Qwen embedding 则通过两级旁路探测：固定文本为 4096 维、177.5ms；目标轨迹中的真实 TDD 片段为 1,200 字符、4096 维、全部 finite、2,692.4ms。后者证明真实内容可被该 endpoint 编码，但必须标记为**旁路兼容性测试**：自动流水线因没有 Atom 而从未进入 embed pool。

隔离 registry 的最终计数为 `watch_dirs=1 / trajectories=1 / llm_usage=39 / atom_adoption=0 / canary_decision=0`。所以本报告能证明真实 daemon 的启动、bridge、discover 和失败重试语义，不能声称已经验证自动 cluster、SkillEdit、生成 Skill 回装或 canary。

安全收尾时已删除隔离 HOME 中复制的模型配置（其中曾含凭据）；主 `~/.xskill/config.yaml` 未改，隔离 trajectory、registry 和 trace 仍保留用于复核。

### 修复实验：Case 2-only 受控重放（2026-08-27）

为排除 Pilot 和会话尾部噪声对 split 的干扰，修复实验从保留的标准 Bridge 中机械截取 Primary Case 的连续区间（原行 `1181–2220`），形成 Case 2-only 派生 Trajectory。它不是新的 Claude 原生会话，也没有改写原始证据；唯一改变的是 TaskAgent 所见的 cohort 边界。

模型名在这里必须分开记录：Claude 发起 coding 请求时配置的是 `MiniMax-M3`，原生 JSONL/Bridge 响应元数据报告的是部署变体 `MiniMax-M3-MXFP8`；TaskAgent 蒸馏请求同样配置为 `MiniMax-M3`。因此不能把 `MiniMax-M3-MXFP8` 写成用户配置的模型名。

| 指标 | Case 2-only 实测值 |
|---|---:|
| 派生 Trajectory | 1,040 行 / 41,629 bytes / 7 个 user header |
| SHA-256 | `90e2f281eaff037dd716ae0351d760617e7023ef2dbc4a2d920dbe102193e613` |
| Claude 请求模型 | `MiniMax-M3` |
| 原生响应模型 ID | `MiniMax-M3-MXFP8` |
| TaskAgent 请求模型 | `MiniMax-M3` |
| 运行耗时 | 24.759s |
| agent trace | 7 rounds / 6 次 `look` / 3 次 `submit_atom` |
| 落盘 Atom | 3 个，起始行 `1 / 817 / 1028` |

工具协议已从原始完整会话的 0-submit 变为成功提交；但逐个审计发现语义切分并未完全正确：

| 输出 | 实际内容 | 质量判断 |
|---|---|---|
| Atom 1 | Case 2 的 TDD red 阶段 | 应与 Atom 2 合并 |
| Atom 2 | 同一 Case 2 的 green 阶段，边界来自用户制止重复测试 | 多余边界，属于过切 |
| Atom 3 | `/exit` 与 `Catch you later!` | 会话控制噪声，不应成为 Atom |

任务级预期是 1 个连续 Atom，实际得到 3 个，其中包含 1 个多余边界和 1 个噪声 Atom。因此修复结论是：**Trajectory → Atom 的工具调用/持久化已通过，语义切分质量仅部分通过。** 本次受控重放有意停在 split 质量审计，没有继续运行 embed、cluster、Candidate、SkillEdit、回注或 Canary；这些阶段仍然是“未验证”，而不是“失败”。这次修复的是实验 cohort，不是 xskill 源码。

这组结果与模型能力有关，但当前证据不能把责任完全归给模型：同一 `MiniMax-M3` 请求模型在缩小 cohort 后从 0-submit 变成 3-submit，说明输入长度与意图混杂显著影响工具遵循；它又把同一目标的纠偏切开，并把 `/exit` 当 Atom，说明语义边界判断仍不够稳定。另一方面，TaskAgent 只先提供“用户提问地图”、需要模型主动 `look` 才能读取正文，这种 prompt/agent 契约同样是变量。`MiniMax-M3-MXFP8` 只是服务端观测到的部署 ID；没有未量化 MiniMax-M3 或另一模型的同轨迹 A/B，不能据此断言量化导致了错误。要回答“是不是模型能力问题”，最小有效补实验是把同一 Case 2 Trajectory、同一 prompt 和采样参数分别重复跑多个模型，再比较 submit 成功率与 Atom 边界准确率。

### 试用结论

| 能力 | 结论 |
|---|---|
| 源码安装与 Claude 注入 | 通过；live symlink，`/xskill` 可发现 |
| Claude/MiniMax 开发任务 | 通过；交付可运行改动，独立复核 71 tests 通过 |
| 真实 Skill tool use | 通过；`diagnosing-bugs`、`tdd` 各 1 次 |
| 原生会话采集与 adapter | 通过；1/1 session 最终 bridge，模型/44 次工具调用保持一致 |
| daemon bridge/discover | 通过；隔离 HOME 中只注册并发现目标 1 条 trajectory |
| 自动 Atom split | **部分通过**；原始完整会话 0 Atom，Case 2-only 重放提交 3 个，但有 1 个过切边界和 1 个噪声 Atom |
| Qwen embedding | 接口与真实片段旁路测试通过；Case 2-only 重放停在 split 审计，自动 embed 仍未到达 |
| Candidate→Skill→canary | 未到达；修复重放没有继续运行下游流水线 |
| 数据范围控制 | 有风险；真实 HOME 首启自动桥接全机历史，必须先隔离或预算化 backfill |
| 使用效率 | 风险明显；Claude 会话 722 万 input tokens、零 cache read；xskill split 又消耗 62.8 万 tokens 仍无 Atom |

## 实验设计与证据口径

> 本节只定义方案和口径，不在这里声明命令是否已执行；实际执行状态仅以上方“实测进度”为准。所有 `<...>` 均为待填值，证据只来自本仓库源码和配置。

### 1. 实验问题与边界

本试用要分别回答五个问题：

1. xskill 是否正确识别真实 Claude Code，会不会把 Skill 安装到 `~/.claude/skills/`。
2. Claude Code 产生的 MiniMax-M3 会话是否在静默窗口后完整进入 `~/.xskill/cc_sessions/`，且 model attribution 不丢失。
3. 轨迹是否依次完成 split、embed、cluster，并留下 Atom、candidate/adoption 和 Skill Git 仓库。
4. 新 Skill 回装后，Claude Code 是否真的发出 `Skill` tool call，而不只是把它列在 available skills 中。
5. 若形成 staging，main/staging 的安装、体验分和 canary 裁决是否能用持久证据复核。

这里有两个容易混淆的“模型”：

- **用户 coding-agent 模型**：Claude Code 原生 JSONL 中 `assistant.message.model`。Claude adapter 取第一条非空值写入 trajectory sidecar 的 `model`，再进入 registry 的 `source_model`（`src/xskill/ecosystems/claude_code.py:630-655,797-810`；`src/xskill/pipeline/registry.py:131-153`）。本试用把实际出现的 MiniMax provider model ID 记录为 `MINIMAX_MODEL_ID`，不预先猜它的拼写。
- **xskill 蒸馏模型**：`~/.xskill/config.yaml` 的 `llm.base_url/model/api_key`，用于 split/cluster/edit。它可以指向任意 OpenAI-compatible chat-completions endpoint；非 DeepSeek 官方 URL 走通用 `OpenAIChat`（`src/xskill/config.py:64-110`；`src/xskill/agents/agno_factory.py:157-204,206-231`）。仓库没有 MiniMax-M3 专属 endpoint 或模型 ID，故实验记录必须另存 `XSKILL_LLM_MODEL`，不能默认它等于 Claude Code 的 `MINIMAX_MODEL_ID`。

仓库也不负责配置 Claude Code 使用 MiniMax。实验前提是外部已把 Claude Code 配好，并且至少完成一条真实会话，使 `~/.claude/projects/*/*.jsonl` 存在。xskill 能证明和观测的是这个文件协议，而不是 Claude/MiniMax 的供应商配置过程（`src/xskill/ecosystems/claude_code.py:60-74,604-623`）。

### 2. 部署形态与隔离原则

核心试用采用 **standalone、真实 HOME、前台启动**：`xskill serve` 会同时启动持久 agent worker 和 Claude ecosystem ingester；后者默认每 0.5 秒轮询，但只有满足 settle barrier 的会话才 bridge（`src/xskill/api/app.py:1208-1271`；`src/xskill/ecosystems/_shared.py:951-1009`）。不采用 team client，以免额外引入 180 秒 quiet、600 秒 hash-stable 和上传链路。

不删除或重置现有 `~/.xskill`。执行前记录 UTC 起点 `T0`、现有文件数和数据库计数，之后所有指标按 `T0` 之后的新会话/新记录计算。这样既不破坏已有数据，也避免把存量成果误报为本次结果。

建议保存以下实验身份字段：

```text
TRIAL_ID=<UTC timestamp or ticket>
T0=<UTC ISO-8601>
T0_SQLITE_UTC=<UTC as YYYY-MM-DD HH:MM:SS>
repo_commit=<git rev-parse HEAD>
xskill_version=<xskill --version>
claude_code_version=<external observation>
MINIMAX_MODEL_ID=<value observed in native JSONL>
XSKILL_LLM_MODEL=<value configured in ~/.xskill/config.yaml>
embedding_model=<value configured in ~/.xskill/config.yaml>
XSKILL_PORT=8877
TRIAL_SKILL=<learned skill slug; set only after a main ref exists>
```

后续 shell/SQL 块假定 `T0`、`T0_SQLITE_UTC` 和 `XSKILL_PORT` 已按上述格式导出为环境变量；形成毕业 Skill 后再导出 `TRIAL_SKILL`。未设置变量时不要直接执行查询，以免空字符串扩大 cohort。

### 3. 安装、初始化与启动命令

项目使用 setuptools build backend，声明 Python `>=3.9`，并把 `xskill` 映射到 `xskill.cli:main`，因此源码试用从仓库根目录执行 editable install（`pyproject.toml:1-10,29-58,89-105`）：

```bash
python -m pip install -e .
xskill --version
command -v jq
command -v sqlite3
command -v git
test -d "$HOME/.claude/projects"
xskill init --skills-only --yes
test -f "$HOME/.claude/skills/xskill/SKILL.md"
test -L "$HOME/.claude/skills/xskill" && readlink -f "$HOME/.claude/skills/xskill"
```

`init --skills-only` 只安装随包分发的 xskill 使用 Skill，不配置 team 连接；该 flag 和安装流程由 `src/xskill/cli.py:179-235,2068-2095` 定义。Claude installer 的目标是 `~/.claude/skills/<name>`，优先 symlink，其次 junction，最后 copy（`src/xskill/ecosystems/claude_code.py:82-124`）。目标 `SKILL.md` 可读证明安装可用；`test -L` 成功且 `readlink -f` 指向预期源仓，才进一步证明本次 Linux 安装是 live symlink。

`jq`、`sqlite3` 和系统 `git` 是本报告的只读审计工具，不是 xskill 的运行依赖；预检缺失时应改用等价只读工具，不能把审计工具缺失误报为 xskill 失败。安装形态不能只看 `readlink -f`，因为它对普通目录也会返回规范路径；Linux 试验以 `test -L` 为 symlink 判据。`init` 是短命令，不配置文件日志，因此本次安装是否 fallback 先看其终端输出；daemon 启动后的再次安装事件才可辅助查看 `xskill.ecosystems.log`（`src/xskill/cli.py:2189-2215`）。

第一次执行 `xskill serve` 时，如果 `~/.xskill/config.yaml` 不存在，CLI 会写模板并正常退出，要求填入 LLM 与 embedding key 后重跑（`src/xskill/cli.py:2289-2300`；`src/xskill/config.py:285-292,468-489`）：

```bash
xskill serve --port "$XSKILL_PORT"
${EDITOR:-vi} "$HOME/.xskill/config.yaml"
```

配置时采用真实 provider 给出的值，不把占位符提交到报告：

```yaml
llm:
  base_url: <openai-compatible-base-url>
  model: <xskill-distillation-model-id>
  api_key: <secret-not-copied-into-report>
  max_tokens: 10000

embedding:
  base_url: <embedding-base-url>
  model: <embedding-model-id>
  api_key: <secret-not-copied-into-report>
  dim: 0

ingest:
  settle_seconds: 10
  mask_patterns: []
```

模板要求 standalone 的 LLM 和 embedding key 都存在（`src/xskill/config.py:51-57,64-134,468-489`）。评测场景使用 `settle_seconds: 10` 是源码模板明确建议的 5–15 秒区间；生产默认是 120 秒（`src/xskill/config.py:225-240`）。该修改必须写进报告，因为它改变了 ingestion latency，不能拿 10 秒结果冒充默认生产时延。

以前台 debug 模式启动，以便同时保留终端和分组件日志：

```bash
xskill --debug serve --port "$XSKILL_PORT"
```

`serve` 有单实例守卫；已有存活 daemon 时默认拒绝，而不是启动第二份争抢 registry（`src/xskill/cli.py:33-67`）。本实验不使用 `--force`。

### 4. 状态与日志检查命令

Standalone 的状态命令是 `xskill stats --json`，其 `status` 字段来自 `~/.xskill/serve_runtime.json`，包含 running、pid、port、mode、llm_model、llm_base_url 和 embed_model（`src/xskill/cli.py:634-673`；`src/xskill/runtime.py:29-59,111-126`）：

```bash
xskill stats --json
xskill registry list
```

不要用 `xskill status` 判断 standalone：该命令只查询 team `connect` 后台服务（`src/xskill/cli.py:622-631,2137-2144,2240-2250`）。`xskill registry list` 会显示 `ECOSYSTEM / TRAJ / INDEXED / PATH`，Claude 自动注册项的 ecosystem 应为 `claude_code`（`src/xskill/cli.py:70-96`）。

长跑命令会把日志写入 `~/.xskill/logs`；汇总真源是 `xskill.log`，细分文件按首次事件懒创建（`src/xskill/utils/logging.py:14-17,24-46,78-101`）：

```bash
tail -F "$HOME/.xskill/logs/xskill.log" \
        "$HOME/.xskill/logs/xskill.ecosystems.log" \
        "$HOME/.xskill/logs/xskill.watcher.log" \
        "$HOME/.xskill/logs/xskill.skill_edit_agent.log" \
        "$HOME/.xskill/logs/xskill.canary.log"
```

日志证据只用于解释状态变化；数量与终态以 JSON 文件、Git refs 和 `registry.db` 为准，避免用 grep 日志行数替代事实表。

### 5. 试用批次

实验分四批，每一条都必须是新 Claude Code session，并记录 native session ID、开始/结束 UTC 和任务标签。

1. **S0 管道烟测（2 条）**：一条短代码修改、一条带工具调用的诊断任务。目标是验证 MiniMax model ID、bridge、Atom 和 `done/error` 状态，不要求生成 Skill。
2. **S1 同域训练（至少 10 条）**：选择一个真实、可重复但非完全相同的工作域，每条任务有独立用户意图和可核验产物。Skill promotion 不是按轨迹条数触发，而是候选 `weightscore` 总和达到 10；单 atom 最高可给 10，因此“10 条”只是试验样本量，不是源码保证的毕业数量（`src/xskill/skill/candidates.py:37-49,287-345`）。
3. **S2 回装后触发评估（正例 10 条、负例 10 条）**：正例应落在新 Skill description 的适用范围，负例与关键词相近但不应调用。真实调用的唯一口径是 Claude JSONL 出现 `tool_use`、`name == "Skill"`、`input.skill == <skill-name>`；仅出现在 available-skills 列表不算使用（`src/xskill/ecosystems/claude_code.py:332-380`）。
4. **S3 canary 扩展**：只有观察到 main、至少一条 main UX score、后续候选达到阈值并形成 staging 后才开始。源码在 main→staging 前强制要求 main 已有真实 UX score（`src/xskill/agents/skill_edit_agent.py:425-506`）。若未满足，就记录“未进入 canary 前置态”，不能伪造 side 或手工写评分。

每条 session 结束后至少等待 `settle_seconds + ingestion/discovery poll`，这个等待只说明会话已经具备 **bridge 与 DB discovery 资格**，不说明 split/cluster/edit 已完成；LLM 流水线必须继续轮询该 cohort 的 `trajectories.status`，直到 `done`、`filtered` 或重试耗尽后的 `error` 等终态。bridge 的真实逻辑以源 JSONL mtime 静默窗口为准，源文件后续增长会 rebridge 并重置下游轨迹（`src/xskill/ecosystems/_shared.py:951-1056`；`src/xskill/pipeline/runner.py:2334-2405,2748-2774`）。

### 6. 轨迹、Atom、Skill 与 canary 的检查命令

#### 6.1 原生会话与标准轨迹

Claude 原生源是 `~/.claude/projects/*/*.jsonl`，bridge 是 `~/.xskill/cc_sessions/traj_cc_<project>_<sid8>.{md,json}`（`src/xskill/ecosystems/_shared.py:145-160`；`src/xskill/ecosystems/claude_code.py:445-470,820-863`）：

```bash
find "$HOME/.claude/projects" -mindepth 2 -maxdepth 2 -type f \
  -name '*.jsonl' -newermt "$T0" -print
find "$HOME/.xskill/cc_sessions" -maxdepth 1 -type f \
  -name 'traj_cc_*.json' -newermt "$T0" -print
find "$HOME/.xskill/cc_sessions" -maxdepth 1 -type f \
  -name 'traj_cc_*.json' -newermt "$T0" \
  -exec jq -r '[.session_id, .model, .total_turns, .total_tool_calls] | @tsv' {} +
```

`-mindepth 2 -maxdepth 2` 有意与生产 `CC_SPEC.sessions_glob="*/*.jsonl"` 保持一致，避免把更深层的其他 JSONL 错算进 bridge-yield 分母（`src/xskill/ecosystems/claude_code.py:584-596`）。`-newermt` 只用于发现候选，正式 cohort 仍以 S0–S3 记录的 session ID 清单为准；存量 session 在 T0 后续写不应被误认成新样本。Adapter 会把 user/assistant/tool call/tool output 转成标准 Markdown，把 model、timeline、tool_names、turn/tool-call 数写入 sidecar（`src/xskill/ecosystems/claude_code.py:604-816`）。每个 `.md`/`.json` 对是持久证据（`src/xskill/ecosystems/_shared.py:674-730`）。

数据库按本次 cohort 检查：

```bash
sqlite3 -header -column "$HOME/.xskill/registry.db" \
  "SELECT t.status,t.source_model,
          COALESCE(NULLIF(t.source_harness,''),w.ecosystem) AS effective_harness,
          COUNT(*) AS n
   FROM trajectories t JOIN watch_dirs w ON w.id=t.watch_dir_id
   WHERE w.ecosystem='claude_code' AND t.discovered_at >= '$T0_SQLITE_UTC'
   GROUP BY t.status,t.source_model,effective_harness ORDER BY n DESC;"

sqlite3 -header -column "$HOME/.xskill/registry.db" \
  "SELECT t.filename,t.status,t.process_action,t.skill_generated,t.skill_used,
          t.canary_side,t.ux_score,t.retry_count,t.error_msg,
          t.discovered_at,t.updated_at
   FROM trajectories t JOIN watch_dirs w ON w.id=t.watch_dir_id
   WHERE w.ecosystem='claude_code' AND t.discovered_at >= '$T0_SQLITE_UTC'
   ORDER BY t.id;"
```

这些列由 registry schema 直接定义（`src/xskill/pipeline/registry.py:121-166`）。本机 Claude sidecar 通常不必重复存 `harness`；统计层的正式语义是 `source_harness` 缺失时回退到非 manual/team 的 `watch_dirs.ecosystem`，所以上述查询显式计算 `effective_harness`（`src/xskill/pipeline/registry.py:1548-1601`）。

#### 6.2 Atom 与候选

Atom 不在 SQLite；每条轨迹的文件位于 bridge 下 `<traj_id>/tasks/atom_*.json`，字段含 intent、summary、used_skills、ux_score、source_model 和持久 `clustered` 标记（`src/xskill/pipeline/atom.py:1-15,33-66,76-113`）：

```bash
find "$HOME/.xskill/cc_sessions" -path '*/tasks/atom_*.json' -type f -newermt "$T0" -print
find "$HOME/.xskill/cc_sessions" -path '*/tasks/atom_*.json' -type f \
  -newermt "$T0" \
  -exec jq -r '[.atom_id,.traj_id,.source_model,.clustered,.ux_score,.intent] | @tsv' {} +

sqlite3 -header -column "$HOME/.xskill/registry.db" \
  "SELECT skill,COUNT(*) AS adoption_events,
          COUNT(DISTINCT atom_id) AS distinct_atoms,
          SUM(CASE WHEN was_new=1 THEN 1 ELSE 0 END) AS first_add_events,
          AVG(weightscore) AS event_weight_avg
   FROM atom_adoption WHERE ts >= '$T0_SQLITE_UTC'
   GROUP BY skill ORDER BY distinct_atoms DESC;"

python - <<'PY'
from pathlib import Path
import yaml

for path in sorted((Path.home() / ".xskill" / "skill").glob("*/.candidates.yml")):
    rows = (yaml.safe_load(path.read_text(encoding="utf-8")) or {}).get("candidates", [])
    print(path.parent.name, "candidates=", len(rows),
          "current_weight_total=", sum(int(row.get("weightscore", 0)) for row in rows))
PY
```

`atom_adoption(atom_id, skill, weightscore, was_new)` 是 append-only 聚类采纳事件；同一 atom+skill 再次 add 会覆盖 candidate 当前分值，但仍可能留下新的 telemetry event，因此 `COUNT(*)`/`SUM(weightscore)` 不能冒充当前候选数/当前 buffer 总分（`src/xskill/pipeline/registry.py:1236-1244`；`src/xskill/agents/agent_tools.py:1353-1397`）。当前待编辑事实在 `<skill>/.candidates.yml`，晋升阈值口径是其中 pending weightscore 的当前求和（`src/xskill/skill/candidates.py:37-68,287-345`）。

#### 6.3 Skill 生成与回装

```bash
find "$HOME/.xskill/skill" -mindepth 2 -maxdepth 2 -type f -name 'SKILL.md' -newermt "$T0" -print
find "$HOME/.xskill/skill" -mindepth 2 -maxdepth 2 -type f -name '.candidates.yml' -print
find -L "$HOME/.claude/skills" -mindepth 2 -maxdepth 2 -name 'SKILL.md' -print
for repo in "$HOME"/.xskill/skill/*; do
  test -d "$repo" || continue
  git -C "$repo" show-ref --verify --quiet refs/heads/main \
    && printf '%s\n' "${repo##*/}"
done
git -C "$HOME/.xskill/skill/$TRIAL_SKILL" show-ref --verify refs/heads/main
git -C "$HOME/.xskill/skill/$TRIAL_SKILL" log --oneline --decorate --all -n 20
git -C "$HOME/.xskill/skill/$TRIAL_SKILL" show-ref --heads
test -L "$HOME/.claude/skills/$TRIAL_SKILL" \
  && readlink -f "$HOME/.claude/skills/$TRIAL_SKILL"
```

新建 Skill 在 `baby` 阶段已经有 stub `SKILL.md`，所以“发现文件”不等于“已经毕业”；本报告只有在 `refs/heads/main` 存在后才设置 `TRIAL_SKILL` 并计入 Skill yield（`src/xskill/skill/git.py:1817-1922`）。生成成功后 runner 会立即实时检测生态并安装；Claude 目标应解析到主 Skill 仓或 canary materialization（`src/xskill/pipeline/runner.py:811-849,1010-1054`；`src/xskill/ecosystems/claude_code.py:82-112`）。Claude 安装目录通常是 symlink，普通 `find` 不会下钻，故检查 discovery root 使用 `find -L`。`git` 命令只是只读审计工具；xskill 本身用 dulwich，不把系统 Git 当运行依赖（`pyproject.toml:45-48`）。

#### 6.4 实际 Skill 调用与 canary

原生调用证据可直接检查目标 session 的 `Skill` tool call：

```bash
jq -c --arg skill "$TRIAL_SKILL" 'select(.type=="assistant")
  | .message.content[]?
  | select(.type=="tool_use" and .name=="Skill" and .input.skill==$skill)
  | {skill:.input.skill,args:.input.args}' \
  "$HOME/.claude/projects/<project>/<session-id>.jsonl"
```

形成 staging 后，依次检查物化内容、当前安装目标、安装账本、归因 header、UX 分和终态裁决：

```bash
test -f "$HOME/.xskill/skill/.canary/$TRIAL_SKILL/SKILL.md"
test -L "$HOME/.claude/skills/$TRIAL_SKILL" \
  && readlink -f "$HOME/.claude/skills/$TRIAL_SKILL"
jq -c --arg skill "$TRIAL_SKILL" 'select(.skill==$skill)' \
  "$HOME/.xskill/install_history.jsonl" | tail -n 50
grep -R "^<!-- xskill:skill=$TRIAL_SKILL " \
  "$HOME/.xskill/cc_sessions" --include='traj_*.md'
jq -s 'sort_by(.side,.commit_sha,.user_model)
  | group_by([.side,.commit_sha,.user_model])
  | map({side:.[0].side,commit_sha:.[0].commit_sha,user_model:.[0].user_model,
         n:length,avg:(map(.score)|add/length)})' \
  "$HOME/.xskill/skill/$TRIAL_SKILL/.ux_scores.jsonl"
sqlite3 -header -column "$HOME/.xskill/registry.db" \
  "SELECT * FROM canary_decision WHERE skill='$TRIAL_SKILL' ORDER BY id DESC;"
```

`.canary/<name>/SKILL.md` 存在才是可安装的 staging（`src/xskill/ecosystems/claude_code.py:383-392`）。Claude ingester 只有确认该 session 真实调用目标 Skill 后，才注入 `skill/side/sha` header、消费灰度配额并进入评分链（`src/xskill/ecosystems/claude_code.py:1299-1317,1459-1544`）。UX 是 atom 级幂等记录，包含 side、commit_sha、score、user_model（`src/xskill/canary.py:1140-1184`）。默认模型分桶 canary 每侧要求 20 个合格样本，staging 均分不低于 main 才 promoted；不足时是 waiting，14 天后仍不足才 timeout（`src/xskill/config.py:145-164`；`src/xskill/canary.py:940-1024`）。

核心试用保持 canary 默认值。若另做“加速灰度”实验，必须单独记录修改后的 `probability / rotate_interval / scope_top_n / total_samples`，并把结果标为 accelerated cohort，不能与默认 20-samples-per-side 口径混合。

### 7. 预注册指标

| 指标 | 计算口径 | 事实源 |
|---|---|---|
| Claude 探测成功 | `registry list` 中 `ecosystem=claude_code` 是否为 1 | `watch_dirs` |
| 安装完整率 | `~/.claude/skills/<skill>/SKILL.md` 存在数 / 本次应安装 Skill 数 | discovery root + skill repo |
| MiniMax 归因覆盖率 | sidecar `model == MINIMAX_MODEL_ID` 的新 Claude 轨迹数 / 新 Claude 轨迹总数 | `traj_cc_*.json`、`trajectories.source_model` |
| bridge yield | 成功生成 `.md`+`.json` 的 session 数 / S0+S1+S2 已完成 native session 数 | native JSONL + bridge |
| registry 可见时延 | 对未发生 rebridge 的新 session，计算 `trajectories.discovered_at - native JSONL final mtime` 的 p50/p95，并注明 settle=10s；rebridge 样本单列。若需纯 bridge 时延，必须由观察器记录文件首次出现时刻，不能用可能被 header/rebridge 改写的最终 md mtime | native mtime + `trajectories.discovered_at` |
| pipeline 完成率 | `status=done` / 本次 cohort trajectory 总数 | `trajectories` |
| pipeline 终态错误率 | 最终 `status=error` / cohort 总数；`retry_count>0` 和 `error_msg` 非空另报为重试/历史诊断，不并入终态错误率 | `trajectories` |
| Atom 产出率 | Atom 总数 / `done` trajectory 数 | `*/tasks/atom_*.json` |
| Atom 聚类率 | `clustered=true` Atom 数 / Atom 总数 | Atom JSON |
| 候选贡献 | telemetry 的 distinct atom/adoption event 数；当前候选数与当前 weightscore 总和只从 `.candidates.yml` 快照计算 | `atom_adoption`、`.candidates.yml` |
| Skill yield | 本 cohort 新建且已存在 `refs/heads/main` 的 Skill 数；baby stub 不计 | Skill Git refs + T0 基线 |
| 正例触发率 | S2 正例中真实 `Skill` tool call 命中目标 Skill 的 session 数 / 10 | native JSONL |
| 负例误触发率 | S2 负例中真实调用目标 Skill 的 session 数 / 10 | native JSONL |
| 版本归因覆盖率 | 有 `xskill:skill/side/sha` header 的目标 Skill 使用轨迹数 / staging 期间真实调用该 Skill 的轨迹数 | bridge header + native JSONL |
| canary 样本与差值 | main/staging 各自 n、均分、`staging_avg-main_avg`，按 `user_model` 分桶 | `.ux_scores.jsonl` + `canary_decision` |
| 蒸馏成本 | 本 cohort 前后 total calls/tokens/USD 差值，并按 step/model 展开 | `xskill stats --json`、`llm_usage` |

预注册的核心通过条件建议为：S0 两条均能 bridge、MiniMax model ID 两条均非空且一致、最终没有 `status=error`；S1 至少产生 Atom，且所有最终 `done` 轨迹的 Atom 都 `clustered=true`；S2 同时报告正例触发率与负例误触发率。`error_msg` 在 error→retry→成功后不一定自动清空，因此只作为发生过异常的诊断证据，不能单独否决最终成功（`src/xskill/pipeline/registry.py:1979-2024`；`src/xskill/pipeline/runner.py:2325-2365`）。是否生成新 Skill、是否形成 staging 和是否达到 canary 终态作为**观察结果**，不因样本不足而伪造成硬通过项。

成本统计的持久口径由 `llm_usage` 汇总为 total tokens、USD、calls，并提供 by-step/by-model 分解（`src/xskill/pipeline/registry.py:156-167,1517-1545`；`src/xskill/cli.py:634-673`）。若价格来自 fallback 而非显式 config，`estimated=true`，报告中必须标为估算而非账单。

### 8. 结果记录规则

- 每个结论至少附一个文件/数据库事实；日志只作为辅助上下文。
- 所有计数限定 `T0` 后 cohort，结果表同时写分子、分母，不能只写百分比。
- `find -newermt` 只是候选发现手段；最终分母按预先记录的 session ID manifest 去重，避免存量文件续写污染 cohort。
- 报告实际的 provider model ID 原文；只有 sidecar 确认后才称其为 MiniMax-M3 cohort。
- 不在报告、命令输出或附件中复制 API key。
- `waiting`、无 staging、无新 Skill 都是合法观测；不得改写 JSONL、Atom、`.ux_scores.jsonl` 或 `canary_decision` 来补齐链路。
- 若源 session 在首次 bridge 后继续增长，以 rebridge 后的最终 trajectory 为准，并记录它重置过下游状态；不能把两版当两条样本。
