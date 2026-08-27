# xskill: A Linear Source Walkthrough

*2026-08-20T06:43:15Z by Showboat 0.6.1*
<!-- showboat-id: 26395772-73d0-4c96-b538-9b6840e467e1 -->

This walkthrough follows one real unit of experience from an agent session to a distributed, versioned Skill. It is deliberately causal rather than directory-ordered: every section answers “what happens next, who owns that transition, and what remains durable if the process dies?”

Scope: the production package under `src/xskill`. Tests, scripts, papers, and historical design documents are useful corroboration but are not needed to understand the runtime. The source is large because it contains platform adapters, safety code, control-plane APIs, and recovery paths; the architectural spine is much smaller.

The most useful invariant to keep in mind is that xskill has three kinds of truth:

- files hold source content: normalized trajectories, atom JSON, candidate YAML, and Skill files;
- SQLite holds scheduling, attribution, telemetry, and read projections;
- each native Skill has its own Git repository, whose branches are the version state machine.

The commands below are executable evidence captured by Showboat, not hand-copied excerpts.

```bash
find src/xskill -type f -name '*.py' -print0 | xargs -0 wc -l | tail -n 1
```

```output
  63825 total
```

## 1. Package boundary and entry point

The package is installed as the `xskill` console command, pointing at `xskill.cli:main`. `python -m xskill` is an equivalent entry used especially by background services on Windows. The dependency list hints at the subsystem boundaries: OpenAI-compatible inference, FastAPI/Uvicorn, Git through Dulwich, SQLite in the standard library, BM25 plus NumPy for retrieval, and optional Milvus for a larger vector index.

The top-level `XSkill` class is a narrow façade. It loads configuration, opens the Registry, points a `SkillRepo` at the configured skill root, lazily constructs LLM/embedding clients, and starts the API application for `serve`. Most thin-client commands intentionally bypass this façade so they can work without local model credentials.

```bash
sed -n '1,75p' pyproject.toml
```

```output
[build-system]
requires = ["setuptools>=68.0", "setuptools-scm>=8"]
build-backend = "setuptools.build_meta"

[project]
name = "xskill"
dynamic = ["version"]
description = "Distill reusable Skills from AI Agent execution trajectories"
readme = "README.md"
requires-python = ">=3.9"
license = { text = "MIT" }
authors = [{ name = "SkillNerds", email = "370025263@qq.com" }]
keywords = ["agent", "skill", "trajectory", "llm", "rag", "self-evolving"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Operating System :: POSIX :: Linux",
    "Operating System :: MacOS",
    "Operating System :: Microsoft :: Windows",
    "Programming Language :: Python :: 3 :: Only",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries :: Python Modules",
]
dependencies = [
    "pyyaml>=6.0",
    "numpy>=1.24",
    "openai>=1.0",
    "agno",
    "tqdm",
    "httpx",
    "fastapi",
    "python-multipart",
    "uvicorn",
    "sse-starlette",
    "packaging>=23",
    "rank-bm25>=0.2",
    # team 模式上传前的轨迹脱敏（team/client/redact.py）。脱敏是上传链路的
    # 强制环节，故列为主依赖而非 optional extra——缺库就该 import 失败。
    "detect-secrets>=1.5",
    # 纯 Python git 实现，替代系统 ``git`` 二进制——容器/受限环境（如
    # ``python:3.9-slim``）默认没装 git，SkillEditAgent 等链路又要跑 git
    # init/commit/branch；走 dulwich 后零依赖系统 git。
    "dulwich>=0.21",
    # DeepSeek Harness (dsh) 默认把会话写成 zstd 帧序列（session.jsonl.zstd），
    # 采集时需要解码。列为主依赖（维护者决定，PR #243）：dsh 是重要生态，
    # 默认安装就应能收到它的轨迹；zstandard 约 1.8 MB，Python 3.9–3.13 各
    # 平台均有预编译包，不存在「部分环境装不上而阻断客户端自动更新」的风险
    # （那类体积大、平台覆盖不全的库应走 optional extra，见下）。
    "zstandard>=0.21",
    # Python 3.9 上 pydantic 需要它来求值 `X | Y` 联合注解（PEP 604 语法
    # 3.10 才原生支持）；3.10+ 不安装。
    "eval_type_backport; python_version < '3.10'",
]

[project.optional-dependencies]
# 嵌入式向量索引（Milvus Lite）。非必需：缺省走 numpy / 内存 fallback，
# 大库上会更慢。team server 建议：pip install 'xskill[milvus]'。
# 不进主依赖——pymilvus 体积大、部分环境装不上，曾阻断 client 自动更新。
milvus = [
    "pymilvus>=2.4.2",
]
dev = [
    "pytest>=7",
    "pytest-timeout>=2",
    "pytest-asyncio>=0.21",
    "pytest-bdd>=8,<9",
    # Known CI flakes (SQLite WAL lock / short TTL cache race) use
    # ``@pytest.mark.flaky`` from this plugin; see marked tests below.
    "pytest-rerunfailures>=14",
    "requests>=2",
```

```bash
sed -n '2222,2310p' src/xskill/cli.py
```

```output
def main() -> int:
    parser = build_parser()
    args = parser.parse_args()
    if not args.command:
        parser.print_help()
        return 1

    set_overrides(debug=args.debug, quiet=args.quiet)
    _setup_logging(args.debug, args.quiet, command=args.command)

    if args.command == "registry" and args.registry_action in ("add", "remove"):
        if not args.path:
            parser.error(f"path is required for 'registry {args.registry_action}'")

    # init 一站式引导：装 skill + connect，同样是瘦客户端侧，不碰 config.yaml。
    if args.command == "init":
        return cmd_init(args)

    # connect 是瘦客户端：不读 config.yaml / 不需要 llm.api_key / 不构造 XSkill 门面
    if args.command == "connect":
        return cmd_connect(args)

    # start/stop/status 管理 connect 常驻任务——同样是瘦客户端侧，不碰 config.yaml。
    if args.command == "start":
        return cmd_start(args)
    if args.command == "stop":
        return cmd_stop(args)
    if args.command == "status":
        return cmd_status(args)
    if args.command == "update":
        return cmd_update(args)
    if args.command == "dashboard":
        return cmd_dashboard(args)

    # skillhub 搜索/下载/上传是瘦客户端侧（走 team server），不碰 config.yaml。
    if args.command == "search":
        return cmd_search(args)
    if args.command == "download":
        return cmd_download(args)
    if args.command == "upload":
        return cmd_upload(args)
    if args.command == "generate":
        return cmd_generate(args)
    if args.command == "import":
        return cmd_import(args)

    # stats 只读 registry，不需要 config.yaml / llm.api_key / facade
    if args.command == "stats":
        return cmd_stats(args)

    # read / rebuild 只动 registry + 文件，不需要 llm.api_key / facade——
    # 重跑由运行中的 watcher 完成，本命令只做"重置/桥接"。
    if args.command == "read":
        return cmd_read(args, None)
    if args.command == "rebuild":
        return cmd_rebuild(args, None)

    # team 客户端的 `registry list`：本机是 client（有 team_client.json）且没有
    # standalone 数据（watch_dirs 为空）时，改走现算视图。放在 config/facade
    # 之前——纯客户端没 config.yaml 也能直接看。standalone/server 机（watch_dirs
    # 非空）走原路，不受影响（哪怕本机也存了 team_client.json）。
    if args.command == "registry" and args.registry_action == "list":
        from xskill.config import get_team_client_state_path
        if (get_team_client_state_path().is_file()
                and _standalone_watch_dir_count() == 0):
            return cmd_registry_list_client()

    # 首次运行 auto-init：serve / registry 都需要 config.yaml。
    # 不存在就写一份模板并要求用户填 key 后重跑——比直接抛 traceback 友好。
    from xskill.config import CONFIG_PATH, ensure_config_exists
    if not ensure_config_exists():
        print(
            f"\n  Created a config template at {CONFIG_PATH}\n"
            f"  Edit it — fill in llm.api_key and embedding.api_key — "
            f"then run `xskill {args.command}` again.\n",
            file=sys.stderr,
        )
        return 0

    from xskill import XSkill
    xskill = XSkill()

    handler = {
        "serve":    cmd_serve,
        "registry": cmd_registry,
    }.get(args.command)
    return handler(args, xskill) if handler else (parser.print_help() or 1)


```

```bash
sed -n '23,70p' src/xskill/core.py
```

```output
class XSkill:
    """xskill 顶层门面。

    用法：
        from xskill import XSkill
        xskill = XSkill()                       # 默认 ~/.xskill/config.yaml
        xskill = XSkill(config_path=Path(...))  # 显式

        # 检索
        hits = xskill.search_skills("django form")
        hits = xskill.search_trajectories("query")

        # daemon
        xskill.serve(host="0.0.0.0", port=8000)

        # 主动 UX 打分（维护性，watcher 会自动跑）
        xskill.score_trajectory_ux(traj)

        # 子系统访问
        xskill.registry.list()
        xskill.skill_repo["fix-foo"]
    """

    def __init__(self, config_path: Optional[Path] = None):
        self.config = load_config(config_path)
        self.registry = Registry()
        self.skill_repo = SkillRepo(get_skill_dir(), registry=self.registry)
        self._llm = None
        self._embed = None

    # ─── lazy LLM / embed clients ──────────────────────────────
    @property
    def llm(self):
        if self._llm is None:
            from xskill.utils.llm import create_llm_client
            self._llm = create_llm_client(self.config)
        return self._llm

    @property
    def embed(self):
        if self._embed is None:
            from xskill.utils.llm import create_embed_client
            self._embed = create_embed_client(self.config)
        return self._embed

    # ─── 检索（跨所有 registry）─────────────────────────────────
    def search_trajectories(self, query: str, top_k: int = 5,
                            min_similarity: float = 0.0) -> list[TrajectoryHit]:
```

## 2. Configuration resolves the physical layout

All normal standalone/server state lives under `~/.xskill`. `config.yaml` supplies the skill root, two model endpoints, four worker-pool sizes, ingestion settling rules, canary thresholds, team slot counts, recommendation settings, and dashboard policy. Runtime normalization validates old and new configuration shapes before consumers see them.

The important locations are derived centrally rather than passed loosely through the code:

| Durable object | Default location | Owner |
|---|---|---|
| configuration | `~/.xskill/config.yaml` | `config.py` |
| pipeline/control DB | `~/.xskill/registry.db` | `pipeline.registry` |
| native Skill repositories | `~/.xskill/skill/<name>/` | `skill.*` |
| normalized local trajectories | `~/.xskill/*_sessions/` | ecosystem ingesters |
| team trajectories | `~/.xskill/team_trajectories/clients/<user>/sessions/` | team server |
| worker traces/status | `~/.xskill/logs/`, status JSON files | workers/API |
| third-party catalog | configured `skillhub.dir` | `recommend.skillhub` |

Configuration is snapshotted by long-lived model clients, while explicitly marked controls—canary probability, recommendation counts, dashboard settings—are read from the shared config dictionary on each relevant request.

```bash
sed -n '1,65p' src/xskill/config.py
```

```output
"""
config.py — 全局路径与配置加载
═════════════════════════════════════
统一从 ~/.xskill/ 读取；无 cwd fallback、无环境变量 fallback、无 ~/.aikey fallback。
缺失即抛异常。
"""

from __future__ import annotations

import copy
import hashlib
import json
import logging
import math
from pathlib import Path
from typing import Optional

import yaml

logger = logging.getLogger("xskill.config")

# ─── 默认根路径 ─────────────────────────────────────────────────
XSKILL_HOME = Path.home() / ".xskill"
CONFIG_PATH = XSKILL_HOME / "config.yaml"
LOGS_DIR = XSKILL_HOME / "logs"

_config: dict = {}
_overrides: dict = {}

DEFAULT_AGENT_WORKER_POOLS = {
    "split": {"workers": 24, "llm_weight": 6},
    "cluster": {"workers": 8, "batch_size": 8, "llm_weight": 3},
    "edit": {"workers": 4, "batch_size": 5, "llm_weight": 1},
    "embed": {"workers": 4},
}
DEFAULT_LLM_RATE_LIMIT = {
    "rpm": 240,
    "request_burst": 8,
    "max_inflight": 8,
}
DEFAULT_EMBEDDING_MAX_INFLIGHT = 4


def set_overrides(**kwargs):
    """CLI flag 覆盖。仅 debug / quiet 两个保留。"""
    for k, v in kwargs.items():
        if v is not None:
            _overrides[k] = v


# 首次运行 auto-init 写出的配置模板。这是配置格式的**唯一真源**——
# 不再单独维护 examples/config.yaml.example，避免两份漂移。
CONFIG_TEMPLATE = """\
# xskill config — fill in the api keys below, then run `xskill serve` again.
#
# xskill does NOT read environment variables or any key file. Missing required
# fields (llm.api_key / embedding.api_key) raise loudly — no silent fallback.

# ===== Skill repository =====
skill_dir: ~/.xskill/skill            # the single global skill repo
interests: []                         # optional top-level interest filter;
                                      # non-empty list enables TaskAgent filtering

# ===== LLM (generation / scoring / chat) =====
# Any OpenAI-compatible chat-completions endpoint works (DeepSeek, OpenAI,
```

```bash
sed -n '884,1045p' src/xskill/config.py
```

```output
def get_skill_dir(
    config_data: Optional[dict] = None,
    *,
    xskill_home: Optional[Path] = None,
) -> Path:
    """skill_dir: config.yaml 字段；默认 ~/.xskill/skill/"""
    config_source = config_data if config_data is not None else get_config()
    state_root = (
        Path(xskill_home) if xskill_home is not None else XSKILL_HOME
    ).expanduser().resolve()
    raw_skill_dir = config_source.get("skill_dir")
    if raw_skill_dir is not None and (
        not isinstance(raw_skill_dir, str) or not raw_skill_dir.strip()
    ):
        raise ValueError("skill_dir 必须是非空字符串路径")
    skill_dir = Path(raw_skill_dir or "skill").expanduser()
    return skill_dir if skill_dir.is_absolute() else state_root / skill_dir


def get_logs_dir() -> Path:
    LOGS_DIR.mkdir(parents=True, exist_ok=True)
    return LOGS_DIR


def get_traj_dir() -> Path:
    """默认轨迹目录 = 第一个已注册的 watch dir。

    轨迹来源的真源是 Registry——dataset 通过 ``xskill registry add <abs-path>``
    注册，daemon 启动时也会自动探测并注册各生态的 session 目录。本函数仅给
    "无显式路径" 的内部调用取一个默认目录用；新代码优先走 Registry / 显式 path。

    一个 watch dir 都没注册时直接抛错——不兜底到某个魔术目录（CLAUDE.md：
    遇到问题 throw error，不写 fallback）。
    """
    # 函数内 import：registry 反过来依赖 config，模块级 import 会成环。
    from xskill.pipeline.registry import list_watch_dirs
    dirs = list_watch_dirs()
    if not dirs:
        raise RuntimeError(
            "没有已注册的 watch dir——先 `xskill registry add <abs-path>`，"
            "或让 daemon 启动时自动探测生态目录后再调用 get_traj_dir()。"
        )
    return Path(dirs[0]["path"])


def get_uploads_dir() -> Path:
    """上传 db 文件的落盘根目录（``~/.xskill/uploads``）。

    HTTP 上传端口把收到的 db 存到 ``uploads/<eco>/<client_id>/`` 下，再由
    ``xskill read`` 入库。按 client 分子目录隔离多用户同名 ``ngagent.db``。
    """
    d = XSKILL_HOME / "uploads"
    d.mkdir(parents=True, exist_ok=True)
    return d


def get_registry_db_path(
    *, xskill_home: Optional[Path] = None,
) -> Path:
    state_root = (
        Path(xskill_home) if xskill_home is not None else XSKILL_HOME
    ).expanduser().resolve()
    return state_root / "registry.db"


# ─── team (C/S 模式) 路径 ───────────────────────────────────────
# 纯路径运算，不读 config.yaml——client 瘦客户端无 llm.api_key，
# get_config() 会抛 KeyError。get_team_trajectories_dir() 是唯一例外
# （只 server 调，server 一定有 key）。

def get_team_server_state_path() -> Path:
    """server join token 落盘位置（~/.xskill/team_server.json，0600）。"""
    return XSKILL_HOME / "team_server.json"


def get_team_clients_db_path() -> Path:
    """server 端 client 注册表 SQLite。"""
    return XSKILL_HOME / "team_clients.db"


def get_team_server_whl_dir() -> Path:
    """server 端静默更新回退 wheel 目录（~/.xskill/whls）。"""
    d = XSKILL_HOME / "whls"
    d.mkdir(parents=True, exist_ok=True)
    return d


def get_team_client_state_path() -> Path:
    """client 端连接信息（server_url / client_id / join_token）。"""
    return XSKILL_HOME / "team_client.json"


def get_connect_daemon_state_path() -> Path:
    """``xskill connect`` 常驻进程的运行态（pid / server_url / task 名）。

    与 ``team_client.json``（连接**身份**，跨重启不变）分开：本文件记的是
    “现在有没有一个后台 connect 进程在跑、它的 pid/宿主任务是谁”，供
    ``xskill start/stop/status`` 管理。进程退出/机器重启后 pid 可能失效，
    读取方须自行校验存活（见 team.client.service）。
    """
    return XSKILL_HOME / "connect_daemon.json"


def _server_scope_id(server_url: str) -> str:
    """把 server_url 映射成文件系统安全、且按 server 唯一的作用域 id。

    形如 ``7.220.144.233_9961-1a2b3c4d``：前半是可读的 host_port（排错时
    一眼能认出连的是哪台），后半是规范化 url 的短哈希消歧（不同 url 规范化
    后撞到同一可读前缀时仍能区分）。规范化会去掉首尾空白与末尾斜杠，所以
    ``http://h:p`` 与 ``http://h:p/`` 视为同一 server。
    """
    import hashlib
    import re
    norm = server_url.strip().rstrip("/")
    netloc = norm.split("://", 1)[-1]
    safe = re.sub(r"[^A-Za-z0-9.]+", "_", netloc).strip("_") or "server"
    digest = hashlib.sha256(norm.encode("utf-8")).hexdigest()[:8]
    return f"{safe}-{digest}"


def get_team_client_dir(server_url: str) -> Path:
    """client 端按 server 隔离的可变状态目录：~/.xskill/clients/<server_id>/。

    上传游标 / 去抖 / 安装历史都落这里。换 server 时天然落到不同目录——不会
    再被上一个 server 的"已上传"游标静默压制对新 server 的上传（方案 A）。
    """
    d = XSKILL_HOME / "clients" / _server_scope_id(server_url)
    d.mkdir(parents=True, exist_ok=True)
    return d


def get_team_client_cursor_path(server_url: str) -> Path:
    """旧版 client 上传游标 JSON 路径，按 server 分目录。

    新版运行时状态落 ``client_state.db``；本路径保留为旧
    ``cursor.json`` / ``cursor.debounce.json`` 的一次性迁移来源。
    """
    return get_team_client_dir(server_url) / "cursor.json"


def get_team_client_state_db_path(server_url: str) -> Path:
    """client 端上传状态 SQLite，按 server 分目录。"""
    return get_team_client_dir(server_url) / "client_state.db"


def get_team_client_history_path(server_url: str) -> Path:
    """client 端安装历史（reconcile 落的 side 时间序列），按 server 分目录。

    注意这与 server/standalone 模式的 ``XSKILL_HOME/install_history.jsonl``
    是不同文件：那条是本机自身 canary 归因用的，与"连了哪个 server"无关。
    """
    return get_team_client_dir(server_url) / "install_history.jsonl"


# 注：team client 不另开 team_skills/ / team_outbox/ 目录——
#  - skill working copies 复用标准 skill_dir（~/.xskill/skill/），与
#    standalone 模式同位置；
#  - 采集的轨迹复用标准 bridge 目录（~/.xskill/<eco>_sessions/），即
#    detect_known_ecosystems 返回的 bridge 路径。


def get_team_trajectories_dir() -> Path:
```

## 3. The durable data model

`registry.db` begins with two pipeline tables. A `watch_dirs` row says which normalized directory may be scanned; a `trajectories` row is the state-machine record for one `traj_*.md`. The latter carries status, source model/harness, user attribution, retry/error fields, timestamps, and selected historical compatibility fields.

The rest of the schema is not another content store. It records usage cost, recommendation exposure, atom adoption, canary decisions, user preferences, lifecycle state, event feeds, UX scores, and materialized catalogs/recommendations. This division matters: candidate YAML and Git remain authoritative for authoring, while high-volume dashboard and sync reads use database projections.

A normalized trajectory itself is a three-file view:

- `<traj>.md`: stable Markdown sections such as `## User` and `## Assistant`;
- `<traj>.json`: adapter output and source metadata, including model/harness;
- `<traj>.md.meta`: optional structured metadata.

After splitting, each atom is an atomic JSON file below `<traj_id>/tasks/`. The durable `clustered` flag is the queue acknowledgement; unlike membership in a candidate buffer, it survives the buffer being consumed during Skill editing.

```bash
sed -n '120,185p' src/xskill/pipeline/registry.py
```

```output

_SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS watch_dirs (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    path       TEXT UNIQUE NOT NULL,
    label      TEXT DEFAULT '',
    auto_index INTEGER DEFAULT 1,
    ecosystem  TEXT DEFAULT 'manual',
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS trajectories (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    watch_dir_id  INTEGER NOT NULL REFERENCES watch_dirs(id) ON DELETE CASCADE,
    filename      TEXT NOT NULL,
    has_meta      INTEGER DEFAULT 0,
    has_embedding INTEGER DEFAULT 0,
    status        TEXT DEFAULT 'discovered',
    process_action TEXT,
    interest_fingerprint TEXT,
    skill_generated TEXT,
    skill_used    TEXT,
    canary_side   TEXT,
    source_model  TEXT,
    source_harness TEXT,
    user_key      TEXT DEFAULT '',
    ux_score      REAL,
    error_msg     TEXT,
    retry_count   INTEGER DEFAULT 0,
    file_mtime    REAL DEFAULT 0,
    discovered_at TEXT DEFAULT (datetime('now')),
    indexed_at    TEXT,
    updated_at    TEXT DEFAULT (datetime('now')),
    UNIQUE(watch_dir_id, filename)
);

CREATE TABLE IF NOT EXISTS llm_usage (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    ts           TEXT DEFAULT (datetime('now')),
    step         TEXT,
    model        TEXT,
    prompt       INTEGER DEFAULT 0,
    completion   INTEGER DEFAULT 0,
    total        INTEGER DEFAULT 0,
    cost_usd     REAL DEFAULT 0,
    price_source TEXT
);
CREATE INDEX IF NOT EXISTS idx_llm_usage_ts ON llm_usage(ts);

-- 埋点(instrumentation,在代码里插记录点):三类事件,供看板算衍生率 --
CREATE TABLE IF NOT EXISTS recommendation_log (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    ts        TEXT DEFAULT (datetime('now')),
    client_id TEXT,
    skill     TEXT,
    side      TEXT,          -- main / staging
    bucket    TEXT           -- ranked / recommended
);
CREATE INDEX IF NOT EXISTS idx_reco_skill ON recommendation_log(skill);

CREATE TABLE IF NOT EXISTS atom_adoption (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ts          TEXT DEFAULT (datetime('now')),
    atom_id     TEXT,
    skill       TEXT,
    weightscore INTEGER,
```

```bash
sed -n '28,95p' src/xskill/pipeline/trajectory.py
```

```output
class Trajectory:
    """一条轨迹的视图。包 .md / .json / .meta 三件套。

    若构造时不传 registry，则 skill_used / skill_generated / canary_side 全返回 None
    （独立加载模式：只读文件本身，不查 DB）。
    """

    def __init__(self, path: Path, registry: Optional["Registry"] = None):
        self.path = Path(path)
        self._registry = registry
        self._meta_cache: Optional[dict] = None

    @classmethod
    def load(cls, path: str | Path,
             registry: Optional["Registry"] = None) -> "Trajectory":
        p = Path(path).expanduser().resolve()
        if not p.is_file():
            raise FileNotFoundError(f"trajectory file not found: {p}")
        return cls(path=p, registry=registry)

    # ─── 文件三件套 ────────────────────────────────────────────────
    @property
    def md_text(self) -> str:
        return self.path.read_text(encoding="utf-8")

    @property
    def meta(self) -> dict:
        """从 <traj>.md.meta 读结构化 meta；不存在返回 {}。"""
        if self._meta_cache is not None:
            return self._meta_cache
        meta_path = self.path.parent / f"{self.path.name}.meta"
        if meta_path.exists():
            self._meta_cache = json.loads(meta_path.read_text(encoding="utf-8"))
        else:
            self._meta_cache = {}
        return self._meta_cache

    @property
    def raw_json(self) -> dict:
        """从 <traj>.json 读上游原始数据；不存在返回 {}。"""
        json_path = self.path.with_suffix(".json")
        if json_path.exists():
            return json.loads(json_path.read_text(encoding="utf-8"))
        return {}

    @property
    def is_success(self) -> bool:
        return bool(self.meta.get("success"))

    # ─── DB 反查 ──────────────────────────────────────────────────
    @property
    def _row(self) -> Optional[dict]:
        if self._registry is None:
            return None
        return self._registry.trajectory_status(self.path)

    @property
    def skill_used(self) -> Optional[str]:
        row = self._row
        return row.get("skill_used") if row else None

    @property
    def skill_generated(self) -> Optional[str]:
        row = self._row
        return row.get("skill_generated") if row else None

    @property
    def canary_side(self) -> Optional[str]:
```

```bash
sed -n '34,118p' src/xskill/pipeline/atom.py
```

```output
class AtomTask:
    """一段完整用户意图的最小提炼单元。

    字段约定：
    - ``offset_start`` / ``offset_end``: 在 ``<traj_id>.md`` 中的 **1-based 行号**
      （半开区间 ``[start, end)``——end 这一行不含；末 atom 的 end = 末行号+1），
      便于 ``ReadTraj`` 按行号读原文。轨迹入库后不变，行号稳定。
    - ``pre_atom_id`` / ``post_atom_id``: 前后 atom 链表，给 cluster/edit
      agent 沿时间线游走。
    - ``context_prefix``: atom 起始行之前内容的省略表示（头 200 字 + 占位）。
    - ``raw_segment``: ``[offset_start, offset_end)`` 行区间内的原文片段。
    """
    atom_id: str
    traj_id: str
    offset_start: int
    offset_end: int
    intent: str
    summary: str
    tags: list[str] = field(default_factory=list)
    used_skills: list[str] = field(default_factory=list)
    ux_score: int | None = None
    pre_atom_id: str | None = None
    post_atom_id: str | None = None
    context_prefix: str = ""
    raw_segment: str = ""
    source_model: str = ""   # 产生该 atom 的用户 agent 模型，继承自所属轨迹的
    #                          <traj>.json sidecar "model"（canary 按模型分桶用）
    clustered: bool = False  # cluster 已消费标记（耐久）。cluster agent 成功把它
    #                          归进某 skill buffer 后由 process_atom_batch/_task 置真。
    #                          watcher 据此跨轨迹池化去重 + 判轨迹 done——比
    #                          .candidates.yml 成员更耐久：SkillEdit 晋升会清空
    #                          .candidates.yml，但本标记留在 atom JSON，跨进程重启
    #                          与 skill 晋升都不丢（rebuild 删 atom 时自然复位）。

    def to_json(self) -> str:
        return json.dumps(asdict(self), ensure_ascii=False, indent=2)

    @classmethod
    def from_json(cls, s: str) -> "AtomTask":
        return cls(**json.loads(s))


class AtomTaskStore:
    """文件系统存储：``<root>/<traj_id>/tasks/atom_*.json`` + ``<root>/index.pkl``。

    设计取舍：
    - **不用 SQLite**: traj 数量级 < 数千；文件读写直接、调试方便、跨平台兼容。
    - **每 traj 一个子目录**: 让 watcher 按 traj 粒度做增量处理，``list_by_traj``
      不需要全表扫。
    - **向量索引增量重建**: TaskAgent 每给一条 traj 拆出新 atom 后由 watcher
      触发一次 ``rebuild_vector_index``；按 ``atom_id`` 复用旧 index.pkl 中同
      模型的向量，只对新原子调 embedding（换模型则整体重算，护栏见方法内），
      避免攒了上万原子的 client 新增几条就全量重 embed。
    """

    INDEX_FILE = "index.pkl"

    def __init__(self, root: Path):
        self.root = Path(root)

    # ── paths ─────────────────────────────────────────────────────

    def _traj_dir(self, traj_id: str) -> Path:
        return self.root / traj_id / "tasks"

    def _path(self, atom: AtomTask) -> Path:
        return self._traj_dir(atom.traj_id) / f"{atom.atom_id}.json"

    def _index_path(self) -> Path:
        return self.root / self.INDEX_FILE

    # ── IO ────────────────────────────────────────────────────────

    def save(self, atom: AtomTask) -> Path:
        atom_path = self._path(atom)
        atom_path.parent.mkdir(parents=True, exist_ok=True)
        temporary_path = atom_path.with_name(f".{atom_path.name}.{uuid.uuid4().hex}.tmp")
        temporary_path.write_text(atom.to_json(), encoding="utf-8")
        temporary_path.replace(atom_path)
        return atom_path

    def load(self, atom_id: str) -> AtomTask:
        """跨 traj_id 子目录查找。命中第一条即返回；找不到抛 FileNotFoundError。

        ``watcher`` 的常态调用是 ``list_by_traj``（O(子目录文件数)），``load``
```

## 4. `serve` creates a control plane and delegates heavy work

`xskill serve` first enforces the single-daemon guard, writes runtime status, and calls `XSkill.serve`. That builds a FastAPI app and runs Uvicorn. Application startup validates both model clients before declaring readiness: a bad LLM or embedding configuration aborts startup instead of leaving a half-working daemon.

The web process is intentionally a coordinator. It owns HTTP routes, a bounded control-plane executor, and lightweight scheduler threads. The expensive pipeline runs in child processes:

1. a persistent `agent-worker` owns the watcher and four bounded thread pools: split, cluster, edit, and embed;
2. standalone mode also runs a persistent ecosystem-ingest worker for low-latency Claude Code session rotation;
3. a periodic `ux-scores-sync` process projects disk UX/candidate state into SQLite;
4. team-server mode periodically runs `recommend-heavy`, which refreshes profiles, reconciles the vector index, and precomputes dirty recommendations.

`IntervalSubprocessScheduler` supervises these children. Persistent children restart after exit; periodic children cannot overlap with themselves. Shutdown raises the global stop flag, stops new control work, terminates schedulers, clears team context, and lets each worker write a final status heartbeat.

```bash
sed -n '33,69p' src/xskill/cli.py
```

```output
def cmd_serve(args, xskill) -> int:
    # --home 用于 debug 模式：生态扫描只看该目录下的 .claude/，不碰真实
    # $HOME。要求顶层 --debug 同时打开，避免生产环境误用。
    home_root = None
    if args.home:
        if not args.debug:
            print("error: --home 仅在 --debug 模式下生效；加 --debug 或去掉 --home",
                  file=sys.stderr)
            return 2
        from pathlib import Path
        home_root = Path(args.home).expanduser().resolve()
        if not home_root.is_dir():
            print(f"error: --home 目录不存在: {home_root}", file=sys.stderr)
            return 2
    from xskill.runtime import read_status, write_running
    # ── 单实例守卫：已有活 daemon 时拒绝启动 ──
    # 双 daemon 会抢同一 registry（rebuild 后旧 daemon 可能用旧模型抢先处理）。
    # read_status 已校验 pid 存活，陈旧运行态文件不会误拦。--force 强行接管。
    status = read_status()
    if status.get("running") and not args.force:
        print(
            f"✗ 已有 xskill daemon 在运行（pid {status.get('pid')}, "
            f"端口 {status.get('port')}）。",
            file=sys.stderr,
        )
        print(
            "  双 daemon 会抢同一 registry，导致换模型 rebuild 被旧 daemon 抢去用旧"
            "模型处理。\n  先停掉它再起；确认要强行接管可加 --force。",
            file=sys.stderr,
        )
        return 2
    write_running(port=args.port, mode="server" if args.server else "standalone")
    xskill.serve(host=args.host, port=args.port, home_root=home_root,
                 server_mode=args.server)
    return 0


```

```bash
sed -n '1100,1282p' src/xskill/api/app.py
```

```output
                    # team_client 生态标签：watcher 的 CS 归因靠 wd.label 反查 client
                    _register_dir(path, label=label, ecosystem="team_client")

                def _team_configure_watch_dir(path, label, auto_index):
                    # clients.ingest_paused 是权威状态；每次上传/导入都据此重放，
                    # 修复 team_clients.db 与 registry.db 跨库写入可能留下的漂移。
                    _register_dir(
                        path,
                        label=label,
                        auto_index=auto_index,
                        ecosystem="team_client",
                    )

                # §5 构造 SkillRecommendEngine 并注入 manifest（staging 优先达量 + 画像推荐）
                from xskill.config import XSKILL_HOME as _xhome
                from xskill.recommend.engine import SkillRecommendEngine
                from xskill.team.server.skill_manifest import set_recommend_engine
                _team_embed = create_embed_client(_config)
                _engine = SkillRecommendEngine(
                    config=_config, skill_dir=_skill_dir, traj_root=traj_root,
                    embed_client=_team_embed,
                    profile_db=_xhome / "team_profile.db",
                    client_registry=client_registry,
                )
                set_recommend_engine(_engine)
                recommend_engine_attached = True

                init_team_context(
                    join_token=join_token,
                    client_registry=client_registry,
                    skill_dir=_skill_dir,
                    traj_root=traj_root,
                    register_dir=_team_register_dir,
                    configure_watch_dir=_team_configure_watch_dir,
                    skillhub=_engine.skillhub,
                    profile_refresh_service=None,
                )
                for client_row in client_registry.list():
                    reconcile_client_ingest_watch_dir(client_row["client_id"])
                team_sync_executor_cleanup_required = True
                start_team_sync_executor(
                    app,
                    max_workers=team_sync_cfg["workers"],
                )
                # 重活合并：画像 + Milvus 对账 + 脏用户推荐预计算，全在独立子进程。
                # Web /sync 只读已落库画像与 client_recommend_slots
                # (profile_refresh_service=None 时 api.team_sync 入队守卫跳过)。
                profile_scheduler = IntervalSubprocessScheduler(
                    "recommend-heavy",
                    [_sys.executable, "-m", "xskill._workers", "recommend-heavy"],
                    interval=profile_refresh_cfg["interval"],
                    timeout=profile_refresh_cfg["timeout"],
                )
                profile_scheduler.start()
                _schedulers.append(profile_scheduler)
                logger.info(
                    "team server context ready (traj_root=%s, "
                    "recommend-heavy every %.0fs via subprocess)",
                    traj_root, profile_refresh_cfg["interval"],
                )
            except Exception:
                if team_sync_executor_cleanup_required:
                    try:
                        from xskill.team.server.api import stop_team_sync_executor
                        stop_team_sync_executor(app)
                    except Exception:  # pylint: disable=broad-exception-caught
                        logger.warning("failed to stop partial team sync executor",
                                       exc_info=True)
                if profile_scheduler is not None and profile_scheduler not in _schedulers:
                    try:
                        profile_scheduler.stop()
                    except Exception:  # pylint: disable=broad-exception-caught
                        logger.warning("failed to stop partial profile scheduler",
                                       exc_info=True)
                for scheduler in _schedulers:
                    try:
                        scheduler.stop()
                    except Exception:  # pylint: disable=broad-exception-caught
                        logger.warning("failed to stop partial team scheduler",
                                       exc_info=True)
                _schedulers.clear()
                registry_owned_by_context = False
                try:
                    from xskill.team.server.api import clear_team_context, team_context
                    registry_owned_by_context = (
                        client_registry is not None
                        and team_context().client_registry is client_registry
                    )
                    clear_team_context(profile_refresh_shutdown_timeout=0)
                except Exception:  # pylint: disable=broad-exception-caught
                    logger.warning("failed to clear partial team context",
                                   exc_info=True)
                if client_registry is not None and not registry_owned_by_context:
                    try:
                        client_registry.close()
                    except Exception:  # pylint: disable=broad-exception-caught
                        logger.warning("failed to close unattached client registry",
                                       exc_info=True)
                if recommend_engine_attached:
                    try:
                        from xskill.team.server.skill_manifest import set_recommend_engine
                        set_recommend_engine(None)
                    except Exception:  # pylint: disable=broad-exception-caught
                        logger.warning("failed to detach recommend engine after "
                                       "partial team init", exc_info=True)
                logger.exception("team server context init failed")
                raise

        # agent-worker 在常驻子进程中持续运行。
        # 进程只留一个轻量守护线程，负责启动、监测和异常退出后重启子进程；重计算
        # 仍与 web 事件循环保持 GIL 隔离。DirectoryWatcher 自己按 poll_interval
        # 持续扫描，Future 跨轮保留，不再等待一批全部结束后才启动下一轮。
        from xskill.pipeline.scheduler import IntervalSubprocessScheduler as _WorkerSched
        poll_interval = float(_config.get("watcher", {}).get("poll_interval", 5))
        agent_worker_command = [
            _sys.executable, "-m", "xskill._workers", "agent-worker",
        ]
        if team_server:
            agent_worker_command.append("--server")
        else:
            agent_worker_command.extend(["--home", str(ecosystem_home_root)])
        agent_worker_scheduler = _WorkerSched(
            "agent-worker", agent_worker_command,
            interval=poll_interval,
            timeout=5.0,
            persistent=True,
        )
        agent_worker_scheduler.start()
        _schedulers.append(agent_worker_scheduler)
        logger.info(
            "persistent agent worker started (team_server=%s, poll every %.0fs)",
            team_server,
            poll_interval,
        )
        from xskill.config import ux_scores_sync_config
        ux_sync_cfg = ux_scores_sync_config(_config)
        ux_sync_scheduler = _WorkerSched(
            "ux-scores-sync",
            [_sys.executable, "-m", "xskill._workers", "ux-scores-sync"],
            interval=ux_sync_cfg["interval"],
            timeout=ux_sync_cfg["timeout"],
        )
        ux_sync_scheduler.start()
        _schedulers.append(ux_sync_scheduler)
        logger.info(
            "ux_scores disk→db sync every %.0fs via subprocess",
            ux_sync_cfg["interval"],
        )
        if not team_server:
            ingest_interval = min(poll_interval, 1.0)
            ingest_scheduler = _WorkerSched(
                "ecosystem-ingest",
                [
                    _sys.executable,
                    "-m",
                    "xskill._workers",
                    "ecosystem-ingest",
                    "--home",
                    str(ecosystem_home_root),
                    "--loop",
                    "--interval",
                    "0.5",
                ],
                interval=max(ingest_interval, 1.0),
                timeout=5.0,
                persistent=True,
            )
            ingest_scheduler.start()
            _schedulers.append(ingest_scheduler)
            logger.info(
                "ecosystem ingest scheduler started "
                "(persistent subprocess, poll every 0.5s)",
            )

        # 依赖初始化全部成功后才创建线程池；如果 startup 在此之前
        # 失败，不会留下需要额外回收的非 daemon 线程。
        _start_control_plane_executor(app)

    @app.on_event("shutdown")
    async def _shutdown():
        # 先竖旗：所有 worker 线程里的 LLM 重试循环见旗即弃，退避睡眠立即
        # 中断。不竖旗的话 join 最坏拖 11 分钟，supervisor 10s 后 SIGKILL。
        from xskill.utils.shutdown import request_shutdown
```

```bash
sed -n '48,175p' src/xskill/_workers.py
```

```output
def run_agent_worker_forever(
    *,
    server: bool = False,
    home: str | None = None,
    stop_event=None,
    status_interval: float = 5.0,
) -> int:
    """构造四池 agent worker 并常驻运行，直到收到 TERM/INT。

    ``DirectoryWatcher.start()`` 的扫描线程每个 poll 都继续提交和收割
    任务，Future 跨轮保留；LLM 长尾任务不会阻止后续扫描。
    ``stop_event`` 仅供测试和嵌入调用注入；生产由信号 handler 设置内部事件。
    """
    import threading

    from xskill.config import (
        XSKILL_HOME,
        get_registry_db_path,
        get_skill_dir,
        load_config,
    )
    from xskill.pipeline.watcher_factory import (
        build_watcher,
        ingest_detected_ecosystems_once,
    )
    from xskill.utils.status_file import (
        AGENT_WORKER_STATUS_FILE,
        WATCHER_STATUS_FILE,
        write_status_file,
    )

    if status_interval <= 0:
        raise ValueError("status_interval 必须 > 0")

    config = load_config()
    home_root = Path(home).expanduser().resolve() if home else Path.home()
    status_path = XSKILL_HOME / WATCHER_STATUS_FILE
    worker_status_path = XSKILL_HOME / AGENT_WORKER_STATUS_FILE
    event = stop_event if stop_event is not None else threading.Event()
    previous_handlers = _install_stop_signal_handlers(event)
    watcher = None
    ok = False
    error = None
    try:
        skill_dir = get_skill_dir(
            config,
            xskill_home=XSKILL_HOME,
        ).expanduser().resolve()
        registry_db_path = get_registry_db_path(
            xskill_home=XSKILL_HOME,
        ).expanduser().resolve()
        install_history_path = (
            XSKILL_HOME / "install_history.jsonl"
        ).expanduser().resolve()

        on_poll_hook = None
        if not server:
            def ingest_on_poll() -> None:
                ingest_detected_ecosystems_once(
                    config,
                    home_root,
                    skill_dir,
                    registry_db_path=registry_db_path,
                    install_history_path=install_history_path,
                    excluded_ecosystems={"claude_code"},
                )

            on_poll_hook = ingest_on_poll

        watcher = build_watcher(
            config,
            xskill_home=XSKILL_HOME,
            config_path=XSKILL_HOME / "config.yaml",
            db_path=registry_db_path,
            skill_dir=skill_dir,
            home_root=home_root,
            server_mode=server,
            on_poll_hook=on_poll_hook,
        )
        watcher.start()
        write_status_file(status_path, watcher.stats, ok=True)
        write_status_file(
            worker_status_path,
            getattr(watcher, "agent_worker_status", watcher.stats),
            ok=True,
        )

        while not event.wait(status_interval):
            if not watcher.is_running:
                raise RuntimeError("watcher thread exited unexpectedly")
            write_status_file(status_path, watcher.stats, ok=True)
            write_status_file(
                worker_status_path,
                getattr(watcher, "agent_worker_status", watcher.stats),
                ok=True,
            )

        ok = True
        return 0
    except Exception as exc:  # noqa: BLE001 — 常驻 worker 顶层边界
        error = str(exc)
        logger.exception("persistent agent worker failed")
        return 1
    finally:
        if watcher is not None:
            from xskill.utils.shutdown import request_shutdown

            request_shutdown()
            watcher.stop()
        write_status_file(
            status_path,
            watcher.stats if watcher is not None else {},
            ok=ok,
            error=error,
        )
        write_status_file(
            worker_status_path,
            (
                getattr(watcher, "agent_worker_status", watcher.stats)
                if watcher is not None
                else {}
            ),
            ok=ok,
            error=error,
        )
        _restore_signal_handlers(previous_handlers)


```

```bash
sed -n '24,155p' src/xskill/pipeline/scheduler.py
```

```output
class IntervalSubprocessScheduler:
    """调度子进程；``persistent=True`` 时守护单个常驻子进程。"""

    def __init__(
        self,
        name: str,
        command: list[str],
        *,
        interval: float,
        timeout: float,
        persistent: bool = False,
    ):
        if interval <= 0:
            raise ValueError("interval 必须 > 0")
        if timeout <= 0:
            raise ValueError("timeout 必须 > 0")
        self._name = name
        self._command = list(command)
        self._interval = float(interval)
        self._timeout = float(timeout)
        self._persistent = bool(persistent)
        self._stop = threading.Event()
        self._thread: threading.Thread | None = None
        self._process: subprocess.Popen | None = None
        self._process_lock = threading.Lock()

    def start(self) -> None:
        """幂等启动调度 daemon 线程。"""
        if self._thread is not None and self._thread.is_alive():
            return
        self._stop.clear()
        self._thread = threading.Thread(
            target=self._loop, name=f"xskill-sched-{self._name}", daemon=True,
        )
        self._thread.start()

    def stop(self, timeout: float = 5.0) -> None:
        """竖停机旗、中断 wait，并终止、回收正在运行的子进程。"""
        self._stop.set()
        with self._process_lock:
            process = self._process
        if process is not None and process.poll() is None:
            try:
                process.terminate()
            except OSError:
                logger.warning(
                    "调度任务 %s 停止时 terminate 失败",
                    self._name,
                    exc_info=True,
                )
        if self._thread is not None:
            self._thread.join(timeout)

        # Popen 与 stop 存在竞态：第一次读取后，调度线程可能刚完成 spawn。
        # join 后必须重读当前句柄，不能只操作旧的 process。
        with self._process_lock:
            process = self._process
        if process is not None and process.poll() is None:
            try:
                process.kill()
            except OSError:
                logger.warning(
                    "调度任务 %s 停止时 kill 失败",
                    self._name,
                    exc_info=True,
                )
            if self._thread is not None:
                self._thread.join(timeout)

        # 正常路径由 communicate()/wait() 完成回收并清空句柄。若调度线程已退但
        # 句柄仍在，最后再 wait 一次，避免已退出的子进程残留为 zombie。
        with self._process_lock:
            process = self._process
        if (
            process is not None
            and (self._thread is None or not self._thread.is_alive())
        ):
            try:
                process.wait(timeout=timeout)
            except subprocess.TimeoutExpired:
                logger.warning("调度任务 %s 停止后子进程未退出", self._name)
            except OSError:
                logger.warning(
                    "调度任务 %s 停止时 wait 失败",
                    self._name,
                    exc_info=True,
                )

    def _loop(self) -> None:
        if self._persistent:
            self._persistent_loop()
            return
        # 先等一个周期再首跑:避免 startup 瞬间与其它初始化抢资源(照 AutoUpdater)。
        # Event.wait 返回 True 表示被 stop 竖旗中断 → 退出循环。
        while not self._stop.wait(self._interval):
            # 本线程是 daemon:任何漏网异常都会让它静默猝死,此后画像
            # 永不再跑(进程还活着,只是不干活了)。照 daemon._tick 兜住并落日志。
            try:
                self._run_once()
            except Exception:  # noqa: BLE001 — 顶层任务边界,吞掉但必须落日志
                logger.warning("调度任务 %s 本轮异常,下轮继续", self._name,
                               exc_info=True)

    def _persistent_loop(self) -> None:
        """守护一个常驻轻量子进程；退出后有界退避重启。"""
        while not self._stop.is_set():
            try:
                process = subprocess.Popen(
                    self._command,
                    stdout=subprocess.DEVNULL,
                    stderr=subprocess.DEVNULL,
                    **windowless_subprocess_kwargs(),
                )
            except OSError:
                logger.warning(
                    "常驻调度任务 %s 启动子进程失败",
                    self._name,
                    exc_info=True,
                )
                if self._stop.wait(self._interval):
                    return
                continue
            with self._process_lock:
                self._process = process
            while process.poll() is None:
                if self._stop.wait(min(self._interval, 0.5)):
                    try:
                        process.terminate()
                    except OSError:
                        logger.warning(
                            "常驻调度任务 %s terminate 失败",
                            self._name,
```

## 5. Native sessions become one normalized trajectory format

The pipeline never teaches downstream agents every vendor format. Ecosystem modules bridge them first.

`detect_known_ecosystems` probes known locations for Claude Code, Codex, OpenCode/ngagent, Cursor, Trae, OpenClaw, nga3, and DeepSeek Harness. JSONL-like tools are described by an immutable `EcosystemSpec`: where sessions live, how to glob them, how to derive a session ID and working directory, which adapter to call, and where Skills are installed. OpenCode/ngagent use a separate `SqliteEcosystemSpec` because a database cursor and a file cursor have genuinely different lifecycles.

`JsonlIngester` waits for the source file to settle, derives a stable `traj_<ecosystem>_<project>_<session>` ID, adapts native events to Markdown, sanitizes control characters, applies configured mask patterns, and writes the normalized files. If a previously bridged source grows, it overwrites the normalized trajectory and resets derived atoms/state for a full re-split. The SQLite ingester queries new sessions by `time_updated`, joins messages to parts, renders the same User/Assistant shape, and submits it through the same write path.

The bridge directory—not the vendor directory—is registered with the watcher. That is the normalization boundary: everything after it is ecosystem-agnostic.

```bash
sed -n '81,142p' src/xskill/ecosystems/_shared.py
```

```output
class EcosystemSpec:
    """描述一个 agent 生态的轨迹来源 + 安装目标。

    Attributes:
        name: 生态标识（``claude_code`` / ``codex`` / ``opencode``）；用于 logging
            和 `detect_known_ecosystems` 上报
        source_kind: 轨迹存储形态。P2 只支持 ``jsonl``；P3 加 ``sqlite``
        sessions_path: ``(home_root) -> Path``，返回该生态的 sessions/projects 根目录
        sessions_glob: 相对 ``sessions_path`` 的 glob，用于扫所有 session 文件
        session_id_from_path: ``(jsonl_path) -> session_id``。CC 用文件名（``stem``），
            codex 用文件名里的 uuid 段
        cwd_from_content: ``(jsonl_content) -> cwd``。CC 扫每条事件找首个 ``cwd``
            字段；codex 只读首行 ``session_meta.payload.cwd``
        adapter_format: 喂给 ``adapt_trajectory`` 的 format 字符串
        traj_id_prefix: 桥过来的 ``traj_*.md`` 文件名 ID 前缀（``traj_cc_`` /
            ``traj_codex_``）
        skills_install_path: ``(home_root) -> Path``，skill 安装目标根目录
        label: 短标签，给 logger 用
    """

    name: str
    source_kind: Literal["jsonl"]
    sessions_path: Callable[[Path], Path]
    sessions_glob: str
    session_id_from_path: Callable[[Path], str]
    cwd_from_content: Callable[[str], str]
    adapter_format: str
    traj_id_prefix: str
    skills_install_path: Callable[[Path], Path]
    label: str
    is_session_complete: Optional[Callable[[str], bool]] = None
    # 可选：自定义「文件 → 文本」读取。默认 None = ``read_text``。给需要先
    # 解码再解析的生态用（DeepSeek Harness 默认写 zstd 帧序列的
    # ``session.jsonl.zstd``）。返回 None 表示本轮读不了（例如缺解码依赖），
    # ingester 跳过该文件并由 reader 自己负责告警。
    read_content: Optional[Callable[[Path], Optional[str]]] = None


@dataclass(frozen=True)
class SqliteEcosystemSpec:
    """SQLite-back 生态系统 spec（独立于 JsonlIngester 的 EcosystemSpec）。

    EcosystemSpec 是 JSONL ingester 专用 spec（含 sessions_glob 等 JSONL-only
    字段），不适合 SQLite。SqliteIngester 用本类，字段集中在 SQLite 视角：
    path_resolver 解析到 .db 文件、cursor_strategy 用 time_updated。

    ``traj_id_prefix`` —— 桥过来的 ``traj_*.md`` 文件名 ID 前缀（``traj_oc_`` /
    ``traj_ng_`` 等）。由 ingester 按 spec 派生 traj_id 而非硬编码——新增同形
    SQLite 生态（如 ngagent，opencode 的企业分支）时只换 spec 即可，避免
    ingester 里出现 ``if spec.name == "ngagent"`` 这种熵增分支。
    """

    name: str                                       # "opencode" | "ngagent" | ...
    source_kind: Literal["jsonl", "sqlite"]
    path_resolver: Callable[[Path], Path]           # (home) -> db file / dir
    cursor_strategy: Literal["mtime_offset", "sqlite_time_updated"]
    label: str                                      # adapter / metadata 标签
    traj_id_prefix: str = "traj_"                   # bridged traj_*.md filename prefix


# ─────────────────────────────────────────────────────────────────
# Ecosystem auto-detection
```

```bash
sed -n '215,272p' src/xskill/ecosystems/_shared.py
```

```output
def bridge_dir_for(eco_id: str, home_root: Path | str | None = None) -> Path:
    """某生态 bridged 轨迹的落盘目录（``<home>/.xskill/<eco>_sessions``）。

    ``xskill read`` / 上传入库把 db 桥成的 ``traj_*.md`` 写到这里——与 daemon
    常驻 ingester 用同一个目录，watcher 注册后即可统一捡起。eco_id 未知直接抛
    （CLAUDE.md：不兜底）。
    """
    home = Path(home_root) if home_root else Path.home()
    for e in _KNOWN_ECOSYSTEMS:
        if e["id"] == eco_id:
            return home / e["bridge_subpath"]
    known = ", ".join(e["id"] for e in _KNOWN_ECOSYSTEMS)
    raise ValueError(f"unknown ecosystem {eco_id!r}; known: {known}")


def detect_known_ecosystems(home_root: Path | str | None = None) -> list[dict]:
    """Probe the user's HOME for known agent tools and report which ones
    have something on disk. Returns a list of detection records:

        {"ecosystem": "claude_code" | "codex" | "opencode",
         "source": <abs path of native session dir or db file>,
         "bridge": <abs path of paired xskill watch dir>}

    A record only appears if the source dir/file exists (按
    ``source_kind`` 区分用 ``is_dir`` 还是 ``is_file``). The bridge dir is
    the path daemon should ``register_dir(..., ecosystem=...)`` to put
    under Registry control — it may or may not exist yet.

    设计：每 install 前 watcher 实时调本函数判 detected list（3 次
    ``Path.is_dir/is_file`` 开销可忽略）——避免启动时缓存导致用户中途
    装了 codex 后 daemon 看不到。
    """
    root = Path(home_root) if home_root else Path.home()
    found: list[dict] = []
    for spec in _KNOWN_ECOSYSTEMS:
        source = root / spec["source_subpath"]
        kind = spec.get("source_kind", "dir")
        if kind == "dir" and not source.is_dir():
            continue
        if kind == "file" and not source.is_file():
            continue
        found.append({
            "ecosystem": spec["id"],
            "source": source.resolve(),
            "bridge": (root / spec["bridge_subpath"]).resolve(),
        })
    # Trae：多路径探测（IDE workspaceStorage / ~/.trae-cn / CLI trajectories）
    from xskill.ecosystems.trae import detect_trae_record

    trae_det = detect_trae_record(root)
    if trae_det is not None:
        found.append(trae_det)
    return found


# ─────────────────────────────────────────────────────────────────
# Shared skill-install implementation
# ─────────────────────────────────────────────────────────────────
```

```bash
sed -n '529,731p' src/xskill/ecosystems/_shared.py
```

```output
def adapt_trajectory(
    content: str,
    format: str,
    metadata: Optional[dict] = None,
) -> tuple[str, dict]:
    """
    Convert various input formats to the standard xskill representation.

    Supported *format* values:

    - ``markdown`` -- passthrough; content is already ``traj_*.md`` format.
    - ``json`` -- JSON object with fields like ``messages``, ``tool_calls``, etc.
      Converted to a markdown trajectory.
    - ``raw`` -- plain text; wrapped in a basic trajectory markdown template.
    - ``claude_code_jsonl`` / ``codex_rollout_jsonl`` /
      ``nga3_jsonl`` / ``zcode_jsonl`` / ``openclaw_trajectory_jsonl`` /
      ``cursor_transcripts_jsonl`` / ``deepseek_harness_session_jsonl`` /
      ``trae_ide_session_json`` /
      ``trae_agent_trajectory_json`` -- 各 agent
      生态原生 session；分发到对应平台模块的 ``_adapt_*``。

    Returns ``(md_content, json_metadata)``.
    """
    # 平台 ``_adapt_*`` 延迟 import，避免 _shared <-> 平台模块循环 import。
    from xskill.ecosystems.claude_code import _adapt_claude_code_jsonl
    from xskill.ecosystems.codex import _adapt_codex_rollout_jsonl
    from xskill.ecosystems.nga3 import _adapt_nga3_jsonl
    from xskill.ecosystems.openclaw import _adapt_openclaw_trajectory_jsonl
    from xskill.ecosystems.cursor import _adapt_cursor_transcripts_jsonl
    from xskill.ecosystems.deepseek_harness import (
        _adapt_deepseek_harness_session_jsonl,
    )
    from xskill.ecosystems.trae import (
        _adapt_trae_agent_trajectory_json,
        _adapt_trae_ide_session_json,
    )

    metadata = metadata or {}

    if format == "markdown":
        return content, metadata

    if format == "json":
        return _adapt_json(content, metadata)

    if format == "raw":
        return _adapt_raw(content, metadata)

    if format == "claude_code_jsonl":
        return _adapt_claude_code_jsonl(content, metadata)

    if format == "codex_rollout_jsonl":
        return _adapt_codex_rollout_jsonl(content, metadata)

    if format in ("nga3_jsonl", "zcode_jsonl"):
        return _adapt_nga3_jsonl(content, metadata)

    if format == "openclaw_trajectory_jsonl":
        return _adapt_openclaw_trajectory_jsonl(content, metadata)

    if format == "cursor_transcripts_jsonl":
        return _adapt_cursor_transcripts_jsonl(content, metadata)

    if format == "deepseek_harness_session_jsonl":
        return _adapt_deepseek_harness_session_jsonl(content, metadata)

    if format == "trae_ide_session_json":
        return _adapt_trae_ide_session_json(content, metadata)

    if format == "trae_agent_trajectory_json":
        return _adapt_trae_agent_trajectory_json(content, metadata)

    raise ValueError(f"unsupported trajectory format: {format!r}")


def _adapt_json(content: str, metadata: dict) -> tuple[str, dict]:
    """Convert a JSON trajectory to markdown + metadata."""
    data = json.loads(content)

    # Merge top-level keys (except messages/tool_calls) into metadata
    meta = dict(metadata)
    for key in ("model", "instance_id", "repo", "task", "result", "exit_status"):
        if key in data and key not in meta:
            meta[key] = data[key]

    # Build markdown from messages / tool_calls
    lines: list[str] = []
    lines.append(f"# Trajectory")
    if meta.get("instance_id"):
        lines.append(f"\n**instance_id**: {meta['instance_id']}")
    if meta.get("model"):
        lines.append(f"**model**: {meta['model']}")
    lines.append("")

    messages = data.get("messages", [])
    tool_calls = data.get("tool_calls", [])

    for msg in messages:
        role = msg.get("role", "unknown")
        text = msg.get("content", "")
        lines.append(f"## {role.capitalize()}")
        lines.append("")
        if isinstance(text, str):
            lines.append(text)
        elif isinstance(text, list):
            # multi-part content
            for part in text:
                if isinstance(part, dict):
                    lines.append(part.get("text", str(part)))
                else:
                    lines.append(str(part))
        lines.append("")

    if tool_calls:
        lines.append("## Tool Calls")
        lines.append("")
        for tc in tool_calls:
            name = tc.get("name", tc.get("function", {}).get("name", "unknown"))
            args = tc.get("arguments", tc.get("function", {}).get("arguments", ""))
            lines.append(f"### {name}")
            lines.append("```")
            lines.append(args if isinstance(args, str) else json.dumps(args, ensure_ascii=False))
            lines.append("```")
            if tc.get("output"):
                lines.append(f"\n**output**:\n```\n{tc['output']}\n```")
            lines.append("")

    md_content = "\n".join(lines)
    return md_content, meta


def _adapt_raw(content: str, metadata: dict) -> tuple[str, dict]:
    """Wrap plain text in a basic trajectory markdown template."""
    lines = [
        "# Trajectory",
        "",
        "## Raw Content",
        "",
        content,
        "",
    ]
    md_content = "\n".join(lines)
    return md_content, dict(metadata)


def submit_trajectory(
    content: str,
    format: str = "markdown",
    metadata: Optional[dict] = None,
    traj_id: Optional[str] = None,
    traj_dir: Optional[Path] = None,
    mask_patterns: Optional[list[str]] = None,
) -> dict:
    """
    Complete submission flow:

    1. Resolve *traj_dir* (from param or ``get_traj_dir()``).
    2. Generate *traj_id* if not provided.
    3. Adapt the input format to standard markdown + JSON metadata.
    4. Write ``traj_{id}.md`` and optionally ``traj_{id}.json``.
    5. Return ``{"traj_id": ..., "path": ..., "status": "stored"}``.

    ``mask_patterns``：去壳掩码正则列表；``None`` 时取 config 的
    ``ingest.mask_patterns``（默认空 = 不替换）。命中段在写 md 之前替换为
    占位符——剥掉评测 harness 的固定外壳，防聚类被任务外壳吸住。
    """
    traj_dir = Path(traj_dir) if traj_dir else get_traj_dir()
    traj_dir.mkdir(parents=True, exist_ok=True)

    if not traj_id:
        traj_id = generate_traj_id(traj_dir)

    md_content, json_metadata = adapt_trajectory(content, format, metadata)

    # 落盘前清洗：去 ANSI 转义 + 控制字符（终端/tool 原始输出常掺入），
    # 保证 splitlines 行数 == \n 行数（atom offset 与人类行号一致）、不喂垃圾给模型。
    from xskill.utils.sanitize import apply_mask_patterns, sanitize_trajectory_text
    md_content = sanitize_trajectory_text(md_content)

    # 去壳掩码：在入库转换阶段（写 md 之前）做，不在拆分阶段——落盘文本
    # 本身已去壳，下游拆分/聚类/embedding 一律看不到外壳原文。
    if mask_patterns is None:
        mask_patterns = ingest_config()["mask_patterns"]
    md_content = apply_mask_patterns(md_content, mask_patterns)

    # Write markdown
    md_path = traj_dir / f"{traj_id}.md"
    md_path.write_text(md_content, encoding="utf-8")

    # Write JSON metadata if non-empty
    if json_metadata:
        json_path = traj_dir / f"{traj_id}.json"
        json_path.write_text(
            json.dumps(json_metadata, ensure_ascii=False, indent=2),
            encoding="utf-8",
        )

    return {
        "traj_id": traj_id,
        "path": str(md_path),
        "status": "stored",
    }

```

```bash
sed -n '738,845p' src/xskill/ecosystems/_shared.py
```

```output
class JsonlIngester:
    """跨生态 JSONL session ingester——spec 化的扫盘 + bridge 逻辑。

    职责（**只**这些，不含 staging / flip / header 注入——那是 CC 专属的
    `CCSessionIngester` 在 wrapper 层做的事）：

    1. 扫 ``spec.sessions_path(home_root)`` 下匹配 ``spec.sessions_glob`` 的文件
    2. 用 ``spec.session_id_from_path`` 抽 session id，跟 ``seen_sessions`` 去重
    3. 用 ``spec.adapter_format`` 喂 ``submit_trajectory`` 桥成 ``traj_*.md``
    4. traj_id 取 ``<spec.traj_id_prefix><project>_<sid8>``——保留 ``traj_`` 前缀
       让 watcher 的 ``traj_*.md`` glob 继续匹配

    用法两种：

    - **One-shot**: ``ingester.scan_and_bridge(target_traj_dir, home_root=)``
      返回 record list 后由调用方处理（live test / 单测 / CLI 单跑用）
    - **Daemon thread**: ``ingester.start()`` 起后台 daemon 线程周期性
      ``_loop`` 调 ``scan_and_bridge``；``ingester.stop()`` 干净退出。
      用于 server.py startup hook 让生态 ingester 与 CC 一样常青运行。
      使用 daemon 模式时 ``target_traj_dir`` / ``home_root`` 必须在
      ``__init__`` 时传入（毕竟 thread 自己跑循环，没有调用方）。
    """

    def __init__(
        self,
        spec: EcosystemSpec,
        *,
        target_traj_dir: Path | str | None = None,
        home_root: Path | str | None = None,
        poll_interval: float = 10.0,
        on_new_sessions: Callable[[list[dict]], None] | None = None,
        settle_seconds: float | None = None,
        registry_db_path: Path | str | None = None,
    ):
        if spec.source_kind != "jsonl":
            # SQLite ingester 用单独的 SqliteIngester；早 fail 避免走错路。
            raise ValueError(
                f"JsonlIngester only supports source_kind='jsonl', got {spec.source_kind!r}"
            )
        self.spec = spec
        # daemon thread 用：one-shot 调用方不传，scan_and_bridge 参数兜底。
        self.target_traj_dir = Path(target_traj_dir) if target_traj_dir else None
        self.home_root = Path(home_root) if home_root else None
        self.poll_interval = poll_interval
        # 入库完成屏障（settle barrier）：源文件 mtime 距今 < settle 秒视为
        # "还在写"，本轮跳过。None = 每次 scan 时从 config 的
        # ingest.settle_seconds 读（daemon 长跑期间改配置即时生效，与
        # detect_known_ecosystems 每轮实测同一设计）；显式传值用于测试 /
        # SDK 调用方覆盖。
        self.settle_seconds = settle_seconds
        self.registry_db_path = (
            Path(registry_db_path)
            if registry_db_path is not None
            else None
        )
        # on_new_sessions: 可选 hook，每轮 scan 桥接到新 session 后调
        # 一次（参数是 submitted records）。openclaw 用这个 hook 做 canary
        # flip——发现新 session → pick_side → 跟 install_history 对比 →
        # 触发 copy-overwrite。codex / claude_code 不用（它们走 symlink，
        # 灰度由 CCSessionIngester / 老逻辑负责）。
        self.on_new_sessions = on_new_sessions
        # daemon thread 内部用：seen_sessions 持久化在 instance 上避免每轮
        # _scan_seen_sessions 重扫（thread 跑期间 traj_*.json 自己也在生成）。
        self._seen: set[str] = set()
        self._stop = threading.Event()
        self._thread: threading.Thread | None = None
        self._stats = {
            "polls": 0, "ingested": 0, "errors": 0, "last_poll": None,
        }

    # ── daemon thread lifecycle ──────────────────────────────────

    def start(self) -> None:
        """起 daemon 线程，周期性调 ``scan_and_bridge``。幂等：已在跑则
        no-op。

        要求 ``__init__`` 传入了 ``target_traj_dir``——daemon 线程没有调
        用方传参数。``home_root`` 可空（fallback ``Path.home()``）。
        """
        if self._thread and self._thread.is_alive():
            return
        if self.target_traj_dir is None:
            raise RuntimeError(
                "JsonlIngester.start() requires target_traj_dir in __init__"
            )
        # 重启场景：从磁盘上已有 traj_*.json 恢复 seen 集合，避免重复桥接。
        self._seen = _scan_seen_sessions(self.target_traj_dir)
        self._stop.clear()
        self._thread = threading.Thread(
            target=self._loop, daemon=True,
            name=f"xskill-{self.spec.name}-ingester",
        )
        self._thread.start()
        logger.info(
            "JsonlIngester(%s) started "
            "(source=%s, target=%s, interval=%.1fs, %d sessions pre-seen)",
            self.spec.name,
            self.spec.sessions_path(self.home_root or Path.home()),
            self.target_traj_dir,
            self.poll_interval,
            len(self._seen),
        )

    def stop(self) -> None:
        """干净停止 daemon 线程（避免 zombie）。"""
        self._stop.set()
        if self._thread:
            self._thread.join(timeout=self.poll_interval + 5)
```

## 6. Registry discovery starts the trajectory state machine

The watcher does not react directly to filesystem events. Every polling round first harvests finished futures, discovers files into SQLite, then submits only work permitted by durable status.

The principal trajectory states are:

`discovered / updated → splitting → split_done → indexed → done`

There are terminal/side states for `filtered`, interest-filtered `not_fit`, and retryable `error`. On restart, a row stuck in `splitting` with no matching future is returned to `discovered`; legacy `clustering` rows return to `indexed`. Errors below the retry limit return to discovery and increment their retry count.

`discover_trajectories` compares file mtimes. A new file inserts a row with source model, harness, and the watch-directory label as canonical user attribution. Growth after a completed state changes the row to `updated`; growth while split work is active leaves the old mtime untouched so the next poll notices it after the in-flight future settles. Invalid, empty, or malformed trajectories are filtered before spending an LLM call.

The loop is non-blocking. Each bounded pool admits at most `workers` running plus `2 × workers` queued tasks. A full pool returns `None`; the durable status stays eligible for a later scan.

```bash
sed -n '84,136p' src/xskill/pipeline/runner.py
```

```output
class DirectoryWatcher:
    """流水线式目录监听器。每条 traj 独立流转，不分批不阻塞。

    v2 状态机：
      discovered → splitting → split_done → indexed → done

    与 v1 (meta-level) 的差异：
    - splitting 阶段调 TaskAgent 拆 AtomTask，落盘到 ``<traj_root>/<traj_id>/tasks/``
    - indexed 阶段以 AtomTask 为单位整批重建 ``<traj_root>/index.pkl``
    - cluster 阶段**跨轨迹池化**：把所有 indexed 轨迹里尚未落地的 atom 汇成一池，
      按 ``cluster.batch_size`` 分批，提交给全局 cluster pool。
    - indexed → done 由 ``_finalize_completed_trajs`` 标：一条轨迹的 atom 全部落进某个
      skill 的 ``.candidates.yml`` 时才 done（文件系统即队列，天然去重+断点续传）。
    """

    def __init__(self, *, llm=None, embed_client=None, config=None,
                 skill_dir=None, poll_interval=5.0, pool_config=None,
                 max_retries=3, db_path=None,
                 store=None, agno_agent_factory=None, home_root=None,
                 xskill_home=None, config_path=None,
                 logs_dir=None,
                 spill_root=None,
                 usage_ledger=None,
                 server_mode=False, install_history_path=None,
                 on_poll_hook=None):
        self.llm = llm
        self.embed_client = embed_client
        self.usage_ledger = usage_ledger
        self.config = config or {}
        self.skill_dir = Path(skill_dir) if skill_dir else None
        # home_root：install_to_claude_code 的 target root。生产 daemon 不
        # 传（None）→ 落到 server._home_root() (默认 Path.home())。测试
        # 必须显式传 tmp_path 防止污染真实 ~/.claude/skills/。
        self.home_root = Path(home_root) if home_root else None
        # server_mode：team server 模式。server 是纯 server——不装 skill 到
        # 本机生态、不做单机灰度轮转、不做本地手改回流（手改走 client
        # push-edit → user-staging/<client_id> 分支）。只跑 agent 流水线
        # （split/cluster/SkillEdit/canary 判定）+ CS 归因打分。
        self.server_mode = bool(server_mode)
        # XSkill 自身状态根与 Agent 生态 home_root 是两类路径，不能混用。
        from xskill.config import XSKILL_HOME
        xskill_state_root = (
            Path(xskill_home)
            if xskill_home is not None
            else XSKILL_HOME
        ).expanduser().resolve()
        self.install_history_path = (
            Path(install_history_path) if install_history_path
            else xskill_state_root / "install_history.jsonl"
        )
        # 冷启动批量 flush：rebuild 写 COLD_START 文件后，watcher 等流水线空闲再
        # 做一次 SkillEdit 扫描；这是 XSkill 状态，不能跟生态 home_root 混用。
        from xskill.pipeline.cold_start import ColdStartSignal
```

```bash
sed -n '325,419p' src/xskill/pipeline/runner.py
```

```output
    def _loop(self):
        # watcher 线程内会懒加载 agno（导入期即构造 asyncio.Lock()）。
        # Python 3.9 非主线程无事件循环时构造会崩 —— 先给本线程装一个。
        _install_thread_event_loop()
        while not self._stop.is_set():
            if not self._pause.is_set():
                if self.on_poll_hook is not None:
                    try:
                        self.on_poll_hook()
                    except Exception:
                        logger.exception("watcher on_poll_hook failed")
                try:
                    self._scan_once()
                except Exception:
                    logger.exception("watcher scan error")
            self._stop.wait(self.poll_interval)

    def _scan_once(self):
        """一次扫描：收割 → 发现 → 提交任务 → 独立扫 pending skill edits。"""
        self._last_poll = time.time()
        self._stats["polls"] += 1
        kw = self._db_kw()
        self._refresh_interests()

        # ── Step 0: 收割已完成的 futures ──
        self._harvest()

        # ── 本轮已消费索引（惰性）：只有真遇到未打 clustered 标记的 atom 才扫盘建
        # atom_id→skill 索引，供 _atom_consumed 走 O(1) 命中，替代逐 atom 全量重读磁盘
        # （O(atoms×skills) 烧核握死 GIL 的根因）。稳态全 clustered → 完全不碰磁盘，
        # 避免 1 万 skill 下每轮白扫 1 万个 .candidates.yml。本轮内快照一致有效。
        from xskill.skill.candidates import LazyConsumedIndex
        consumed_index = LazyConsumedIndex(self.skill_dir)

        # ── Step 1-4: 对每个目录扫描 + 提交 split/embed + 收集 cluster atom ──
        watch_dirs = list_watch_dirs(**kw)
        active_watch_dirs = []
        for wd in watch_dirs:
            if self._stop.is_set():
                break
            if not wd.get("auto_index"):
                continue
            active_watch_dirs.append(wd)
            self._scan_dir(wd, consumed_index, **kw)

        # 全部 watch_dir 共用一个 pending/claimed 队列；持续提交到 cluster 池满。
        # 最后不足 batch_size 的尾批也立即提交。
        self._submit_cluster_batches()

        # 已完成归类的轨迹在下一次 durable atom 检查中进入 done。
        if self.skill_dir:
            for wd in active_watch_dirs:
                dir_path = Path(wd["path"])
                if dir_path.is_dir():
                    self._finalize_completed_trajs(
                        wd["id"], dir_path, consumed_index, **kw,
                    )

        # ── Step 5a: 用户点名的 generate 入队到 SkillEdit 同一线程池 ──
        # 先于自动 SkillEdit 提交，避免后台整理把用户任务挤到池外。
        self._submit_generate_jobs()

        # ── Step 5: 独立扫所有 skill 目录的 candidates buffer ──
        # 这步与具体 atom 处理解耦：即便某些 atom cluster 失败，buffer
        # 已满阈值的 skill 仍能在每轮 scan 中被检出 + 触发 SkillEdit。
        # 不放在 _scan_dir 内是因为 skill_dir 不是 watch_dir，跟 wd 循环
        # 无关——每个 watcher 只有一个全局 skill_dir。
        self._run_skill_edit_step()
        self._check_scripting_requests()

        # ── Step 6: 灰度判定独立轮询 ──
        # 对每个 staging 分支存在的 skill 跑 AtomCanary.check_and_decide：
        # 收齐 5 条评分就裁决 promote/reject，超时 max_days_hold 就 discard。
        # 这条与 cluster / score 链路彻底解耦——灰度系统自治。
        self._check_canary_decisions()

        # ── Step 7: 用户手改回流检测 ──
        # 用户改 ~/.claude/skills/<name>/* (symlink 指向源仓库) 后 ≥3 分钟
        # 没新改动 → 触发 UserEditAbsorbAgent 把手改吸回 main，并删除任何
        # 在飞 staging（用户改是 ground truth，优先级压过灰度）。
        # server 模式跳过：server 本机没有 symlink 出去的 skill 给用户改；
        # client 手改走 push-edit 进 user-staging/<client_id> 分支。
        if not self.server_mode:
            self._check_user_edits()

        # ── Step 8: 单机 canary 流量入口轮转 ──
        # 周期性（每 canary.rotate_interval 秒）按概率把每个有 staging 分支
        # 的 skill 子仓 checkout 到 main 或 staging——这是 staging 拿到真实
        # ux_score 样本的唯一入口。否则 staging 永远没流量 → check_and_decide
        # 永远 waiting → 最终 timeout_discarded，灰度形同虚设。
        # server 模式跳过：server 不装 skill 到本机，无"流量入口"概念。
        # CS 模式的分桶在 client 的 reconcile_skill_sides 里按 client_id 做。
        if not self.server_mode:
            self._reconcile_skill_sides()

```

```bash
sed -n '1731,1823p' src/xskill/pipeline/registry.py
```

```output
def discover_trajectories(
    watch_dir_id: int,
    dir_path: Path,
    *,
    db_path: Optional[Path] = None,
) -> list[str]:
    """扫描目录中的 traj_*.md，upsert 到 DB。返回新发现的文件名列表。

    续写重拆触发：已存在的文件若 mtime 增大（客户端追加内容后重传覆盖写,
    mtime 变更），把它从"已落定"状态翻回 ``updated``——watcher 下一轮会像
    ``discovered`` 一样重新提交 split，TaskAgent 用 ``last_offset`` 续接点
    只拆新增内容。``updated`` 不计入返回的 new_files（只统计真·新文件）。
    """
    dir_path = Path(dir_path)
    new_files: list[str] = []
    with pooled_connection(db_path) as conn:
        # P2-2.1 归因(D5):入库即把 watch_dir 的 label 写进 user_key,聚合层
        # 不再 JOIN watch_dirs.label——source 唯一。CS 模式各用户桶(label=
        # sessions 桶目录名=user_name/client_id)不论 ecosystem 是 team_client
        # 还是 bridge 检出的真实生态(ngagent 等)都归因;无 label 的本地目录
        # 留空,聚合层显示 '(local)'。
        wd = conn.execute(
            "SELECT label FROM watch_dirs WHERE id=?",
            (watch_dir_id,),
        ).fetchone()
        user_key = (wd["label"] or "") if wd else ""

        existing = {
            row["filename"]: row
            for row in conn.execute(
                "SELECT filename, status, process_action, file_mtime FROM trajectories"
                " WHERE watch_dir_id=?",
                (watch_dir_id,),
            ).fetchall()
        }

        for md in sorted(dir_path.glob("traj_*.md")):
            if md.name.endswith(".meta"):
                continue
            mtime = md.stat().st_mtime
            row = existing.get(md.name)
            if row is None:
                conn.execute(
                    "INSERT INTO trajectories"
                    " (watch_dir_id, filename, file_mtime, source_model,"
                    "  source_harness, user_key)"
                    " VALUES (?, ?, ?, ?, ?, ?)",
                    (watch_dir_id, md.name, mtime, _sidecar_model(md),
                     _sidecar_field(md, "harness"), user_key),
                )
                new_files.append(md.name)
                continue

            stored_mtime = row["file_mtime"] or 0
            if mtime <= stored_mtime:
                continue  # 没变化
            status = row["status"]
            if status in _ACTIVE_STATUSES:
                # 正在 split/cluster——别打架,留旧 mtime,落定后下一轮再检出。
                continue
            if status == "discovered":
                # 还没开拆,后续 split 会读到最新内容（last_offset=0 全量拆）。
                # 只更 mtime,不必翻 updated。
                conn.execute(
                    "UPDATE trajectories SET file_mtime=?"
                    " WHERE watch_dir_id=? AND filename=?",
                    (mtime, watch_dir_id, md.name),
                )
                continue
            if (
                status == TrajectoryStatus.FILTERED.value
                and row["process_action"] == ProcessAction.NOT_FIT.value
            ):
                # Interest-filtered trajectories re-enter only after interests change.
                conn.execute(
                    "UPDATE trajectories SET file_mtime=?"
                    " WHERE watch_dir_id=? AND filename=?",
                    (mtime, watch_dir_id, md.name),
                )
                continue
            # 已落定（done/indexed/split_done/error/filtered/updated）+ 内容变更
            # → 翻 updated,等下一轮重新 split（续接点续拆）。
            conn.execute(
                "UPDATE trajectories SET status='updated', file_mtime=?,"
                " updated_at=datetime('now')"
                " WHERE watch_dir_id=? AND filename=?",
                (mtime, watch_dir_id, md.name),
            )

        conn.commit()
        return new_files


```

```bash
sed -n '2334,2442p' src/xskill/pipeline/runner.py
```

```output
    def _scan_dir(self, wd, consumed_index, **kw):
        wd_id = wd["id"]
        dir_path = Path(wd["path"])
        if not dir_path.is_dir():
            return

        # 清理僵尸 splitting：``_do_split`` 在跑（stage='split'）。一旦 DB 里
        # 有 splitting 但没对应 in-flight future = 上次 daemon 退出时 future 被切
        # / 进程崩。回退到 discovered 让 watcher 下轮重新调度。
        # （cluster 无此问题：watcher 不再把轨迹置 clustering，崩溃时轨迹停在
        #  indexed，下一轮天然重新进池。遗留 clustering 在下方无条件回退 indexed。）
        for fname in get_trajs_by_status(wd_id, "splitting", **kw):
            if not any(
                future_info.get("stage") == "split"
                and future_info.get("wd_id") == wd_id
                and future_info.get("fname") == fname
                for future_info in self._futures.values()
            ):
                update_traj_status(wd_id, fname, "discovered", **kw)

        # 跨轨迹批处理下 watcher 不再把轨迹置 "clustering"（done 由
        # _finalize_completed_trajs 按 atom 落地情况标）。任何遗留的 "clustering"
        # （旧 daemon 升级残留 / 历史数据）一律回退 "indexed" 让其重新进池——
        # 已落地的 atom 会在 _collect_cluster_batch 被去重跳过，不会重复消费。
        for fname in get_trajs_by_status(wd_id, "clustering", **kw):
            update_traj_status(wd_id, fname, "indexed", **kw)

        # 重试 error
        for fname in get_trajs_by_status(wd_id, "error", max_retries=self.max_retries, **kw):
            update_traj_status(wd_id, fname, "discovered", **kw)
            increment_retry(wd_id, fname, **kw)
            self._stats["retries"] += 1

        # 发现新文件
        new = discover_trajectories(wd_id, dir_path, **kw)
        if new:
            self._stats["new_trajs"] += len(new)
            logger.info("[%s] discovered %d new", dir_path.name, len(new))

        # ── 提交 split 任务（discovered / updated → splitting）──
        # 需要 llm；缺则 traj 留在 discovered 等条件齐备。
        # ``updated``（续写重传后 discover 翻的状态）与 ``discovered`` 同等处理：
        # 同样跑 _do_split，TaskAgent 用 last_offset 续接点只拆新增内容。
        if self.llm is not None:
            for status in ("discovered", "updated"):
                for fname in get_trajs_by_status(
                    wd_id, status,
                    limit=self._pools["split"].total_capacity,
                    **kw,
                ):
                    validation = validate_trajectory_source(dir_path / fname)
                    if not validation.valid:
                        update_traj_status(
                            wd_id, fname, "filtered",
                            error_msg=validation.reason or "invalid_trajectory",
                            **kw,
                        )
                        logger.info(
                            "%s filtered before split: %s",
                            fname, validation.reason,
                        )
                        continue
                    fut = self._pools["split"].submit(
                        self._do_split,
                        dir_path,
                        fname,
                        task={
                            "kind": "traj",
                            "traj_id": (dir_path / fname).stem,
                            "watch_dir": dir_path.name,
                        },
                    )
                    if fut is None:
                        break
                    update_traj_status(wd_id, fname, "splitting", **kw)
                    self._futures[fut] = {
                        "wd_id": wd_id, "fname": fname, "stage": "split",
                    }

        # ── 提交 embed 任务（split_done → indexed，整批一个任务） ──
        if self.embed_client is not None:
            split_done_files = get_trajs_by_status(wd_id, "split_done", **kw)
            if split_done_files and not any(
                i["stage"] == "embed" and i["wd_id"] == wd_id for i in self._futures.values()
            ):
                fut = self._pools["embed"].submit(
                    self._do_atom_index,
                    dir_path,
                    wd_id,
                    split_done_files,
                    task={"kind": "embed", "count": len(split_done_files)},
                )
                if fut is not None:
                    self._futures[fut] = {
                        "wd_id": wd_id,
                        "fname": "_batch_embed",
                        "stage": "embed",
                    }

        # ── Cluster：只收集到全局 pending，不在目录循环内等待或串行 ──
        if self.skill_dir:
            self._collect_cluster_atoms(
                dir_path, wd_id, consumed_index, **kw,
            )

    # ───────────────────────────────────────────────────────────
    # Helpers: store / agno factory 按需获取
    # ───────────────────────────────────────────────────────────

```

## 7. TaskAgent turns a transcript into retryable atoms

A trajectory is too coarse to become a reusable Skill: one session can contain several unrelated user intents. `TaskAgent` performs a single agentic split over the normalized Markdown.

Before calling the model it extracts legal user-section line numbers. The model can inspect bounded context with tools, but it can submit an atom only at one of those verified boundaries. Each submission provides intent, summary, tags, used Skills, and the already-observable UX score. The framework derives half-open line ranges, so the model never invents end offsets.

Several invariants make splitting restart-safe:

- atom IDs append deterministically as `atom_<traj_id>_NNNN`;
- the previous and next atoms form a linked list;
- `raw_segment` preserves the exact source slice;
- an appended trajectory resumes from the last atom's end;
- all newly emitted ranges must cover the remaining file continuously through EOF;
- a run that saw user turns but submitted zero atoms is an error, not silent success.

Atoms are saved atomically before the split future updates SQLite to `split_done`. The embed stage then rebuilds the directory's `index.pkl`, reusing vectors for unchanged atom IDs when the embedding model is unchanged, and moves every included trajectory to `indexed`.

```bash
sed -n '351,474p' src/xskill/agents/task_agent.py
```

```output
    def run(self, *, traj_id: str, traj_path: Path) -> list[AtomTask]:
        """弃窗单趟拆分整条轨迹：抽全轨迹 User 地图喂一次,agent 一趟出全部 atom。

        EOF 硬校验：首 atom offset_start=1、末 atom offset_end=total+1；有 User
        轮却 0 提交 → 抛错；落盘后断言区间铺满 [resume, total+1)。
        增量续拆：resume_line = last_offset,只切 ≥ resume 的新意图。
        """
        traj_path = Path(traj_path)
        text = traj_path.read_text(encoding="utf-8")
        lines = text.splitlines(keepends=True)
        total_lines = len(lines)
        source_model = _sidecar_model(traj_path)

        resume_line = self.store.last_offset(traj_id) or 1
        if resume_line > total_lines:
            return []  # 没有新增行,省一次 LLM 调用

        queries = _extract_user_queries(lines)
        # 续拆：只保留 ≥ resume_line 的 User 回合作为可切边界。
        new_queries = [(ln, snip) for ln, snip in queries if ln >= resume_line]
        if not new_queries:
            # 续接点之后没有真正的用户意图回合（全是机器噪声 / 无 User）→
            # 无新 atom。但若是首轮（resume==1）且全文确实无任何 User 回合,
            # 则整条无可拆边界,合法返回空（无 User 轮,不触发 0 提交抛错）。
            return []

        prior_atoms = self.store.list_by_traj(traj_id)
        prior_atom = prior_atoms[-1] if prior_atoms else None
        valid_lines = [ln for ln, _ in new_queries]

        submitted = self._run_agent(
            traj_id=traj_id, traj_path=traj_path, source_model=source_model,
            resume_line=resume_line, prior_atoms=prior_atoms,
            all_lines=lines, total_lines=total_lines,
            queries=queries, valid_lines=valid_lines,
        )
        if not submitted:
            # 有 User 轮却 0 提交 → 静默空,绝不收下（设计 §4.4）。
            raise RuntimeError(
                f"TaskAgent: traj {traj_id} 有 {len(new_queries)} 个待拆 User "
                f"回合却 0 提交（疑似 LLM 静默失败 / 限流空返）,标记重拆")

        parsed = self._derive_ranges(
            submitted, floor_line=resume_line, eof_line=total_lines + 1)

        next_idx = len(prior_atoms) + 1
        new_atoms: list[AtomTask] = []
        for i, p in enumerate(parsed):
            atom_id = f"atom_{traj_id}_{next_idx + i:04d}"
            os_line = p["offset_start"]
            oe_line = p["offset_end"]
            new_atoms.append(AtomTask(
                atom_id=atom_id,
                traj_id=traj_id,
                offset_start=os_line,
                offset_end=oe_line,
                intent=p["intent"],
                summary=p["summary"],
                tags=p["tags"],
                used_skills=p["used_skills"],
                ux_score=p.get("ux_score"),
                pre_atom_id=None,
                post_atom_id=None,
                context_prefix=self._context_prefix(text, lines, os_line),
                raw_segment="".join(lines[os_line - 1:oe_line - 1]),
                source_model=source_model,
            ))

        # 本批内部相邻 atom 互填 pre/post
        for i in range(1, len(new_atoms)):
            new_atoms[i].pre_atom_id = new_atoms[i - 1].atom_id
            new_atoms[i - 1].post_atom_id = new_atoms[i].atom_id

        # 与 store 末尾 atom 衔接（续拆场景）
        if prior_atom is not None:
            new_atoms[0].pre_atom_id = prior_atom.atom_id
            prior_atom.post_atom_id = new_atoms[0].atom_id
            self.store.save(prior_atom)

        for a in new_atoms:
            self.store.save(a)

        self._assert_eof_coverage(traj_id, resume_line=resume_line,
                                  total_lines=total_lines)
        return new_atoms

    # ── EOF 覆盖硬校验 ────────────────────────────────────────────

    def _assert_eof_coverage(self, traj_id: str, *, resume_line: int,
                             total_lines: int) -> None:
        """落盘后断言本 traj 全部 atom 区间无缝无叠地铺满 [resume, total+1)。

        续拆场景下 [1, resume) 已由历史 atom 覆盖,本次只校验
        [resume_line, total_lines+1) 这段被新 atom 无缝铺满。
        """
        atoms = sorted(
            self.store.list_by_traj(traj_id),
            key=attrgetter("offset_start"),
        )
        # 取覆盖 [resume, ...) 的尾段（新 atom 起点 ≥ floor=resume 或并入首 atom）。
        floor = resume_line
        seg = [a for a in atoms if a.offset_end > floor]
        if not seg:
            raise RuntimeError(
                f"TaskAgent: traj {traj_id} 落盘后无 atom 覆盖 [{floor}, "
                f"{total_lines + 1})")
        cursor = min(seg[0].offset_start, floor)
        if cursor > floor:
            raise RuntimeError(
                f"TaskAgent: traj {traj_id} 覆盖起点 {cursor} > 续接点 {floor}")
        for a in seg:
            if a.offset_start != cursor:
                raise RuntimeError(
                    f"TaskAgent: traj {traj_id} atom 区间不连续：期望起点 "
                    f"{cursor},实得 {a.offset_start}（atom {a.atom_id}）")
            cursor = a.offset_end
        if cursor != total_lines + 1:
            raise RuntimeError(
                f"TaskAgent: traj {traj_id} 末 atom 未覆盖到 EOF：终点 "
                f"{cursor} != total+1 {total_lines + 1}")

    # ── agentic 拆分（单趟）───────────────────────────────────────

    def _run_agent(self, *, traj_id, traj_path, source_model, resume_line,
```

```bash
sed -n '474,574p' src/xskill/agents/task_agent.py
```

```output
    def _run_agent(self, *, traj_id, traj_path, source_model, resume_line,
                   prior_atoms, all_lines, total_lines, queries,
                   valid_lines) -> list[dict]:
        """构造 agent + run-scoped 工具,跑一趟工具调用循环。

        返回按提交顺序的 ``submitted`` 列表。``submit_atom`` 把校验通过的 atom
        append 进闭包捕获的本地列表——每次 run 各自一份,线程安全（watcher 并发
        拆多条 traj 不串）。``agent.run()`` 后查 run_response.status,error 即抛。
        """
        submitted: list[dict] = []
        not_fit_reasons: list[str] = []
        user_msg = self._build_user_msg(
            traj_id=traj_id, traj_path=traj_path, source_model=source_model,
            resume_line=resume_line, prior_atoms=prior_atoms,
            total_lines=total_lines, queries=queries,
        )
        from xskill.agents import agent_tools
        tools = agent_tools.make_task_agent_tools(
            submitted=submitted,
            valid_lines=valid_lines,
            resume_line=resume_line,
            total_lines=total_lines,
            all_lines=all_lines,
            user_msg=user_msg,
            not_fit_reasons=not_fit_reasons if self.interests else None,
        )
        agent = self.agno_agent_factory(
            instructions=[self._system_prompt],
            tools=tools,
        )
        # 把这次拆分的逐轮 CoT/工具调用流式写进 logs/agents/task_agents/<traj_id>.log
        from xskill.agents.agent_trace import trace_to
        sink = (
            self.logs_dir / "agents" / "task_agents" / f"{traj_id}.log"
            if self.logs_dir is not None
            else None
        )
        with trace_to(sink):
            run_response = agent.run(user_msg)
        self._check_run_status(traj_id, run_response)
        if not_fit_reasons and not submitted:
            raise TrajectoryNotFit(not_fit_reasons[-1])
        return submitted

    @staticmethod
    def _check_run_status(traj_id: str, run_response: Any) -> None:
        """查 agno run_response.status,error 即抛（绝不把静默失败当成功收下）。

        stub / 旧返回对象没有 status 字段时视为正常（测试注入的假 run 结果）。
        """
        status = getattr(run_response, "status", None)
        if status is None:
            return
        sval = getattr(status, "value", status)
        if str(sval).upper() == "ERROR":
            raise RuntimeError(
                f"TaskAgent: traj {traj_id} agent.run() 返回 status=ERROR,"
                "标记重拆（不静默收空）")

    @staticmethod
    def _derive_ranges(submitted: list[dict], *,
                       floor_line: int, eof_line: int) -> list[dict]:
        """把按序提交的 start_line 推成 [start, end) 行区间。

        首 atom 从 ``floor_line`` 起（含续接点到首 ## User 之间的衔接行/前言）；
        其余从各自 start_line 起。终点 = 下一 atom 起点 / EOF（total+1）。
        ``submit_atom`` 已保证严格递增 + 合法行,这里只做纯几何推导,并由
        ``run()`` 末尾的 ``_assert_eof_coverage`` 兜死覆盖。
        """
        out: list[dict] = []
        for i, p in enumerate(submitted):
            os_line = floor_line if i == 0 else p["start_line"]
            if i + 1 < len(submitted):
                oe_line = submitted[i + 1]["start_line"]
            else:
                oe_line = eof_line
            out.append({**p, "offset_start": os_line, "offset_end": oe_line})
        return out

    # ── prompt helpers ────────────────────────────────────────────

    def _context_prefix(self, text: str, lines: list[str],
                        start_line: int) -> str:
        """生成 atom 起始行之前内容的省略表示（给 prompt / ux 评分用）。"""
        char_off = sum(len(ln) for ln in lines[:start_line - 1])
        if char_off <= 200:
            return text[:char_off]
        return text[:200] + f"\n\n[省略 {char_off - 200} 字符]\n\n"

    def _build_user_msg(self, *, traj_id, traj_path, source_model, resume_line,
                        prior_atoms, total_lines, queries) -> str:
        """构造 user 消息（discover / update 共用一份模板）。

        context-0 只放 User 提问地图（不含 assistant 正文）+ 元信息 + 续拆衔接块。
        """
        if prior_atoms:
            prior_block = "".join(
                _inject(
                    PRIOR_ATOM_TEMPLATE,
                    atom_id=str(a.atom_id),
                    atom_path=str(traj_path),
```

```bash
sed -n '2478,2563p' src/xskill/pipeline/runner.py
```

```output
    def _do_split(self, dir_path, fname):
        """跑 TaskAgent 拆 AtomTask。返回 (fname, num_atoms_added, last_offset, last_atom_id, err)。

        v2.3: TaskAgent 走 agentic 工具调用（submit_atom/readfile/grep），用
        和 cluster/edit 同一个 agno 工厂。``updated`` 状态的续写轨迹和首次
        ``discovered`` 走同一条路径——TaskAgent 内部用 last_offset 续接点只拆
        新增内容。
        """
        import time
        from xskill.agents.task_agent import TaskAgent, TrajectoryNotFit
        md_path = dir_path / fname
        validation = validate_trajectory_source(md_path)
        if not validation.valid:
            logger.info("⊘ split 跳过 %s（%s）", fname, validation.reason or "invalid")
            return (
                fname, 0, 0, None,
                validation.reason or "invalid_trajectory",
            )
        traj_id = md_path.stem
        store = self._store_for(dir_path)
        # 处理前：打一条"开始拆"(带行数)——这是真正干活的边界,让人看到它在跑、
        # 跑哪条、多大,而不是只看 cluster 阶段无脑刷 0-total。
        try:
            n_lines = sum(1 for _ in md_path.open(encoding="utf-8", errors="ignore"))
        except OSError:
            n_lines = -1
        logger.info("⟳ split 开始 %s（%d 行）", fname, n_lines)
        t0 = time.monotonic()
        current_interests = list(self.interests)
        current_interest_fingerprint = self.interest_fingerprint
        from xskill.agents import agent_tools
        tool_context = agent_tools.create_agent_tool_context(
            skill_dir=self.skill_dir,
            data_dir=self.skill_dir,
            config=self.config,
            atom_skill_dir=self.skill_dir,
            atom_store=store,
            default_traj_root=dir_path,
            spill_root=self.spill_root,
            usage_ledger=self.usage_ledger,
            registry_db_path=self.db_path,
        )
        try:
            with agent_tools.use_agent_tool_context(tool_context):
                atoms = TaskAgent(
                    agno_agent_factory=self._factory(),
                    store=store,
                    traj_root=dir_path,
                    skill_dir=self.skill_dir,
                    interests=current_interests,
                    logs_dir=self.logs_dir,
                ).run(traj_id=traj_id, traj_path=md_path)
        except TrajectoryNotFit as not_fit_error:
            logger.info(
                "⊘ split not_fit %s（interest_fingerprint=%s）: %s",
                fname,
                current_interest_fingerprint[:12],
                not_fit_error.reason,
            )
            return (
                fname,
                0,
                store.last_offset(traj_id),
                store.last_atom_id(traj_id),
                {
                    "process_action": ProcessAction.NOT_FIT.value,
                    "reason": not_fit_error.reason,
                    "interest_fingerprint": current_interest_fingerprint,
                },
            )
        last_off = store.last_offset(traj_id)
        last_id = store.last_atom_id(traj_id)
        # 处理后：打一条"拆完"(带 atom 数 + 耗时),0 个也明确说明是"无可拆 User 回合"。
        dt = time.monotonic() - t0
        if atoms:
            logger.info("✓ split 完成 %s → %d atoms（%.1fs）", fname, len(atoms), dt)
        else:
            logger.info("✓ split 完成 %s → 0 atoms（无可拆 User 回合,%.1fs）", fname, dt)
        return (fname, len(atoms), last_off, last_id, None)

    def _do_atom_index(self, dir_path, wd_id, filenames):
        """整批重建 AtomTask 向量索引。返回 (wd_id, filenames)。"""
        store = self._store_for(dir_path)
        store.rebuild_vector_index(self.embed_client)
        return (wd_id, filenames)

```

## 8. ClusterAgent routes atoms into candidate buffers

All `indexed` trajectories across all watch directories feed one global atom queue. The watcher skips atoms whose JSON already says `clustered=true`, and uses a lazy scan of candidate buffers only as a compatibility check for older unmarked atoms. `claimed_atoms` prevents duplicate submission while a batch is queued or running.

The cluster pool takes up to the configured batch size—eight by default—and sends one possibly cross-trajectory batch to a `TaskClusterAgent`. A `MultiAtomTaskStore` lets its tools find atoms spread across different team-client directories. The agent sees a compact catalog of existing Skill names/descriptions/states and can:

- read atoms and their source trajectory slices;
- inspect a Skill and its pending tasks;
- create a new Skill repository;
- add one or many atoms to a Skill;
- move an atom between Skills;
- assign the atom's observable UX score.

An atom may support more than one Skill. Tool writes are recorded in a run-scoped `ClusterResultRecorder`, while a single-thread `ClusterWriteQueue` serializes mutations to shared candidate files. Successful tool writes remain authoritative even if the model call later fails. Every atom with at least one recorded assignment is atomically resaved with `clustered=true`; unassigned or failed atoms become eligible for the next scan.

Each target repository has a `.candidates.yml` buffer. Entries are keyed by atom ID and carry a 0–10 `weightscore`. The file is atomically replaced under the per-repository lock, and the same write updates the database projection. A cumulative score of ten makes the Skill eligible for editing. This file is a durable handoff between two independent agents, not an in-memory message queue.

```bash
sed -n '2564,2747p' src/xskill/pipeline/runner.py
```

```output
    def _collect_cluster_atoms(self, dir_path, wd_id, consumed_index, **kw):
        """Collect every new unclustered atom into the global pending queue.

        过滤靠 atom 的耐久 ``clustered`` 标记——已消费 atom（含上一批刚写入的、
        以及进程被 kill 前已消费的）一律跳过。这从机制上同时实现了**去重**与
        **断点续传**：文件系统即队列（atom json = 待消费池，atom.clustered =
        已消费标记），不需要额外的 DB 表或游标。用 atom 上的耐久标记而非
        ``.candidates.yml`` 成员判定——后者会被 SkillEdit 晋升清空，会让已消费
        atom 看起来又"未消费"而被重复送 LLM。

        ``claimed_atoms`` covers both pending and running batches, so repeated
        scans never enqueue the same atom twice.
        """
        store = self._store_for(dir_path)
        for fname in get_trajs_by_status(wd_id, "indexed", **kw):
            traj_id = (dir_path / fname).stem
            for atom in store.list_by_traj(traj_id):
                if self._atom_consumed(atom, consumed_index):
                    continue  # 已消费 → 跳过（去重 + 断点续传）
                if atom.atom_id in self.claimed_atoms:
                    continue
                self.claimed_atoms.add(atom.atom_id)
                self.pending_atoms.append(atom.atom_id)

    def _submit_cluster_batches(self) -> None:
        """Fill the cluster pool without ever waiting in the watcher thread."""
        while self.pending_atoms:
            batch_size = min(self.cluster_batch_size, len(self.pending_atoms))
            atom_ids = list(self.pending_atoms)[:batch_size]
            future = self._pools["cluster"].submit(
                self._do_cluster_batch,
                atom_ids,
                task={"kind": "atom_batch", "atom_ids": list(atom_ids)},
            )
            if future is None:
                return
            for _ in atom_ids:
                self.pending_atoms.popleft()
            self._futures[future] = {
                "stage": "cluster",
                "atom_ids": atom_ids,
            }
            self.cluster_futures.add(future)

    def _atom_consumed(self, atom, consumed_index) -> bool:
        """atom 是否已被 cluster 消费。耐久标记 ``clustered`` 为主（O(1)，扛得住
        SkillEdit 晋升清空 .candidates.yml 与进程重启）；未打标记的（旧 daemon 落
        的 / 外部预置的）回退查本轮 ``consumed_index``（惰性索引）——任何 skill buffer
        里出现过即视为消费过。常态走快路径不碰索引；回退时索引惰性建一次、本轮复用
        （稳态全 clustered → 索引根本不构建，1 万 skill 下零扫盘）。"""
        if atom.clustered:
            return True
        return consumed_index.contains(atom.atom_id)

    def _do_cluster_batch(self, atom_ids):
        """对一批（可能跨多条轨迹）atom 调**一个** ClusterAgent，只跑 cluster。

        把"逐 atom 一次 LLM 往返"压成"一批一次往返"。edit 触发独立由
        ``_check_pending_skill_edits`` 每轮 scan 完成，不依赖本批成功。整批
        抛异常（如 LLM 余额耗尽）由 ``_harvest`` 记日志后忽略：atom 留在未
        落地池，下一轮 scan 重新进池——已落地的会被 ``_collect_cluster_batch``
        去重跳过，不重复烧 token。

        返回 ``[result_dict, ...]``（顺序同 atom_ids）。
        """
        from xskill.pipeline.atom import MultiAtomTaskStore

        stores = [
            self._store_for(Path(wd["path"]))
            for wd in list_watch_dirs(**self._db_kw())
            if wd.get("auto_index") and Path(wd["path"]).is_dir()
        ]
        if not stores:
            raise RuntimeError("cluster batch has no active AtomTaskStore")
        store = stores[0] if len(stores) == 1 else MultiAtomTaskStore(stores)
        factory = self._factory()
        return process_atom_batch(
            atom_ids=atom_ids,
            config=self.config,
            skill_dir=self.skill_dir,
            store=store,
            embed_client=self.embed_client,
            agno_agent_factory=factory,
            db_path=self.db_path,
            usage_ledger=self.usage_ledger,
            logs_dir=self.logs_dir,
            spill_root=self.spill_root,
            cluster_write_queue=self.cluster_write_queue,
        )

    # ───────────────────────────────────────────────────────────
    # 收割回调
    # ───────────────────────────────────────────────────────────

    def _on_split_done(self, wd_id, fname, result, **kw):
        from xskill.pipeline.registry import update_traj_offset
        _fname, n_atoms, last_off, last_id, err = result
        if err is not None:
            if (
                isinstance(err, dict)
                and err.get("process_action") == ProcessAction.NOT_FIT.value
            ):
                mark_not_fit(
                    wd_id,
                    fname,
                    str(err.get("reason") or "not fit"),
                    str(err.get("interest_fingerprint") or self.interest_fingerprint),
                    **kw,
                )
                return
            update_traj_status(
                wd_id,
                fname,
                TrajectoryStatus.FILTERED.value,
                error_msg=str(err),
                **kw,
            )
            return
        update_traj_status(wd_id, fname, TrajectoryStatus.SPLIT_DONE.value, **kw)
        # tasks_extracted 用全量口径：TaskAgent 对追加轨迹只返回本次续拆的
        # **新增** atom 数，直接落库会把累计值覆盖成增量（审计 P0-3）。
        # 全量 = 该轨迹 atom 文件的当前总数。
        matched = [w for w in list_watch_dirs(**kw) if w["id"] == wd_id]
        if not matched:
            raise RuntimeError(
                f"watch_dir id={wd_id} vanished during split of {fname}")
        stem = fname[:-3] if fname.endswith(".md") else fname
        total_atoms = len(
            self._store_for(Path(matched[0]["path"])).list_by_traj(stem))
        update_traj_offset(
            wd_id, fname,
            last_offset=last_off, last_atom_id=last_id,
            tasks_extracted=total_atoms, **kw,
        )
        self._stats["atoms_extracted"] += n_atoms

    def _on_embed_done(self, wd_id, _filename, result, **kw):
        _wd_id, filenames = result
        for f in filenames:
            update_traj_status(wd_id, f, "indexed", **kw)
            mark_indexed(wd_id, f, **kw)
            self._stats["indexed"] += 1

    def _on_cluster_batch_done(self, results):
        """cluster batch 收割：只记日志（落地审计 + silent-drop 告警），不改
        轨迹状态。

        轨迹 done 与具体 batch 解耦——一个 batch 跨多条轨迹，done 由
        ``_finalize_completed_trajs`` 按"该轨迹 atom 是否全落地"独立判定。
        """
        n_total = len(results)
        in_skills = [r for r in results if _result_skill_assignments(r)]
        dropped = [
            r for r in results
            if (
                r.get("action") == "clustered"
                and not _result_skill_assignments(r)
            )
        ]

        _emit = logger.info if n_total > 0 else logger.debug
        _emit(
            "cluster batch → %d total, %d in skills, %d dropped",
            n_total, len(in_skills), len(dropped),
        )
        # 每个 atom→skill 关联一行 info（per-association 审计链）。
        for r in in_skills:
            for assignment in _result_skill_assignments(r):
                logger.info(
                    "  %s → %s @ ws=%s",
                    r.get("atom_id"),
                    assignment.get("skill_name"),
                    assignment.get("weightscore"),
                )
        # drop 的 atom 走 WARNING 让人 grep 得到。新 prompt 改完不应再出现，
        # 但作为 defensive 保留——cluster agent 真违反"任何分数都必须 add"
        # 这条硬约束时必须立刻被发现。
        if dropped:
            logger.warning(
                "%d atom(s) DROPPED (silent in cluster agent): %s",
                len(dropped), [r.get("atom_id") for r in dropped],
            )
        self._stats["atoms_clustered"] += len(in_skills)

```

```bash
sed -n '3207,3325p' src/xskill/pipeline/runner.py
```

```output
def process_atom_batch(*, atom_ids: list[str], config: dict, skill_dir: Path,
                       store, embed_client, agno_agent_factory,
                       db_path: Path | None = None,
                       usage_ledger=None,
                       logs_dir: Path | None = None,
                       spill_root: Path | None = None,
                       cluster_write_queue=None) -> list[dict]:
    """Run one atom batch with an isolated AgentToolContext."""
    from xskill.agents import agent_tools

    from xskill.pipeline.worker_runtime import ClusterResultRecorder
    recorder = ClusterResultRecorder()
    tool_context = agent_tools.create_agent_tool_context(
        skill_dir=skill_dir,
        data_dir=skill_dir,
        config=config,
        atom_skill_dir=skill_dir,
        atom_store=store,
        default_traj_root=store.root,
        spill_root=spill_root,
        usage_ledger=usage_ledger,
        cluster_write_queue=cluster_write_queue,
        cluster_result_recorder=recorder,
        registry_db_path=db_path,
    )
    with agent_tools.use_agent_tool_context(tool_context):
        return _process_atom_batch_bound(
            atom_ids=atom_ids,
            config=config,
            skill_dir=skill_dir,
            store=store,
            embed_client=embed_client,
            agno_agent_factory=agno_agent_factory,
            db_path=db_path,
            logs_dir=logs_dir,
            cluster_result_recorder=recorder,
        )


def _process_atom_batch_bound(*, atom_ids: list[str], config: dict,
                              skill_dir: Path, store, embed_client,
                              agno_agent_factory,
                              db_path: Path | None = None,
                              logs_dir: Path | None = None,
                              cluster_result_recorder=None) -> list[dict]:
    """批量版 ``process_atom_task``：一次 LLM 会话覆盖**多个 atom 的位置**，只跑 cluster。

    与单 atom 版语义等价，只是把"逐 atom 一次往返"压成"一批一次往返"——
    ``atom_ids`` 可能跨多条轨迹（watcher 跨轨迹池化后传入）。batch 跑完后直接读取
    当前执行上下文的写入结果，构造与单 atom 版同形的 result dict 列表（顺序同
    ``atom_ids``），不再遍历所有 candidates 文件反查落点。

    Args 同 ``process_atom_task``，只是 ``atom_id`` → ``atom_ids``（list）。

    Returns:
        list[dict]，每条含 keys: action / atom_id / skill_name / weightscore /
        skill_assignments / cluster_log。``skill_name`` 和 ``weightscore`` 保留
        最近一次写入作为兼容视图；完整结果以 ``skill_assignments`` 为准。
    """
    from xskill.agents.task_cluster_agent import TaskClusterAgent
    from xskill.agents import agent_tools

    del embed_client  # kept for API compatibility; cluster tools no longer use it

    atoms = [store.load(aid) for aid in atom_ids]
    atom_by_id = {a.atom_id: a for a in atoms}

    cluster = TaskClusterAgent(
        skill_dir=skill_dir, store=store,
        agno_agent_factory=agno_agent_factory,
        llm_cfg=config.get("llm", {}),
        logs_dir=logs_dir,
        tools=[
            agent_tools.atom_task_read, agent_tools.read_traj,
            agent_tools.skill_read, agent_tools.read_skill_tasks,
            agent_tools.new_skill_folder, agent_tools.add_tasks_to_skill,
            agent_tools.move_task_to,
            agent_tools.score_task,
        ],
    )
    cluster_error = None
    cluster_content = ""
    try:
        cluster_content = cluster.process_batch(atoms)
    except BaseException as error:  # tools may already have committed part of the batch
        cluster_error = (error, error.__traceback__)

    results: list[dict] = []
    for aid in atom_ids:
        skill_assignments = _recorded_skill_assignments(
            cluster_result_recorder,
            aid,
        )
        latest_assignment = (
            skill_assignments[-1] if skill_assignments else None
        )
        skill_name = (
            latest_assignment["skill_name"] if latest_assignment else None
        )
        weightscore = (
            latest_assignment["weightscore"] if latest_assignment else None
        )
        # 落地即打耐久消费标记（在 SkillEdit 可能清空 .candidates.yml 之前完成
        # 这次回查），让 watcher 的去重/done 判定不受后续 skill 晋升影响。
        if skill_assignments and aid in atom_by_id:
            latest_atom = store.load(aid)
            if not latest_atom.clustered:
                latest_atom.clustered = True
                store.save(latest_atom)
        # 埋点：atom 落到某 skill = 一次采纳（best-effort，失败不阻断）。
        for assignment in skill_assignments:
            try:
                from xskill.pipeline.registry import record_atom_adoption
                record_atom_adoption(
                    atom_id=aid,
                    skill=assignment["skill_name"],
                    weightscore=assignment["weightscore"],
                    was_new=True,
                    db_path=db_path,
```

```bash
sed -n '35,160p' src/xskill/skill/candidates.py
```

```output
logger = logging.getLogger("xskill.candidates")

CANDIDATES_FILENAME = ".candidates.yml"
FUZZY_PREFIX = 60

# v2 schema：以 atom_id 为单位累计 weightscore（cluster agent 给的 0-10 分）。
# buffer 中 pending（非 promoted）的 weightscore_total 累加 ≥ 这个阈值
# 就让 SkillEditAgent 触发一次 SKILL.md 整理。
# 单 atom 给 10 分立即触发——cluster agent 的提示词鼓励"非常确信时打高分"。
ATOM_PROMOTION_THRESHOLD = 10

# main→staging / jam 旧渐进链的单轮候选数。baby 冷启动不再使用这个常量：
# 它从 ``agent_worker.pools.edit.batch_size`` 读取 N（默认 5），每 N 个候选
# 直接 checkpoint 到 baby 并立即消费。
SKILL_EDIT_BATCH_SIZE = 20


# ═══════════════════════════════════════════════════════════════════
# I/O
# ═══════════════════════════════════════════════════════════════════

def _candidates_path(skill_dir: Path) -> Path:
    return Path(skill_dir) / CANDIDATES_FILENAME


def _load_candidates_unlocked(skill_dir: Path) -> dict:
    candidate_path = _candidates_path(skill_dir)
    if not candidate_path.exists():
        return {"candidates": []}
    try:
        raw_data = candidate_path.read_text(encoding="utf-8")
        data = yaml.safe_load(raw_data)
    except (OSError, UnicodeDecodeError, yaml.YAMLError):
        logger.exception("failed to load candidates file %s", candidate_path)
        raise
    if data is None:
        return {"candidates": []}
    if not isinstance(data, dict):
        raise ValueError(
            f"candidates file must contain a mapping: {candidate_path}",
        )
    if data.get("candidates") is None:
        data["candidates"] = []
    if not isinstance(data["candidates"], list):
        raise ValueError(
            f"candidates field must be a list: {candidate_path}",
        )
    for index, candidate in enumerate(data["candidates"]):
        if not isinstance(candidate, dict):
            raise ValueError(
                "candidate entry must be a mapping: "
                f"{candidate_path} index={index}",
            )
    return data


def load_candidates(skill_dir: Path) -> dict:
    """Read a complete candidates snapshot or raise on invalid data."""
    return _load_candidates_unlocked(skill_dir)


def _atomic_save_candidates_unlocked(skill_dir: Path, data: dict) -> None:
    candidate_path = _candidates_path(skill_dir)
    candidate_path.parent.mkdir(parents=True, exist_ok=True)
    serialized = yaml.safe_dump(
        data,
        allow_unicode=True,
        sort_keys=False,
    )
    temporary_path: Path | None = None
    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=candidate_path.parent,
            prefix=f".{candidate_path.name}.",
            suffix=".tmp",
            delete=False,
        ) as temporary_file:
            temporary_path = Path(temporary_file.name)
            temporary_file.write(serialized)
            temporary_file.flush()
            os.fsync(temporary_file.fileno())
        os.replace(temporary_path, candidate_path)
        temporary_path = None
        if os.name != "nt":
            directory_fd = os.open(
                candidate_path.parent,
                os.O_RDONLY | os.O_DIRECTORY,
            )
            try:
                os.fsync(directory_fd)
            finally:
                os.close(directory_fd)
    finally:
        if temporary_path is not None:
            temporary_path.unlink(missing_ok=True)
    from xskill.skill.catalog_store import notify_native_candidates_count
    notify_native_candidates_count(
        skill_dir,
        len(data.get("candidates", []) or []),
    )
    from xskill.pipeline.registry import notify_atom_pending_sync
    notify_atom_pending_sync(
        skill_dir,
        data.get("candidates", []) or [],
    )


def save_candidates(skill_dir: Path, data: dict) -> None:
    """Atomically replace one candidate snapshot under its repository lock."""
    with skill_repo_lock(skill_dir, use_git_write_limit=False):
        _atomic_save_candidates_unlocked(skill_dir, data)


# ═══════════════════════════════════════════════════════════════════
# Fuzzy matching + merge
# ═══════════════════════════════════════════════════════════════════

def _norm(s: str) -> str:
    return re.sub(r"\s+", " ", (s or "").strip().lower())


def _fuzzy_equal(a: str, b: str) -> bool:
    a_n = _norm(a)
    b_n = _norm(b)
```

```bash
sed -n '333,365p' src/xskill/skill/candidates.py
```

```output
def ready_for_promotion_v2(
    data: dict, threshold: int = ATOM_PROMOTION_THRESHOLD,
) -> list[dict]:
    """v2.1 简化：candidates 全是 pending（无 promoted 字段），sum 所有
    ``weightscore`` ≥ threshold 即 ready。

    返回 buffer 中所有 candidates iff 总分 ≥ threshold；否则空列表。
    """
    cands = data.get("candidates", []) or []
    total = sum(int(c.get("weightscore", 0)) for c in cands)
    if total >= threshold:
        return list(cands)
    return []


def clear_candidates(skill_dir: Path) -> None:
    """v2.1: SkillEditAgent 成功落盘 SKILL.md 后调用——清空 buffer。

    写入 ``{candidates: []}`` 而不是删文件，保留 yaml 形态便于下次 cluster
    直接追加。
    """
    with skill_repo_lock(skill_dir, use_git_write_limit=False):
        data = _load_candidates_unlocked(skill_dir)
        _atomic_save_candidates_unlocked(
            skill_dir,
            {**data, "candidates": []},
        )


def remove_candidates(
    skill_dir: Path,
    atom_ids: set[str],
) -> tuple[list[str], int]:
```

## 9. SkillEditAgent turns evidence into a versioned Skill

Creating a candidate target creates a standalone Git repository on branch `baby` with a stub `SKILL.md`. The watcher scans every Skill buffer independently of cluster completion, so a full buffer is not stranded when an unrelated cluster call fails.

`SkillEditAgent.maybe_run` is the gatekeeper:

1. a baby Skill must reach cumulative weight ten, unless it already has a partial checkpoint that must be drained;
2. an established `main` must reach the same threshold and must have at least one real main-side UX observation before an update is allowed;
3. an existing `staging` normally holds all new edits while the A/B test runs;
4. only a “jam” (old enough staging, no recent UX progress, and at least the higher jam weight threshold) may combine main, staging, and accumulated candidates directly.

The editor consumes a snapshot of current atom IDs, leaving candidates added concurrently for the next round. It reads the exact atom and source slices, edits the Skill body plus optional reference/script files, and validates strict frontmatter before publication.

Baby work is checkpointed in small framework-bound batches. Each successful checkpoint commits a real change and removes exactly its bound atom IDs; only after the buffer is empty does framework code rename `baby` to `main`. An existing main is edited in progressive turn branches, but the final tool commits the new version to `staging` and restores the working branch to main. Description trigger optimization runs at the publication boundary. Only after the final Git state is verified does `maybe_run` remove the original snapshot from the candidate buffer.

The outcome is deliberately simple:

`baby → main → staging → (main if promoted | rejected ref if rejected)`

The temporary turn branches and materialized `.canary/<skill>` copy are mechanics around that state machine, not additional public states.

```bash
sed -n '1271,1352p' src/xskill/agents/agent_tools.py
```

```output
def new_skill_folder(skill_name: str, description: str) -> str:
    """v2: 创建 skill 目录 → git init → checkout baby 分支 → 首次 commit
    （含 stub SKILL.md + .gitignore）。

    description 必填，落到 stub SKILL.md 的 frontmatter 中。后续：
    - 路由表（``build_skill_catalog_block``）从 SKILL.md frontmatter 取 desc
      展示
    - state 由 git 分支决定（baby/main/staging），不再单写 .meta.yml
    - 后续 SkillEditAgent 触发时拿到 candidates 来填正文，调
      ``commit_baby_to_main`` graduate 到 main
    """
    skill_dir = agent_tool_config.atom_skill_dir
    if skill_dir is None:
        return "error: atom task tool context not initialized"
    desc = (description or "").strip()
    if not desc:
        return ("error: description 必填——简述这个 skill 服务于什么类型的 atom "
                "（2-3 句中文，让后续 cluster agent 能判断同类）")
    slug = _slugify(skill_name)
    target = skill_dir / slug

    def mutate():
        if target.exists():
            return f"already exists: {target}"
        from xskill.skill.git import init_skill_repo_on_baby
        init_skill_repo_on_baby(str(target), name=slug, description=desc)
        return f"created on baby branch: {target}  desc={desc[:60]!r}"

    return _run_cluster_mutation(mutate)


@tool(name="skill_read")
def skill_read(skill_name: str) -> str:
    """读 skill 的 SKILL.md 正文 + 目录内其余文件树（路径可直接喂 read_file）。"""
    skill_dir = agent_tool_config.atom_skill_dir
    if skill_dir is None:
        return "error: atom task tool context not initialized"
    slug = _slugify(skill_name)
    skill_path = (skill_dir / slug).resolve()
    header = f"skill_dir: {skill_path}"
    try:
        from xskill.skill.git import current_branch
        header += f"   (branch: {current_branch(str(skill_path))})"
    except Exception:
        logger.debug("failed to read branch for %s", skill_path, exc_info=True)
    markdown_path = skill_path / "SKILL.md"
    if markdown_path.is_file():
        markdown_text = markdown_path.read_text(encoding="utf-8")
        _mark_file_read(markdown_path)
        body = (f"--- SKILL.md ({len(markdown_text.splitlines())} lines) ---\n"
                f"{markdown_text}")
    else:
        body = f"(skill {slug} has no SKILL.md yet — only candidates buffer)"
    tree_lines: list[str] = []
    if skill_path.is_dir():
        for current_dir, dir_names, file_names in os.walk(skill_path):
            walk_depth = len(Path(current_dir).relative_to(skill_path).parts)
            # 剪枝进不去 .git / 第 4 层以下，避免白遍历上千 git 对象。
            dir_names[:] = sorted(
                name for name in dir_names if name != ".git" and walk_depth < 3
            )
            for file_name in sorted(file_names):
                file_path = Path(current_dir) / file_name
                relative_path = file_path.relative_to(skill_path)
                if relative_path.as_posix() == "SKILL.md":
                    continue
                size_kb = file_path.stat().st_size / 1024
                annotation = ("  (用 list_candidates 读)"
                              if relative_path.name == "candidates.yml" else "")
                tree_lines.append(
                    f"{relative_path.as_posix()}  ({size_kb:.1f} KB){annotation}",
                )
        if len(tree_lines) > 100:
            tree_lines = tree_lines[:100] + ["(+more, use list_files)"]
    if not tree_lines:
        return f"{header}\n{body}"
    files_block = "\n".join(tree_lines)
    return (f"{header}\n{body}\n"
            f"--- other files（相对 {skill_path}；用 read_file(绝对路径) 精读）---\n"
            f"{files_block}")


```

```bash
sed -n '425,610p' src/xskill/agents/skill_edit_agent.py
```

```output
    def maybe_run(self) -> bool:
        """检查所有守门条件 → 触发 agent → 验证落盘 → 清 buffer。

        守门顺序（任一失败即 return False）：
          1. 该 skill 有 staging 分支 → 灰度中：
             - age≥min_jam_age ∧ plateau≥jam_plateau ∧ ws≥jam_threshold → jam
             - 否则维持 hold（并打门控日志）
          2. 无 staging 时：candidates 累计 weightscore < threshold → 没攒够
          3. 触发场景是 "create staging"（main 已存在）：
             - .ux_scores.jsonl 必须至少有 1 条 side=main → 证明 main 真有人用
             - 否则保留 candidates 等用户用过 main 再触发

        baby 全过 → 按 N 逐批 checkpoint + 消费，清空后框架晋升 main。
        main/jam 全过 → 保留旧渐进分支链，终态成功后摘除入口快照 atom_id。
        """
        from xskill.skill.git import current_branch, run_git
        from xskill.canary import CanaryConfig, evaluate_jam_gates

        self._recover_crashed_turns()

        # 守门 1：staging 存在时——三条件合取才 jam；否则 hold。
        staging_exists = run_git(
            ["rev-parse", "--verify", "staging"], cwd=str(self.skill_dir),
        )[0] == 0
        data = C.load_candidates(self.skill_dir)
        total_ws = sum(
            int(c.get("weightscore", 0))
            for c in (data.get("candidates", []) or [])
        )
        jam = False
        if staging_exists:
            jam_cfg = CanaryConfig(
                jam_threshold=self.jam_threshold,
                min_jam_age_sec=self.min_jam_age_sec,
                jam_plateau_sec=self.jam_plateau_sec,
            )
            gates = evaluate_jam_gates(
                self.skill_dir, total_ws=total_ws, config=jam_cfg,
            )
            logger.info(
                "jam_gate %s: ok=%s age=%.1f plateau_s=%.1f "
                "main_n=%s/%s staging_n=%s/%s ws=%s/%s "
                "main_sha=%s staging_sha=%s reason=%s",
                self.skill_dir.name, gates.get("ok"), gates.get("age"),
                gates.get("plateau_s"),
                gates.get("main_n"), gates.get("need"),
                gates.get("staging_n"), gates.get("need"),
                gates.get("ws"), gates.get("jam_threshold"),
                gates.get("main_sha"), gates.get("staging_sha"),
                gates.get("reason"),
            )
            jam = bool(gates.get("ok"))
            if not jam:
                return False

        if not staging_exists:
            cur = current_branch(str(self.skill_dir))
            if cur == "baby":
                partial_baby = self._baby_has_checkpoint()
                # 首次启动仍要求累计达到 promotion threshold；一旦已有 baby
                # checkpoint，就必须无视剩余权重继续 drain，避免最后几个低分
                # atom 永远无法完成毕业。candidate 已空则补完 crash window 中
                # “remove 成功、rename 前进程退出”的最终晋升。
                if not partial_baby:
                    if not C.ready_for_promotion_v2(
                        data, threshold=self.threshold,
                    ):
                        return False
                return self._run_baby_until_empty()
            if cur == "main":
                # 守门 2: main 的新一轮更新仍要求累计达到阈值。
                ready = C.ready_for_promotion_v2(data, threshold=self.threshold)
                if not ready:
                    return False
                # 守门 3: create staging 前要求 main 真有人用过。
                if not self._main_has_ux_score():
                    logger.info(
                        "skip SkillEdit: %s main 还没真实 ux_score，"
                        "保留 candidates 等用户用 main 后再产 staging",
                        self.skill_dir.name,
                    )
                    return False
            else:
                logger.warning(
                    "skip SkillEdit: %s 在异常分支 %r (期望 baby 或 main)",
                    self.skill_dir.name, cur,
                )
                return False
        else:
            ready = list(data.get("candidates", []) or [])
            cur = "main"

        skill_md = self.skill_dir / "SKILL.md"
        mtime_before = skill_md.stat().st_mtime if skill_md.is_file() else 0.0
        size_before = skill_md.stat().st_size if skill_md.is_file() else 0
        main_sha_before = ""
        if jam:
            code, out, _ = run_git(["rev-parse", "main"], cwd=str(self.skill_dir))
            if code == 0:
                main_sha_before = out.strip()

        # 快照：本次消化范围 = 当前 buffer 里的这些 atom_id。多轮渐进式消化
        # 期间 cluster 并发 add 进来的新候选不在这个集合里，不受影响，留给
        # 下一次 maybe_run 处理（避免竞争/饿死）。
        snapshot_atom_ids = {
            c.get("atom_id") for c in ready if c.get("atom_id")
        }

        try:
            run_ok = self._run(ready, current_branch_name=cur, jam=jam)
        except Exception:
            logger.exception("SkillEditAgent _run failed: %s", self.skill_dir.name)
            run_ok = False

        # 实测落盘：mtime 推进 + 非空 = agent 真写了
        wrote = (
            skill_md.is_file()
            and skill_md.stat().st_size > 0
            and (
                skill_md.stat().st_mtime > mtime_before
                or skill_md.stat().st_size != size_before
            )
        )
        if not wrote:
            logger.warning(
                "SkillEditAgent ran but SKILL.md not written/empty: %s — "
                "保留 candidates 等下轮重试",
                self.skill_dir.name,
            )
            return False

        # 发布门兜底：write_file 已挡住非法 frontmatter，但 agent 可能绕开
        # write_file（或别的路径）落了坏 SKILL.md。commit 前再跑一次 parse_strict，
        # 非法 → 不清 buffer、标重试，绝不把坏 skill 静默发布出去。
        from xskill.skill.frontmatter import (
            parse_strict as fm_parse_strict,
            FrontmatterError,
        )
        try:
            fm_parse_strict(skill_md.read_text(encoding="utf-8"))
        except FrontmatterError as e:
            logger.warning(
                "SkillEditAgent 落了非法 SKILL.md: %s — %s；保留 candidates 重试",
                self.skill_dir.name, e,
            )
            return False

        if not run_ok:
            logger.warning(
                "SkillEditAgent _run 未完整跑完终态提交（多轮渐进式消化中途失败/"
                "崩溃）: %s — 保留 candidates，下轮 maybe_run 会清残留 turn 分支"
                "重新开始",
                self.skill_dir.name,
            )
            return False

        if jam:
            from xskill.canary import discard_staging

            code, out, _ = run_git(["rev-parse", "main"], cwd=str(self.skill_dir))
            main_sha_after = out.strip() if code == 0 else ""
            if not main_sha_after or main_sha_after == main_sha_before:
                logger.warning(
                    "SkillEditAgent jam-merge did not advance main: %s — "
                    "保留 candidates/staging 等下轮重试",
                    self.skill_dir.name,
                )
                return False
            if not discard_staging(self.skill_dir):
                logger.warning(
                    "SkillEditAgent jam-merge could not discard staging: %s — "
                    "保留 candidates 等下轮重试",
                    self.skill_dir.name,
                )
                return False
            self._cleanup_turn_branches("main")

        # commit 工具的成功效应（baby→main 或 main→staging）通过当前分支变化
        # 自然反映，不需要在这里做额外检查（非 jam 路径的 turn 分支已在
        # ``_run`` 尾部的终态校验通过后就地清理）
        C.remove_candidates(self.skill_dir, snapshot_atom_ids)
        logger.info("SkillEditAgent done + %d candidate(s) removed: %s",
                    len(snapshot_atom_ids), self.skill_dir.name)
        return True

    def _baby_has_checkpoint(self) -> bool:
```

```bash
sed -n '1050,1220p' src/xskill/agents/skill_edit_agent.py
```

```output
    def _run(self, ready: list[dict], current_branch_name: str, jam: bool = False) -> bool:
        """跑完整条（可能多轮的）消化链。

        返回 True = 终态 commit 完整跑完（baby→main / main 直接更新走
        staging / jam 强砍合并）且 turn 分支已清理；False/异常 = 某一环节
        失败——调用方保留 candidates，下轮 ``maybe_run`` 触发崩溃恢复重来。
        """
        from xskill.skill import git as skillgit
        from xskill.skill.git import current_branch as _current_branch
        from xskill.skill.git import run_git as _run_git

        batches = self._make_batches(ready)
        num_batches = len(batches)
        if num_batches == 0:
            return False

        for turn_idx, batch in enumerate(batches, start=1):
            is_last = turn_idx == num_batches
            if is_last and num_batches > 1:
                # 最后一轮开跑前把 HEAD 切回原分支（工作区不动，仍是最后一个
                # turn 分支演化出的内容）——终态 commit 工具的分支校验才能过。
                skillgit.checkout_head_ref_only(
                    str(self.skill_dir), current_branch_name,
                )
            if jam:
                self._run_jam_round(
                    batch, turn_idx=turn_idx, num_batches=num_batches, is_last=is_last,
                )
            else:
                self._run_normal_round(
                    batch, current_branch_name=current_branch_name,
                    turn_idx=turn_idx, num_batches=num_batches, is_last=is_last,
                )
            if not is_last:
                turn_branch = f"{current_branch_name}_turn{turn_idx}"
                skillgit.commit_progressive_turn(
                    str(self.skill_dir), turn_branch,
                    f"skilledit progressive turn {turn_idx}/{num_batches} "
                    f"({len(batch)} atoms, not final)",
                )

        if jam:
            # jam 的终态校验（main sha 是否真推进）+ discard_staging + turn 分支
            # 清理都在 maybe_run 里做（它已经拿着 main_sha_before 了）。这里只
            # 负责把多轮循环跑完，不重复判定。
            return True

        final_branch = _current_branch(str(self.skill_dir))
        if current_branch_name == "baby":
            ok = final_branch == "main"
        else:
            staging_ok = _run_git(
                ["rev-parse", "--verify", "staging"], cwd=str(self.skill_dir),
            )[0] == 0
            ok = staging_ok and final_branch == "main"
        if ok:
            self._cleanup_turn_branches(current_branch_name)
        return ok

    def _run_jam_round(
        self, batch: list[dict], *, turn_idx: int, num_batches: int, is_last: bool,
    ) -> None:
        from xskill.agents import agent_tools

        skill_md = self.skill_dir / "SKILL.md"
        staging_body = (
            self.skill_dir.parent / ".canary" / self.skill_dir.name / "SKILL.md"
        )
        if not staging_body.is_file():
            from xskill.canary import materialize_staging
            materialize_staging(self.skill_dir, self.skill_dir.parent / ".canary")
        if not staging_body.is_file():
            raise RuntimeError(
                f"jam-merge: staging body for {self.skill_dir.name} could not be "
                "materialized; refusing to merge and discard"
            )

        lines = [
            MERGE_DISCIPLINE_BLOCK,
            *self._round_info_lines(turn_idx, num_batches),
            "",
            f"skill_name: {self.skill_dir.name}（**main 分支 · 轨迹堰塞强砍合并**）",
            f"现有 main 正文：用 skill_read('{self.skill_dir.name}') 读。",
            f"staging 正文路径（用 read_file 读）：{staging_body}",
            *self._skill_tree_context_lines(),
            "# 待合并候选（按 weightscore 倒序）",
        ]
        for c in sorted(batch, key=_candidate_weight, reverse=True):
            note = c.get("note", "")
            lines.append(
                f"- atom_id={c['atom_id']}  weightscore={c['weightscore']}"
                + (f"  note: {note}" if note else "")
            )
        lines += [
            "",
            f"目标 skill 目录: {self.skill_dir}",
            f"目标 SKILL.md 路径: {skill_md}",
        ]
        scenario_block = "\n".join(lines)
        sysprompt = build_system_prompt(scenario_block=scenario_block, branch_now="main")

        tools = [
            agent_tools.atom_task_read,
            agent_tools.read_traj,
            agent_tools.skill_read,
            agent_tools.read_file,
            agent_tools.list_files,
            agent_tools.grep_files,
            agent_tools.write_file,
        ]
        if is_last:
            tools.append(agent_tools.commit_update_main)

        agent = self.agno_agent_factory(instructions=[sysprompt], tools=tools)
        self._trace_run(agent, scenario_block)

    def _run_normal_round(
        self, batch: list[dict], *, current_branch_name: str,
        turn_idx: int, num_batches: int, is_last: bool,
    ) -> None:
        from xskill.agents import agent_tools
        from xskill.skill.frontmatter import parse as fm_parse

        skill_md = self.skill_dir / "SKILL.md"
        scenario_lines: list[str] = []
        if current_branch_name == "baby":
            scenario_lines.append(
                "skill_name: " + self.skill_dir.name + "（**baby 分支**——首次出版本）"
            )
            scenario_lines.extend(self._round_info_lines(turn_idx, num_batches))
            scenario_lines.append(
                "本轮只处理下面这批候选。写完后必须调 "
                "``commit_baby(skill_name, message)``：它会 checkpoint 当前改动"
                "并消费系统绑定的 atom_id，但保持在 baby；buffer 清空后由框架"
                "自动 graduate 到 main。"
            )
        else:
            scenario_lines.append(
                "skill_name: " + self.skill_dir.name + "（**main 分支** —— 更新现有 skill）"
            )
            scenario_lines.extend(self._round_info_lines(turn_idx, num_batches))
            if is_last:
                scenario_lines.append(
                    "写完 SKILL.md 后调 ``commit_to_staging(skill_name, message)`` "
                    "把更新作为灰度候选 commit 到 staging。"
                )
            else:
                scenario_lines.append(
                    "本轮没有提供任何 commit 工具——分支推进由系统在全部批次渐进式"
                    "消化完成后自动处理，你只需要把本轮候选编辑进 SKILL.md。"
                )

        # 现有 SKILL.md 是 stub (baby 时) 或正式版 (main 时)；多轮消化中间态时
        # 是前几轮已经融合过候选的正文。
        if skill_md.is_file():
            try:
                fm, _ = fm_parse(skill_md.read_text(encoding="utf-8"))
                cur_desc = (fm.get("description") or "").strip().replace("\n", " ")
                cur_ver = (fm.get("metadata", {}) or {}).get("version", "?")
                scenario_lines.append("")
                scenario_lines.append(f"现有 SKILL.md description: {cur_desc[:200]}")
                scenario_lines.append(f"现有 SKILL.md version: {cur_ver}")
            except Exception:
                logger.warning("failed to read existing skill metadata: %s",
                               skill_md, exc_info=True)
        scenario_lines.extend(self._skill_tree_context_lines())
        scenario_lines.append("")
        scenario_lines.append("# 待整理 candidates（按 weightscore 倒序）")
        for c in sorted(batch, key=_candidate_weight, reverse=True):
            note = c.get("note", "")
            ext = f"  note: {note}" if note else ""
```

```bash
sed -n '1886,2055p' src/xskill/agents/agent_tools.py
```

```output
def graduate_baby_to_main(target: Path, slug: str, message: str) -> bool:
    """Framework-controlled final baby graduation.

    Description optimization runs once here, after every candidate checkpoint
    has been consumed.  The legacy ``commit_baby_to_main`` tool delegates to
    this function and remains available for compatibility, but checkpointed
    baby turns only expose ``commit_baby`` to the model.
    """
    from xskill.skill.frontmatter import FrontmatterError
    from xskill.skill.git import (
        commit_baby_to_main_branch,
        skill_md_still_baby_stub,
    )

    skill_md = target / "SKILL.md"
    try:
        fm_parse_strict(skill_md.read_text(encoding="utf-8"))
    except (OSError, UnicodeDecodeError, FrontmatterError) as error:
        logger.warning("baby graduation frontmatter invalid for %s: %s", slug, error)
        return False
    if skill_md_still_baby_stub(target):
        logger.warning(
            "baby graduation refused: %s SKILL.md still has init stub placeholder",
            slug,
        )
        return False
    _run_description_optimization(target, slug)
    return commit_baby_to_main_branch(str(target), message)


@tool(name="commit_baby")
def commit_baby(skill_name: str, message: str) -> str:
    """Commit the current framework-bound atom batch on ``baby``.

    Candidate IDs are deliberately not arguments: the framework binds the
    exact ordered batch in ``AgentToolContext`` before this turn.  A successful
    call creates one non-empty commit, then removes only those bound IDs while
    holding the same repository lock.  It never renames the branch and never
    runs description optimization.
    """
    from xskill.agents import agent_trace
    from xskill.skill import candidates as candidate_buffer
    from xskill.skill.frontmatter import FrontmatterError
    from xskill.skill.git import (
        commit_baby_checkpoint,
        current_branch,
        skill_repo_lock,
    )

    context = current_agent_tool_context()
    skill_root = context.atom_skill_dir
    if skill_root is None:
        return "error: atom task tool context not initialized"
    slug = _slugify(skill_name)
    if not context.skill_edit_skill_name or not context.skill_edit_batch_ids:
        return "error: no framework-bound SkillEdit batch"
    if slug != context.skill_edit_skill_name:
        return (
            "error: commit_baby can only commit the framework-bound skill "
            f"{context.skill_edit_skill_name}"
        )
    target = skill_root / slug
    if not target.is_dir() or not (target / ".git").is_dir():
        return f"error: skill {slug} not found or has no git repository"
    msg = (message or "").strip()
    if not msg:
        return "error: commit message 必填"

    skill_md = target / "SKILL.md"
    try:
        fm_parse_strict(skill_md.read_text(encoding="utf-8"))
    except (OSError, UnicodeDecodeError, FrontmatterError) as error:
        return f"error: invalid SKILL.md frontmatter: {error}"

    batch_ids = tuple(context.skill_edit_batch_ids)
    with skill_repo_lock(target):
        if current_branch(str(target)) != "baby":
            return "error: commit_baby only works on the baby branch"
        commit_sha = commit_baby_checkpoint(str(target), msg)
        if commit_sha is None:
            return "error: commit_baby requires a real non-empty change"
        consumed, remaining = candidate_buffer.remove_candidates(
            target,
            set(batch_ids),
        )

    lines = [
        f"Created baby checkpoint {commit_sha[:7]}.",
        "Consumed atoms:",
        *[f"  {atom_id}" for atom_id in consumed],
        f"{remaining} atoms remain.",
    ]
    agent_trace.event("INFO", "\n".join(lines), include_timestamp=False)
    return "\n".join(lines)


@tool(name="commit_baby_to_main")
def commit_baby_to_main(skill_name: str, message: str) -> str:
    """SkillEditAgent 首次为某 skill 出版本时调用。

    前提：该 skill 当前在 baby 分支（cluster 创建后未 graduate）。
    行为：git add . + commit + git branch -m baby main → 该 skill 第一次
    有 main 版本。**之后** SkillEditAgent 再触发只能调 commit_to_staging。

    Args:
        skill_name: 目标 skill 的 slug（如 ``django-fix``）
        message: commit message，应该写明本次基于哪些 atom_id

    Returns:
        成功："graduated baby → main: <skill_name>"
        失败："error: ..."
    """
    skill_dir = agent_tool_config.atom_skill_dir
    if skill_dir is None:
        return "error: atom task tool context not initialized"
    slug = _slugify(skill_name)
    target = skill_dir / slug
    if not target.is_dir():
        return f"error: skill {slug} not found"
    if not (target / ".git").is_dir():
        return f"error: skill {slug} 没 git 仓库（new_skill_folder 出问题？）"
    msg = (message or "").strip()
    if not msg:
        return "error: commit message 必填"
    from xskill.skill.git import skill_md_still_baby_stub

    if skill_md_still_baby_stub(target):
        return (
            "error: SKILL.md 仍是 baby stub（含 init placeholder），"
            "禁止 graduate；请先 write_file 写完整正文再调用"
        )
    ok = graduate_baby_to_main(target, slug, msg)
    if not ok:
        return "error: commit_baby_to_main 失败（不在 baby 分支？看 daemon 日志）"
    return f"graduated baby → main: {slug}"


@tool(name="commit_to_staging")
def commit_to_staging(skill_name: str, message: str) -> str:
    """SkillEditAgent 在 skill 已有 main 时调用——产出灰度候选。

    前提：该 skill 当前在 main 且不存在 staging。
    行为：从 main 切 staging 分支 + add . + commit + 物化到
    ``<skill_dir>/../.canary/<name>/`` 让 install_to_claude_code(side='staging')
    能装到。

    Args:
        skill_name: 目标 skill 的 slug
        message: commit message

    Returns:
        成功："committed to staging: <skill_name>"
        失败："error: ..."
    """
    from xskill.skill.git import commit_to_staging_branch
    skill_dir = agent_tool_config.atom_skill_dir
    if skill_dir is None:
        return "error: atom task tool context not initialized"
    slug = _slugify(skill_name)
    target = skill_dir / slug
    if not target.is_dir():
        return f"error: skill {slug} not found"
    if not (target / ".git").is_dir():
        return f"error: skill {slug} 没 git 仓库"
    msg = (message or "").strip()
    if not msg:
        return "error: commit message 必填"
    # commit 前先跑 description 触发优化（best desc 写回 frontmatter）。失败只
    # log，不阻断 commit。
    _run_description_optimization(target, slug)
```

## 10. Real use closes the canary loop

A staging branch is useful only if real agent sessions exercise it. The canary layer therefore has both routing and evidence collection.

Routing is deterministic: a hash of trajectory/client plus Skill gives the same side for the same scope. Only selected top source-model cohorts may enter staging in model-scoped mode. Deficit filling overrides the probabilistic fallback until both current branch SHAs have enough samples—staging first, then main—so small deployments cannot accidentally starve one side.

The normalized atom already carries the interaction's `ux_score`; after every atom in a trajectory has been clustered, the watcher appends that observation to the used Skill's `.ux_scores.jsonl`. Standalone trajectories use the xskill header that recorded installed Skill, side, and SHA. Team trajectories may report several `used_skills`, so the server resolves and scores each one against the uploading client and current model cohort. Appends are idempotent on atom/Skill/side and are mirrored to SQLite; a periodic projection worker repairs any missed mirror.

A decision considers only scores for the current main and staging SHAs. Unscoped mode requires `min_samples` on both sides and compares simple means. Scoped mode requires `total_samples`, computes a mean within each eligible model cohort, then weights cohort means by observed model share. If staging is at least as good as main, it is merged and becomes main. If worse, it is discarded; a rejected ref preserves the commit for lineage and diffs. A staging branch that cannot gather enough evidence by `max_days_hold` is also discarded.

Canary planning is read-only; applying the terminal decision performs Git/materialization changes and records telemetry. The watcher additionally carries recovery markers around terminal transitions so a crash between “Git changed” and “install/history recorded” can converge on the next scan.

```bash
sed -n '489,575p' src/xskill/canary.py
```

```output
def pick_side(traj_id: str, skill_name: str, probability: float) -> str:
    """同一条轨迹对同一个 skill 始终返回同一个 side。

    伪随机源：sha256(traj_id : skill_name)。返回 'main' 或 'staging'。
    probability=0.2 表示 20% 概率给 staging。
    """
    if probability <= 0:
        return "main"
    if probability >= 1:
        return "staging"
    h = hashlib.sha256(f"{traj_id}:{skill_name}".encode("utf-8")).digest()
    r = int.from_bytes(h[:4], "big") / (1 << 32)
    return "staging" if r < probability else "main"


def pick_side_scoped(traj_id: str, skill_name: str, probability: float,
                     *, user_model: str, eligible: dict[str, float] | None) -> str:
    """模型分桶路由(batch3):只有 top-N 用户模型的流量才可能进 staging。

    - ``eligible`` 为 None → 未启用模型分桶,退回 :func:`pick_side`(老行为)。
    - ``eligible`` 给定(``{model: weight}``)→ ``user_model`` 不在其中(含
      unknown / 非 top-N)一律返回 ``main``,**不进灰度**;在其中则照常按
      ``pick_side`` 确定性分流(各模型的灰度量天然 ∝ 其流量,即"等比推送")。
    """
    if eligible is None:
        return pick_side(traj_id, skill_name, probability)
    if user_model not in eligible:
        return "main"
    return pick_side(traj_id, skill_name, probability)


def fill_deficit_side(
    *,
    staging_n: int,
    main_n: int,
    need: int,
    fallback: str,
) -> str:
    """体验分补漏：staging 不够先喂 staging，够了 main 还不够就喂 main。

    两侧都达到 ``need`` 后返回 ``fallback``（通常是 CanaryRouter / pick_side）。
    只改路由，不虚补分数。
    """
    if need <= 0:
        return fallback if fallback in ("main", "staging") else "main"
    if staging_n < need:
        return "staging"
    if main_n < need:
        return "main"
    return fallback if fallback in ("main", "staging") else "main"


def auto_canary_side(
    skill_dir: Path,
    *,
    main_sha: str,
    staging_sha: str,
    need: int,
    fallback: str,
) -> str:
    """按当前这对 sha 的体验分条数做补漏换侧。"""
    s_sha = staging_sha or ""
    m_sha = main_sha or ""
    staging_n = len(recent_scores(
        skill_dir, side="staging", commit_sha=s_sha, n=max(need, 1),
    )) if s_sha else 0
    main_n = len(recent_scores(
        skill_dir, side="main", commit_sha=m_sha, n=max(need, 1),
    )) if m_sha else 0
    return fill_deficit_side(
        staging_n=staging_n, main_n=main_n, need=need, fallback=fallback,
    )


# ═══════════════════════════════════════════════════════════════════
# 有状态分流：CanaryRouter —— 在线偏差最小化 + pick_side hash 随机
# ═══════════════════════════════════════════════════════════════════
# pick_side 是无状态哈希：client 很少时（team-CS 常少量 worker）可能全落 main，
# staging 饿死。CanaryRouter 按 skill 记账，新 client 选「加入后 staging 比例
# 最接近 probability」的一侧。
#
# 随机手段沿用 pick_side（sha256），不用 random.random()：
#   - 首个 client（种子）→ pick_side(client_id, skill, p)
#   - 误差打平（破平）→ pick_side(client_id, skill, 0.5)
# 高基数路径（traj_id）仍直接用 pick_side。


```

```bash
sed -n '833,1026p' src/xskill/canary.py
```

```output
def recent_scores(
    skill_dir: Path,
    *,
    side: str,
    commit_sha: str,
    n: int,
) -> list[dict]:
    """取某 side+sha 最近 n 条 UX 分（优先 registry.db，空则回退 jsonl）。

    镜像写入失败且 sync 尚未追上时，DB 可能暂时缺最新分——回退盘文件；
    若 DB 已有旧行但缺最新，会偏保守（样本偏少），有意为之。
    """
    skill_name = Path(skill_dir).name
    all_: list[dict] = []
    try:
        from xskill.pipeline.ux_scores_store import load_ux_scores_for_skill
        all_ = load_ux_scores_for_skill(skill_name, side=side, days=0)
    except Exception:
        logger.debug("ux_scores db read failed; fallback jsonl", exc_info=True)
    if not all_:
        all_ = load_ux_scores(skill_dir)
    filtered = [
        s for s in all_
        if s.get("side") == side and s.get("commit_sha") == commit_sha
    ]
    filtered.sort(key=lambda s: s.get("scored_at", ""), reverse=True)
    return filtered[:n]


def aggregate_ux_by_version(rows: list[dict]) -> list[dict]:
    """按 ``commit_sha`` 分组聚合 ux 分记录（共享给 ``Skill`` / ``SkillHub``）。

    每组返回 ``{"commit_sha", "side", "count", "avg", "first_scored_at",
    "last_scored_at"}``。``side`` 字段：组内记录同侧 → 该 side 字符串；
    多侧混在一起 → ``"mixed"``（即调用方传 ``side=None`` 两侧合并的口径）。
    按 ``last_scored_at`` 降序（最新版本在前）。无评分记录的 sha 不出组。
    """
    by_sha: dict[str, list[dict]] = {}
    for r in rows:
        sha = r.get("commit_sha") or ""
        by_sha.setdefault(sha, []).append(r)
    out: list[dict] = []
    for sha, items in by_sha.items():
        scores = [r.get("score") for r in items
                  if isinstance(r.get("score"), (int, float))]
        if not scores:
            continue
        sides = {r.get("side") for r in items}
        side_label = next(iter(sides)) if len(sides) == 1 else "mixed"
        timestamps = [r.get("scored_at", "") for r in items]
        out.append({
            "commit_sha": sha,
            "side": side_label,
            "count": len(scores),
            "avg": round(sum(scores) / len(scores), 4),
            "first_scored_at": min(timestamps),
            "last_scored_at": max(timestamps),
        })
    out.sort(key=lambda d: d["last_scored_at"], reverse=True)
    return out


# ═══════════════════════════════════════════════════════════════════
# Controller：事件触发判定
# ═══════════════════════════════════════════════════════════════════

def eligible_models(model_share: list[dict], top_n: int) -> dict[str, float]:
    """从 registry.model_share() 结果选出"使用量 top-N 的用户模型"并归一成权重。

    - 排除 ``unknown`` / 空 / ``<synthetic>``（来源不可信，不参与灰度——见设计）。
    - 按 ``trajs`` 降序取前 ``top_n``，权重 = 各自 trajs / Σ(top-N trajs)。
    - 返回 ``{model: weight}``，Σweight=1.0；无合格模型时返回 ``{}``。

    ``model_share`` 形如 ``[{"model": "claude-opus-4-7", "trajs": 102, ...}, ...]``。
    """
    excluded = {"", "unknown", "<synthetic>"}
    rows = [r for r in model_share
            if str(r.get("model", "")).strip() not in excluded
            and int(r.get("trajs", 0)) > 0]
    rows.sort(key=lambda r: int(r.get("trajs", 0)), reverse=True)
    top = rows[:max(0, top_n)]
    total = sum(int(r["trajs"]) for r in top)
    if total <= 0:
        return {}
    return {str(r["model"]): int(r["trajs"]) / total for r in top}


def _cohort_weighted(scores: list[dict], weights: dict[str, float]
                     ) -> tuple[float | None, dict[str, int]]:
    """按 user_model 分桶求各桶均分，再按 ``weights`` 加权汇总成"真正体验分"。

    只统计 model ∈ weights 的样本（unknown / 非 top-N 被丢弃）。权重在"实际有
    样本的桶"上重新归一。返回 (加权分 or None, 各桶样本数)；无任一合格桶→None。
    """
    by_model: dict[str, list[float]] = {}
    for s in scores:
        m = str(s.get("user_model", ""))
        if m in weights:
            by_model.setdefault(m, []).append(float(s["score"]))
    if not by_model:
        return None, {}
    wsum = sum(weights[m] for m in by_model)
    weighted = sum((weights[m] / wsum) * (sum(v) / len(v))
                   for m, v in by_model.items())
    return weighted, {m: len(v) for m, v in by_model.items()}


def plan_decision(skill_dir: Path, config: CanaryConfig | None = None,
                  *, weights: dict[str, float] | None = None) -> dict:
    """只读计算灰度决策，不修改 Git、物化目录或裁决记录。

    返回结果字典的 ``action`` 字段含义：

    - no_staging     :  该 skill 无 staging 分支，什么都不做
    - waiting        :  样本不足，继续收集
    - timeout_discarded : 超过 max_days 仍不足 → 丢弃 staging
    - promoted       :  加权 staging 分 ≥ 加权 main → 合入 main
    - rejected       :  加权 staging 分 < 加权 main → 丢弃 staging

    ``weights``: ``{user_model: 权重}``（来自 :func:`eligible_models`）。
    - 给定时走**模型分桶加权**:只统计 top-N 模型样本(unknown 等被排除)，每侧
      需 ≥ ``total_samples`` 个合格样本；加权体验分 = Σ 桶均分 × 桶人口权重。
    - 为 None 时退化为**单桶**(全部样本一个桶、权重 1)，阈值用 ``min_samples``——
      等价于旧的简单均分(单机/未开模型分桶场景)。两者同一套分桶算法，非两条路径。
    """
    cfg = config or CanaryConfig()
    skill_dir = Path(skill_dir)

    # P2-2.4c:已下线 skill 不再参与灰度判定——不晋升、不翻牌,staging 原样
    # 冻结(数据保留;恢复在役后按剩余样本继续)。
    from xskill.pipeline.registry import retired_skills
    if skill_dir.name in retired_skills():
        return {"action": "retired"}

    if not has_staging(skill_dir):
        return {"action": "no_staging"}

    m_sha = main_sha(skill_dir)
    s_sha = staging_sha(skill_dir)
    if not m_sha or not s_sha:
        return {"action": "no_staging"}

    created = staging_created_at(skill_dir)
    age_days = None
    if created is not None:
        age_days = (datetime.now(timezone.utc) - created.astimezone(timezone.utc)).days

    scoped = weights is not None
    need = cfg.total_samples if scoped else cfg.min_samples
    # 单桶用通配权重 {"*": 1.0}，并把样本的 user_model 临时视作 "*"。
    eff_weights = weights if scoped else {"*": 1.0}

    n_collect = max(need * (len(eff_weights) or 1), need)
    main_all = recent_scores(skill_dir, side="main", commit_sha=m_sha, n=n_collect)
    staging_all = recent_scores(skill_dir, side="staging", commit_sha=s_sha, n=n_collect)
    if not scoped:
        for s in main_all + staging_all:
            s["user_model"] = "*"

    main_n = sum(1 for s in main_all if s.get("user_model") in eff_weights)
    staging_n = sum(1 for s in staging_all if s.get("user_model") in eff_weights)
    enough = main_n >= need and staging_n >= need

    if not enough:
        if age_days is not None and age_days >= cfg.max_days_hold:
            return {"action": "timeout_discarded", "age_days": age_days,
                    "main_samples": main_n, "staging_samples": staging_n,
                    "main_sha": m_sha, "staging_sha": s_sha}
        return {"action": "waiting", "age_days": age_days,
                "main_samples": main_n, "staging_samples": staging_n, "need": need}

    main_w, main_cohorts = _cohort_weighted(main_all, eff_weights)
    staging_w, staging_cohorts = _cohort_weighted(staging_all, eff_weights)
    if main_w is None or staging_w is None:
        return {"action": "waiting", "age_days": age_days,
                "main_samples": main_n, "staging_samples": staging_n, "need": need}

    summary = {
        "main_avg": round(main_w, 3),
        "staging_avg": round(staging_w, 3),
        "main_samples": main_n,
        "staging_samples": staging_n,
        "main_cohorts": main_cohorts,
        "staging_cohorts": staging_cohorts,
        "age_days": age_days,
        "main_sha": m_sha,
        "staging_sha": s_sha,
    }

    if staging_w >= main_w:
        return {"action": "promoted", **summary}
    return {"action": "rejected", **summary}


```

```bash
sed -n '1045,1080p' src/xskill/canary.py
```

```output
def apply_decision(
    skill_dir: Path,
    decision: dict,
    *,
    record_telemetry: bool = True,
) -> dict:
    """应用 :func:`plan_decision` 的终态，并在成功后记录裁决遥测。"""
    action = decision.get("action", "")
    if action not in ("promoted", "rejected", "timeout_discarded"):
        return decision
    skill_dir = Path(skill_dir)
    if action == "promoted":
        if not merge_staging_to_main(skill_dir):
            return {
                **decision,
                "action": "merge_failed",
                "attempted_action": action,
            }
    elif not discard_staging(skill_dir):
        return {
            **decision,
            "action": "discard_failed",
            "attempted_action": action,
        }
    if record_telemetry:
        record_decision_telemetry(skill_dir, decision)
    return decision


def check_and_decide(skill_dir: Path, config: CanaryConfig | None = None,
                     *, weights: dict[str, float] | None = None) -> dict:
    """计算并立即应用灰度决策；事务调用方应改用 :func:`plan_decision`。"""
    decision = plan_decision(skill_dir, config=config, weights=weights)
    return apply_decision(skill_dir, decision)


```

```bash
sed -n '1150,1199p' src/xskill/canary.py
```

```output
class AtomCanary:
    skill_dir: Path

    def append(self, *, atom_id: str, skill_name: str, side: str,
               commit_sha: str, score: float, reasons: str,
               user_model: str = "") -> bool:
        """幂等追加一条 atom 体验分。

        同一 (atom_id, skill_name, side) 三元组已存在则返回 False，不重复写入。
        ``user_model``: 产生该 atom 的用户模型，供模型分桶加权裁决用。
        """
        existing = load_ux_scores(self.skill_dir)
        for e in existing:
            if (e.get("atom_id") == atom_id
                    and e.get("skill_name") == skill_name
                    and e.get("side") == side):
                return False
        record = {
            "atom_id": atom_id,
            "skill_name": skill_name,
            "side": side,
            "commit_sha": commit_sha,
            "score": float(score),
            "reasons": reasons,
            "user_model": user_model,
            "scored_at": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        }
        p = self.skill_dir / UX_SCORES_FILENAME
        p.parent.mkdir(parents=True, exist_ok=True)
        with p.open("a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")
        db_rec = dict(record)
        db_rec["skill_name"] = Path(self.skill_dir).name
        _mirror_ux_score_to_db(db_rec)
        return True

    def recent(self, *, side: str, commit_sha: str, n: int) -> list[dict]:
        """与 ``recent_scores`` 同语义，但读 atom_id 字段。"""
        return recent_scores(
            self.skill_dir, side=side, commit_sha=commit_sha, n=n,
        )

    def check_and_decide(self, *, config: "CanaryConfig | None" = None,
                         weights: "dict[str, float] | None" = None) -> dict:
        """代理 ``check_and_decide``——判定逻辑不区分 atom/traj 粒度。

        ``weights`` 透传:给定则按模型分桶加权裁决,None 则单桶(等价旧均分)。
        """
        return check_and_decide(self.skill_dir, config=config, weights=weights)

```

```bash
sed -n '2748,2895p' src/xskill/pipeline/runner.py
```

```output
    def _finalize_completed_trajs(self, wd_id, dir_path, consumed_index, **kw):
        """把"所有 atom 都已落进某个 skill .candidates.yml"的 indexed 轨迹标
        done，并触发该轨迹的 ux 打分。

        这是跨轨迹批处理下 done 的唯一判据：cluster batch 不再 1:1 对应一条
        轨迹，所以每轮 scan 重新核对每条 indexed 轨迹是否已被完全消费。0-atom
        轨迹（无可拆 User 回合）视为已消费 → 直接 done。标 done 后该轨迹离开
        indexed，下一轮不再重复处理 → 打分每条只触发一次。

        判据用 atom 的耐久 ``clustered`` 标记（非 .candidates.yml 成员）——
        SkillEdit 晋升会清空 .candidates.yml，用它判 done 会让已消费 atom 看起来
        又未消费、轨迹永不 done。
        """
        store = self._store_for(dir_path)
        for fname in get_trajs_by_status(wd_id, "indexed", **kw):
            traj_id = (dir_path / fname).stem
            atoms = store.list_by_traj(traj_id)
            if any(not self._atom_consumed(a, consumed_index) for a in atoms):
                continue  # 还有未消费 atom → 等后续 batch 消费
            update_traj_status(
                wd_id, fname, "done", process_action="clustered", **kw,
            )
            # 该轨迹所有 atom 已落盘——ux_score 应当跑的时机。
            if self.server_mode:
                self._score_atoms_for_traj_server(wd_id, fname, **kw)
            else:
                self._score_atoms_for_traj(wd_id, fname, **kw)

    # ───────────────────────────────────────────────────────────
    # ux_score
    # ───────────────────────────────────────────────────────────

    def _score_atoms_for_traj(self, wd_id, fname, **kw):
        """对一条已跑完 cluster 的 traj 扫所有 atom 打 ux_score。

        前置：
        - traj.md 顶部含 ``<!-- xskill:skill=X side=Y sha=Z -->`` header
        - 该 traj 已拆出 atom

        每个 atom 直接使用 TaskAgent 拆分时生成的 ``ux_score``，再调
        ``AtomCanary.append``。同一 atom
        在同 (skill, side) 上幂等：``AtomCanary.append`` 自带去重。
        所有 atom 处理完调一次 ``check_and_decide`` 让 staging 该升的升 /
        该弃的弃。

        skill 定位走两步查找：先自有 ``skill_dir/<name>``（有 git / 灰度），
        未命中再查三方 ``skillhub_dir/<name>``（无 git → side 恒 ``main``、
        sha = ``SKILL.md`` 内容哈希前 16 位）。两处都无 → 该 skill 未装/未索引，
        跳过（不报错）。
        """
        if self.skill_dir is None:
            return
        from xskill.canary import AtomCanary
        # 找到该 wd 的 dir_path
        for wd in list_watch_dirs(**kw):
            if wd["id"] == wd_id:
                dir_path = Path(wd["path"])
                break
        else:
            return
        md_path = dir_path / fname
        if not md_path.is_file():
            return
        md_text = md_path.read_text(encoding="utf-8")
        header = parse_traj_header(md_text)
        if not header or not header.get("skill") or not header.get("side"):
            return
        skill_name = header["skill"]
        traj_id = md_path.stem
        store = self._store_for(dir_path)
        atoms = store.list_by_traj(traj_id)
        if not atoms:
            return
        skill_sub, side, commit_sha = self._resolve_skill_for_scoring(
            skill_name, header)
        if skill_sub is None:
            return
        ac = AtomCanary(skill_dir=skill_sub)
        new_scores: list[float] = []
        for atom in atoms:
            try:
                if atom.ux_score is None:
                    continue
                if ac.append(
                    atom_id=atom.atom_id, skill_name=skill_name,
                    side=side, commit_sha=commit_sha,
                    score=atom.ux_score,
                    reasons=["TaskAgent split ux_score"],
                    user_model=atom.source_model,
                ):
                    new_scores.append(float(atom.ux_score))
                self._stats["scores"] += 1
            except Exception:
                logger.exception("score_atom failed: %s/%s",
                                 fname, atom.atom_id)
        # 翻牌判定
        # check_and_decide 不再绑在打分链路里——移到 watcher 周期性
        # _check_canary_decisions() 独立轮询，保证灰度系统自治不依赖
        # traj 触发。这里只负责打分落盘。
        mark_skill_used(wd_id, fname, skill_name, side, **kw)
        if new_scores:
            self._emit_feedback_event(
                wd_id, fname, skill_name=skill_name, traj_id=traj_id,
                scores=new_scores, side=side, sha=commit_sha, **kw)

    def _resolve_skill_for_scoring(
        self, skill_name: str, header: dict,
    ) -> tuple[Path | None, str, str]:
        """两步定位打分目标 skill：先 ``skill_dir``（自有），后 ``skillhub_dir``（三方）。

        返回 ``(skill_sub, side, commit_sha)``；两处都无 → ``(None, "", "")``，
        调用方据此跳过（该 skill 未装/未索引，非错误）。

        - 自有 skill：side/sha 取自 traj header（client 在推荐时写入）。
        - 三方 skill：无 git/staging → side 恒 ``"main"``、sha = ``SKILL.md``
          内容 sha256 前 16 位（``SkillHub.content_sha``）。
        """
        own = self.skill_dir / skill_name
        if own.is_dir():
            return own, header["side"], header.get("sha", "")
        from xskill.recommend.skillhub import SkillHub
        hub = SkillHub.from_config(self.config, self.embed_client)
        hub_sub = hub.skill_path(skill_name)
        if hub_sub is None:
            return None, "", ""
        return hub_sub, "main", hub.content_sha(skill_name) or ""

    def _score_atoms_for_traj_server(self, wd_id, fname, **kw):
        """CS 模式打分：遍历每个 atom 的 used_skills，对每个用到的 team skill
        用 pick_side(client_id, ...) 现算 side，逐个 score + AtomCanary.append。

        与单机 _score_atoms_for_traj 的差异：
        - 不读 traj header（一条上传轨迹可能用多个 team skill）
        - client_id 从 watch_dir 的 label 取（upload 端点注册时 label=client_id）
        - side 由 pick_side 现算，不是 header 里写死的

        skill 定位同样走两步查找：先自有 ``skill_dir/<name>``（有 git → 走灰度
        路由），未命中再查三方 ``skillhub_dir/<name>``（无 git → side 恒 ``main``、
        sha = 内容哈希）。两处都无 → 跳过该 skill（不报错）。
        """
        if self.skill_dir is None:
            return
        from xskill.canary import AtomCanary
        from xskill.canary import CanaryConfig, eligible_models
        from xskill.pipeline.registry import model_share
        from xskill.recommend.skillhub import SkillHub

        # 找到该 wd 的 dir_path + client_id（label）
```

## 11. A chosen version is installed into agent ecosystems

The native Skill repository is not necessarily where an agent loads Skills. Installation projects the chosen side into each detected harness:

- Claude Code receives `~/.claude/skills/<name>`;
- Codex, OpenCode, and OpenClaw share `~/.agents/skills/<name>`;
- Cursor, Trae, nga3, and DeepSeek Harness use their native roots.

The common installer prefers a symlink, uses a Windows junction when appropriate, and falls back to a real copy. Metadata and an install ledger record ownership, source identity, side, and SHA. Reconciliation is idempotent when the destination already matches. Removal and copy replacement are conservative: xskill deletes only targets it can prove it owns and protects edits whose content no longer matches the recorded baseline.

Standalone mode periodically materializes staging and rotates installed sides, then records session-to-side assignments so the later UX score is attributed to the version actually used. When a local user edits an installed Skill, the reverse-sync and `UserEditAbsorbAgent` paths wait for a quiet window, preserve filesystem safety, absorb the edit to main, and supersede an in-flight staging candidate because explicit user work is treated as ground truth.

Team mode moves that trust boundary to the server: the client commits a dirty working copy to a temporary `_useredit` branch and uploads a branch bundle; the server stores it as isolated `user-staging/<client_id>`, never permitting a client to write main directly.

```bash
sed -n '275,382p' src/xskill/ecosystems/_shared.py
```

```output
def _source_md_for_side(skill_path: Path, side: str) -> Path:
    """根据 side 选磁盘上的内容源。

    main:     ``<skill_path>/SKILL.md``                       (git@main 的工作树)
    staging:  ``<skill_path>/../.canary/<name>/SKILL.md``    (canary.materialize_staging 物化)

    两侧都不存在则抛 FileNotFoundError——daemon 翻牌子时这两个文件应当**已经**
    都准备好了；找不到说明灰度状态不一致，应该 fail-loud 而不是 silently
    fall back（CLAUDE.md "遇到问题 throw error"）。
    """
    if side == "main":
        src = skill_path / "SKILL.md"
        if not src.is_file():
            raise FileNotFoundError(f"main SKILL.md not found: {src}")
        return src
    if side == "staging":
        canary_md = skill_path.parent / ".canary" / skill_path.name / "SKILL.md"
        if not canary_md.is_file():
            raise FileNotFoundError(
                f"staging SKILL.md not found: {canary_md} "
                f"(did you forget canary.materialize_staging?)"
            )
        return canary_md
    raise ValueError(f"side must be 'main' or 'staging', got {side!r}")


def _install_skill_into(
    skill_path: Path,
    skills_root: Path,
    side: str,
    *,
    ecosystem_label: str,
) -> Path:
    """共享的 skill 安装实现：把 ``skill_path``（或其 staging 物化版）装到
    ``skills_root/<name>``，走三阶 fallback。

    被 ``install_to_claude_code`` / ``install_to_codex`` / ``install_to_opencode``
    共用——它们只在 ``skills_root`` 的解析上不同，安装语义完全一致。

    ``ecosystem_label`` 仅用于 warning log 时打"是哪个生态遇到 copy fallback"，
    便于运维定位。

    Args:
        skill_path: ``main`` 时即源 skill 目录；``staging`` 时取 ``..canary/<name>``
        skills_root: 安装目标根（``<home>/.claude/skills`` 或
            ``<home>/.agents/skills``）
        side: ``main`` / ``staging``
        ecosystem_label: ``claude_code`` / ``codex`` /... 用于日志

    Returns:
        ``<skills_root>/<name>/SKILL.md`` 路径
    """
    skill_path = Path(skill_path).resolve()
    if not skill_path.is_dir():
        raise NotADirectoryError(f"skill_path is not a directory: {skill_path}")

    # 校验源是否齐备（main: SKILL.md 必须有；staging: .canary/<name>/SKILL.md 必须有）
    _source_md_for_side(skill_path, side)

    if side == "main":
        src_dir = skill_path
    elif side == "staging":
        src_dir = (skill_path.parent / ".canary" / skill_path.name).resolve()
    else:
        raise ValueError(f"side must be 'main' or 'staging', got {side!r}")

    name = skill_path.name
    skills_root.mkdir(parents=True, exist_ok=True)
    dest = skills_root / name

    # 已有 symlink/junction 且指向正确：no-op。``is_link_or_junction``
    # 而非 ``is_symlink`` —— pathlib 在 Windows 对 junction 返回 False
    # （issue #35 同源 bug），统一处理 link/junction 两种 reparse point。
    if is_link_or_junction(dest):
        try:
            cur = dest.resolve(strict=False)
        except OSError:
            cur = None
        if cur == src_dir:
            ensure_link_install_metadata(dest, src_dir)
            return dest / "SKILL.md"
        # 指向别处的 link/junction 或断链 → ``unlink`` 删 reparse 本体
        # （不递归动 target）
        dest.unlink()
    elif dest.exists():
        # copy 模式且源未变：no-op——避免每轮 reconcile 重拷 + 重试 junction（Windows 弹 cmd 窗）
        if copy_install_is_current(src_dir, dest):
            return dest / "SKILL.md"
        # 旧 install 留下的真实目录或文件 → 删（保留备份避免误删用户手写）。
        # ``.replaced-by-symlink`` 备份保留——这是用户手写 skill 目录的保护机制，
        # 不是 boilerplate；不能直接走 ``_reset_dest`` 删掉。
        if dest.is_dir():
            backup = skills_root / f".{name}.replaced-by-symlink"
            if backup.exists():
                shutil.rmtree(backup)
            dest.rename(backup)
        else:
            dest.unlink()

    mode: InstallMode = install_dir(src_dir, dest)
    if mode == "copy":
        # copy 模式下 UserEditAbsorbAgent 失效 —— 用户改副本源仓看不到。
        logger.warning(
            "install_to_%s(%s): copy-mode install at %s — "
            "live-update / user-edit-absorb are disabled on this destination",
            ecosystem_label, name, dest,
        )
    return dest / "SKILL.md"
```

```bash
sed -n '1009,1180p' src/xskill/pipeline/runner.py
```

```output
    def _install_skill_to_all_detected(
        self,
        skill_path,
        *,
        excluded_ecosystems: set[str] | None = None,
    ):
        """把该 skill 装到**当前 detected 的所有 agent 生态**。

        每次调用实时跑 ``detect_known_ecosystems`` 决定要装哪些 agent
        ——3 次 ``Path.is_dir/is_file`` 开销可忽略，比启动时缓存稳定（用户
        中途装新 agent 也能被发现）。

        每个 installer 独立 ``try/except``：一个失败不影响其它 agent 继续
        装；失败记录写到 ``~/.xskill/install_history.jsonl`` 的同一个文件
        （加 ``action="fail"`` 字段）。至少一个成功就算整体 OK——daemon
        不抛异常给上层 watcher loop。

        Args:
            skill_path: ``self.skill_dir / <name>`` 的 Path 对象

        Returns:
            dict[str, Path | Exception]: agent → 安装结果（成功为 dest 路径，
            失败为异常对象）。便于调用方 / 测试断言。
        """
        # server 模式：纯 server 不装 skill 到本机生态，直接 no-op。
        if self.server_mode:
            return {}
        from xskill.ecosystems import (
            detect_known_ecosystems,
            install_to_claude_code,
            install_to_codex,
            install_to_nga3,
            install_to_opencode,
            install_to_ngagent,
            install_to_openclaw,
            install_to_cursor,
            install_to_deepseek_harness,
            install_to_trae,
        )

        target_root = self._resolve_target_root()
        # 实时 detect。测试场景下 self.home_root 是 tmp_path，detect 也
        # 走 tmp_path——只有 tmp_path 里真造了 .claude/projects 之类目录，
        # 该生态才会被探到，不会污染用户真目录。
        detect_root = self.home_root or target_root
        detections = detect_known_ecosystems(home_root=detect_root) if detect_root else []

        installer_by_ecosystem = {
            "claude_code": install_to_claude_code,
            "codex": install_to_codex,
            "nga3": install_to_nga3,
            "opencode": install_to_opencode,
            "ngagent": install_to_ngagent,  # opencode 企业分支，独立 skill 目录
            "openclaw": install_to_openclaw,  # copy 模式，详见 install_to_openclaw docstring
            "cursor": install_to_cursor,
            "trae": install_to_trae,
            "deepseek_harness": install_to_deepseek_harness,
        }

        results: dict = {}
        any_ok = False
        excluded = excluded_ecosystems or set()
        attempted_count = 0
        for det in detections:
            agent = det["ecosystem"]
            if agent in excluded:
                continue
            installer = installer_by_ecosystem.get(agent)
            if installer is None:
                continue
            attempted_count += 1
            try:
                dest = installer(skill_path, target_root=target_root, side="main")
                results[agent] = dest
                any_ok = True
                logger.info("installed (symlink) to %s: %s", agent, dest)
            except Exception as e:
                results[agent] = e
                logger.warning(
                    "install_to_%s failed for %s: %s",
                    agent, skill_path.name, e,
                )
                self._record_install_fail(
                    skill=skill_path.name, agent=agent, reason=str(e)[:200],
                )
        if not detections:
            logger.debug(
                "_install_skill_to_all_detected(%s): no agent detected under %s",
                skill_path.name, detect_root,
            )
        elif attempted_count and not any_ok:
            logger.warning(
                "_install_skill_to_all_detected(%s): all %d attempted agent(s) failed",
                skill_path.name,
                attempted_count,
            )
        return results

    def _record_install_fail(self, *, skill: str, agent: str, reason: str) -> None:
        """把一条 install 失败写到当前 XSkill 实例的 install_history。

        失败记录走 ``InstallHistory.record_fail``（带 ``action="fail"``
        字段），与成功 install 记录在同一文件，不分两份避免 source 熵增。

        写盘本身失败不传播——失败日志的失败只能 logger.warning。
        """
        try:
            from xskill.ecosystems._history import InstallHistory
            InstallHistory(self.install_history_path).record_fail(
                skill=skill, agent=agent, reason=reason,
            )
        except Exception:
            logger.exception(
                "record_install_fail failed (skill=%s agent=%s)",
                skill, agent,
            )

    def _install_skill_to_cc(self, skill_path):
        """Backward-compat thin wrapper for ``_install_skill_to_all_detected``.

        旧调用路径 / 旧测试可能直接调本方法，保留它走多 agent install
        逻辑（不是只装 CC）。新代码应直接调 ``_install_skill_to_all_detected``。
        """
        return self._install_skill_to_all_detected(skill_path)

    def _check_user_edits(self):
        """检测每个 skill 是否有用户手改且静默 ≥3 分钟 → 触发 absorb agent。

        对每个 skill 先扫一遍 openclaw dest 看有没有用户改要回流——openclaw
        装的 skill 是 copy 不是 symlink，dest 跟源仓解耦。reverse_sync 把 dest
        改动灌回源仓 + touch source mtime，让 detect_user_edits 在**同一轮**内
        看到 pending edit，直接走原有 absorb 链路。
        """
        if self.skill_dir is None or not self.skill_dir.is_dir():
            return
        from xskill.agents.user_edit_absorb_agent import (
            ReverseSyncStatus,
            UserEditAbsorbAgent,
            detect_user_edits,
            reverse_sync_openclaw_dest,
        )
        from xskill.agents import agent_tools
        target_root = self._resolve_target_root()
        factory = self._factory()
        tool_context = agent_tools.create_agent_tool_context(
            skill_dir=self.skill_dir,
            data_dir=self.skill_dir,
            config=self.config,
            atom_skill_dir=self.skill_dir,
            default_traj_root=self.skill_dir,
            spill_root=self.spill_root,
            usage_ledger=self.usage_ledger,
            registry_db_path=self.db_path,
        )
        for d in sorted(self.skill_dir.iterdir()):
            if not d.is_dir() or d.name.startswith("."):
                continue
            try:
                # openclaw 回流（dest → source）— 没装 openclaw / dest 不存在
                # / dest 没改 → no-op。返回 True 意味着 source mtime 刚被 touch，
                # 下面 detect_user_edits 会立刻看到 pending edit。
                if target_root is not None:
                    dest_dir = target_root / ".agents" / "skills" / d.name
                    try:
                        reverse_status = reverse_sync_openclaw_dest(
                            dest_dir, d,
                        )
                    except Exception:
                        logger.warning(
                            "openclaw reverse sync stopped skill_id_hash=%s "
                            "error_type=REVERSE_SYNC_UNEXPECTED",
                            hashlib.sha256(
```

```bash
sed -n '1134,1230p' src/xskill/pipeline/runner.py
```

```output
    def _check_user_edits(self):
        """检测每个 skill 是否有用户手改且静默 ≥3 分钟 → 触发 absorb agent。

        对每个 skill 先扫一遍 openclaw dest 看有没有用户改要回流——openclaw
        装的 skill 是 copy 不是 symlink，dest 跟源仓解耦。reverse_sync 把 dest
        改动灌回源仓 + touch source mtime，让 detect_user_edits 在**同一轮**内
        看到 pending edit，直接走原有 absorb 链路。
        """
        if self.skill_dir is None or not self.skill_dir.is_dir():
            return
        from xskill.agents.user_edit_absorb_agent import (
            ReverseSyncStatus,
            UserEditAbsorbAgent,
            detect_user_edits,
            reverse_sync_openclaw_dest,
        )
        from xskill.agents import agent_tools
        target_root = self._resolve_target_root()
        factory = self._factory()
        tool_context = agent_tools.create_agent_tool_context(
            skill_dir=self.skill_dir,
            data_dir=self.skill_dir,
            config=self.config,
            atom_skill_dir=self.skill_dir,
            default_traj_root=self.skill_dir,
            spill_root=self.spill_root,
            usage_ledger=self.usage_ledger,
            registry_db_path=self.db_path,
        )
        for d in sorted(self.skill_dir.iterdir()):
            if not d.is_dir() or d.name.startswith("."):
                continue
            try:
                # openclaw 回流（dest → source）— 没装 openclaw / dest 不存在
                # / dest 没改 → no-op。返回 True 意味着 source mtime 刚被 touch，
                # 下面 detect_user_edits 会立刻看到 pending edit。
                if target_root is not None:
                    dest_dir = target_root / ".agents" / "skills" / d.name
                    try:
                        reverse_status = reverse_sync_openclaw_dest(
                            dest_dir, d,
                        )
                    except Exception:
                        logger.warning(
                            "openclaw reverse sync stopped skill_id_hash=%s "
                            "error_type=REVERSE_SYNC_UNEXPECTED",
                            hashlib.sha256(
                                d.name.encode("utf-8"),
                            ).hexdigest()[:12],
                        )
                        continue
                    if reverse_status in {
                        ReverseSyncStatus.RECENT_EDIT,
                        ReverseSyncStatus.FAILED,
                    }:
                        logger.warning(
                            "openclaw reverse sync stopped "
                            "skill_id_hash=%s error_type=%s",
                            hashlib.sha256(
                                d.name.encode("utf-8"),
                            ).hexdigest()[:12],
                            (
                                "REVERSE_SYNC_RECENT_EDIT"
                                if reverse_status
                                == ReverseSyncStatus.RECENT_EDIT
                                else "REVERSE_SYNC_FAILED"
                            ),
                        )
                        continue
                    if reverse_status not in {
                        ReverseSyncStatus.NO_EDIT,
                        ReverseSyncStatus.SYNCED,
                    }:
                        logger.warning(
                            "openclaw reverse sync stopped "
                            "skill_id_hash=%s "
                            "error_type=REVERSE_SYNC_INVALID_STATUS",
                            hashlib.sha256(
                                d.name.encode("utf-8"),
                            ).hexdigest()[:12],
                        )
                        continue

                if not detect_user_edits(d):
                    continue
                logger.info("user edit detected (stable for 3+ min): %s", d.name)
                with agent_tools.use_agent_tool_context(tool_context):
                    ok = UserEditAbsorbAgent(
                        skill_dir=d,
                        agno_agent_factory=factory,
                        llm_cfg=self.config.get("llm", {}),
                    ).run()
                if ok:
                    self._install_skill_to_all_detected(d)
            except Exception:
                logger.exception("user edit absorb failed: %s", d.name)

```

```bash
sed -n '1486,1520p' src/xskill/team/server/api.py
```

```output
async def team_push_edit(
    request: Request,
    x_xskill_token: str | None = Header(default=None),
    x_xskill_client: str | None = Header(default=None),
    x_xskill_skill: str | None = Header(default=None),
) -> PushEditResponse:
    client_id = _auth(x_xskill_token, x_xskill_client)
    if not x_xskill_skill:
        raise HTTPException(status_code=400, detail="X-Xskill-Skill header required")
    repo_dir = _ctx.skill_dir / x_xskill_skill
    if not (repo_dir / ".git").is_dir():
        raise HTTPException(status_code=404, detail=f"skill not found: {x_xskill_skill}")
    bundle = await request.body()
    if not bundle:
        raise HTTPException(status_code=400, detail="empty bundle")
    dest_ref = f"refs/heads/user-staging/{client_id}"
    # git 子进程是阻塞调用，卸到线程池，别占事件循环
    sha = await run_in_threadpool(
        fetch_branch_from_bundle, bundle, repo_dir, "_useredit", dest_ref)
    logger.info("team push-edit: %s -> %s (%s)", x_xskill_skill, dest_ref, sha[:8])
    # P3-3.1 埋点:手改分支即修改意见——通知该 skill 贡献者(旁路,失败不阻断)
    try:
        from xskill.events import EventStore
        row = _ctx.client_registry.get(client_id) or {}
        EventStore().emit_push_edit(
            actor=row.get("user_name") or client_id,
            skill=x_xskill_skill,
            branch=f"user-staging/{client_id}", ref_sha=sha)
    except Exception:  # pylint: disable=broad-exception-caught
        logger.debug("push-edit event emit skipped", exc_info=True)
    return PushEditResponse(branch=f"user-staging/{client_id}", ref_sha=sha)


@router.post("/generate", response_model=GenerateAccepted)
def team_generate(
```

## 12. Team mode reuses the same pipeline behind a thin client

`xskill connect` does not load `config.yaml`, construct an LLM, or run the distillation pipeline. The registration handshake stores server URL, join token, and stable client identity. The long-lived `TeamClient` then does a fixed reconciliation tick:

`collect/upload → sync manifest → reconcile versions → refresh explicit downloads → push edits → cleanup`

Its `TeamCollector` runs the same ecosystem ingesters locally, but only as mirrors into normal bridge directories. Before upload, a trajectory must pass two quiet gates: its file mtime must be old enough not to be mid-write, and its redacted content hash must have stopped changing for the longer debounce interval. `detect-secrets`-based redaction happens before hashing and transport. SQLite records file identity, clean hash, debounce start, and the last accepted upload, so restarts and metadata-only changes do not resend content.

The team server authenticates both the join token and client ID. Registration can derive stable identity from an employee/user name. Uploads are placed in a per-user session bucket. The server verifies the supplied SHA-256, writes model/harness sidecar metadata before the Markdown (so discovery cannot race ahead), sanitizes content, and ensures that bucket's watch-directory pause state matches the client registry.

From that point the ordinary watcher takes over: the uploaded trajectory is discovered, split, indexed, clustered, edited, and scored exactly like local input. The watch-directory label gives atoms and feedback a canonical user attribution. This is the key team design: transport changes, but distillation does not fork into a second implementation.

On sync, the client receives `SkillSlot` records naming source, side, and immutable SHA. Native Skills arrive as Git bundles and are reconciled to that exact commit; SkillHub entries arrive as validated ZIP archives. Anything removed from the manifest is uninstalled only when ownership can be proven, then its working copy is cleaned up.

```bash
sed -n '100,250p' src/xskill/team/client/daemon.py
```

```output
class TeamClient:
    """team 瘦客户端。http 接受 httpx.Client 或 FastAPI TestClient。"""

    def __init__(
        self,
        *,
        state: ClientState,
        http,
        skill_dir: Path,
        cursor_path: Path,
        history_path: Path,
        home_root: Path | None = None,
        poll_interval: float = 30.0,
        quiet_seconds: int = 180,
        min_change_interval: int = 600,
        auto_update: bool = True,
        use_proxy: bool = False,
    ):
        self.state = state
        self.http = http
        # skill working copies 落标准 skill_dir（= ~/.xskill/skill/）——与
        # standalone 模式同一个位置，不另开 team_skills/。一台机器要么
        # standalone 要么 client，这个目录谁来管取决于模式。
        self.skill_dir = Path(skill_dir)
        self.skill_dir.mkdir(parents=True, exist_ok=True)
        self.home_root = Path(home_root) if home_root else Path.home()
        self.poll_interval = poll_interval
        self.history = InstallHistory(history_path)
        self.collector = TeamCollector(
            cursor_path=Path(cursor_path),
            quiet_seconds=quiet_seconds, home_root=self.home_root,
            min_change_interval=min_change_interval,
        )
        self._stop = threading.Event()
        self.auto_update = auto_update
        # updater 的 server 方向请求跟随 connect 的 --use-proxy；默认直连内网 server。
        self.use_proxy = use_proxy

    # ── HTTP 鉴权头 ──────────────────────────────────────────────
    def _hdr(self, extra: dict | None = None) -> dict:
        from xskill import __version__ as _xskill_version
        h = {"X-Xskill-Token": self.state.join_token,
             "X-Xskill-Client": self.state.client_id,
             # P2-2.10:每次 sync 携带版本,server touch 时 upsert 进 clients 表
             "X-Xskill-Version": _xskill_version}
        if extra:
            h.update(extra)
        return h

    # ── ① 采集 + 上传 ────────────────────────────────────────────
    def collect_and_upload(self) -> int:
        """扫 outbox 静默轨迹，脱敏后上传 server。返回成功上传条数。"""
        pending = self.collector.pending()
        if not pending:
            return 0
        req = UploadRequest(trajectories=[
            UploadTrajectory(traj_id=p.traj_id, content=p.content, sha256=p.sha256,
                             model=p.model, harness=p.harness)
            for p in pending
        ])
        resp = self.http.post("/api/v1/team/upload", headers=self._hdr(),
                              json=req.model_dump())
        if resp.status_code != 200:
            logger.warning("upload failed http_status=%s", resp.status_code)
            return 0
        accepted = set(resp.json().get("accepted", []))
        for p in pending:
            if p.traj_id in accepted:
                self.collector.mark_uploaded(p.traj_id, p.sha256)
        logger.info("uploaded %d trajectories", len(accepted))
        return len(accepted)

    # ── ② sync ──────────────────────────────────────────────────
    def sync(self) -> SyncResponse:
        """拉 server 现算的 skill manifest。"""
        resp = self.http.get("/api/v1/team/sync", headers=self._hdr())
        if resp.status_code != 200:
            raise RuntimeError(f"sync failed: HTTP {resp.status_code} — {resp.text}")
        return SyncResponse.model_validate(resp.json())

    @staticmethod
    def apply_client_take(manifest: SyncResponse) -> SyncResponse:
        """按 server 下发的 ``take_n`` 截取安装队列；``None``=装全部（旧 server）。

        截取后的 slots 同时驱动 reconcile 与 cleanup——减小 N 时尾部会被卸掉。
        """
        if manifest.take_n is None:
            return manifest
        n = max(0, int(manifest.take_n))
        if n >= len(manifest.slots):
            return manifest
        manifest.slots = list(manifest.slots[:n])
        return manifest

    # ── ③ reconcile ─────────────────────────────────────────────
    def reconcile_skill_sides(self, manifest: SyncResponse) -> None:
        """对 manifest 每个 slot：拉 bundle → 对齐 side → 装到本机生态。

        这是设计里约定的 reconcile_skill_sides——契约步骤 1（决定 target）
        就是读 manifest slot 的 side/sha；步骤 2/3/4 走共享
        reconcile_skill_side。
        """
        for slot in manifest.slots:
            repo_dir = self.skill_dir / slot.skill_name
            # 拉 bundle 落地/刷新本地 working copy
            r = self.http.get(f"/api/v1/team/skill/{slot.skill_name}/bundle",
                              headers=self._hdr())
            if r.status_code != 200:
                logger.warning(
                    "bundle fetch failed skill_id_hash=%s http_status=%s",
                    hashlib.sha256(
                        slot.skill_name.encode("utf-8"),
                    ).hexdigest()[:12],
                    r.status_code,
                )
                continue
            if getattr(slot, "source", "repo") == "skillhub":
                # 与 repo slot 的 on_changed 语义对齐：内容没变不重装生态
                if self._apply_skillhub_archive(
                    r.content, repo_dir, expected_sha=slot.sha,
                    display_name=slot.display_name,
                    source_path=slot.source_path,
                ):
                    self._install_to_ecosystems(repo_dir)
                continue
            apply_repo_bundle(r.content, repo_dir)
            # 步骤 1 = manifest 给的 (side, sha)；2/3/4 = 共享助手
            reconcile_skill_side(
                repo_dir=repo_dir, target_side=slot.side, target_sha=slot.sha,
                history=self.history, on_changed=self._install_to_ecosystems,
            )
        logger.info("reconciled %d skills", len(manifest.slots))

    def _apply_skillhub_archive(
        self, archive_bytes: bytes, dest_dir: Path, *, expected_sha: str,
        display_name: str | None, source_path: str | None,
    ) -> bool:
        return apply_skillhub_archive(
            archive_bytes, dest_dir, expected_sha=expected_sha,
            display_name=display_name, source_path=source_path,
        )

    def _install_to_ecosystems(self, repo_dir: Path) -> None:
        install_skill_to_ecosystems(repo_dir, home_root=self.home_root)

    def reconcile_downloaded_skills(self) -> int:
        """刷新显式下载项；持久下载不占 search LRU，随服务端版本继续更新。"""
        from xskill.team.client.search_slots import (
            DownloadedSkills,
            _valid_slot_id,
        )
```

```bash
sed -n '83,190p' src/xskill/team/client/collector.py
```

```output
class TeamCollector:
    """采集本机生态轨迹 → 标准 bridge 目录；吐 pending 给 TeamClient 上传。"""

    def __init__(
        self,
        *,
        cursor_path: Path,
        quiet_seconds: int = 180,
        min_change_interval: int = 600,
        home_root: Path | None = None,
        poll_interval: float = 10.0,
        time_fn: Callable[[], float] = time.time,
        state_db_path: Path | None = None,
    ):
        self.cursor_path = Path(cursor_path)
        self.quiet_seconds = quiet_seconds
        # 上传频率拦截：同一条 traj 的内容（hash）距上次变更必须
        # 静默 ≥ min_change_interval 秒才允许上传。用户代理工具调用会让轨迹文件
        # 每 ~30s 追加一次，若每次增量都上传，server 每次跑全量流水线，原子会被
        # 切碎、不成体系。这里按 hash-变更去抖（debounce）：内容只要还在变，计时
        # 就一直重置，直到稳定满 10 分钟（默认）才放行——保证上传的是一段相对
        # 完整、可被连贯拆分的轨迹。
        self.min_change_interval = min_change_interval
        self.home_root = Path(home_root) if home_root else Path.home()
        self.poll_interval = poll_interval
        self._now = time_fn
        # 标准 bridge 目录都落在 <home_root>/.xskill/ 下（cc_sessions /
        # codex_sessions / opencode_sessions）——与 detect_known_ecosystems
        # 返回的 bridge 路径一致。
        self._bridge_root = self.home_root / ".xskill"
        self._ingesters: list = []
        self.state_db_path = (
            Path(state_db_path) if state_db_path
            else self.cursor_path.with_name("client_state.db")
        )
        self._state_store = TrajectoryUploadStateStore(
            db_path=self.state_db_path,
            legacy_cursor_path=self.cursor_path,
            home_root=self.home_root,
            time_fn=self._now,
        )

    def mark_uploaded(self, traj_id: str, sha256: str) -> None:
        """记录某 traj 的某版本已上传。同时清掉它的去抖状态（该版本已落地）。"""
        self._state_store.mark_uploaded(traj_id, sha256)

    # ── ingester 生命周期 ────────────────────────────────────────
    def start_ingesters(self) -> None:
        """探测本机生态，对每个起一个纯镜像 ingester 写进标准 bridge 目录。"""
        from xskill.ecosystems import (
            detect_known_ecosystems, JsonlIngester, SqliteIngester,
            TraeIngester,
            CC_SPEC, CODEX_SPEC, DSH_SPEC, NGA3_SPEC, OPENCODE_SPEC,
            NGAGENT_SPEC,
        )
        for det in detect_known_ecosystems(home_root=self.home_root):
            eco = det["ecosystem"]
            bridge = det["bridge"]   # 标准路径 ~/.xskill/<eco>_sessions
            bridge.mkdir(parents=True, exist_ok=True)
            if eco == "claude_code":
                ing = JsonlIngester(CC_SPEC, target_traj_dir=bridge,
                                    home_root=self.home_root,
                                    poll_interval=self.poll_interval)
            elif eco == "codex":
                ing = JsonlIngester(CODEX_SPEC, target_traj_dir=bridge,
                                    home_root=self.home_root,
                                    poll_interval=self.poll_interval)
            elif eco == "nga3":
                # nga3 / CodeAgent3（~/.cac/projects）。daemon 侧
                # （api/app.py）一直有这条分支，collector 这条平行分发链
                # 漏掉了它——.cac 用户的轨迹从未被采集。
                ing = JsonlIngester(NGA3_SPEC, target_traj_dir=bridge,
                                    home_root=self.home_root,
                                    poll_interval=self.poll_interval)
            elif eco == "opencode":
                ing = SqliteIngester(target_traj_dir=bridge,
                                     home_root=self.home_root,
                                     spec=OPENCODE_SPEC,
                                     poll_interval=self.poll_interval)
            elif eco == "ngagent":
                # ngagent = opencode 企业分支，复用 SqliteIngester，只换 spec
                ing = SqliteIngester(target_traj_dir=bridge,
                                     home_root=self.home_root,
                                     spec=NGAGENT_SPEC,
                                     poll_interval=self.poll_interval)
            elif eco == "trae":
                ing = TraeIngester(target_traj_dir=bridge,
                                   home_root=self.home_root,
                                   poll_interval=self.poll_interval)
            elif eco == "deepseek_harness":
                # DeepSeek Harness（~/.dsh/sessions）。与 nga3 同一教训：
                # daemon 侧 watcher_factory 有这条分支，collector 这条
                # 平行分发链也必须接上，否则 connect 后 dsh 会话不会被
                # 镜像进 bridge，团队模式下永远采不到（PR #243 评审发现）。
                ing = JsonlIngester(DSH_SPEC, target_traj_dir=bridge,
                                    home_root=self.home_root,
                                    poll_interval=self.poll_interval)
            else:
                continue
            ing.start()
            self._ingesters.append(ing)
            logger.info("collector ingester started: %s -> %s", eco, bridge)

    def stop_ingesters(self) -> None:
        for ing in self._ingesters:
            try:
                ing.stop()
            except Exception:
```

```bash
sed -n '255,341p' src/xskill/team/client/collector.py
```

```output
                    content = redact_text(raw)
                    sha = hashlib.sha256(content.encode("utf-8")).hexdigest()
                    self._state_store.record_seen_file(
                        trajectory_id=traj_id,
                        file_path=str(md),
                        harness_name=harness_name,
                        model_name=model_name,
                        file_size_bytes=stat.st_size,
                        file_modified_time_nanoseconds=stat.st_mtime_ns,
                        file_changed_time_nanoseconds=stat.st_ctime_ns,
                        original_content_hash=raw_sha,
                        cleaned_content_hash=sha,
                    )
                else:
                    content = None
            else:
                raw = self._read_trajectory_text(md)
                raw_sha = hashlib.sha256(raw.encode("utf-8")).hexdigest()
                if state is not None and state["original_content_hash"] == raw_sha:
                    sha = state["cleaned_content_hash"]
                    content = None
                    self._state_store.record_seen_file(
                        trajectory_id=traj_id,
                        file_path=str(md),
                        harness_name=harness_name,
                        model_name=model_name,
                        file_size_bytes=stat.st_size,
                        file_modified_time_nanoseconds=stat.st_mtime_ns,
                        file_changed_time_nanoseconds=stat.st_ctime_ns,
                        original_content_hash=raw_sha,
                    )
                    refreshed = self._state_store.get(traj_id)
                    if (
                        sha
                        and refreshed is not None
                        and refreshed["uploaded_cleaned_content_hash"] == sha
                    ):
                        if refreshed["waiting_content_hash"] is not None:
                            self._state_store.clear_waiting(traj_id)
                        continue
                else:
                    content = redact_text(raw)
                    sha = hashlib.sha256(content.encode("utf-8")).hexdigest()
                    self._state_store.record_seen_file(
                        trajectory_id=traj_id,
                        file_path=str(md),
                        harness_name=harness_name,
                        model_name=model_name,
                        file_size_bytes=stat.st_size,
                        file_modified_time_nanoseconds=stat.st_mtime_ns,
                        file_changed_time_nanoseconds=stat.st_ctime_ns,
                        original_content_hash=raw_sha,
                        cleaned_content_hash=sha,
                    )
                    refreshed = self._state_store.get(traj_id)
                    if (
                        refreshed is not None
                        and refreshed["uploaded_cleaned_content_hash"] == sha
                    ):
                        if refreshed["waiting_content_hash"] is not None:
                            self._state_store.clear_waiting(traj_id)
                        continue
            if not sha:
                continue
            # 闸 2：hash-变更去抖。内容（hash）每变一次就把 since 重置成此刻,
            # 必须自上次变更起稳定满 min_change_interval 秒才放行。
            state = self._state_store.get(traj_id)
            if state is None or state["waiting_content_hash"] != sha:
                self._state_store.set_waiting(
                    trajectory_id=traj_id,
                    waiting_content_hash=sha,
                    waiting_started_at_seconds=now,
                )
                waiting_started_at = now
            else:
                waiting_started_at = float(state["waiting_started_at_seconds"] or now)
            if (now - waiting_started_at) < self.min_change_interval:
                continue  # 还没稳定满窗口,继续拦（min_change_interval<=0 时恒放行）
            if content is None:
                raw = self._read_trajectory_text(md)
                content = redact_text(raw)
            out.append(PendingTrajectory(traj_id=traj_id, content=content,
                                         sha256=sha, model=model_name,
                                         harness=harness_name))
        # 清理已消失的 traj 的去抖状态,避免无限增长
        self._state_store.clear_waiting_for_missing(seen_ids)
        return out
```

```bash
sed -n '651,776p' src/xskill/team/server/api.py
```

```output
async def team_register(req: RegisterRequest) -> RegisterResponse:
    if _ctx.client_registry is None:
        raise HTTPException(status_code=503, detail="team context not initialized")
    if req.token != _ctx.join_token:
        raise HTTPException(status_code=401, detail="invalid join token")
    user_name = (req.user_name or "").strip() or None
    # 每请求现取：面板改 allow_anonymous_user 后下一次 connect 即生效，无需重启
    # serve（同 live_manifest_tuning 的理由——admin_config_reload 原地 mutate
    # app._config）。函数内 import：app 模块 import 本模块，模块级会循环。
    from xskill.api import app as app_mod
    from xskill.config import allow_anonymous_user
    if not user_name and not allow_anonymous_user(app_mod._config or {}):  # pylint: disable=protected-access
        raise HTTPException(
            status_code=403, detail="anonymous users not allowed"
        )
    client_id = _ctx.client_registry.register(
        label=req.client_label,
        hostname=req.hostname,
        claimed_client_id=req.claimed_client_id,
        user_name=user_name,
        client_version=req.client_version,
    )
    logger.info("team client registered: %s (label=%s, name=%s)",
                client_id, req.client_label, user_name or "<anonymous>")
    # P2-2.2(Q2a):命名用户发放 dashboard 登录 token(幂等,已有则原样返回)。
    # 匿名用户无 user_name 身份键,dashboard 登录不适用 → None。
    dashboard_token = (
        _ctx.client_registry.ensure_dashboard_token(client_id)
        if user_name else None
    )
    return RegisterResponse(client_id=client_id, dashboard_token=dashboard_token)


@router.post("/upload", response_model=UploadResponse)
async def team_upload(
    req: UploadRequest,
    x_xskill_token: str | None = Header(default=None),
    x_xskill_client: str | None = Header(default=None),
) -> UploadResponse:
    client_id = _auth(x_xskill_token, x_xskill_client)
    watch_state = reconcile_client_ingest_watch_dir(
        client_id, ensure_directory=True,
    )
    sessions_dir = watch_state["sessions_dir"]

    accepted: list[str] = []
    rejected: list[UploadRejection] = []
    for t in req.trajectories:
        if not t.traj_id.startswith("traj_"):
            rejected.append(UploadRejection(traj_id=t.traj_id,
                                            reason="traj_id must start with 'traj_'"))
            continue
        actual = hashlib.sha256(t.content.encode("utf-8")).hexdigest()
        # sha256 不匹配 → 传输损坏，拒收（CLAUDE.md：遇问题 throw，不静默接受）
        if t.sha256 and actual != t.sha256:
            rejected.append(UploadRejection(traj_id=t.traj_id, reason="sha256 mismatch"))
            continue
        # model / harness 非空时先落 .json sidecar，再落 .md：watcher 只 glob
        # traj_*.md，必须保证它发现新 .md 时同名 sidecar 已就位，否则 discover 会
        # INSERT source_model/source_harness=NULL 且永不回读（已存在的行只更 mtime）。
        sidecar = {}
        if t.model:
            sidecar["model"] = t.model
        if t.harness:
            sidecar["harness"] = t.harness
        if sidecar:
            (sessions_dir / f"{t.traj_id}.json").write_text(
                json.dumps(sidecar, ensure_ascii=False), encoding="utf-8")
        # sha256 完整性校验已过（上面），落盘前再做一遍内容清洗：客户端桥接常把
        # 终端 ANSI 码 / 控制字符灌进 .md，会让 splitlines 行号错位、污染模型输入。
        clean = sanitize_trajectory_text(t.content)
        (sessions_dir / f"{t.traj_id}.md").write_text(clean, encoding="utf-8")
        accepted.append(t.traj_id)
    logger.info("team upload from %s: %d accepted, %d rejected",
                client_id, len(accepted), len(rejected))
    return UploadResponse(accepted=accepted, rejected=rejected)


@router.post("/ingest-db")
async def team_ingest_db(
    file: UploadFile = File(...),
    eco: str = Form("ngagent"),
    x_xskill_token: str | None = Header(default=None),
    x_xskill_client: str | None = Header(default=None),
) -> dict:
    """收一个原始 db 文件（ngagent/opencode SQLite），落盘后桥接入库。

    给没装 sshpass / 不愿手敲密码的 Windows 用户用：``upload_ngagent_db.ps1``
    直接 POST db 文件到这里，免 scp。落盘到 ``uploads/<eco>/<client_id>/``，
    再 ``read_db_files`` 桥成 traj 落到该 client 的 sessions 桶（label=client_id
    让 watcher 做 CS 归因），watcher 后续按常规流水线出 skill。
    """
    client_id = _auth(x_xskill_token, x_xskill_client)

    from xskill.config import get_uploads_dir
    from xskill.pipeline.db_ingest import read_db_files
    watch_state = reconcile_client_ingest_watch_dir(
        client_id, ensure_directory=True,
    )
    sessions_dir = watch_state["sessions_dir"]
    dir_name = watch_state["dir_name"]

    # 落盘：uploads/<eco>/<client_id>/<安全文件名>
    safe_name = Path(file.filename or "upload.db").name
    dest_dir = get_uploads_dir() / eco / client_id
    dest_dir.mkdir(parents=True, exist_ok=True)
    dest = dest_dir / safe_name
    dest.write_bytes(await file.read())

    # 桥接到该 client 的 sessions 桶，label=dir_name（与 team_upload 一致）
    try:
        # SQLite 解析 + 批量落盘是阻塞调用，卸到线程池，别占事件循环
        summary = await run_in_threadpool(
            read_db_files,
            dest, eco=eco, target_dir=sessions_dir, register=False,
            register_label=dir_name,
        )
    except (FileNotFoundError, ValueError) as e:
        raise HTTPException(status_code=400, detail=str(e)) from e

    logger.info("team ingest-db from %s: %s → bridged %d traj",
                client_id, safe_name, summary["bridged"])
    return {"client_id": client_id, "saved": str(dest),
            "bridged": summary["bridged"]}


```

```bash
sed -n '606,655p' src/xskill/team/client/daemon.py
```

```output
    def _uninstall_from_ecosystems(self, repo_dir: Path) -> None:
        uninstall_skill_from_ecosystems(
            repo_dir.name, home_root=self.home_root, source_dir=repo_dir,
        )

    # ── 守护循环 ─────────────────────────────────────────────────
    def _tick(self) -> None:
        try:
            self.collect_and_upload()
            manifest = self.apply_client_take(self.sync())
            self.reconcile_skill_sides(manifest)
            self.reconcile_downloaded_skills()
            self.push_user_edits()
            self.cleanup(manifest)
        except Exception as tick_error:
            logger.warning(
                "team client tick failed error_type=%s",
                type(tick_error).__name__,
            )

    def run_forever(self) -> None:
        """阻塞循环。先起 collector ingester，再每 poll_interval 跑一轮 _tick。"""
        from xskill.team.client.updater import AutoUpdater
        updater = AutoUpdater(
            server_url=self.state.server_url,
            client_id=self.state.client_id,
            join_token=self.state.join_token,
            use_proxy=self.use_proxy,
        ) if self.auto_update else None
        if updater:
            updater.start()
        self.collector.start_ingesters()
        logger.info(
            "team client running server_hash=%s client_id_hash=%s",
            hashlib.sha256(
                self.state.server_url.encode("utf-8"),
            ).hexdigest()[:12],
            hashlib.sha256(
                self.state.client_id.encode("utf-8"),
            ).hexdigest()[:12],
        )
        try:
            while not self._stop.is_set():
                self._tick()
                self._stop.wait(self.poll_interval)
        finally:
            if updater:
                updater.stop()
            self.collector.stop_ingesters()

```

## 13. Recommendation determines which Skills a client receives

A team server may contain more distributable Skills than a client should install. The manifest builds a bounded, policy-aware view in this order:

1. remove globally/user-blocked and retired Skills;
2. reserve slots for valid pinned Skills;
3. take the configured number of quality-ranked native Skills, ordered by current-main UX and use count;
4. fill remaining recommendation slots from the client's precomputed interest results;
5. resolve each native Skill to an exact main/staging SHA, including a user's explicit side override;
6. return the server limit plus the user's `take_n`, which lets the client trim the queue and uninstall the tail.

Profiles are derived from that user's atom summaries. A stable revision hash avoids recomputation; unchanged atom/model pairs reuse stored vectors. The remaining vectors are clustered into a small set of interest centers and persisted with used-Skill aggregates and scatter metadata.

Recommendation itself is kept off the `/sync` request path. The periodic heavy worker first refreshes profiles, then reconciles the catalog into Milvus Lite when installed (or an in-memory NumPy index), consumes dirty-user markers, searches each interest center, and saves ordered names in `client_recommend_slots`. `/sync` only reads that table. If no profile/precomputed row exists, the defined cold-start behavior is to continue down the UX-ranked tail, never to perform an expensive all-Skill matrix multiply in the request.

The candidate catalog combines two sources:

- native repositories with a `main` branch, which support Git history and canary sides;
- optional SkillHub directories, which are third-party immutable content identified by content hash and always use side `main`.

Interactive `xskill search` is separate from passive recommendation. SkillHub search builds BM25 and semantic rankings over native plus third-party metadata and fuses them with reciprocal-rank fusion; when query embedding is disabled, busy, or times out, BM25 remains usable. A normal search returns metadata only. Explicit `download` is persistent; legacy `search --download` uses a separate ten-slot LRU.

```bash
sed -n '412,528p' src/xskill/recommend/engine.py
```

```output
    def update_user_interest(
        self,
        client_interest: "ClientInterest",
        task_atom=None,
        *,
        should_commit: Optional[Callable[[], bool]] = None,
    ) -> ProfileUpdateResult:
        """atom 触发：重扫用户 atom 摘要 → 重新聚类 → upsert 画像。

        ``task_atom`` 为触发事件（增量优化预留，当前以 atom store 为单一真源重扫）。
        新鲜度版本持久化到 SQLite；只有完整计算和 upsert 成功后才推进版本。
        """
        del task_atom  # Preserve keyword compatibility; the store is authoritative.
        user_id = client_interest.user_id
        snapshot = sorted(self._user_atoms(user_id), key=attrgetter("atom_id"))
        revision = self._atom_revision(snapshot)
        model = getattr(self.embed_client, "model", "") or ""
        persisted = self.profile_store.get_revision(user_id)
        if (persisted is not None
                and persisted["source_revision"] == revision
                and persisted["embed_model"] == model):
            return ProfileUpdateResult(
                changed=False, embed_items=0, source_revision=revision,
            )

        used_skills = self._used_skills_from_atoms(snapshot)
        atoms = [atom for atom in snapshot if atom.summary]
        if not atoms:
            if should_commit is not None and not should_commit():
                return ProfileUpdateResult(
                    changed=False,
                    embed_items=0,
                    source_revision=revision,
                    cancelled=True,
                )
            self.profile_store.upsert(
                user_id, feature_tensor=None, mean_tensor=None, used_skills=used_skills,
                embed_model=model, source_revision=revision,
            )
            self._publish_profile_cache(
                user_id,
                feature_tensor=None,
                mean_tensor=None,
                used_skills=used_skills,
            )
            return ProfileUpdateResult(
                changed=True, embed_items=0, source_revision=revision,
            )
        if should_commit is not None and not should_commit():
            return ProfileUpdateResult(
                changed=False,
                embed_items=0,
                source_revision=revision,
                cancelled=True,
            )
        vecs, embed_items, reused_items = self._embed_atoms_incremental(user_id, atoms)
        client_interest.reset_points(vecs)
        ft = client_interest.feature_tensor
        mt = client_interest.mean_tensor
        # P3-3.4(Q4):原子点向量 + 逐点元数据随画像顺手落盘,供散点图直读投影
        point_meta = [{"atom_id": a.atom_id, "summary": a.summary,
                       "ux": a.ux_score, "tags": list(a.tags or [])}
                      for a in atoms]
        # stop() 可能在慢 embedding 执行期间发生。后台服务传入的检查必须放在
        # 最终 upsert 之前，避免服务停止后把已经过时的快照写回数据库。
        if should_commit is not None and not should_commit():
            return ProfileUpdateResult(
                changed=False,
                embed_items=embed_items,
                source_revision=revision,
                embed_batches=int(embed_items > 0),
                reused_vector_items=reused_items,
                cancelled=True,
            )
        self.profile_store.upsert(
            user_id, feature_tensor=ft, mean_tensor=mt, used_skills=used_skills,
            points=vecs, point_meta=point_meta,
            embed_model=model, source_revision=revision,
        )
        self._publish_profile_cache(
            user_id,
            feature_tensor=ft,
            mean_tensor=mt,
            used_skills=used_skills,
        )
        return ProfileUpdateResult(
            changed=True, embed_items=embed_items, source_revision=revision,
            embed_batches=int(embed_items > 0),
            reused_vector_items=reused_items,
        )

    def _embed_atoms_incremental(
        self, user_id: str, atoms: list,
    ) -> tuple[np.ndarray, int, int]:
        """只对新增或 summary 变化的原子调 embedding。

        只有 ``atom_id`` 和 ``summary`` 都一致才复用；summary 原地变化时只重算该条。
        换 embedding 模型时 ``load_vector_cache`` 返回空 → 整体重算（护栏在存储层）。
        避免一个攒了上万原子的用户每次新增几条就全量重 embed。
        """
        model = getattr(self.embed_client, "model", "")
        cache = self.profile_store.load_vector_cache_entries(user_id, model)
        missing = [
            atom for atom in atoms
            if atom.atom_id not in cache
            or cache[atom.atom_id]["summary"] != (atom.summary or "")
        ]
        if missing:
            fresh = _normalize_rows(np.asarray(
                self.embed_client.encode_batch([a.summary for a in missing]),
                dtype=float,
            ))
            cache = dict(cache)
            for a, v in zip(missing, fresh):
                cache[a.atom_id] = {"summary": a.summary or "", "vector": v}
        return (
            np.asarray([cache[a.atom_id]["vector"] for a in atoms], dtype=float),
```

```bash
sed -n '534,707p' src/xskill/recommend/engine.py
```

```output
    def get_skill_for_client(
        self, client_user: "ClientUser", skill_num: int,
        *, exclude_names: Optional[set[str]] = None,
        candidate_pool: Optional[list["Skill"]] = None,
        candidate_refs: Optional[dict[str, tuple[str, str | None]]] = None,
        persist_recommendations: bool = True,
        candidate_pool_quality_ordered: bool = False,
    ) -> list["Skill"]:
        """recommended 纯相关性：按兴趣中心轮询（每中心每轮 1 个），多轮填满
        ``skill_num``；冷启动或相关性不足时 UX 序回填；记录推荐 + resolve side。

        ``exclude_names``：从候选池排除的 skill 名（如已占 ranked 槽位的），供
        ``_pick_recommended`` 在 ranked 之外选 recommended 位用。
        ``candidate_pool_quality_ordered`` 表示调用方已按同一质量键排好候选，
        可避免 manifest 热路径重复读取每个 skill 的评分文件（仅用于 UX 回填）。
        """
        source_pool = (
            list(candidate_pool)
            if candidate_pool is not None
            else self._distributable_skills()
        )
        pool = source_pool
        if exclude_names:
            pool = [s for s in pool if s.name not in exclude_names]

        if candidate_pool_quality_ordered:
            quality_ordered = pool
        else:
            quality_keys = {
                s.name: self._quality_key(
                    s,
                    candidate_refs.get(s.name)
                    if candidate_refs is not None else None,
                )
                for s in pool
            }
            # decorate-sort-undecorate：排序键是"拿 skill.name 查表"，itemgetter
            # 表达不了对象→键的映射，故先把键贴成元组首位再排（禁 lambda）。
            # list.sort 稳定且只比较首位，与原 sorted(reverse=True) 的并列次序一致。
            decorated = [(quality_keys[skill.name], skill) for skill in pool]
            decorated.sort(key=itemgetter(0), reverse=True)
            quality_ordered = [skill for _key, skill in decorated]

        relevance: list["Skill"] = []
        picked: set[str] = set()
        ci = client_user.client_interest
        if ci is not None and ci.feature_tensor is not None:
            names, embs, is_hub = self._combined_relevance(source_pool)
            by_name = {s.name: s for s in pool}  # pool 已排除 exclude_names
            centers = list(ci.feature_tensor)
            if len(centers) > 0 and embs.shape[0] > 0:
                while len(relevance) < skill_num:
                    progress = False
                    for center in centers:
                        if len(relevance) >= skill_num:
                            break
                        sims = embs @ np.asarray(center, dtype=float)
                        order = np.argsort(-sims)
                        for i in order:
                            nm = names[i]
                            if exclude_names and nm in exclude_names:
                                continue
                            if nm in picked:
                                continue
                            if is_hub.get(nm):
                                entry = self.skillhub.entry(nm)
                                if entry is None:
                                    continue
                                relevance.append(entry)
                            elif nm in by_name:
                                relevance.append(by_name[nm])
                            else:
                                continue
                            picked.add(nm)
                            progress = True
                            break  # 每中心每轮只取 1 个
                    if not progress:
                        break

        chosen = relevance
        # 回填：冷启动或相关性不足时从 pool（ux 序）补齐至 skill_num
        if len(chosen) < skill_num:
            for s in quality_ordered:
                if len(chosen) >= skill_num:
                    break
                if s.name not in picked:
                    chosen.append(s)
                    picked.add(s.name)

        chosen = chosen[:skill_num]
        # 记录推荐 + resolve side（双向）
        client_user.recommended_skills = []
        recommendation_records: list[tuple[str, str, str]] = []
        for s in chosen:
            if isinstance(s, dict) and s.get("source") == "skillhub":
                side = "main"
                sha = s["content_sha"]
                skill_name = s["skill_id"]
                rec = {
                    "skill": skill_name,
                    "branch": side,
                    "hash": sha,
                    "source": "skillhub",
                    "display_name": s["display_name"],
                    "source_path": s["source_path"],
                }
            else:
                cached = candidate_refs.get(s.name) if candidate_refs is not None else None
                side = self.resolve_side(s, client_user, refs=cached)
                if cached is not None:
                    sha = cached[1] if side == "staging" else cached[0]
                else:
                    sha = staging_sha(s.path) if side == "staging" else (main_sha(s.path) or "")
                skill_name = s.name
                rec = {"skill": skill_name, "branch": side, "hash": sha}
            recommendation_records.append((skill_name, side, sha))
            client_user.recommended_skills.append(rec)
        if persist_recommendations:
            self.reco_store.record_many(
                user_id=client_user.user_id,
                records=recommendation_records,
            )
        return chosen

    # ── 5.4 resolve_side：staging 优先达量 ───────────────────────
    def _side_count(self, skill_dir: Path, side: str, sha: str) -> int:
        from xskill.canary import recent_scores
        return len(recent_scores(skill_dir, side=side, commit_sha=sha, n=self.staging_need + 1))

    def resolve_side(
        self, skill: "Skill", client_user: "ClientUser",
        *, refs: tuple[str, str | None] | None = None,
    ) -> str:
        """staging 优先达量：未达量→staging；staging 达量 main 未达量→main；双侧达量→pick_side。

        双侧达量时用 ``pick_side(user_id, skill_name, probability)`` 做确定性分流
        （main 分支上的既有机制）。

        「最可能用该 skill 的用户」优先：staging 在推荐链路中只被分给**已被推荐该 skill**
        的用户（``get_skill_for_client`` 按用户画像相关性 + ux 质量选出），即被推荐者本身
        就是该 skill 的最可能用户——故 staging 未达量时直接给 staging 即满足 spec D6 的
        「最可能用户优先消费 staging」。跨用户的显式时间序排序留作后续优化。
        """
        if refs is None:
            m_sha = main_sha(skill.path) or ""
            s_sha = staging_sha(skill.path)
        else:
            m_sha, s_sha = refs
        if not s_sha:
            return "main"
        staging_n = self._side_count(skill.path, "staging", s_sha)
        main_n = self._side_count(skill.path, "main", m_sha)
        fallback = pick_side(
            client_user.user_id, skill.name, self.canary_cfg.probability)
        return fill_deficit_side(
            staging_n=staging_n, main_n=main_n,
            need=self.staging_need, fallback=fallback,
        )

    # ── 5.6 find_friend ──────────────────────────────────────────
    def relevance_search(self, query_vec, top_k: int = 5) -> list[tuple[str, bool]]:
        """在合并检索池（可分发 + 三方 skill）做 KNN，返回 ``(name, is_skillhub)``。

        优先读重活进程维护的 Milvus Lite 索引；不可用时退回原 numpy 全库乘。
        """
        milvus_hits = self._relevance_search_milvus(query_vec, top_k=top_k)
        if milvus_hits is not None:
            return milvus_hits
        names, embs, is_hub = self._combined_relevance()
        if embs.shape[0] == 0:
            return []
        sims = embs @ np.asarray(query_vec, dtype=float)
        order = np.argsort(-sims)[:top_k]
        return [(names[i], is_hub.get(names[i], False)) for i in order]
```

```bash
sed -n '175,250p' src/xskill/recommend/heavy_worker.py
```

```output
    """从引擎 embed_client 构造 embed(text)->list[float]；不可用则 None。"""
    client = getattr(engine, "embed_client", None)
    if client is None or not hasattr(client, "encode"):
        return None

    def _embed(text: str) -> list[float]:
        vec = client.encode(text)
        return [float(x) for x in vec]

    return _embed


def run_recommend_heavy_once(
    *,
    engine,
    db_path: Path | None = None,
    vector_db_path: Path | None = None,
    memory_index=None,
    mark_catalog_dirty: bool = True,
) -> dict:
    """对账向量索引并消化推荐脏队列（画像刷新由调用方先跑）。"""
    from xskill.config import XSKILL_HOME, get_registry_db_path
    from xskill.recommend.recommend_store import mark_all_recommend_dirty
    from xskill.recommend.skill_vector_store import (
        DEFAULT_DIM,
        default_vector_db_path,
        fake_embed,
        open_skill_vector_index,
    )

    registry = Path(db_path) if db_path else get_registry_db_path()
    vdb = Path(vector_db_path) if vector_db_path else default_vector_db_path(XSKILL_HOME)
    embed_fn = _embed_fn_from_engine(engine)
    if embed_fn is None:
        embed_fn = lambda text: fake_embed(text, DEFAULT_DIM)  # noqa: E731
        dim = DEFAULT_DIM
    else:
        dim = len(embed_fn("dimension probe"))
    # open_skill_vector_index：无 pymilvus 时退回内存索引并 hourly warn
    index = memory_index or open_skill_vector_index(vdb, dim=dim)
    vec_stats = run_vector_reconcile(
        db_path=registry,
        vector_db_path=vdb,
        embed=embed_fn,
        memory_index=index,
    )
    if mark_catalog_dirty and (
        vec_stats.get("upserted", 0) or vec_stats.get("deleted", 0)
    ):
        mark_all_recommend_dirty(reason="catalog_vector_changed", db_path=registry)
    n = process_dirty_recommends(
        db_path=registry, vector_index=index, engine=engine,
    )
    return {"vector": vec_stats, "recommends": n}
```

```bash
sed -n '310,445p' src/xskill/team/server/skill_manifest.py
```

```output
def build_manifest(
    *,
    client_id: str,
    skill_dir: Path | str,
    probability: float,
    ranked_slots: int = 80,
    total_slots: int = 100,
    traj_root: Path | str | None = None,
    prefs: dict | None = None,
    retired: set | None = None,
    telemetry_submit: Callable[[Callable[[], None]], bool] | None = None,
    user_key: str = "",
    fill_need: int | None = None,
    db_path: Path | str | None = None,
) -> SyncResponse:
    """为 ``client_id`` 现算 manifest。skill 总数不足 total_slots 时全发。

    只分发**已 graduate 到 main 分支**的 skill。``baby`` 分支上的 stub
    （cluster 建了目录但 SkillEditAgent 还没跑过、没正文）没有 main，本来
    就不该下发给 client——这里直接过滤掉，不是 fallback 而是正确的可分发
    集合判定。

    P2-2.4 注入顺序（含 ``prefs``/``retired`` 时）：**blocked 先排除 →
    pinned 占位 → ranked → recommended 回填**。
    - ``prefs`` = ``registry.effective_prefs`` 的合并视图（调用方现查，
      None = 无控制面语义，纯 ranked+recommended）。
    - ``retired`` = 下线 skill 集合，无条件不分发（即便被 pin）。
    - pinned 超量在**写入侧**拒绝（D8）,这里对"pin 的 skill 已不可分发"
      只跳过不报错——sync 路径绝不 500。

    slot 分段：
    - pinned —— prefs 里的钉住 skill（全局在前），占位不参与排名。
    - ranked —— 按 ux 滑窗均分降序取高分（配额 = min(ranked_slots, 余量)）。
    - recommended —— SP3 画像推荐位：基于该 client 用过的 skill 的
      质心，从「distributable 且不在 pinned/ranked、且 client 没用过」的候选
      里取 cosine 最近邻。``traj_root`` 为 None（非 team server 调用）或该
      client 没有任何带 used_skills 的 atom（冷启动、无画像）时，退回 ux
      排序往下接着取——这不是 fallback，是画像不存在时的正确定义。
    """
    skill_dir = Path(skill_dir)
    if total_slots <= 0:
        # 控制面压测和显式禁用分发的部署不需要触碰 skill 仓；否则短 TTL
        # 过期时仍会让一批无槽位请求等待一次无意义的全量扫描。
        return SyncResponse(slots=[], server_time=time.time())
    prefs = prefs or {"pinned": [], "blocked": set()}
    retired = retired or set()
    excluded = set(prefs.get("blocked") or set()) | retired

    catalog = _catalog_cache.get(skill_dir)
    distributable = [s for s in catalog.skills if s.name not in excluded]
    skills = distributable
    by_name = {s.name: s for s in distributable}

    # pinned 占位:不存在/尚不可分发(无 main)/已下线的 pin 跳过——读路径
    # 绝不 throw(D8),写入侧校验已把守常规超量。
    pinned = [by_name[n] for n in (prefs.get("pinned") or []) if n in by_name]
    pinned = pinned[:total_slots]
    pinned_names = {s.name for s in pinned}

    remaining = total_slots - len(pinned)
    non_pinned = [s for s in skills if s.name not in pinned_names]
    ranked = non_pinned[:min(ranked_slots, remaining)]
    taken_names = pinned_names | {s.name for s in ranked}
    reco_slots = max(remaining - len(ranked), 0)

    # exclude 集合并入 blocked/retired:推荐引擎有自己的候选索引(skillhub 等),
    # 不经 distributable 过滤,必须显式排除
    chosen = pinned + ranked + _pick_recommended(
        client_id=client_id,
        skill_dir=skill_dir,
        ranked=ranked,
        ranked_names=taken_names | excluded,
        ux_ordered=non_pinned,
        reco_slots=reco_slots,
        traj_root=traj_root,
        candidate_pool=list(catalog.skills),
        candidate_refs=catalog.refs,
        persist_recommendations=False,
    )

    side_overrides = prefs.get("side") or {}
    need = CanaryConfig().min_samples if fill_need is None else max(int(fill_need), 1)
    origin_db = Path(db_path) if db_path else None
    slots: list[SkillSlot] = []
    for idx, skill in enumerate(chosen):
        if idx < len(pinned):
            bucket = "pinned"
        elif idx < len(pinned) + len(ranked):
            bucket = "ranked"
        else:
            bucket = "recommended"
        slot = _resolve_slot(
            skill, client_id, probability, bucket, refs=catalog.refs,
            user_key=user_key, fill_need=need, db_path=origin_db,
        )
        if slot is not None:
            ov = side_overrides.get(slot.skill_name)
            if ov in ("main", "staging") and not (
                    isinstance(skill, dict) and skill.get("source") == "skillhub"):
                cached_main, cached_staging = (
                    catalog.refs[skill.name] if skill.name in catalog.refs else
                    (main_sha(skill.path) or "", staging_sha(skill.path))
                )
                if ov == "staging" and not cached_staging:
                    ov = "main"
                new_sha = cached_staging if ov == "staging" else cached_main
                if new_sha:
                    slot = SkillSlot(
                        skill_name=slot.skill_name,
                        side=ov,
                        sha=new_sha,
                        bucket=slot.bucket,
                        source=slot.source,
                        display_name=slot.display_name,
                        source_path=slot.source_path,
                    )
            slots.append(slot)
    # 埋点：只记画像推荐位(recommended bucket)。team server 将写入提交给
    # 独立的有界单线程 executor，避免 SQLite 写锁进入 /sync 响应路径；直接
    # 调用 build_manifest 的场景仍同步落盘，保持原有 API 行为。
    records = [
        (s.skill_name, s.side or "main", s.bucket, s.sha or "")
        for s in slots if s.bucket == "recommended"
    ]
    recorder = partial(
        _record_recommendation_telemetry,
        engine=_engine,
        client_id=client_id,
        records=records,
    )
    if telemetry_submit is None:
        recorder()
    elif not telemetry_submit(recorder):
        _logger.debug("recommendation telemetry queue full; event skipped")
    return SyncResponse(slots=slots, server_time=time.time())

```

```bash
sed -n '92,180p' src/xskill/recommend/skillhub.py
```

```output
class SkillHub:
    """三方 skill 扫描器 + ux 查询。``enabled=False``（缺省）时为 no-op。"""

    def __init__(self, *, enabled: bool, hub_dir: Path | str, embed_client,
                 scan_ttl_seconds: float = 3600.0,
                 search_max_embed: int = 2, search_timeout_s: float = 3.0):
        self.enabled = bool(enabled)
        self.dir = Path(hub_dir)
        self.embed_client = embed_client
        self.scan_ttl_seconds = float(scan_ttl_seconds)
        self.search_max_embed = int(search_max_embed)
        self.search_timeout_s = float(search_timeout_s)
        # L3 备忘录：SKILL.md 路径 →
        # (st_mtime_ns, st_size, sha16, display_name | None,
        #  description | None, retry_after)。display_name/description 均为 None
        # 表示该内容版本解码或字段校验失败；按限额定期重验可发现 metadata 未变
        # 的原位修复，sha16 则避免同一坏内容重复刷 warning。
        self._file_memo: dict[
            Path, tuple[int, int, str, str | None, str | None, float]
        ] = {}
        # stat/read 的 OSError 可能是瞬时故障，独立短时限频但不写入内容失败
        # 备忘录；冷却到期自动重试，成功或文件删除后清理。
        self._file_io_retry_after: dict[Path, float] = {}
        # 稳定内容错误的重验使用去重队列轮转；每轮只检查固定数量，既不集中
        # 重读全部坏文件，也不会让目录排序靠后的文件长期得不到重验。
        self._invalid_recheck_queue: deque[Path] = deque()
        self._invalid_recheck_paths: set[Path] = set()
        # L1 快照：require_description=False 的全集（不含 vec），single-flight 保护。
        self._scan_snapshot_entries: list[dict] | None = None
        self._scan_snapshot_expires_at: float = 0.0
        self._scan_lock = threading.Lock()
        # 派生结构随快照身份缓存：_snapshot 内容不变时列表身份不变，故 entry() /
        # fingerprint() 不变盘时零重建、不再每次调用深拷全量 N 条。single-flight
        # 由独立锁保护。strong_index：skill_id / source_path → entry(O(1) 定位)；
        # display_unique：display_name → entry(仅唯一时命中，兜底键)。
        self._derived_for: list[dict] | None = None
        self._strong_index: dict[str, dict] = {}
        self._display_unique: dict[str, dict | None] = {}
        self._fingerprint_cache: tuple[tuple[str, str, str], ...] = ()
        self._derived_lock = threading.Lock()
        # 混合检索索引：随 fingerprint 变化整体重建，在扫描 single-flight 之外自成一锁。
        self._search_index: dict | None = None
        self._search_index_lock = threading.Lock()
        # query embed 四护栏：非阻塞信号量 + 独立短超时线程 + fingerprint 感知 LRU
        # + 后端失败短时冷却。
        self._query_embed_semaphore = threading.Semaphore(max(self.search_max_embed, 0))
        self._query_embed_executor: concurrent.futures.ThreadPoolExecutor | None = None
        self._query_embed_executor_lock = threading.Lock()
        self._query_vector_cache: OrderedDict[str, tuple[tuple, np.ndarray]] = OrderedDict()
        self._query_vector_cache_lock = threading.Lock()
        self._query_embed_inflight: set[str] = set()
        self._corpus_embed_inflight: set[str] = set()
        self._query_embed_retry_after = 0.0
        self._ux_avg_cache: dict[
            tuple[str, int, str | None], tuple[float, str, float | None]
        ] = {}
        self._ux_avg_cache_lock = threading.Lock()

    @classmethod
    def from_config(cls, config: dict, embed_client) -> "SkillHub":
        cfg = skillhub_config(config)
        search_cfg = embedding_search_config(config)
        return cls(
            enabled=cfg["enabled"], hub_dir=cfg["dir"], embed_client=embed_client,
            scan_ttl_seconds=cfg["scan_ttl_seconds"],
            search_max_embed=search_cfg["max_embed"],
            search_timeout_s=search_cfg["search_timeout_s"],
        )

    def _queue_invalid_recheck(self, skill_file: Path) -> None:
        if skill_file not in self._invalid_recheck_paths:
            self._invalid_recheck_paths.add(skill_file)
            self._invalid_recheck_queue.append(skill_file)

    def _read_skill_file(
        self, skill_file: Path, skill_dir: Path, relative_path: str,
        stat_result: os.stat_result, scan_time: float,
        memo: tuple[int, int, str, str | None, str | None, float] | None,
    ) -> bool:
        try:
            raw_bytes = skill_file.read_bytes()
        except OSError as scan_error:
            self._file_io_retry_after[skill_file] = (
                scan_time + SKILL_FILE_IO_RETRY_COOLDOWN_SECONDS
            )
            logger.warning(
                "skillhub scan skipped SKILL.md path=%s error_type=%s",
                f"{relative_path}/SKILL.md" if relative_path != "." else "SKILL.md",
                type(scan_error).__name__,
```

## 14. HTTP APIs and dashboard expose the same durable state

`create_app` mounts three API families:

- core `/api/v1/*` routes for health/status, trajectory submission/search, Skill inspection and lifecycle operations, registry management, and reindexing;
- SSE routes for long-running index/process operations;
- in team-server mode, authenticated `/api/v1/team/*` registration, upload, sync, bundles, SkillHub, import, edit, and generate routes.

Synchronous Git/SQLite control work is moved off the event loop into bounded executors. Long-running agents are not executed inside HTTP handlers: endpoints write jobs or durable inputs, and workers consume them. Watcher and profile status endpoints read heartbeat files because those objects live in child processes, not in the API process.

The dashboard is optional. When enabled it mounts a static HTML/JavaScript shell plus aggregate routes. The built-in form adds signed login, role checks, and admin/user control routes; an independent read-only form physically omits sensitive routes rather than depending on a late authorization check. Public aggregate endpoints show overview, costs, domains, canary state, catalog pages, and pipeline occupancy. Sensitive built-in endpoints may read trajectory/atom content, Git diffs and lineage, user routing, and agent traces.

For hot catalog, UX, candidate, preference, event, and recommendation paths, SQLite projection tables keep request handlers from repeatedly walking the authoring filesystem. Content remains on disk/Git and is read only for views that actually need content. This makes the API a read/control surface over the same state machines, not a competing owner of them.

```bash
sed -n '897,1005p' src/xskill/api/app.py
```

```output
def create_app(home_root: Path | str | None = None,
               *, team_server: bool = False) -> FastAPI:
    """Build the FastAPI app. Calls ``_ensure_loaded`` first so all module-level
    config globals (``_config``/``_skill_dir``/...) are populated before any
    endpoint or startup hook reads them.

    Args:
        home_root: 可选，覆盖生态扫描的 home root。debug 模式下设成自选目录
                   （只扫描该目录下的 ``.claude/``），生产环境留 None 用真
                   实 ``$HOME``。
        team_server: True = team server 模式。挂 /api/v1/team/* 路由、跳过
                   本机生态自动探测（纯 server 不采集自己的轨迹）、watcher
                   开 server_mode。
    """
    global _home_root_override
    _home_root_override = (
        Path(home_root).expanduser().resolve()
        if home_root is not None
        else None
    )
    ecosystem_home_root = _home_root().resolve()
    _ensure_loaded()
    """Create and configure the FastAPI application."""
    app = FastAPI(
        title="xskill",
        description="Trajectory-to-Skill distillation API",
        version=__version__,
    )
    app.include_router(router)

    # team server 模式：挂 /api/v1/team/* 路由
    if team_server:
        from xskill.team.server.api import router as team_router
        app.include_router(team_router)

    # SSE 长耗时接口
    from xskill.api.sse import sse_router
    app.include_router(sse_router)

    # 轨迹提交接口
    from xskill.ecosystems import submit_trajectory
    from pydantic import BaseModel as _BaseModel

    class _SubmitRequest(_BaseModel):
        content: str
        format: str = "markdown"
        metadata: dict | None = None
        traj_id: str | None = None

    @app.post("/api/v1/trajectories/submit")
    async def api_submit_trajectory(req: _SubmitRequest):
        try:
            # traj_dir 不传 → submit_trajectory 落到 get_traj_dir()
            # （第一个已注册的 watch dir）；没注册目录会抛错。
            result = submit_trajectory(
                content=req.content,
                format=req.format,
                metadata=req.metadata or {},
                traj_id=req.traj_id,
            )
            return result
        except Exception as e:
            raise HTTPException(status_code=400, detail=str(e))

    # -- Watcher status endpoint --
    @app.get("/api/v1/watcher/status")
    async def api_watcher_status():
        # watcher 位于常驻子进程，web 进程不持有其内存对象；
        # 读子进程定期落盘的心跳和统计。
        from xskill.config import XSKILL_HOME
        from xskill.utils.status_file import WATCHER_STATUS_FILE, read_status_file
        status = read_status_file(XSKILL_HOME / WATCHER_STATUS_FILE)
        if status is None:
            return {"running": False, "message": "watcher has not started yet"}
        return status

    @app.get("/api/v1/agent-worker/status")
    async def api_agent_worker_status():
        from xskill.config import XSKILL_HOME
        from xskill.utils.status_file import (
            AGENT_WORKER_STATUS_FILE,
            read_status_file,
        )

        status = read_status_file(XSKILL_HOME / AGENT_WORKER_STATUS_FILE)
        if status is None:
            return {
                "running": False,
                "message": "agent-worker has not started yet",
            }
        return status

    # -- Usage / cost stats (Issue #43) --
    @app.get("/api/v1/stats")
    async def api_stats():
        # watcher 读常驻子进程心跳；画像仍读短命子进程的最近一轮状态。
        from xskill.config import XSKILL_HOME
        from xskill.utils.status_file import (
            AGENT_WORKER_STATUS_FILE,
            PROFILE_STATUS_FILE,
            WATCHER_STATUS_FILE,
            read_status_file,
        )
        watcher = read_status_file(XSKILL_HOME / WATCHER_STATUS_FILE)
        agent_worker = read_status_file(
            XSKILL_HOME / AGENT_WORKER_STATUS_FILE,
        )
        profile_refresh = read_status_file(XSKILL_HOME / PROFILE_STATUS_FILE)

```

```bash
rg -n '^@router\.(get|post|delete)|^@sse_router\.(get|post)' src/xskill/api/app.py src/xskill/api/sse.py
```

```output
src/xskill/api/app.py:254:@router.post("/trajectories/search", response_model=TrajectorySearchResponse)
src/xskill/api/app.py:303:@router.get("/trajectories/content")
src/xskill/api/app.py:339:@router.get("/skills", response_model=SkillListResponse)
src/xskill/api/app.py:351:@router.get("/skills/{name}", response_model=SkillDetailResponse)
src/xskill/api/app.py:366:@router.get("/skills/{name}/log", response_model=SkillLogResponse)
src/xskill/api/app.py:381:@router.get("/skills/{name}/diff", response_model=SkillDiffResponse)
src/xskill/api/app.py:396:@router.post("/skills/{name}/rollback", response_model=MessageResponse)
src/xskill/api/app.py:412:@router.post("/skills/{name}/freeze", response_model=MessageResponse)
src/xskill/api/app.py:427:@router.post("/skills/{name}/unfreeze", response_model=MessageResponse)
src/xskill/api/app.py:442:@router.delete("/skills/{name}", response_model=MessageResponse)
src/xskill/api/app.py:457:@router.get("/skills/{name}/export")
src/xskill/api/app.py:484:@router.post("/skills/import", response_model=MessageResponse)
src/xskill/api/app.py:514:@router.post("/skills/search")
src/xskill/api/app.py:540:@router.post("/skills/resolve")
src/xskill/api/app.py:609:@router.get("/skills/{name}/candidates")
src/xskill/api/app.py:621:@router.get("/skills/{name}/canary")
src/xskill/api/app.py:668:@router.get("/canary/overview")
src/xskill/api/app.py:695:@router.get("/registry/dirs")
src/xskill/api/app.py:703:@router.post("/registry/dirs")
src/xskill/api/app.py:718:@router.delete("/registry/dirs")
src/xskill/api/app.py:731:@router.get("/trajectories/logs")
src/xskill/api/app.py:756:@router.get("/trajectories/list")
src/xskill/api/app.py:796:@router.get("/health", response_model=HealthResponse)
src/xskill/api/app.py:802:@router.get("/status", response_model=StatusResponse)
src/xskill/api/app.py:823:@router.post("/init", response_model=MessageResponse)
src/xskill/api/app.py:835:@router.post("/reindex", response_model=MessageResponse)
```

```bash
sed -n '1,65p' src/xskill/dashboard/mount.py
```

```output
"""把看板挂到一个 FastAPI app:include_router + 访问中间件。仅在 enabled 时动。"""
from __future__ import annotations

from pathlib import Path
from typing import Optional

from xskill.config import XSKILL_HOME, dashboard_config
from xskill.dashboard.router import build_dashboard_router
from xskill.dashboard.security import DashboardAccessMiddleware


def _team_registry_provider():
    """登录时解引用 team ctx 的 ClientRegistry（app startup 后才存在）。

    非 team 模式 / ctx 未初始化 → None（普通用户登录不可用，仅 admin 口令）。
    """
    from xskill.team.server.api import team_context
    return getattr(team_context(), "client_registry", None)


def mount_dashboard(app, cfg: dict, *, db_path: Optional[Path] = None,
                    serve_builtin: bool = True) -> None:
    """``serve_builtin=False`` = 独立只读实例（D4）：只挂聚合 GET 端点；
    登录、写操作及内容级敏感路由均物理不挂载。"""
    dc = dashboard_config(cfg)
    if not dc["enabled"]:
        return
    app.include_router(build_dashboard_router(
        db_path=db_path,
        default_harness=dc["default_harness"],
        default_model=dc["default_model"],
        expose_sensitive=serve_builtin))
    if serve_builtin:
        # P2-2.2 登录与角色:仅 serve 内置形态挂载(D4)。
        from xskill.dashboard.auth import (
            build_auth_router, configure_auth, ensure_dashboard_secret,
        )
        configure_auth(
            secret=ensure_dashboard_secret(XSKILL_HOME / "dashboard_secret.json"),
            admins=dc["admins"],
            admin_password=dc["admin_password"],
            registry_provider=_team_registry_provider,
        )
        app.include_router(build_auth_router())
        # P2 控制面(我的/管理):同样只在 serve 内置形态
        from xskill.dashboard.console import build_console_router
        app.include_router(build_console_router(db_path=db_path))
    app.add_middleware(DashboardAccessMiddleware, public=dc["public"],
                       password=dc["password"])
```

```bash
sed -n '35,155p' src/xskill/dashboard/router.py
```

```output
        return Path(db_path).parent / "skill"
    from xskill.config import get_skill_dir
    return get_skill_dir()


def build_dashboard_router(db_path: Optional[Path] = None, *,
                           default_harness: Optional[str] = None,
                           default_model: Optional[str] = None,
                           expose_sensitive: bool = True) -> APIRouter:
    """``expose_sensitive=False`` = 公网只读实例内容白名单（§1.3）：轨迹原文、
    原子详情、用户连接状态、skill 文件/版本/评测 case 等内容级端点和所有写
    端点**物理不注册**（404），只保留聚合数字类 GET 端点。这是给独立只读部署
    （dashboard_standalone）用的闸，不是中间件式拦截——路由根本不存在。
    serve 内置挂载保持默认 True。"""
    # 看板归类口径：缺 source_harness/source_model 的历史轨迹归到哪个桶。
    # 显式传入优先（serve 挂载从 dashboard_config 传）；否则直接读 config.yaml 的
    # dashboard 段（独立只读实例走这条，不需要 api_key）。留空均退 'unknown'。
    if default_harness is None or default_model is None:
        from xskill.config import dashboard_attribution_defaults
        attr = dashboard_attribution_defaults()
        default_harness = default_harness or attr["harness"]
        default_model = default_model or attr["model"]

    router = APIRouter()
    sensitive_router = APIRouter()
    skill_dir = _skill_dir_for(db_path)
    metrics = DashboardMetrics(db_path=db_path, skill_dir=skill_dir,
                               unknown_harness=default_harness,
                               unknown_model=default_model)

    @router.get("/", response_class=HTMLResponse)
    def index() -> str:
        return (_STATIC / "index.html").read_text(encoding="utf-8")

    @router.get("/app.js")
    def appjs() -> Response:
        return Response((_STATIC / "app.js").read_text(encoding="utf-8"),
                        media_type="application/javascript")

    @router.get("/api/v1/dashboard/overview")
    def overview() -> dict:
        return {**metrics.overview(), "price_health": _price_health()}

    @router.get("/api/v1/dashboard/by-domain")
    def by_domain() -> dict:
        return {"by_ecosystem": metrics.by_ecosystem(), "by_model": metrics.by_model()}

    @router.get("/api/v1/dashboard/rates")
    def rates() -> dict:
        """三个需埋点的衍生率:推荐触发率 / 原子采纳率 / canary 晋升率。"""
        return {"trigger": metrics.trigger_rate(),
                "adoption": metrics.adoption_rate(),
                "promotion": metrics.promotion_rate()}

    @router.get("/api/v1/dashboard/cost")
    def cost() -> dict:
        return usage_summary(db_path)

    @router.get("/api/v1/dashboard/models")
    def models() -> dict:
        return {
            "models": model_share(
                db_path,
                unknown_label=default_model,
                exclude_paused_backlog=True,
            ),
            "harnesses": harness_share(
                db_path,
                unknown_label=default_harness,
                exclude_paused_backlog=True,
            ),
        }

    @router.get("/api/v1/dashboard/dirs")
    def dirs() -> dict:
        rows = list_watch_dirs(
            db_path=db_path,
            exclude_paused_backlog=True,
        )
        if not expose_sensitive:
            return {"dirs": [{
                "ecosystem": row.get("ecosystem"),
                "traj_count": row.get("traj_count"),
                "indexed_count": row.get("indexed_count"),
            } for row in rows]}
        return {"dirs": [{"ecosystem": r.get("ecosystem"), "path": r.get("path"),
                          "label": r.get("label"), "traj_count": r.get("traj_count"),
                          "indexed_count": r.get("indexed_count")} for r in rows]}

    @router.get("/api/v1/dashboard/canary")
    def canary() -> dict:
        return {"sides": metrics.canary_sides()}

    @sensitive_router.get("/api/v1/dashboard/users")
    def users() -> dict:
        """团队用户(client)列表 + 总数（纯 registry 分析式）。"""
        u = metrics.users()
        return {"total": len(u), "users": u}

    @router.get("/api/v1/dashboard/tags")
    def tags() -> dict:
        """标签云/关键词（扫原子 tags 聚合，分析式）。"""
        tag_rows = metrics.tag_cloud()
        if not expose_sensitive:
            return {"tags": [{
                "tag": row["tag"],
                "count": row["count"],
            } for row in tag_rows]}
        return {"tags": tag_rows}

    @router.get("/api/v1/dashboard/skills")
    def skills(request: Request, limit: int = 0, offset: int = 0, name: str = "",
               q: str = "") -> dict:
        """skill 库存清单(分析式：读 skill 目录,不依赖埋点)。

        自产 git 技能标 ``source="native"``；skillhub 三方技能（启用时）合入
        并标 ``source="skillhub"`` + ``hub`` + ``skill_id``。skillhub 缺省禁用
        → ``_build_skillhub()`` 返回 None → no-op，列表只有自产技能。

        **分页**(海量 skill,如 1 万条,别让前端一次性拉全量炸锅):``limit``>0 时只返回
        ``skills[offset:offset+limit]`` 这一页;``limit``=0(默认)返回全部,向后兼容。
```

## 15. On-demand commands enter through durable boundaries

Several CLI commands look special but deliberately reuse existing machinery.

### Generate

`xskill generate` posts an instruction and optional user names to the team server, then streams job events over SSE. It does not synthesize a Skill from the instruction alone: the server grants the GenerateAgent read roots containing allowed trajectories, and the instruction directs what to distill or rewrite. The endpoint atomically enqueues a job on disk. The persistent watcher claims it into the same bounded edit pool used by automatic SkillEdit work, so user jobs get priority without an unbounded fifth LLM pool. The GenerateAgent searches/reads evidence, creates or edits repositories, commits directly to main with validated frontmatter, records origin, and pins committed Skills for the initiating user.

### Import versus upload

`import` brings an existing Skill into the native repository and therefore into Git/version/recommendation lifecycle. It preserves or safely replaces appropriate repository state, keeps an active staging comparison where defined, installs the resulting main, and pins team imports for the initiator. `upload` instead publishes a validated directory into the user's third-party SkillHub area; it is searchable and hash-versioned but has no native canary Git lifecycle.

### Read and rebuild

`read <db> --eco opencode|ngagent` passes an arbitrary SQLite file through the existing SQLite bridge, then registers the resulting normalized directory. `rebuild` is not a second pipeline: it deletes derived atom directories and indexes, resets selected trajectory rows to discovery, and lets the running watcher execute the usual state machine. A `COLD_START` file snapshots exactly the reset trajectory IDs. Skill editing is held until that batch reaches terminal states, then one normal threshold scan flushes the accumulated candidates. `--force` additionally removes distilled (not imported) Skills and derived telemetry.

These boundaries make failure recovery straightforward: HTTP/CLI producers finish once durable input exists; workers may restart and replay. Split failures become retryable trajectory rows, cluster failures leave unacknowledged atoms, edit failures retain candidates, and terminal canary/install transitions carry recovery state.

```bash
sed -n '1543,1665p' src/xskill/cli.py
```

```output
def _parse_sse_block(block: str) -> dict | None:
    import json as _json
    for line in block.splitlines():
        if line.startswith("data:"):
            payload = line[5:].strip()
            if not payload:
                continue
            try:
                parsed = _json.loads(payload)
            except _json.JSONDecodeError:
                return None
            if isinstance(parsed, dict):
                return parsed
    return None


def cmd_generate(args, http=None, headers=None) -> int:
    """`xskill generate "指令"` —— 在 team server 上即时生成或改写 skill。"""
    import httpx

    instruction = " ".join(args.instruction).strip()
    if not instruction:
        print("error: instruction 不能为空", file=sys.stderr)
        return 2
    names = [
        part.strip()
        for part in str(getattr(args, "name", "") or "").split(",")
        if part.strip()
    ]
    if http is None:
        http, headers = _team_client_http_and_headers()
        if http is None:
            return 1
    try:
        resp = http.post(
            "/api/v1/team/generate",
            json={"instruction": instruction, "names": names},
            headers=headers,
        )
        if resp.status_code == 404:
            print(
                "error: server 版本过旧，不支持 generate，请管理员先升级 server",
                file=sys.stderr,
            )
            return 1
        if resp.status_code != 200:
            print(
                f"error: generate 提交失败 HTTP {resp.status_code}: "
                f"{resp.text[:300]}",
                file=sys.stderr,
            )
            return 1
        job_id = resp.json().get("job_id")
        if not job_id:
            print("error: server 未返回 job_id", file=sys.stderr)
            return 1
        print(f"generate job {job_id}", flush=True)
        stream_timeout = httpx.Timeout(None)
        with httpx.Client(
            base_url=str(http.base_url),
            timeout=stream_timeout,
            trust_env=False,
        ) as stream_http:
            with stream_http.stream(
                "GET",
                f"/api/v1/team/generate/{job_id}/events",
                headers=headers,
            ) as stream:
                if stream.status_code != 200:
                    print(
                        f"error: 无法读取 generate 日志 HTTP {stream.status_code}",
                        file=sys.stderr,
                    )
                    return 1
                buffer = ""
                final = None
                for text in stream.iter_text():
                    buffer += text
                    while "\n\n" in buffer:
                        block, buffer = buffer.split("\n\n", 1)
                        event = _parse_sse_block(block)
                        if event is None:
                            continue
                        if event.get("type") == "log":
                            chunk = event.get("chunk") or ""
                            if chunk:
                                sys.stdout.write(chunk)
                                sys.stdout.flush()
                        elif event.get("type") == "ping":
                            status = event.get("status") or ""
                            if status == "queued":
                                print("仍在排队，等待席位…", flush=True)
                            else:
                                print("仍在执行…", flush=True)
                        elif event.get("type") == "done":
                            final = event
                if final is None and buffer.strip():
                    final = _parse_sse_block(buffer)
    except (httpx.HTTPError, OSError) as network_error:
        print(
            f"error: 无法连接 team server（{type(network_error).__name__}），"
            "server 可能未响应，请检查网络或联系管理员",
            file=sys.stderr,
        )
        return 1
    if not final:
        print("error: generate 结束但没有收到完成事件", file=sys.stderr)
        return 1
    if not final.get("ok"):
        err = final.get("error") or "generate 失败"
        print(f"error: {err}", file=sys.stderr)
        return 1
    skills = final.get("skill_names") or []
    pinned = final.get("pinned") or []
    if skills:
        print("generate 完成: " + "、".join(skills))
    else:
        print("generate 完成")
    if pinned:
        print("已钉到发起人推荐列表: " + "、".join(pinned))
    return 0


```

```bash
sed -n '58,158p' src/xskill/team/server/generate_jobs.py
```

```output
def create_job(
    *,
    client_id: str,
    user_id: str,
    instruction: str,
    preferred_names: list[str],
    logs_dir: Path,
) -> dict[str, Any]:
    job_id = uuid.uuid4().hex
    log_path = _job_dir(logs_dir, user_id) / f"{job_id}.log"
    log_path.write_text(
        f"generate queued job_id={job_id} user={user_id}\n"
        "waiting for SkillEdit pool seat\n",
        encoding="utf-8",
    )
    job = {
        "job_id": job_id,
        "client_id": client_id,
        "user_id": user_id,
        "instruction": instruction,
        "preferred_names": list(preferred_names),
        "status": "queued",
        "log_path": str(log_path),
        "skill_names": [],
        "pinned": [],
        "error": "",
        "created_at": time.time(),
    }
    with _JOBS_LOCK:
        _JOBS[job_id] = job
    _write_status(job)
    return dict(job)


def enqueue_generate_job(job: dict[str, Any], *, logs_dir: Path) -> None:
    """把任务写进 pending，供 agent-worker 的 edit 池领取。"""
    payload = {
        "job_id": job["job_id"],
        "client_id": job["client_id"],
        "user_id": job["user_id"],
        "instruction": job["instruction"],
        "preferred_names": list(job.get("preferred_names") or []),
        "log_path": job["log_path"],
        "created_at": job.get("created_at") or time.time(),
        "status": "queued",
    }
    path = _pending_dir(logs_dir) / f"{job['job_id']}.json"
    _atomic_write_json(path, payload)


def list_pending_paths(logs_dir: Path) -> list[Path]:
    pending = jobs_root(logs_dir) / "pending"
    if not pending.is_dir():
        return []
    return sorted(
        (p for p in pending.glob("*.json") if p.is_file()),
        key=lambda p: p.stat().st_mtime,
    )


def try_claim(logs_dir: Path, pending_path: Path) -> dict[str, Any] | None:
    claimed = _claimed_dir(logs_dir) / pending_path.name
    try:
        pending_path.replace(claimed)
    except FileNotFoundError:
        return None
    except OSError:
        logger.warning("generate claim failed for %s", pending_path, exc_info=True)
        return None
    try:
        return json.loads(claimed.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError):
        logger.warning("generate claimed file unreadable %s", claimed, exc_info=True)
        return None


def release_claim(logs_dir: Path, job_id: str) -> None:
    claimed = _claimed_dir(logs_dir) / f"{job_id}.json"
    if not claimed.is_file():
        return
    pending = _pending_dir(logs_dir) / claimed.name
    try:
        claimed.replace(pending)
    except OSError:
        logger.warning("generate release claim failed for %s", job_id, exc_info=True)


def finish_claim(logs_dir: Path, job_id: str) -> None:
    claimed = _claimed_dir(logs_dir) / f"{job_id}.json"
    try:
        claimed.unlink(missing_ok=True)
    except TypeError:
        # Python 3.9: Path.unlink 无 missing_ok
        try:
            claimed.unlink()
        except FileNotFoundError:
            pass
    except OSError:
        logger.warning("generate finish claim failed for %s", job_id, exc_info=True)


```

```bash
sed -n '420,472p' src/xskill/pipeline/runner.py
```

```output
    def _submit_generate_jobs(self):
        """把 web 进程入队的 generate 任务提交到 SkillEdit 同一线程池。

        不另开池：占 edit 席位、走 edit 的 llm_weight。pending 文件在
        ``<home>/generate_jobs/pending/``，与 logs 同级，web 与 worker 都能见。
        """
        if self.skill_dir is None or not self.skill_dir.is_dir():
            return
        from xskill.team.server import generate_jobs as gen_jobs

        inflight = {
            info.get("job_id")
            for info in self._futures.values()
            if info.get("stage") == "generate" and info.get("job_id")
        }
        gen_jobs.reclaim_orphans(self.logs_dir, inflight)
        for pending_path in gen_jobs.list_pending_paths(self.logs_dir):
            job = gen_jobs.try_claim(self.logs_dir, pending_path)
            if job is None:
                continue
            job_id = job.get("job_id") or pending_path.stem
            if job_id in inflight:
                gen_jobs.release_claim(self.logs_dir, job_id)
                continue

            def _run(claimed=job):
                traj_root = None
                if self.server_mode:
                    from xskill.config import get_team_trajectories_dir
                    try:
                        traj_root = get_team_trajectories_dir()
                    except Exception:
                        logger.debug(
                            "generate traj_root unavailable", exc_info=True,
                        )
                gen_jobs.run_claimed_generate_job(
                    claimed,
                    skill_dir=self.skill_dir,
                    config=self.config,
                    db_path=self.db_path,
                    logs_dir=self.logs_dir,
                    traj_root=traj_root,
                )

            fut = self._pools["edit"].submit(
                _run,
                task=gen_jobs.monitor_task(job),
            )
            if fut is None:
                gen_jobs.release_claim(self.logs_dir, job_id)
                break
            self._futures[fut] = {"stage": "generate", "job_id": job_id}

```

```bash
sed -n '1845,1957p' src/xskill/cli.py
```

```output
def cmd_read(args, xskill) -> int:
    """`xskill read <PATH> --eco ngagent` —— 批量把 db 文件桥接入库。"""
    del xskill  # CLI handler signature compatibility.
    from xskill.pipeline.db_ingest import read_db_files
    try:
        summary = read_db_files(
            args.path,
            eco=args.eco,
            register=not args.no_register,
            recursive=args.recursive,
        )
    except (FileNotFoundError, ValueError) as e:
        print(f"error: {e}", file=sys.stderr)
        return 2
    print(
        f"read: {len(summary['db_files'])} db 文件 → 桥接 {summary['bridged']} "
        f"条轨迹到 {summary['target_dir']}"
    )
    if not args.no_register:
        print("已注册为 watch_dir —— 启动 `xskill serve` 后将自动拆分入库。")
    return 0


def cmd_rebuild(args, _xskill) -> int:
    """`xskill rebuild [--force]` —— 用现有原始轨迹重跑蒸馏。

    默认：删除已拆 atom + index.pkl、轨迹状态翻回 discovered，让运行中的 watcher
    从头重拆重聚（删 atom 是真正触发重拆的动作——splitter 续接点取自 atom 文件，
    不读 DB offset）。``--force``：额外先清空蒸馏所得 skill（``xskill import``
    纳入的留下）、看板派生埋点和安装历史。

    换模型护栏：rebuild 的重跑是交给**正在运行的 daemon**，而 daemon 的模型是
    启动时缓存的（改 config 不重启不生效）。若 daemon 在跑且其模型 ≠ 当前 config
    模型 → 默认拒绝并提示先重启 serve，否则会静默用旧模型重生成（`--ignore-
    model-mismatch` 可强行用当前运行的模型重跑）。
    """
    from xskill.config import XSKILL_HOME
    from xskill.pipeline.registry import reset_trajectories
    from xskill.runtime import config_models, read_status

    # ── 换模型护栏（先于任何清仓/重置）──
    status = read_status()
    if status.get("running") and not args.ignore_model_mismatch:
        daemon_model = status.get("llm_model")
        config_model = config_models().get("llm_model")
        if daemon_model != config_model:
            print(
                f"✗ 运行中的 daemon 在用模型 {daemon_model!r}，但 config.yaml "
                f"现在是 {config_model!r}。",
                file=sys.stderr,
            )
            print(
                "  daemon 的模型是启动时缓存的——直接 rebuild 会用旧模型重生成。\n"
                "  换模型请先干净重启：停掉 serve（确认进程真退了）→ 重新 "
                "`xskill serve` → 再 rebuild。\n"
                "  确认就是要用当前运行的模型重跑，可加 --ignore-model-mismatch。",
                file=sys.stderr,
            )
            return 2

    if args.force:
        from xskill.config import get_registry_db_path, get_skill_dir
        from xskill.pipeline.registry import clear_rebuild_derived_state
        from xskill.skill.repo import SkillRepo
        skill_count, kept_names = SkillRepo(get_skill_dir()).wipe_all_skills(
            db_path=get_registry_db_path(),
        )
        print(f"--force: 清空蒸馏 skill（删 {skill_count} 个）")
        if kept_names:
            print(
                "--force: 保留 "
                f"{len(kept_names)} 个 import 纳入的技能"
            )
        deleted_counts = clear_rebuild_derived_state()
        print(
            "--force: 清空看板派生数据（"
            f"recommendation_log={deleted_counts['recommendation_log']}, "
            f"atom_adoption={deleted_counts['atom_adoption']}, "
            f"canary_decision={deleted_counts['canary_decision']}, "
            f"skill_trigger_eval={deleted_counts['skill_trigger_eval']}）"
        )
        install_history_path = XSKILL_HOME / "install_history.jsonl"
        if install_history_path.is_file():
            install_history_path.unlink()
            print("--force: 删除安装历史 install_history.jsonl")
        else:
            print("--force: 安装历史为空")

    reset_trajectory_ids = reset_trajectories(eco=args.eco, traj_id=args.traj)
    print(
        f"rebuild: 重置 {len(reset_trajectory_ids)} 条轨迹"
        "（已删 atom + index.pkl，将从头重拆）"
    )

    from xskill.pipeline.cold_start import ColdStartSignal
    cold_start_signal = ColdStartSignal(XSKILL_HOME)
    cold_start_signal.create(reset_trajectory_ids)
    print(
        "cold-start: 已写入本批轨迹快照信号，watcher 会在这批轨迹处理完成后 flush "
        f"({cold_start_signal.file_path})"
    )

    if read_status().get("running"):
        print("watcher 运行中 —— 30s 内将自动重跑这些轨迹。")
    else:
        print("⚠ 未检测到运行中的 daemon —— 请 `xskill serve` 启动后才会重跑。")
    return 0


# ═══════════════════════════════════════════════════════════════
# argparse
# ═══════════════════════════════════════════════════════════════

```

```bash
sed -n '1,63p' src/xskill/pipeline/cold_start.py
```

```output
"""冷启动一次性批量 flush 信号。

冷启动不是可配置的线上状态机，也没有多轮概念。``xskill rebuild`` 把本批被
重置的轨迹 id 快照写进当前 XSkill 实例的 ``COLD_START``，watcher 只等这批轨迹全部
到达终态（离开 pending 状态）就按既有 ``ATOM_PROMOTION_THRESHOLD`` 做一次
SkillEdit 扫描并删除该文件——rebuild 之后新进的轨迹不延长等待。
"""
from __future__ import annotations

import json
import os
import time
from dataclasses import dataclass
from pathlib import Path


COLD_START_FILENAME = "COLD_START"

# 快照内个别轨迹卡死时的安全网：超过该时长强制 flush，不再 hold。
COLD_START_MAX_HOLD_SECONDS = 24 * 3600


@dataclass(frozen=True)
class ColdStartSignal:
    """管理一次 cold-start flush 的文件信号。"""

    xskill_home: Path

    @property
    def file_path(self) -> Path:
        return self.xskill_home / COLD_START_FILENAME

    @property
    def exists(self) -> bool:
        return self.file_path.exists()

    def create(self, trajectory_ids: list[int]) -> dict:
        self.file_path.parent.mkdir(parents=True, exist_ok=True)
        payload = {
            "trajectory_ids": list(trajectory_ids),
            "created_at": time.time(),
        }
        # 原子写：CLI 与 watcher 补录跨进程双写，不能让对方读到半截 JSON。
        temp_path = self.file_path.with_suffix(".tmp")
        temp_path.write_text(json.dumps(payload), encoding="utf-8")
        os.replace(temp_path, self.file_path)
        return payload

    def snapshot(self) -> dict | None:
        """读快照。≤0.6.11 的空 touch 文件/坏 JSON 返回 None，调用方补录。"""
        try:
            payload = json.loads(self.file_path.read_text(encoding="utf-8"))
        except (OSError, ValueError):
            return None
        if not isinstance(payload, dict):
            return None
        if not isinstance(payload.get("trajectory_ids"), list):
            return None
        return payload

    def consume(self) -> None:
        if self.exists:
            self.file_path.unlink()
```

## 16. The complete runtime story

Putting the pieces back into one sequence:

1. The CLI validates configuration and starts FastAPI.
2. FastAPI validates model clients, initializes optional team/recommendation context, and supervises worker subprocesses.
3. Ecosystem ingesters wait for stable native sessions, normalize them to `traj_*.md` plus sidecars, and write into registered bridge directories.
4. Registry discovery inserts or updates a durable trajectory row.
5. The split pool asks TaskAgent for contiguous intent atoms and atomically stores them.
6. The embed pool updates the atom vector index and marks the trajectory indexed.
7. The watcher pools all unclustered atoms; TaskClusterAgent routes each into one or more per-Skill candidate buffers.
8. Recorded assignments set the atom's durable `clustered` acknowledgement; once every atom is acknowledged, the trajectory becomes done.
9. A buffer reaching weight ten schedules SkillEditAgent.
10. A baby repository drains checkpoints and graduates to main; later edits become staging candidates only after real main use.
11. Installed Skills generate future session evidence. Atom UX observations accumulate against exact branch SHAs.
12. Canary decisions merge a non-worse staging branch or preserve-and-reject a worse/expired one.
13. In standalone mode the chosen repository is projected directly into local harnesses. In team mode the server builds a personalized immutable manifest, and thin clients reconcile it locally.
14. SQLite projections, event telemetry, API endpoints, and the dashboard explain and control the same durable flow.
15. Any crash resumes from the last durable boundary rather than reconstructing an in-memory workflow.

The smallest useful mental model is therefore:

`session → normalized trajectory → atom → candidate evidence → Git Skill version → real-use score → rollout decision → installed Skill → next session`

### Source map for future changes

| Concern | Start here | Shared boundary to preserve |
|---|---|---|
| add a harness | `ecosystems/<name>.py`, `ecosystems/_shared.py` | normalized trajectory format |
| change scheduling | `pipeline/runner.py`, `pipeline/registry.py` | durable status/atom acknowledgement |
| change atom semantics | `agents/task_agent.py`, `pipeline/atom.py` | contiguous line coverage |
| change routing | `agents/task_cluster_agent.py`, `agents/agent_tools.py` | candidate file + recorded assignment |
| change Skill authoring | `agents/skill_edit_agent.py`, `skill/git.py` | per-Skill branch state machine |
| change rollout | `canary.py` | scores bound to exact SHAs |
| change team transport | `team/client/*`, `team/server/*` | same server-side pipeline |
| change recommendation | `recommend/*`, `team/server/skill_manifest.py` | precompute heavy work; sync reads |
| change dashboard | `dashboard/*`, catalog/UX projections | DB reads for business lists/metrics |

That is how the package fits together end to end; the remaining modules chiefly implement platform-specific parsing, defensive filesystem/Git handling, retrieval details, authentication, logging, and compatibility around these boundaries.

```bash
rg -n '^class (XSkill|DirectoryWatcher|TaskAgent|TaskClusterAgent|SkillEditAgent|SkillRecommendEngine|TeamClient|TeamCollector|SkillHub|AtomCanary)' src/xskill | sort
```

```output
src/xskill/agents/skill_edit_agent.py:368:class SkillEditAgent:
src/xskill/agents/task_agent.py:312:class TaskAgent:
src/xskill/agents/task_cluster_agent.py:254:class TaskClusterAgent:
src/xskill/canary.py:1150:class AtomCanary:
src/xskill/core.py:23:class XSkill:
src/xskill/pipeline/runner.py:84:class DirectoryWatcher:
src/xskill/recommend/engine.py:64:class SkillRecommendEngine:
src/xskill/recommend/skillhub.py:92:class SkillHub:
src/xskill/team/client/collector.py:83:class TeamCollector:
src/xskill/team/client/daemon.py:100:class TeamClient:
```
