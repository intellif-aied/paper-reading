---
theme: default
title: XSkill：把会话经验变成可试用、可回滚的 Skill
aspectRatio: 16/9
canvasWidth: 1280
transition: slide-left
colorSchema: light
mdc: true
---

<div class="xslide dark cover" data-slide="cover">
<p class="kicker">Repo Reading · XSkill</p>
    <h1 id="cover-title">把会话经验变成<br><span class="cyan">可试用、可回滚</span>的 Skill</h1>
    <div class="cover-rule"></div>
    <p class="subtitle">从轨迹提炼，到 Git 版本，再到真实流量 Canary</p>
    <div class="cover-meta"><span class="pill">paper v4 · WIP</span><span class="pill">source · bc9bf94</span><span class="pill">wiki · bc7ac83</span></div>
    <p class="cover-date">2026-08-28 · paper reading</p>
</div>

---

<div class="xslide dark" data-slide="problem">
<p class="kicker">Why lifecycle management</p>
    <h2 id="problem-title">Skill 写出来以后，问题才开始</h2>
    <div class="problem-layout">
      <p class="problem-copy">哪个经验值得写？<br>更新后能否<span>安全上线</span>？<br>退化时怎样<span>回滚</span>？</p>
      <div class="dark-stats">
        <div class="dark-stat"><span class="number small">139</span><p>作者报告的一台重度用户机器上的 Skill 文件数</p></div>
        <div class="dark-stat"><span class="number small">26,448</span><p>仅 description listing 的字符数</p></div>
        <div class="dark-stat"><span class="number small">3.3×</span><p>相对 8,000 字符 listing budget 的溢出</p></div>
      </div>
    </div>
    <p class="source">作者单用户 inventory：paper Table 5 / §5.1。论文结论中的 13× 与 26,448÷8,000 不一致，本页使用 3.3×。</p>
    <span class="page-no">02</span>
</div>

---

<div class="xslide light" data-slide="questions">
<p class="kicker">Talk map</p>
    <h2 id="questions-title">这次只回答三个问题</h2>
    <div class="grid three questions">
      <article class="card question" v-click><h3>一段工作会话怎样变成可复用的 Skill？</h3><p>先统一日志格式，再切成单一任务，最后积累到一次版本改动。</p></article>
      <article class="card question" v-click><h3>一个 Skill 的“版本”是什么？</h3><p>完整目录的 Git commit；Main 是当前版本，Staging 是待检验版本。</p></article>
      <article class="card question" v-click><h3>真实使用怎样决定下一版？</h3><p>Canary 把小部分流量给候选版，收集 UX 证据后选择胜者。</p></article>
    </div>
    <p class="source">叙事顺序：对象定义 → 数据流 → 版本 → Canary → 证据。</p><span class="page-no">03</span>
</div>

---

<div class="xslide white" data-slide="skill-version">
<p class="kicker">Definition first</p>
    <h2 id="skill-version-title">一个 Skill 是目录；一个版本是这个目录的 Git commit</h2>
    <div class="skill-layout">
      <pre class="file-tree"><span class="root">deploy-python/</span>
├── <span class="hot">SKILL.md</span>
├── scripts/
├── references/
└── assets/</pre>
      <div class="version-stack">
        <div class="version-row draft"><strong>Baby</strong><span>第一版草稿，尚未分发</span><code>draft</code></div>
        <div class="version-row current"><strong>Main</strong><span>当前给用户使用的完整目录</span><code>6aa1e9…</code></div>
        <div class="version-row candidate"><strong>Staging</strong><span>新改动的候选目录，小流量试用</span><code>b71c42…</code></div>
      </div>
    </div>
    <p class="definition">Git 带来三件现成能力：看 diff、锁定精确 SHA、把失败候选留档。</p>
    <p class="source">source · src/xskill/skill/git.py · paper §3.2.3 / §3.3</p><span class="page-no">04</span>
</div>

---

<div class="xslide light" data-slide="objects">
<p class="kicker">Vocabulary</p>
    <h2 id="objects-title">五个对象，按产生顺序读</h2>
    <div class="object-flow">
      <article class="object"><span class="tag">Native log</span><h3>原生会话日志</h3><p>Claude、Codex 等运行时自己写出的 JSONL 或数据库记录。</p></article>
      <article class="object"><span class="tag">Bridge file</span><h3>统一会话副本</h3><p>XSkill 把不同日志转换成 Markdown 与元数据，供后续统一处理。</p></article>
      <article class="object"><span class="tag">Atom</span><h3>单一用户任务</h3><p>从长会话中切出的一段连续意图，保留原始行号范围。</p></article>
      <article class="object"><span class="tag">Candidate</span><h3>版本改动证据</h3><p>某个 Atom 对某个 Skill 有复用价值，先进入候选账本。</p></article>
      <article class="object"><span class="tag">Version</span><h3>完整 Skill 版本</h3><p>候选累计后改写整个目录，并提交到 Baby、Main 或 Staging。</p></article>
    </div>
    <p class="source">“Bridge”在本报告中专指统一后的 Markdown 会话副本；它不承担模型代理或网络转发。</p><span class="page-no">05</span>
</div>

---

<div class="xslide diagram-slide" data-slide="architecture">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/xskill-system-map.html" title="XSkill 接在 coding agent 文件两端的架构图"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="capture">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/trajectory-normalization.html" title="不同格式会话转换成统一文本的数据流图"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="distill">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/distillation-pipeline.html" title="XSkill 三 Agent 提炼流程图"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="two-scores">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/evidence-control-points.html" title="Weightscore 与 UX score 控制不同决策"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="version-state">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/skill-version-state.html" title="Baby Main Staging 与 Frozen 的状态机"></iframe>
</div>

---

<div class="xslide light" data-slide="modes">
<p class="kicker">Deployment modes</p>
    <h2 id="modes-title">Standalone 与 Team，只是部署边界不同</h2>
    <div class="grid two">
      <article class="card mode">
        <div class="mode-title"><span class="badge">1</span><h3>Standalone · 单机模式</h3></div>
        <p>采集、提炼、Git 仓库与安装都在同一台开发机。版本只影响这台机器的 agent。</p>
        <div class="mode-flow"><span>本机会话</span><b>→</b><span>本地 XSkill</span><b>→</b><span>本地 Skill 目录</span></div>
      </article>
      <article class="card mode">
        <div class="mode-title"><span class="badge">N</span><h3>Team · 团队模式</h3></div>
        <p>每台机器运行 client；中央 server 汇总会话、维护版本并给每个 client 下发精确 side 与 SHA。</p>
        <div class="mode-flow"><span>多台 client</span><b>→</b><span>中央 server</span><b>→</b><span>指定版本同步回 client</span></div>
      </article>
    </div>
    <p class="definition">“同步版本”指把某个 Skill 目录对齐到指定 Git commit，再放进 agent 的原生 discovery 目录。</p>
    <p class="source">source · team/client/daemon.py · team/shared/reconcile.py · ecosystem installers</p><span class="page-no">11</span>
</div>

---

<div class="xslide white" data-slide="canary-definition">
<p class="kicker">The central idea</p>
    <h2 id="canary-definition-title">已有 Skill 的自动更新，默认先进入 Staging</h2>
    <p class="lead">新 Skill 没有旧版本可比较：它在 Baby 完成第一版后发布为 Main。Canary 从已有 Main 的下一次更新开始。</p>
    <div class="versus">
      <article class="version-card main"><h3><span>Main · 当前版</span><span class="share">≈80%</span></h3><p class="lead">已有稳定内容，继续给大部分合适流量使用。</p><span class="sha">skill=deploy · sha=6aa1e9…</span></article>
      <span class="vs">VS</span>
      <article class="version-card staging"><h3><span>Staging · 候选版</span><span class="share">≈20%</span></h3><p class="lead">包含新经验，只在小流量上试用，尚未覆盖 Main。</p><span class="sha">skill=deploy · sha=b71c42…</span></article>
    </div>
    <p class="definition"><strong>Canary：</strong>两个不可变版本同时存在；真实任务产生的体验证据决定下一轮 Main。</p>
    <p class="source">当前默认 staging probability = 0.2。`xskill generate` 与 jam breaker 是两条显式旁路。来源：config.py / canary.py / generate_agent.py。</p><span class="page-no">12</span>
</div>

---

<div class="xslide diagram-slide" data-slide="canary-loop">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/canary-feedback-loop.html" title="新版本先接受小流量检验的 Canary 反馈循环"></iframe>
</div>

---

<div class="xslide light compact" data-slide="ux-score">
<p class="kicker">Who scores</p>
    <h2 id="ux-score-title">用户提供真实任务结果；分数由系统生成</h2>
    <p class="lead">当前实现没有星级弹窗。用户照常使用 agent，评分器读取后续交互。</p>
    <div class="compare">
      <article class="compare-panel">
        <span class="panel-tag">PAPER PROTOCOL</span><h3>行为信号公式</h3>
        <div class="formula">5 × completion<br>+ 3 × (1 − min(corrections / 3, 1))<br>+ 2 × skill attribution</div>
        <div class="code-line"><b>来源</b><span>任务完成、用户修正次数、是否实际使用该 Skill。</span></div>
        <div class="code-line"><b>状态</b><span>论文设计；当前源码没有计算这条公式。</span></div>
      </article>
      <article class="compare-panel">
        <span class="panel-tag">CURRENT SOURCE</span><h3>LLM 直接给 1–10</h3>
        <div class="formula">trajectory text<br>→ TaskAgent rubric<br>→ integer ux_score ∈ [1,10]</div>
        <div class="code-line"><b>来源</b><span>LLM 阅读轨迹，综合完成、修正和结果后选择分档。</span></div>
        <div class="code-line"><b>风险</b><span>未做人工满意度标定；排序偏差会直接改变 Canary 结果。</span></div>
      </article>
    </div>
    <p class="verdict">Canary 的反馈回路已经存在；UX 指标还不能称为经过验证的用户价值度量。</p>
    <p class="source">paper Eq. 1 / §3.3 · source agents/task_agent.py · pipeline/runner.py</p><span class="page-no">14</span>
</div>

---

<div class="xslide diagram-slide" data-slide="canary-drift">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/canary-paper-vs-code.html" title="论文 Canary 与当前实现的规则对照"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="attribution">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/canary-attribution-gap.html" title="评分版本与实际下发版本可能不一致"></iframe>
</div>

---

<div class="xslide light" data-slide="paper-evidence">
<p class="kicker">Evidence ledger</p>
    <h2 id="paper-evidence-title">论文是 WIP 系统设计；四类数字要分开读</h2>
    <div class="ledger">
      <article class="ledger-card real"><span class="kind">OBSERVED</span><h3>作者机器库存</h3><span class="bigline">139 skills</span><p>单个重度用户的真实快照。说明 catalog 压力存在，不代表总体分布。</p></article>
      <article class="ledger-card hypo"><span class="kind">HYPOTHETICAL</span><h3>FastAPI 生命周期</h3><span class="bigline">p = .038</span><p>论文明确写为说明机制的构造示例，不能当作上线结果。</p></article>
      <article class="ledger-card estimate"><span class="kind">ESTIMATED</span><h3>吞吐与收敛</h3><span class="bigline">1–2 weeks</span><p>基于假设流量和推理速度的容量推演，没有实测 wall clock。</p></article>
      <article class="ledger-card claim"><span class="kind">LATER CLAIM</span><h3>README benchmark</h3><span class="bigline">+5.58 pp</span><p>当前 README 的三项均值；仓库快照没有逐题结果和对应运行产物。</p></article>
    </div>
    <p class="source">paper review flag / §5 · current xskill README benchmark table · reanalysis-research.md</p><span class="page-no">17</span>
</div>

---

<div class="xslide white" data-slide="experiment-trace">
<p class="kicker">Observed trace</p>
    <h2 id="experiment-trace-title">旧实验沿一条 trace 检查四阶段</h2>
    <div class="trace" role="img" aria-label="Coding task、采集、学习、下游四阶段的实验结果">
      <article class="trace-step pass"><span class="step">01 · CODING</span><h3>真实任务交付</h3><span class="metric">71 passed</span><p>17 行实现、102 行测试；本次重新复跑仍为 71 passed。</p></article>
      <article class="trace-step pass"><span class="step">02 · CAPTURE</span><h3>统一会话副本</h3><span class="metric">44 / 44</span><p>Native 与 sidecar 的工具名称序列逐项相同；146 turns。</p></article>
      <article class="trace-step partial"><span class="step">03 · LEARN</span><h3>自动切分</h3><span class="metric">0 Atom</span><p>39 次成功 LLM 调用，628,205 tokens；完整会话没有提交 Atom。</p></article>
      <article class="trace-step stop"><span class="step">04 · DOWNSTREAM</span><h3>未到达</h3><span class="metric">—</span><p>Candidate、SkillEdit、重新安装与 Canary 都没有自动执行。</p></article>
    </div>
    <div class="readout"><b>READOUT</b><span>采集链保留了工具序列；学习结果未达到可用条件；这条 trace 没有检验版本演化和 Canary。</span></div>
    <p class="source">真实证据：references/claude-minimax-trial-evidence.json；复核：experiment audit。628,205 是 split pipeline token，不是 native session token。</p><span class="page-no">18</span>
</div>

---

<div class="xslide white" data-slide="judgment">
<p class="kicker">Assessment</p>
    <h2 id="judgment-title">对 XSkill 的判断</h2>
    <div class="judgment">
      <article class="card"><h3>工程骨架已存在</h3><ul><li>沿用 agent 原生 Skill loader</li><li>每个 Skill 独立 Git 仓库</li><li>Main / Staging 精确到 commit SHA</li><li>拒绝版本仍可审计和回滚</li></ul></article>
      <article class="card"><h3>Canary 是核心贡献</h3><ul><li>候选版只接小流量</li><li>真实工作继续产生反馈</li><li>旧版本始终可用</li><li>下一轮继承胜出版本</li></ul></article>
      <article class="card"><h3>当前证据仍缺四项</h3><ul><li>UX score 与用户价值的标定</li><li>实际下发 side / SHA 的可靠归因</li><li>可控误晋升率的决策规则</li><li>受控端到端效果实验</li></ul></article>
    </div>
    <p class="source">判断基于 paper v4、source bc9bf94、wiki bc7ac83 与旧实验复核；三者的时间点和口径分开记录。</p><span class="page-no">19</span>
</div>

---

<div class="xslide light compact" data-slide="next-experiment">
<p class="kicker">Minimum meaningful eval</p>
    <h2 id="next-experiment-title">下一轮先测 Canary 数据是否可信</h2>
    <div class="experiment">
      <article class="experiment-card"><span class="exp-no">EXPERIMENT A</span><h3>版本归因一致性</h3><ol><li>固定一个 Main 与一个 Staging SHA。</li><li>覆盖普通分流、管理员 override、配额修正、server restart。</li><li>记录 manifest 实际下发的 (user, skill, side, SHA)。</li><li>逐行比较 UX ledger，目标一致率 100%。</li></ol><div class="metric-row"><span class="metric-chip">primary · attribution accuracy</span><span class="metric-chip">no LLM required</span></div></article>
      <article class="experiment-card"><span class="exp-no">EXPERIMENT B</span><h3>误晋升仿真</h3><ol><li>构造同分布、Staging 略差、Staging 略优三种场景。</li><li>分数按 user / session 聚类，保留 1–10 有界分布。</li><li>比较当前均值规则、论文 Welch、用户级 permutation / sequential test。</li><li>报告误晋升率、正确晋升率与等待时长。</li></ol><div class="metric-row"><span class="metric-chip">primary · false promotion</span><span class="metric-chip">10k simulations</span></div></article>
    </div>
    <p class="definition">先确认 exposure 与评分版本一致，再讨论哪种统计检验更好。继续补跑生成模型无法回答这个前置问题。</p>
    <p class="source">实验设计来自源码归因链审计与论文 §3.3 的统计有效性限制。</p><span class="page-no">20</span>
</div>

---

<div class="xslide dark takeaways" data-slide="takeaways">
<p class="kicker">Takeaways</p>
    <h2 id="takeaways-title">四个结论</h2>
    <div class="takeaway-list">
      <div class="takeaway"><b>01</b><p>XSkill 给 Skill 目录增加了证据账本、Git 版本、分发与回滚。</p></div>
      <div class="takeaway"><b>02</b><p>Canary 是最值得保留的设计：候选版本先接小流量，真实使用决定下一轮。</p></div>
      <div class="takeaway"><b>03</b><p>当前产品使用 LLM UX 分与均值门槛；论文的行为公式和 Welch 检验没有落地。</p></div>
      <div class="takeaway"><b>04</b><p>版本归因必须先准确，后续统计才有意义。当前 team 上传协议在这里留有缺口。</p></div>
    </div>
    <p class="question-end">讨论：XSkill 现在能否证明“这条 UX 分来自用户实际拿到的那个 commit”？</p>
    <p class="source"><a href="/paper-reading/2026-08-28/skill/narratives.html">三种叙事方案</a> · <a href="/paper-reading/2026-08-28/skill/reanalysis.html">重新分析笔记</a> · ↓ 附录继续</p><span class="page-no">21</span>
</div>

---

<div class="xslide light" data-slide="appendix-drift">
<p class="appendix-label">APPENDIX A</p><h2 id="appendix-drift-title">论文、Wiki、当前源码的关键差异</h2>
    <table class="matrix"><thead><tr><th>议题</th><th>论文 v4</th><th>Wiki</th><th>当前源码 bc9bf94</th></tr></thead><tbody>
      <tr><td>UX score</td><td>5/3/2 行为公式</td><td>交互产生 1–10</td><td>TaskAgent LLM 直接给 1–10</td></tr>
      <tr><td>候选路由</td><td>embedding top-k</td><td>与已有 Skills 比较</td><td>扫描全 catalog；弱 Atom 仍须归类</td></tr>
      <tr><td>weight ≥ 10</td><td>声称约 2–3 条轨迹</td><td>有用材料累计</td><td>单个 10 或十个 1 都能触发</td></tr>
      <tr><td>Canary 样本</td><td>每侧 20</td><td>未分桶每侧 5</td><td>未分桶 5；分桶每侧总计 20</td></tr>
      <tr><td>决策</td><td>单侧 Welch，p&lt;.05</td><td>Staging 均值不低于 Main</td><td>简单/加权均值；平局晋升</td></tr>
      <tr><td>超时</td><td>30 天；正文口径不一</td><td>14 天丢弃</td><td>14 天丢弃；rejected ref 留档</td></tr>
      <tr><td>生态</td><td>5 个，含 Hermes</td><td>6 个，不含 Hermes</td><td>9 个可探测；team collector 覆盖更少</td></tr>
    </tbody></table>
    <p class="source">完整表与精确行号见 reanalysis-research.md §6。</p>
</div>

---

<div class="xslide diagram-slide" data-slide="appendix-experiment">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/experiment-cohort-trace.html" title="完整会话与 Case2-only 的实验双轨数据"></iframe>
</div>

---

<div class="xslide light" data-slide="appendix-probe">
<p class="appendix-label">APPENDIX C</p>
    <h2 id="appendix-probe-title">缩小输入后出现提交动作，边界质量仍不合格</h2>
    <div class="probe">
      <article class="probe-panel"><h3>完整会话</h3><div class="arrow-result"><div><b>2,220</b><br><span class="muted">行 · 14 个意图</span></div><span>→</span><div><b>0</b><br><span class="muted">Atom</span></div></div><p class="lead">长会话混合 pilot、TDD、权限中断与退出指令。</p></article>
      <article class="probe-panel"><h3>机械截取 Case 2</h3><div class="arrow-result"><div><b>1,040</b><br><span class="muted">行 · 7 个 user header</span></div><span>→</span><div><b>3</b><br><span class="muted">submitted Atom</span></div></div><div class="audit-list"><div class="audit-item"><span>任务级预期</span><b>1</b></div><div class="audit-item"><span>语义过切</span><b>1</b></div><div class="audit-item"><span>/exit 噪声</span><b>1</b></div></div></article>
    </div>
    <p class="definition">固定模型与工具，只收窄输入范围，提交从 0 变成 3。工具能够被调用；任务边界仍未切准。</p>
    <p class="source">这项诊断只解释 0 Atom 的边界条件，不承担 XSkill 主结论。Case 2 输出只保留汇总，无法再次独立审计。</p>
</div>

---

<div class="xslide white" data-slide="appendix-audit">
<p class="appendix-label">APPENDIX D</p><h2 id="appendix-audit-title">两处复核改变了数字口径</h2>
    <div class="grid two">
      <article class="card"><span class="number small">91 → 47</span><h3>event rows 与响应数口径不同</h3><p>44 个响应分别以 text 与 tool_use 两行保存，usage 完全重复。按 message.id 去重后，native input / output 为 3,719,340 / 11,441；旧值分别虚高 94.22% / 93.49%。</p></article>
      <article class="card"><span class="number small">1 → 315</span><h3>默认 discovery 远超目标 cohort</h3><p>目标只有 1 条 session；首次启动在 108 秒内带入 315 条无关历史 session、630 个文件、131.60 MiB。部署前要限定数据范围和预算。</p></article>
    </div>
    <p class="definition">Trace 可用于评估的前提：先定义 event、response、session、trajectory 的计数单位。</p>
    <p class="source">复算自保留的 native JSONL、Bridge sidecar 与隔离目录；详见 experiment audit。</p>
</div>

---

<div class="xslide light" data-slide="sources">
<p class="appendix-label">APPENDIX E</p><h2 id="sources-title">材料与设计源</h2>
    <div class="source-list">
      <a class="source-link" href="/paper-reading/2026-08-28/skill/reanalysis.html"><strong>重新分析笔记</strong><span>paper / source / wiki 三方核对与精确行号</span></a>
      <a class="source-link" href="/paper-reading/2026-08-28/skill/narratives.html"><strong>三种叙事方案</strong><span>生命周期、证据审计、Canary 在线评估</span></a>
      <a class="source-link" href="/paper-reading/2026-08-28/skill/references/claude-minimax-trial-evidence.json"><strong>脱敏实验摘要</strong><span>保留配置口径、阶段状态与聚合指标</span></a>
      <a class="source-link" href="https://github.com/SkillNerds/xskill/tree/bc9bf941662467ac711523e450968f2677cd230e"><strong>XSkill 源码快照</strong><span>bc9bf941662467ac711523e450968f2677cd230e</span></a>
      <a class="source-link" href="https://xskill.wiki/wiki.html"><strong>官方 Wiki</strong><span>本地镜像 commit bc7ac834…</span></a>
      <a class="source-link" href="/paper-reading/2026-08-28/skill/diagrams/canary-feedback-loop.html"><strong>图表 HTML 源</strong><span>Deck 使用 9 张自包含、可单独打开的 1280×720 图</span></a>
    </div>
    <p class="source">页面不包含模型端点、API key 或原始私有轨迹。</p>
</div>
