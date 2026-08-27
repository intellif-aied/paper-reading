# SWE-bench 后续 xskill / Claude Code 实验候选

> 结论：首轮建议按 `pytest-dev__pytest-7220` → `pallets__flask-4992` 运行；若要加入第三种任务形态，再加入 `mwaskom__seaborn-3010`。三者分别覆盖“环境状态导致的根因 bug”“边界清晰的兼容性功能”“数值管线的缺失值健壮性”。本文只做一手资料静态核验，没有启动 agent、容器或测试。

## 证据口径与防泄漏

- 候选来自 SWE-bench 官方 `SWE-bench_Lite` test split。本文固定数据版本为 Hugging Face 数据仓库提交 [`b0dde1093fe417d83b7184254edf8199c1f0dff5`](https://huggingface.co/datasets/SWE-bench/SWE-bench_Lite/tree/b0dde1093fe417d83b7184254edf8199c1f0dff5)，文件标识为 `data/test-00000-of-00001.parquet[instance_id=…]`。官方数据说明列出了 `base_commit`、问题文本、gold `patch`、`test_patch`、`FAIL_TO_PASS` 与 `PASS_TO_PASS` 等字段，并明确警告解题时不要查看 gold patch。[SWE-bench dataset schema](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/docs/guides/datasets.md#L41-L60)
- 本文中的“测试确定性”是静态判断：官方 test patch 的断言只依赖本地文件、进程状态或内存数据，官方 `eval_script` 也只运行一个测试模块；**没有实际执行验证**。正式实验仍应以官方镜像内的 harness 结果为准。
- 这是一份 evaluator-side 文档。不要把本文、gold `patch`、`test_patch`、测试名或历史 PR 放进 Claude Code 的上下文，也不要把本文放进目标 repo checkout。解题 agent 只应看到原始 issue/problem statement 与 `base_commit` 工作树；结束后由隔离的 evaluator 应用 test patch 并评分。
- 官方评分把 F2P 与 P2P 分开计算，只有两者比例都为 1 才是 `FULL` resolved；因此实验至少记录 `F2P passed / total`、`P2P passed / total`，不能只记目标测试。[grading implementation](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/swebench/harness/grading.py#L234-L258) [resolution criteria](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/swebench/harness/grading.py#L288-L315)

## 候选总览

| 优先级 | instance_id | 任务形态 | 官方测试范围 | 环境负担 | 适合观察的学习信号 |
|---|---|---|---:|---|---|
| 1 | `pytest-dev__pytest-7220` | cwd 改变后错误路径失真；根因定位 | 1 F2P + 11 P2P，单模块 | 低 | 是否形成“可变进程状态 vs 稳定 invocation 基准”的 Atom |
| 2 | `pallets__flask-4992` | 为 loader 增加文本/二进制模式；小型 API 功能 | 1 F2P + 18 P2P，单模块 | 低；必须是 Python 3.11+ | 是否形成“调用方协议差异应在边界显式建模”的 Candidate |
| 3 | `mwaskom__seaborn-3010` | PolyFit 遇到缺失值崩溃；数值健壮性 | 1 F2P + 2 P2P，单模块 | 中等，科学计算依赖较重 | 是否形成“先清洗、再按组变换、用等价性断言”的 Candidate |

表中测试数、镜像、安装命令均来自下列各实例的官方数据行；“学习信号”是本实验的研究假设，不是 SWE-bench 官方结论。

## 1. `pytest-dev__pytest-7220`（首选：根因型 bug）

### 身份与问题

- **repo / base commit**：`pytest-dev/pytest` @ [`56bf819c2f4eaf8b36bd8c42c06bb59d5a3bfc0f`](https://github.com/pytest-dev/pytest/tree/56bf819c2f4eaf8b36bd8c42c06bb59d5a3bfc0f)。`environment_setup_commit` 为 `678c1a0745f1cf175c442c719906a1f13e496910`，数据版本标签为 `5.4`。[official task row, row 177](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=177&length=1)
- **原始问题**：fixture 把 cwd 从启动目录切到子目录后，失败报告按新 cwd 输出 `../test_path_error.py`，编辑器无法从原启动目录正确跳转；issue 给出了最小复现和 expected/displayed 对比。[issue #6428](https://github.com/pytest-dev/pytest/issues/6428)
- **基线根因位置**：`Node._repr_failure_py` 只调用 `os.getcwd()` 判断 cwd 是否存在；只要调用成功就设 `abspath=False`，没有比较当前 cwd 与 `config.invocation_dir`。[base source, `src/_pytest/nodes.py` L333-L372](https://github.com/pytest-dev/pytest/blob/56bf819c2f4eaf8b36bd8c42c06bb59d5a3bfc0f/src/_pytest/nodes.py#L333-L372)
- **历史修复关联**：PR #7220 的标题是 “Use abspath for errors when cwd changes during testing”，正文写明 `Fixes #6428`。[PR #7220](https://github.com/pytest-dev/pytest/pull/7220)

### 官方 patch / tests metadata

- Gold patch 只改 `src/_pytest/nodes.py`：比较 `Path(os.getcwd())` 与 `Path(self.config.invocation_dir)`，cwd 改变时请求绝对路径，同时保留 `getcwd()` 抛 `OSError` 时的绝对路径回退。Test patch 只向 `testing/test_nodes.py` 加一个 pytester 回归测试，在 fixture 中 `chdir` 并匹配失败输出中的绝对路径。[official task row, `patch` and `test_patch`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=177&length=1)
- **FAIL_TO_PASS（1）**：`testing/test_nodes.py::test_failure_with_changed_cwd`。
- **PASS_TO_PASS（11）**：
  - `testing/test_nodes.py::test_ischildnode[--True]`
  - `testing/test_nodes.py::test_ischildnode[-foo-True]`
  - `testing/test_nodes.py::test_ischildnode[-foo/bar-True]`
  - `testing/test_nodes.py::test_ischildnode[-foo/bar::TestBaz-True]`
  - `testing/test_nodes.py::test_ischildnode[foo-food-False]`
  - `testing/test_nodes.py::test_ischildnode[foo/bar::TestBaz-foo/bar-False]`
  - `testing/test_nodes.py::test_ischildnode[foo/bar::TestBaz-foo/bar::TestBop-False]`
  - `testing/test_nodes.py::test_ischildnode[foo/bar-foo/bar::TestBop-True]`
  - `testing/test_nodes.py::test_node_from_parent_disallowed_arguments`
  - `testing/test_nodes.py::test__check_initialpaths_for_relpath`
  - `testing/test_nodes.py::test_std_warn_not_pytestwarning`

上述测试清单逐项来自固定版本官方数据行的 `FAIL_TO_PASS` / `PASS_TO_PASS` 字段。[official task row](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=177&length=1)

### 负担、实验价值与 caveat

- 官方镜像为 `swebench/sweb.eval.x86_64.pytest-dev_1776_pytest-7220:latest`；`eval_script` 激活预置 `testbed` conda 环境，执行 `python -m pip install -e .`，应用 test patch 后仅运行 `pytest -rA testing/test_nodes.py`。测试代码只创建本地临时文件/目录并启动嵌套 pytest，不需要外部服务或网络。[official task row, `image` and `eval_script`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=177&length=1)
- 这是最适合观察 Atom 的候选：表面是“路径打印错”，真正决策点是把 mutable cwd 与 stable invocation directory 分开；轨迹中应能观察到复现、定位格式化层、保留 `OSError` 回退、最小回归测试四段知识是否被提取并在后续 Candidate 中抽象。
- **兼容 caveat**：这是 pytest 5.4 时代的提交。若脱离官方镜像在最新 Python/依赖上安装，兼容性噪声可能盖过任务本身；应预拉官方镜像并固定环境，解题阶段断网。
- **许可**：该基线提交的 `LICENSE` 是 MIT License，实验产生或分发的 pytest 派生代码需遵守其版权与许可声明。[pytest license at base commit](https://github.com/pytest-dev/pytest/blob/56bf819c2f4eaf8b36bd8c42c06bb59d5a3bfc0f/LICENSE)

## 2. `pallets__flask-4992`（次选：边界清晰的功能）

### 身份与问题

- **repo / base commit**：`pallets/flask` @ [`4c288bc97ea371817199908d0d9b12de9dae327e`](https://github.com/pallets/flask/tree/4c288bc97ea371817199908d0d9b12de9dae327e)。`environment_setup_commit` 为 `182ce3dd15dfa3537391c3efaf9c3ff407d134d4`，数据版本标签为 `2.3`。[official task row, row 148](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=148&length=1)
- **原始问题**：Python 3.11 的 `tomllib.load` 要求 binary-mode 文件，而 `Config.from_file()` 总以文本模式打开；issue 提议让 API 暴露文件模式，从而避免调用方手工重复 root-path 拼接与 `from_mapping()`。[issue #4989](https://github.com/pallets/flask/issues/4989)
- **基线代码**：`from_file(filename, load, silent=False)` 在 `open(filename)` 后直接把 handle 交给 loader，因此 loader 的文本/二进制协议无法表达。[base source, `src/flask/config.py` L232-L273](https://github.com/pallets/flask/blob/4c288bc97ea371817199908d0d9b12de9dae327e/src/flask/config.py#L232-L273)
- **历史修复关联**：PR #4992 正文写明 `fixes #4989`。[PR #4992](https://github.com/pallets/flask/pull/4992)

### 官方 patch / tests metadata

- Gold patch 只改 `src/flask/config.py`：增加默认值为 `True` 的 `text` 参数，用 `"r" if text else "rb"` 选择 open mode，并同步文档示例。Test patch 新增 `tests/static/config.toml`，把原 JSON 测试更名，并新增 TOML 回归测试。[official task row, `patch` and `test_patch`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=148&length=1)
- **FAIL_TO_PASS（1）**：`tests/test_config.py::test_config_from_file_toml`。
- **PASS_TO_PASS（18）**：
  - `tests/test_config.py::test_config_from_pyfile`
  - `tests/test_config.py::test_config_from_object`
  - `tests/test_config.py::test_config_from_file_json`
  - `tests/test_config.py::test_from_prefixed_env`
  - `tests/test_config.py::test_from_prefixed_env_custom_prefix`
  - `tests/test_config.py::test_from_prefixed_env_nested`
  - `tests/test_config.py::test_config_from_mapping`
  - `tests/test_config.py::test_config_from_class`
  - `tests/test_config.py::test_config_from_envvar`
  - `tests/test_config.py::test_config_from_envvar_missing`
  - `tests/test_config.py::test_config_missing`
  - `tests/test_config.py::test_config_missing_file`
  - `tests/test_config.py::test_custom_config_class`
  - `tests/test_config.py::test_session_lifetime`
  - `tests/test_config.py::test_get_namespace`
  - `tests/test_config.py::test_from_pyfile_weird_encoding[utf-8]`
  - `tests/test_config.py::test_from_pyfile_weird_encoding[iso-8859-15]`
  - `tests/test_config.py::test_from_pyfile_weird_encoding[latin-1]`

上述测试清单逐项来自固定版本官方数据行。[official task row](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=148&length=1)

### 负担、实验价值与 caveat

- 官方镜像为 `swebench/sweb.eval.x86_64.pallets_1776_flask-4992:latest`；`eval_script` 激活 `testbed`，执行 `python -m pip install -e .`，应用 test patch 后仅运行 `pytest -rA tests/test_config.py`。测试只读仓库内 config fixture，不依赖外部服务或网络。[official task row, `image` and `eval_script`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=148&length=1)
- 它适合检验 Candidate 的可复用性：知识不是“TOML 特判”，而是“高阶 loader 的输入协议不同，应在公共边界以向后兼容参数表达，并同时保住 JSON/编码回归面”。可检查 xskill 是否把一次局部修改提炼成跨 parser/wrapper API 可召回的原则。
- **版本 caveat**：test patch 用 `pytest.importorskip("tomllib", reason="tomllib added in 3.11")`；在 Python 3.10 以下目标 F2P 会被跳过，因而不能把本地绿色误当作任务解决。必须用官方镜像或明确固定 Python 3.11+。[official test patch](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=148&length=1)
- **许可**：基线 `LICENSE.rst` 是三条款 BSD 文本；分发派生代码时保留其版权、条件与免责声明。[Flask license at base commit](https://github.com/pallets/flask/blob/4c288bc97ea371817199908d0d9b12de9dae327e/LICENSE.rst)

## 3. `mwaskom__seaborn-3010`（可选：数据/数值健壮性）

### 身份与问题

- **repo / base commit**：`mwaskom/seaborn` @ [`0f5a013e2cf43562deec3b879458e59a73853813`](https://github.com/mwaskom/seaborn/tree/0f5a013e2cf43562deec3b879458e59a73853813)。`environment_setup_commit` 为 `d25872b0fc99dbf7e666a91f59bd4ed125186aa1`，数据版本标签为 `0.12`。[official task row, row 144](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=144&length=1)
- **原始问题**：`so.Plot(... None ...).add(so.Line(), so.PolyFit())` 在 `np.polyfit` / `lstsq` 内以 `LinAlgError: SVD did not converge` 崩溃；issue 提供了最小输入与完整调用栈。[issue #2992](https://github.com/mwaskom/seaborn/issues/2992)
- **基线代码**：`PolyFit._fit_predict` 直接把 `data["x"]` / `data["y"]` 交给 `np.polyfit`，而 `__call__` 直接对未经清洗的 data 做 `groupby.apply`。[base source, `seaborn/_stats/regression.py` L22-L41](https://github.com/mwaskom/seaborn/blob/0f5a013e2cf43562deec3b879458e59a73853813/seaborn/_stats/regression.py#L22-L41)
- **历史修复关联**：PR #3010 标题是 “Make PolyFit stat robust to missing data”，正文写明 `Fixes #2992`。[PR #3010](https://github.com/mwaskom/seaborn/pull/3010)

### 官方 patch / tests metadata

- Gold patch 只改 `seaborn/_stats/regression.py`：在 grouped apply 前对 `x`、`y` 子集 `dropna`。Test patch 只向 `tests/_stats/test_regression.py` 加 `test_missing_data`，并用 `pandas.testing.assert_frame_equal` 验证“含 NaN 输入的结果”等于“显式 dropna 后的结果”。[official task row, `patch` and `test_patch`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=144&length=1)
- **FAIL_TO_PASS（1）**：`tests/_stats/test_regression.py::TestPolyFit::test_missing_data`。
- **PASS_TO_PASS（2）**：
  - `tests/_stats/test_regression.py::TestPolyFit::test_no_grouper`
  - `tests/_stats/test_regression.py::TestPolyFit::test_one_grouper`

上述测试清单逐项来自固定版本官方数据行。[official task row](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=144&length=1)

### 负担、实验价值与 caveat

- 官方镜像为 `swebench/sweb.eval.x86_64.mwaskom_1776_seaborn-3010:latest`；`eval_script` 激活 `testbed`，执行 `python -m pip install -e .[dev]`，应用 test patch 后仅运行 `pytest --no-header -rA tests/_stats/test_regression.py`。单模块测试本身使用内存 DataFrame、NumPy 与 pandas，不依赖外部服务或网络。[official task row, `image`, `eval_script`, `test_patch`](https://datasets-server.huggingface.co/rows?dataset=SWE-bench%2FSWE-bench_Lite&config=default&split=test&offset=144&length=1)
- 它适合观察另一类 Candidate：从下游线性代数异常反推上游数据契约，把清洗放在分组变换边界，并用行为等价性而非具体系数做 oracle。若前两题形成的知识过度绑定 Web/测试框架，这题能检验抽象是否仍可迁移。
- **环境 caveat**：`.[dev]` 会牵涉 NumPy、pandas、matplotlib 等科学计算/开发依赖，冷安装成本明显高于前两题。为减少时间与网络噪声，应在计时前预拉官方镜像，解题阶段禁网；不要在会重新解析最新依赖的裸环境中比较轨迹效率。
- **许可**：基线 `LICENSE.md` 是三条款 BSD 文本；分发派生代码时保留其版权、条件与免责声明。[seaborn license at base commit](https://github.com/mwaskom/seaborn/blob/0f5a013e2cf43562deec3b879458e59a73853813/LICENSE.md)

## 建议的实验顺序与量化指标

1. 先跑 `pytest-dev__pytest-7220`：它需要真实根因定位，最能区分“agent 偶然补丁”与“可解释 Atom”。
2. 再跑 `pallets__flask-4992`：它短而边界清晰，适合观察上一步抽取的调试/验证习惯是否被召回，同时形成 API 设计 Candidate。
3. 资源允许时加入 `mwaskom__seaborn-3010`：用不同技术域检查 Candidate 是否过拟合。

每题建议记录：官方 `FULL/PARTIAL/NO`、F2P 比例、P2P 比例、首次正确复现前的 tool calls、首次定位 gold production file 前的 tool calls、总修改文件数、无效补丁/回滚次数、最终补丁相对 gold 的文件集合差异，以及本次产生/命中的 Atom 与 Candidate 数。前三项直接对应官方评分语义；后几项是 xskill 实验指标，需由同一采集口径在 baseline 与 xskill 组中一致计算。[official grading metrics](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/swebench/harness/grading.py#L288-L315)

## 网络与许可证总说明

- 运行时应提前获取固定的官方 image，并在正式 agent 轨迹开始前完成所有 clone/image/dependency 下载。三个官方 `eval_script` 的测试阶段均不要求网络服务，但镜像拉取和首次依赖准备需要网络；把这些步骤放进计时会污染“xskill 是否提升解题效率”的比较。[三个固定 task rows](https://huggingface.co/datasets/SWE-bench/SWE-bench_Lite/tree/b0dde1093fe417d83b7184254edf8199c1f0dff5/data)
- SWE-bench 工具仓库自述为 MIT license；这不替代目标仓库自身的许可。三个候选分别受 pytest 的 MIT、Flask 的三条款 BSD、seaborn 的三条款 BSD 约束，尤其在保存/分享 agent patch、轨迹中的源码片段或构建镜像时应保留相应声明。[SWE-bench license statement](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/README.md#L150-L151) [pytest license](https://github.com/pytest-dev/pytest/blob/56bf819c2f4eaf8b36bd8c42c06bb59d5a3bfc0f/LICENSE) [Flask license](https://github.com/pallets/flask/blob/4c288bc97ea371817199908d0d9b12de9dae327e/LICENSE.rst) [seaborn license](https://github.com/mwaskom/seaborn/blob/0f5a013e2cf43562deec3b879458e59a73853813/LICENSE.md)

## 一手来源索引

- SWE-bench official repo：`SWE-bench/SWE-bench@7a21e05772954cc81471ae19d56f436cecf43c54`；[dataset guide](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/docs/guides/datasets.md)，[grading code](https://github.com/SWE-bench/SWE-bench/blob/7a21e05772954cc81471ae19d56f436cecf43c54/swebench/harness/grading.py)。
- SWE-bench Lite official data：`SWE-bench/SWE-bench_Lite@b0dde1093fe417d83b7184254edf8199c1f0dff5:data/test-00000-of-00001.parquet`；精确行标识为 `row_idx=144`、`148`、`177`。[data tree](https://huggingface.co/datasets/SWE-bench/SWE-bench_Lite/tree/b0dde1093fe417d83b7184254edf8199c1f0dff5/data)
- 原始 issues：[Flask #4989](https://github.com/pallets/flask/issues/4989)、[pytest #6428](https://github.com/pytest-dev/pytest/issues/6428)、[seaborn #2992](https://github.com/mwaskom/seaborn/issues/2992)。
- 对应历史 PRs：[Flask #4992](https://github.com/pallets/flask/pull/4992)、[pytest #7220](https://github.com/pytest-dev/pytest/pull/7220)、[seaborn #3010](https://github.com/mwaskom/seaborn/pull/3010)。这些 PR 仅用于 evaluator-side 核验，不能暴露给解题 agent。
