---
theme: default
title: XSkill：把会话经验变成可试用、可回滚的 Skill
aspectRatio: 16/9
canvasWidth: 1280
transition: slide-left
colorSchema: light
routerMode: hash
mdc: true
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
    <h2 id="questions-title">四个问题，正好组成一个闭环</h2>
    <div class="grid two questions">
      <article class="card question" v-click><h3>Session 如何采集？</h3><p>从 Agent 原生日志得到统一、可处理的会话副本。</p></article>
      <article class="card question" v-click><h3>Session 如何生成 Skill？</h3><p>服务端把长会话切成任务证据，累计后改写 Skill。</p></article>
      <article class="card question" v-click><h3>如何评估 Skill 效果？</h3><p>Main 与 Staging 并存，用真实任务反馈选择下一版。</p></article>
      <article class="card question" v-click><h3>Skill 如何注入回系统？</h3><p>服务端选择精确版本，客户端安装回本机 Agent。</p></article>
    </div>
</div>

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

<div class="xslide diagram-slide" data-slide="capture">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/trajectory-normalization.html" title="Session 从本机原生日志进入服务端会话库"></iframe>
</div>

---

<div class="xslide diagram-slide" data-slide="distill">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/distillation-pipeline.html" title="服务端三个 Agent 把 Session 变成 Skill 版本"></iframe>
</div>

---

<div class="xslide light fit" data-slide="two-scores">
<p class="kicker">Two separate decisions</p>
    <h2 id="two-scores-title">别把两个 1–10 分混在一起</h2>
    <div class="compare">
      <article class="compare-panel">
        <span class="panel-tag">是否值得改 SKILL？</span><h3>Weightscore · 复用证据</h3>
        <div class="formula">每条 Candidate：1–10<br>按 Skill 累计：Σ weightscore<br>默认达到 10 → 触发 SkillEditAgent</div>
        <div class="code-line"><b>谁给分</b><span>TaskClusterAgent</span></div>
        <div class="code-line"><b>回答</b><span>这个任务对这个 Skill 有多大复用价值？</span></div>
      </article>
      <article class="compare-panel">
        <span class="panel-tag">新版本是否更好？</span><h3>UX score · 使用结果</h3>
        <div class="formula">每个真实任务：1–10<br>记录：skill + side + commit SHA<br>样本够用 → 比较 Main / Staging</div>
        <div class="code-line"><b>谁给分</b><span>TaskAgent 根据会话结果生成</span></div>
        <div class="code-line"><b>回答</b><span>用户使用这个具体版本完成任务的体验如何？</span></div>
      </article>
    </div>
    <p class="definition">Weightscore 决定“要不要写新版本”；UX score 决定“新版本要不要成为 Main”。</p>
</div>

---

<div class="xslide diagram-slide" data-slide="version-state">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/skill-version-state.html" title="Skill 的 Git 版本管理"></iframe>
</div>

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

<div class="xslide diagram-slide" data-slide="canary-loop">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/canary-feedback-loop.html" title="真实任务反馈驱动 Skill 版本演进"></iframe>
</div>

---

<div class="xslide light fit" data-slide="ux-score">
<p class="kicker">Evaluation data path</p>
    <h2 id="ux-score-title">用户只需正常工作；系统把结果归到实际版本</h2>
    <div class="grid four feedback-steps">
      <article class="card"><span class="feedback-no">01</span><h3>记录曝光版本</h3><p>Client 安装时记下 Skill、Main/Staging 与 commit SHA。</p></article>
      <article class="card"><span class="feedback-no">02</span><h3>完成真实任务</h3><p>用户继续使用原来的 Coding Agent，不需要切换版本。</p></article>
      <article class="card"><span class="feedback-no">03</span><h3>生成 UX score</h3><p>TaskAgent 根据完成情况、修正和结果给出 1–10 分。</p></article>
      <article class="card"><span class="feedback-no">04</span><h3>按版本比较</h3><p>分数进入对应 SHA 的账本；样本达到门槛后裁决。</p></article>
    </div>
    <p class="definition">这里没有“请给 Skill 打五星”的弹窗：用户行为提供反馈，版本归因把反馈变成可比较的数据。</p>
</div>

---

<div class="xslide diagram-slide" data-slide="return-flow">
<iframe class="diagram" src="/paper-reading/2026-08-28/skill/skill-return-flow.html" title="服务端选择版本，客户端安装回本机 Agent"></iframe>
</div>

---

<div class="xslide dark takeaways" data-slide="takeaways">
<p class="kicker">The whole loop</p>
    <h2 id="takeaways-title">四个问题，四个答案</h2>
    <div class="takeaway-list">
      <div class="takeaway"><b>01</b><p>采集：Client 把本机原生日志转换为统一会话文本；Team 模式再脱敏上传。</p></div>
      <div class="takeaway"><b>02</b><p>生成：服务端三个 Agent 把 Session 收窄成 Candidate，累计后修改团队 Skill Git 仓库。</p></div>
      <div class="takeaway"><b>03</b><p>评估：Main 与 Staging 同时服务真实任务，UX score 决定候选版本是否胜出。</p></div>
      <div class="takeaway"><b>04</b><p>回注：服务端下发 side + SHA；每台 Client 对齐本地副本并安装到自己的 Agent。</p></div>
    </div>
    <p class="question-end">贯穿全链路的主键不是“最新版”，而是一个不可变的 Git commit。</p>
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
      <a class="source-link" href="/paper-reading/2026-08-28/skill/diagrams/xskill-system-map.html"><strong>图表 HTML</strong><span>Deck 使用的 6 张可独立打开图表</span></a>
    </div>
</div>
