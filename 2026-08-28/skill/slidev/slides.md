---
theme: default
title: XSkill：把会话经验变成可试用、可回滚的 Skill
aspectRatio: 16/9
canvasWidth: 1280
transition: slide-left
colorSchema: light
routerMode: hash
hideInToc: true
---

<div class="xslide dark cover" data-slide="cover">
<p class="kicker">XSkill · Paper Reading</p>
    <h1 id="cover-title">把会话经验变成<br><span class="cyan">可试用、可回滚</span>的 Skill</h1>
    <div class="cover-rule"></div>
    <p class="subtitle">从 Session 采集，到 Skill 生成、金丝雀评估与版本回注</p>
    <div class="cover-meta"><span class="pill">Session → Skill</span><span class="pill">Git versions</span><span class="pill">Canary feedback</span></div>
    <p class="cover-date">2026-08-28 · paper reading</p>
</div>

---

<div class="xslide dark" data-slide="problem">
<p class="kicker">Why lifecycle management</p>
    <h2 id="problem-title">Skill 写出来以后，问题才开始</h2>
    <div class="problem-layout">
      <p class="problem-copy">哪个经验值得写？<br>新版本是否<span>真的更好</span>？<br>退化时怎样<span>安全回滚</span>？</p>
      <div class="dark-stats">
        <div class="dark-stat"><span class="number small">139</span><p>论文记录的一台重度用户机器上的 Skill 文件数</p></div>
        <div class="dark-stat"><span class="number small">26,448</span><p>只列出这些 Skill description 所需的字符数</p></div>
        <div class="dark-stat"><span class="number small">3.3×</span><p>相对 8,000 字符 listing budget 的溢出</p></div>
      </div>
    </div>
</div>

---

<div class="xslide light fit" data-slide="questions">
<p class="kicker">Talk map</p>
    <h2 id="questions-title">四个机制、一个案例与附录</h2>
    <div class="chapter-toc"><Toc :columns="2" :maxDepth="1" /></div>
</div>

---
layout: full
hideInToc: true
---

<div class="xslide diagram-slide" data-slide="architecture">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/xskill-system-map.html" title="XSkill 本地端与服务端的完整闭环"></iframe>
</div>

---

<div class="xslide light fit" data-slide="objects">
<p class="kicker">Read the architecture</p>
    <h2 id="objects-title">五个对象是一条证据链</h2>
    <div class="object-flow">
      <article class="object"><span class="tag">Native log</span><h3>原生会话日志</h3><p>Claude、Codex 等 Agent 写出的 JSONL 或数据库记录。</p></article>
      <article class="object"><span class="tag">Bridge</span><h3>统一会话文本</h3><p>把不同日志转成相同的 Markdown 结构，供服务端统一读取。</p></article>
      <article class="object"><span class="tag">Atom</span><h3>单一任务片段</h3><p>长会话里一个连续用户意图，保留原文位置与结果分。</p></article>
      <article class="object"><span class="tag">Candidate</span><h3>候选证据</h3><p>这个任务对某个 Skill 有复用价值，先进入该 Skill 的证据账本。</p></article>
      <article class="object"><span class="tag">Version</span><h3>Skill 版本</h3><p>完整 Skill 目录的一次 Git commit，可精确分发和回滚。</p></article>
    </div>
    <p class="definition"><strong>Bridge</strong> 只是“统一格式的会话副本”；<strong>Candidate</strong> 只是“还没写进 Skill 的证据”。</p>
</div>

---
layout: section
class: xskill-section
title: 01 · Session 如何采集？
level: 1
routeAlias: chapter-capture
---

<div class="chapter-content" data-slide="chapter-capture">
<p class="kicker">Chapter 01 · Capture</p>
    <h1>Session 如何进入 XSkill？</h1>
    <p class="lead">先把不同 Agent 的原生日志变成同一种、可追溯的会话文本。</p>
    <span class="chapter-number" aria-hidden="true">01</span>
    <div class="chapter-track"><Link to="chapter-capture" class="active">01 采集</Link><Link to="chapter-generate">02 生成</Link><Link to="chapter-evaluate">03 评估</Link><Link to="chapter-return">04 回注</Link><Link to="chapter-example">05 案例</Link><Link to="chapter-appendix">附录</Link></div>
</div>

---
layout: full
title: 会话采集链路
level: 2
---

<div class="xslide diagram-slide" data-slide="capture">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/trajectory-normalization.html" title="Session 从本机原生日志进入服务端会话库"></iframe>
</div>

---
layout: section
class: xskill-section
title: 02 · Session 如何生成 Skill？
level: 1
routeAlias: chapter-generate
---

<div class="chapter-content" data-slide="chapter-generate">
<p class="kicker">Chapter 02 · Generate</p>
    <h1>Session 如何变成 Skill？</h1>
    <p class="lead">三个服务端 Agent 把长会话逐步收窄为候选证据，再提交新的 Git 版本。</p>
    <span class="chapter-number" aria-hidden="true">02</span>
    <div class="chapter-track"><Link to="chapter-capture">01 采集</Link><Link to="chapter-generate" class="active">02 生成</Link><Link to="chapter-evaluate">03 评估</Link><Link to="chapter-return">04 回注</Link><Link to="chapter-example">05 案例</Link><Link to="chapter-appendix">附录</Link></div>
</div>

---
title: Algorithm 1 · 原始伪代码
level: 2
---

<div class="xslide light fit algorithm-page" data-slide="algorithm-1-code">
<p class="kicker">Paper · Algorithm 1</p>
    <h2>Three-Agent Skill Distillation Pipeline</h2>
    <pre class="paper-algorithm"><code>Require: Trajectory delta δ, skill catalog C, threshold θ = 10
Ensure: Updated skill repository
 1  Stage 1: TaskAgent — Decompose δ into AtomTasks
 2  A ← TaskAgent.decompose(δ)   {Segment by semantic boundaries}
 3  for each atom a ∈ A do
 4    Assign a.ux_score ← fUX(a) {Behavioral UX scoring (Eq. 1)}
 5    Record a.intent, a.tags, a.used_skills
 6  end for
 8  Stage 2: TaskClusterAgent — Route atoms to skills
 9  for each atom a ∈ A do
10    (action, skill, w) ← TCA.route(a, C)
11    if action = CREATE then
12      Initialize new skill folder in “baby” state with a as first evidence
13      C ← C ∪ {skill}
14    else if action = ROUTE then
15      Append (a, w) to skill.candidates.yml
16    else if action = RECLASSIFY then
17      Move a from current skill to better-fit target
18    end if
19  end for
21  Stage 3: SkillEditAgent — Produce or update skills
22  for each skill s ∈ C do
23    if Σ(a,w)∈s.candidates w ≥ θ and canary guards pass then
24      SKILL.md ← SEA.generate(s.candidates, s.existing_body)
25      if s.branch = baby then
26        git commit to main       {First public version}
27      else
28        git commit to staging    {Canary candidate}
29      end if
30    end if
31  end for</code></pre>
    <p class="algorithm-note"><strong>ux_score：</strong>TaskAgent 从任务完成、用户修正与 Skill 归因提取的 1–10 分；它随 Atom 保存，留给后面的金丝雀版本比较。</p>
</div>

---
layout: full
title: Algorithm 1 · 蒸馏循环
level: 2
---

<div class="xslide diagram-slide" data-slide="distill">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/algorithm1-distillation-loop.html" title="Algorithm 1 的三 Agent 蒸馏与跨 Session 证据循环"></iframe>
</div>

---
layout: full
title: 三个 Agent 的输出如何进入版本管理
level: 2
---

<div class="xslide diagram-slide" data-slide="version-state">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/skill-version-state.html" title="Skill 的 Git 版本管理"></iframe>
</div>

---
layout: section
class: xskill-section
title: 03 · 如何评估 Skill 效果？
level: 1
routeAlias: chapter-evaluate
---

<div class="chapter-content" data-slide="chapter-evaluate">
<p class="kicker">Chapter 03 · Evaluate</p>
    <h1>如何判断新版本更好？</h1>
    <p class="lead">Main 与 Staging 同时接受真实任务；归到具体 commit 的 UX score 决定胜负。</p>
    <span class="chapter-number" aria-hidden="true">03</span>
    <div class="chapter-track"><Link to="chapter-capture">01 采集</Link><Link to="chapter-generate">02 生成</Link><Link to="chapter-evaluate" class="active">03 评估</Link><Link to="chapter-return">04 回注</Link><Link to="chapter-example">05 案例</Link><Link to="chapter-appendix">附录</Link></div>
</div>

---
title: Main 与 Staging
level: 2
---

<div class="xslide white fit" data-slide="canary-definition">
<p class="kicker">Canary rollout</p>
    <h2 id="canary-definition-title">更新已有 Skill 时，先保留两个版本</h2>
    <p class="lead">EditAgent 把新改动提交为 Staging，不覆盖 Main。服务端在不同 Client 的清单里写入不同的 side 与 commit SHA。</p>
    <div class="versus">
      <article class="version-card main"><h3><span>Main · 当前版</span><span class="share">≈80%</span></h3><p class="lead">继续服务大部分合适用户；它也是失败时的安全退路。</p><span class="sha">deploy-python · 6aa1e9…</span></article>
      <span class="vs">VS</span>
      <article class="version-card staging"><h3><span>Staging · 候选版</span><span class="share">≈20%</span></h3><p class="lead">只交给小部分合适用户；真实任务会产生独立反馈。</p><span class="sha">deploy-python · b71c42…</span></article>
    </div>
    <p class="definition"><strong>金丝雀的巧妙处：</strong>候选版不能直接覆盖 Main；它必须先在真实任务中与 Main 比较。</p>
</div>

---
title: Algorithm 2 · 原始伪代码
level: 2
---

<div class="xslide light fit algorithm-page" data-slide="algorithm-2-code">
<p class="kicker">Paper · Algorithm 2</p>
    <h2>Canary Evaluation Protocol</h2>
    <pre class="paper-algorithm"><code>Require: Skill s with new staging commit, user set U, routing fraction
         ρ ∈ (0, 0.5], minimum samples nmin = 20,
         significance level α = 0.05, timeout Tmax days
Ensure: Decision ∈ {PROMOTE, FREEZE, TIMEOUT}
 1  Guard conditions:
 2    Assert no active staging branch for s
      (one canary at a time; new candidates queue)
 3    Assert ≥ 1 real side=main UX score exists
      (baseline established)
 5  User routing:
 6    Route ceil(ρ|U|) users to staging; remainder to main
 8  Score collection:
 9    while |Smain| &lt; nmin or |Sstaging| &lt; nmin do
10      if elapsed &gt; Tmax then
11        return TIMEOUT (freeze staging, retain for audit)
12      end if
13      Collect UX scores si from AtomTasks using skill s
14      Attribute each si to Smain or Sstaging based on user's branch
15    end while
17  Statistical comparison:
18    Compute s̄main, s̄staging, σmain, σstaging
19    Perform one-sided Welch's t-test: H0: μstaging ≤ μmain
20    if p &lt; α and s̄staging &gt; s̄main then
21      Merge staging → main
22      return PROMOTE
23    else
24      Freeze staging (retain for audit, hide from distribution)
25      return FREEZE
26    end if</code></pre>
    <p class="algorithm-note"><strong>规范冲突：</strong>Algorithm 2 超时即冻结；正文另述用 Δmin=0.5 做均值 fallback。两者不能同时成立。</p>
</div>

---
layout: full
title: Algorithm 2 · 金丝雀协议
level: 2
---

<div class="xslide diagram-slide" data-slide="canary-loop">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/algorithm2-canary-protocol.html" title="Algorithm 2 的用户分流、样本门槛、统计检验与版本裁决"></iframe>
</div>

---
title: UX score 如何归到具体版本
level: 2
---

<div class="xslide light fit" data-slide="ux-score">
<p class="kicker">Evaluation data path</p>
    <h2 id="ux-score-title">UX score 衡量一个版本在真实任务中的使用结果</h2>
    <div class="grid four feedback-steps">
      <article class="card"><span class="feedback-no">01</span><h3>记录曝光版本</h3><p>Client 安装时记下 Skill、Main/Staging 与 commit SHA。</p></article>
      <article class="card"><span class="feedback-no">02</span><h3>完成真实任务</h3><p>用户继续使用原来的 Coding Agent，不需要切换版本。</p></article>
      <article class="card"><span class="feedback-no">03</span><h3>生成 UX score</h3><p>TaskAgent 根据完成情况、修正和结果给出 1–10 分。</p></article>
      <article class="card"><span class="feedback-no">04</span><h3>按版本比较</h3><p>分数进入对应 SHA 的账本；样本达到门槛后裁决。</p></article>
    </div>
    <p class="definition"><strong>UX score：</strong>TaskAgent 根据任务完成、用户修正与 Skill 归因生成 1–10 分；它只用于比较 Main / Staging，不参与生成新版本的证据门槛。</p>
</div>

---
layout: full
title: XSkill 与其他系统的设计分界线
level: 2
---

<div class="xslide diagram-slide" data-slide="landscape-comparison">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/xskill-landscape-differences.html" title="XSkill 在原生交付、金丝雀评估和 Git 版本控制上的设计差异"></iframe>
</div>

---
layout: section
class: xskill-section
title: 04 · Skill 如何注入回系统？
level: 1
routeAlias: chapter-return
---

<div class="chapter-content" data-slide="chapter-return">
<p class="kicker">Chapter 04 · Distribute</p>
    <h1>Skill 如何回到用户的 Agent？</h1>
    <p class="lead">服务端只选择版本；每台 Client 自己对齐 commit，并安装到本机 Skill 目录。</p>
    <span class="chapter-number" aria-hidden="true">04</span>
    <div class="chapter-track"><Link to="chapter-capture">01 采集</Link><Link to="chapter-generate">02 生成</Link><Link to="chapter-evaluate">03 评估</Link><Link to="chapter-return" class="active">04 回注</Link><Link to="chapter-example">05 案例</Link><Link to="chapter-appendix">附录</Link></div>
</div>

---
layout: full
title: 客户端回注流程
level: 2
---

<div class="xslide diagram-slide" data-slide="return-flow">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/skill-return-flow.html" title="服务端选择版本，客户端安装回本机 Agent"></iframe>
</div>

---
layout: section
class: xskill-section
title: 05 · 真实案例
level: 1
routeAlias: chapter-example
---

<div class="chapter-content" data-slide="chapter-example">
<p class="kicker">Chapter 05 · Worked Example</p>
    <h1>真实案例</h1>
    <p class="lead">FastAPI Deployment Skill 生命周期；过程具体，效果数字为论文构造。</p>
    <span class="chapter-number" aria-hidden="true">05</span>
    <div class="chapter-track"><Link to="chapter-capture">01 采集</Link><Link to="chapter-generate">02 生成</Link><Link to="chapter-evaluate">03 评估</Link><Link to="chapter-return">04 回注</Link><Link to="chapter-example" class="active">05 案例</Link><Link to="chapter-appendix">附录</Link></div>
</div>

---
layout: full
title: FastAPI Deployment Skill 生命周期
level: 2
---

<div class="xslide diagram-slide" data-slide="fastapi-lifecycle">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/fastapi-lifecycle-swimlane.html" title="FastAPI Deployment Skill 从本地会话到 staging 晋升的构造案例"></iframe>
</div>

---

<div class="xslide dark takeaways" data-slide="takeaways">
<p class="kicker">Summary</p>
    <h2 id="takeaways-title">总结</h2>
    <div class="takeaway-list">
      <div class="takeaway"><b>01</b><p>采集：Client 把本机原生日志转换为统一会话文本；Team 模式再脱敏上传。</p></div>
      <div class="takeaway"><b>02</b><p>生成：服务端三个 Agent 把 Session 收窄成 Candidate，累计后修改团队 Skill Git 仓库。</p></div>
      <div class="takeaway"><b>03</b><p>评估：Main 与 Staging 同时服务真实任务，UX score 决定候选版本是否胜出。</p></div>
      <div class="takeaway"><b>04</b><p>回注：服务端下发 side + SHA；每台 Client 对齐本地副本并安装到自己的 Agent。</p></div>
    </div>
    <p class="question-end">贯穿全链路的主键不是“最新版”，而是一个不可变的 Git commit。</p>
</div>

---
layout: section
class: xskill-section
title: 附录
level: 1
routeAlias: chapter-appendix
---

<div class="chapter-content" data-slide="chapter-appendix">
<p class="kicker">Appendix</p>
    <h1>附录</h1>
    <p class="lead">运行模式、论文与实现差异、参考材料。</p>
    <span class="chapter-number" aria-hidden="true">A</span>
    <div class="chapter-track"><Link to="chapter-capture">01 采集</Link><Link to="chapter-generate">02 生成</Link><Link to="chapter-evaluate">03 评估</Link><Link to="chapter-return">04 回注</Link><Link to="chapter-example">05 案例</Link><Link to="chapter-appendix" class="active">附录</Link></div>
</div>

---

<div class="xslide light fit" data-slide="appendix-modes">
<p class="appendix-label">APPENDIX A</p>
    <h2 id="appendix-modes-title">Standalone 与 Team 的区别</h2>
    <div class="grid two">
      <article class="card mode">
        <div class="mode-title"><span class="badge">1</span><h3>Standalone · 单机模式</h3></div>
        <p>采集、三个 Agent、Git 仓库和安装都在同一台开发机；版本只影响这台机器。</p>
        <div class="mode-flow"><span>本机会话</span><b>→</b><span>本地 XSkill</span><b>→</b><span>本机 Agent</span></div>
      </article>
      <article class="card mode">
        <div class="mode-title"><span class="badge">N</span><h3>Team · 团队模式</h3></div>
        <p>Client 负责采集和安装；服务端运行三个 Agent、维护共享 Git 仓库，并为每个 Client 选择版本。</p>
        <div class="mode-flow"><span>多台 Client</span><b>→</b><span>中央服务端</span><b>→</b><span>逐 Client 回注</span></div>
      </article>
    </div>
    <p class="definition">正文架构图采用 Team 模式，因为它最能说明“服务端共享学习、客户端各自安装”的边界。</p>
</div>

---

<div class="xslide light fit" data-slide="appendix-implementation-notes">
<p class="appendix-label">APPENDIX B</p><h2 id="appendix-notes-title">论文描述与仓库实现的差异</h2>
    <table class="matrix"><thead><tr><th>环节</th><th>论文描述</th><th>仓库实现</th></tr></thead><tbody>
      <tr><td>UX score</td><td>任务完成、用户修正、Skill 归因组成行为公式</td><td>TaskAgent 阅读会话后直接给 1–10 分</td></tr>
      <tr><td>候选路由</td><td>Embedding 检索 top-k Skill</td><td>TaskClusterAgent 对比可见 Skill catalog</td></tr>
      <tr><td>触发编辑</td><td>候选权重累计达到 10</td><td>同样使用 10；该门槛是 SkillEditAgent 配置</td></tr>
      <tr><td>Canary 决策</td><td>每侧 20 个样本后做单侧 Welch 检验</td><td>达到配置样本门槛后比较均值</td></tr>
      <tr><td>候选超时</td><td>30 天口径</td><td>Staging 默认 14 天未胜出则丢弃</td></tr>
    </tbody></table>
</div>

---

<div class="xslide light fit" data-slide="sources">
<p class="appendix-label">APPENDIX C</p><h2 id="sources-title">材料</h2>
    <div class="source-list">
      <a class="source-link" href="https://github.com/SkillNerds/xskill/blob/bc9bf941662467ac711523e450968f2677cd230e/paper/xskill_v4.pdf"><strong>XSkill paper v4</strong><span>论文 PDF</span></a>
      <a class="source-link" href="https://github.com/SkillNerds/xskill/tree/bc9bf941662467ac711523e450968f2677cd230e"><strong>XSkill repository</strong><span>源码快照 bc9bf94</span></a>
      <a class="source-link" href="https://xskill.wiki/wiki.html"><strong>XSkill Wiki</strong><span>官方使用与部署文档</span></a>
      <a class="source-link" href="/paper-reading/2026-08-28/skill/diagrams/xskill-system-map.html"><strong>图表 HTML</strong><span>Deck 使用的 8 张可独立打开图表</span></a>
    </div>
</div>
