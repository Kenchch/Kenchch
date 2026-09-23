# 质量审计报告：软件工程、测试质量、打包与可维护性

**审计对象**：`Kenchch/nz-ets-forest-eligibility-screening` @ `622e3b8`
**审计视角**：软件工程 / 测试质量 / 打包 / 可维护性（QA）
**工作方式**：所有命令都在仓库副本 `scratchpad/quality/repo` 中执行，原仓库未被修改。另建了几个独立 venv：`quality/venv`（Python 3.11 + coverage/ruff/mypy/build/nbclient）、`quality/wheelvenv`（非 editable 的 wheel 安装）、`quality/minvenv`（依赖取声明的最低版本）、`quality/venv313`（Python 3.13 + wheel）。
**主要依赖版本（最新环境）**：pandas 3.0.6、geopandas 1.1.4、shapely 2.1.2、pyogrio 0.13.0（GDAL 3.12.4）、pyproj 3.7.2（PROJ 9.5.1）。

---

## 0. 测得的覆盖率

命令：`pytest --cov=ets_screening --cov-branch --cov-report=term-missing`（结果：55 passed）

| 模块 | Stmts | Miss | Branch | BrPart | Cover | 未覆盖（实际行号） |
|---|---|---|---|---|---|---|
| `__init__.py` | 4 | 0 | 0 | 0 | 100% | |
| `arcgis_paths.py` | 29 | 0 | 14 | 0 | 100% | |
| `demo_data.py` | 26 | 9 | 6 | 1 | 56% | `write_demo_layers`、`__main__` |
| `geometry.py` | 21 | 0 | 4 | 0 | 100% | |
| `io_utils.py` | 21 | 0 | 0 | 0 | 100% | |
| `load.py` | 45 | 7 | 20 | 6 | **80%** | 27（无 CRS）、37（空图层）、39（null/empty geometry）、42-43（invalid geometry）、52（缺列）、76（`read_layer` 缺列） |
| `report.py` | 90 | 0 | 12 | 4 | 96% | |
| `review_labels.py` | 46 | 0 | 8 | 1 | 98% | |
| `rules.py` | 77 | 0 | 16 | 0 | 100% | |
| `sample_review.py` | 55 | 14 | 8 | 2 | **71%** | 39-43（`sample_ids` 路径，`reproduce.py` 走的正是这条）、111-118（CLI `main`） |
| `screen.py` | 101 | 25 | 22 | 3 | **72%** | 184-187、199（覆盖已有输出目录）、210-241（CLI `main`） |
| **TOTAL** | 515 | 55 | 110 | 17 | **87%** | |

把 CI 的端到端步骤也算进来（`coverage run` 依次执行 pytest、`scripts/reproduce.py`、`semantic_hashes.py`、`verify_checksums.py`，再 combine）：`src/` 与 `scripts/` 合计 **68%**。其中 `download_gisborne_data.py` 和 `download_review_cards.py` 为 **0%**（需要联网）。`screen.main()`、`sample_review.main()` 这两个 console-script 入口在 pytest 和 CI 里都**从未被执行**。

其他测试运行结果：
- `pytest-randomly` 用种子 1/2/3 打乱顺序，三次都是 55 passed，未发现顺序依赖。
- 依赖取最低声明版本（pandas 2.2.0、geopandas 1.0.0、shapely 2.0.0、pyogrio 0.10.0、numpy 1.26.4）：55 passed。在这个环境下跑 `reproduce.py`，CSV 与 semantic hash 和提交版本完全一致。
- 在 Python 3.13 上安装 wheel（非 editable），从 sdist 取出 tests 运行：55 passed。

---

## 1. 发现清单

| ID | 严重度 | 标题 |
|---|---|---|
| QA-01 | **High** | `validate_candidates` 的重叠检查用了 1e-6 m² 绝对容差，而误差来自浮点抵消，合法数据会被误判为重叠 |
| QA-02 | Medium | 自定义 `RuleConfig` 只影响规则判定，宽度对比表、规则名和图表仍按硬编码的 30 m 输出，结果自相矛盾 |
| QA-03 | Medium | `run_manifest.json` 缺少审计溯源：没有配置、输入哈希、代码版本和依赖版本 |
| QA-04 | Medium | `rules/rule_register.csv` 与代码完全脱钩，已经出现漂移 |
| QA-05 | Medium | 审计长表 `rule_results.csv` 中 R-02/R-03/R-04 的 `observed` 只是通过与否的布尔值，没有测量值 |
| QA-06 | Medium | src、scripts、notebook 三处重复实现业务逻辑；`build_findings` 用四舍五入后的列重新判定阈值（潜在漂移） |
| QA-07 | Medium | `ets-screen` CLI：预期错误直接抛 traceback、参数不校验、`--demo` 静默忽略输入、没有 `--layer` |
| QA-08 | Medium | `ets-review-sample` CLI：负数 sample size 静默出错、不按 status 过滤、输出不可复现 |
| QA-09 | Medium | 测试缺口：CLI、错误路径、覆盖旧输出、`sample_ids`、非默认配置、真实规模数据都没测；有一个同义反复测试 |
| QA-10 | Medium | `review_map.html` 把 API key 明文写进受版本控制的文件；JSON 与 innerHTML 都没有转义 |
| QA-11 | Medium | README 中多瓣核心（multi-lobe）的数字没有任何提交代码能生成，且"每个部分都 >500 m²"的描述与 53 这个数不符 |
| QA-12 | Low | `reproduce.py` 把自己的 argv 泄漏给 `ingest_review_labels`：`--help` 会先跑完整流水线并改写 8 个受版本控制的文件 |
| QA-13 | Low | 审核标签校验可被绕过：纯空白字段能通过；非 YYYY-MM-DD 格式的日期也被接受，min/max 按字典序算错 |
| QA-14 | Low | ArcGIS `.pyt`：阈值 0 被 `or` 静默改成 0.8；异常没有转换成 `arcpy.AddError` |
| QA-15 | Low | `build_layout.py` 硬编码结果数字；README 写"54 tests"（实际 55） |
| QA-16 | Low | 依赖没有锁定、CI 只测单一版本，受版本控制的二进制输出在不同环境下会反复变动 |
| QA-17 | Low | 库级副作用和硬编码：import 时执行 `matplotlib.use("Agg")`、PDF 作者写死、staging 目录落在 cwd |

---

### QA-01 — High — 重叠检查用绝对容差 1e-6 m² 对大数相减，合法的真实数据会被拒

**位置**：`src/ets_screening/load.py:60-63`

```python
overlap_area = float(frame.geometry.area.sum() - frame.geometry.union_all().area)
if overlap_area > 1e-6:
    raise InputValidationError(
        f"candidate polygons overlap by {overlap_area:.3f} m2; submitted mapping must not overlap"
```

**问题**：两个约 1e9 m² 的浮点和相减后，与 1e-6 这个绝对阈值比较。1.1e9 量级下 float64 的 ULP 已是 2.4e-7。再加上 1500 个面积累加的误差和 `union_all` 的 noding 误差，差值的符号和大小基本是随机的。完整的 Gisborne 数据集能通过，只是因为差值恰好为负（`-1.955e-05`）。

**证据**（对项目自带的 `data/processed/gisborne_candidates.gpkg` 做随机子集，模拟"只筛选某个子区域"的常见用法）：
```
gisborne overlap_area = -1.9550323486328125e-05 sum=3662711860.0 union 5.00s
0 5.484e-06 REJECTED: candidate polygons overlap by 0.000 m2; submitted mapping must not overlap
1 1.669e-06 REJECTED: ...
2 2.384e-07 accepted
...
false rejections: 7          # 12 个随机 1500 要素子集中有 7 个被拒
```
对被拒的第 0 个子集逐对求交：
```
touching pairs: 326 max pairwise intersection area m2: 0.0 sum: 0.0
sum of areas: 1130924967.3 m2  float64 ulp at that magnitude: 2.384185791015625e-07
```
也就是说，这个子集根本没有任何重叠。

**影响**：主入口（CLI、`run_screening`、ArcGIS 工具）在"fail-loud"的输入校验这一步随机拒绝合法数据。报错信息是 "overlap by 0.000 m2"，既不能照着处理，也没有指出是哪些 `unit_id`。数据集越大，误判概率越高。另外，对全区做 `union_all` 本身要 5 秒。

**修复**：改为基于 sindex 的逐对检查，用有物理意义的面积容差，并在报错里列出冲突的 ID。我已在本地验证：Gisborne 全集 0 误报，12 个子集 0 误报，原测试里的真实重叠（A/B 各 5000 m²）照常报出，耗时 1.9 s（原来 5 s）。
```python
import shapely

def _overlapping_pairs(frame, tolerance_m2: float = 0.01):
    left, right = frame.sindex.query(frame.geometry, predicate="intersects")
    keep = left < right
    left, right = left[keep], right[keep]
    geoms = frame.geometry.values
    areas = shapely.area(shapely.intersection(geoms[left], geoms[right]))
    bad = areas > tolerance_m2
    ids = frame["unit_id"].to_numpy()
    return list(zip(ids[left[bad]], ids[right[bad]], areas[bad]))

pairs = _overlapping_pairs(frame)
if pairs:
    raise InputValidationError(
        f"{len(pairs)} candidate pairs overlap (> 0.01 m2), e.g. {pairs[:5]}"
    )
```
同时补一个回归测试：两个相邻但不重叠、坐标在 NZTM 量级（~2e6/5.7e6）的大面积网格应当通过。

---

### QA-02 — Medium — 非默认 `RuleConfig` 产出自相矛盾的结果

**位置**：`src/ets_screening/screen.py:101`；`src/ets_screening/rules.py:82, 134-135`；`src/ets_screening/report.py:122`

```python
# screen.py:101
comparison = compare_width_methods(results)            # 默认 threshold_m=30.0
# rules.py:82
widths = compare_width_methods(out, config.width_threshold_m)
# rules.py:134-135
"R-01": ("Area at least 1 ha", ...),
"R-02": ("30 m width erosion proxy", ...),
# report.py:122
ax.axhline(30.0, ..., label="30 m threshold")
```

**证据**：`run_screening(..., config=RuleConfig(width_threshold_m=45.0))`
```
rules (45 m): G11 width_core_pass= False  methods_disagree= False  status= quarantine  failed= R-02
width_method_comparison.csv G11: {'width_area_perimeter_m': 35.294, 'area_perimeter_pass': True, 'erosion_core_pass': True, 'methods_disagree': False}
rule_results R-02 name: 30 m width erosion proxy
```
同一次运行里，G11 按 45 m 判为 R-02 失败，但 `width_method_comparison.csv`、`summary.csv` 的 `width_method_disagreements`、宽度图和规则名都还是 30 m 口径。

**影响**：`RuleConfig` 是公开 API（在 `__init__` 中导出），README 也把 1% 阈值称作"documented sensitivity threshold"。但只要做敏感性分析，产物之间就互相矛盾，而且 manifest 里也不记录用了哪个配置（见 QA-03）。

**修复**：`run_screening` 里解析出 `config = config or RuleConfig()` 一次，然后 `compare_width_methods(results, config.width_threshold_m)`。更好的做法是直接复用 `evaluate_rules` 已经算好的 `width_*` 列，不再算第二遍。规则名用 f-string 从 config 生成（例如 `f"Area at least {config.minimum_area_ha:g} ha"`）。`plot_width_comparison` 增加 `threshold_m` 参数。再补一个非默认配置的端到端测试。

---

### QA-03 — Medium — `run_manifest.json` 不足以支撑审计溯源

**位置**：`src/ets_screening/screen.py:151-166`

**证据**（demo 运行的完整 manifest 键）：
```
manifest keys: ['scope', 'crs', 'input_feature_type', 'feature_count', 'overlap_materiality', 'reject_rate', 'reject_rate_threshold', 'unresolved_rules']
```
缺少的内容：`minimum_area_ha`、`width_threshold_m`、`plantable_lcdb_classes`；输入路径、layer 名和 SHA-256；包版本（包里也没有 `__version__`）；geopandas/shapely/GEOS/PROJ/GDAL 版本；review 的 seed 和 sample size。`unresolved_rules` 硬编码为 `["R-03","R-06","R-07","R-08"]`，而 `summary.csv` 又写 `manual_rules_out_of_scope = "3"`（`screen.py:48`），两处口径不一致。

**影响**：项目定位是"auditable spatial triage"，但对 CLI 或 ArcGIS 用户的任意一次运行，事后都无法确认用了什么输入、什么阈值、什么代码版本。Gisborne 那次运行能溯源，靠的是仓库外部的 `checksums.sha256` 和 git 历史，而不是 manifest 本身。

**修复**：
```python
from dataclasses import asdict
from importlib.metadata import version
import shapely, pyproj, pyogrio

cfg = config or RuleConfig()
manifest["rule_config"] = {k: sorted(v) if isinstance(v, frozenset) else v
                           for k, v in asdict(cfg).items()}
manifest["software"] = {
    "ets_screening": version("nz-ets-forest-eligibility-screening"),
    "geopandas": gpd.__version__, "shapely": shapely.__version__,
    "geos": shapely.geos_version_string, "proj": pyproj.proj_version_str,
    "gdal": pyogrio.__gdal_version_string__,
}
manifest["inputs"] = input_provenance   # main()/pyt 传入 {name: {"path":..., "layer":..., "sha256":...}}
```

---

### QA-04 — Medium — 规则登记表与代码脱钩，已出现漂移

**位置**：`rules/rule_register.csv`（没有任何代码读取它）；`src/ets_screening/rules.py:127, 133-165`；`src/ets_screening/screen.py:48, 165`

**证据**：`grep -rn "rule_register"` 在 `*.py/*.toml/*.yml/*.ipynb` 中**没有任何结果**。两边已经不一致：

| rule | `rule_register.csv` | 代码（`rule_results.csv`） |
|---|---|---|
| R-01 name | Area at least 1 hectare | Area at least 1 ha |
| R-02 name / testable | Average width at least 30 metres / `proxy` | 30 m width erosion proxy / `True` |
| R-03 name / testable | Not materially mapped as pre-1990 forest land / `partial` | No material mapped pre-1990 overlap / `True` |
| R-05 testable | `proxy` | `True` |
| R-07 name | Crown cover more than 30 percent in each hectare | Crown cover at maturity |
| R-05 implementation | "Configurable LCDB class allow-list" | CLI 和 ArcGIS 均不可配置（只能通过 Python API） |

**影响**：登记表本应是审计的权威来源，但代码可以悄悄偏离它。审阅者看到的 `testable_from_open_data=True` 与登记表上的 proxy/partial 定性相互矛盾。

**修复**：把登记表打进 package data（或放进 `ets_screening/rules/`），由 `evaluate_rules` 读取 name 和 testable 字段来生成审计行。至少加一个一致性测试：
```python
def test_audit_rows_match_rule_register():
    reg = pd.read_csv(ROOT / "rules/rule_register.csv").set_index("rule_id")
    _, audit = evaluate_rules(*build_demo_layers())
    for rid, grp in audit.groupby("rule_id"):
        assert grp["rule_name"].iloc[0] == reg.loc[rid, "name"]
```

---

### QA-05 — Medium — 审计表的 `observed` 列对 3/5 条自动规则只记录布尔值

**位置**：`src/ets_screening/rules.py:133-149`

```python
"R-02": ("30 m width erosion proxy", "r02_width_proxy_pass", "width_core_pass"),
"R-03": ("No material mapped pre-1990 overlap", "r03_no_pre1990_overlap", "r03_no_pre1990_overlap"),
"R-04": ("No material public conservation overlap", "r04_no_conservation_overlap", "r04_no_conservation_overlap"),
```

**证据**（提交的 `outputs/gisborne/rule_results.csv`，R-03 的 `passed|observed` 组合）：
```
    691 False|False
   5021 True|True
```
`observed` 完全等于 `passed`，看不到重叠 m²、百分比或 2A/P 宽度。R-01 的 `observed` 是四舍五入到 4 位的 `area_ha`，所以 0.99996 ha 这种边界案例在审计表里显示为 `observed=1.0, passed=False`。

**影响**：长表格式的审计表本该让人不回头查 GeoPackage 也能复核每个判定，而现在 R-02/R-03/R-04 的判定依据在表里看不到。

**修复**：`observed` 改为结构化的测量值，例如 R-03 写 `f"{m2:.2f} m2; {pct:.4f}%"`，R-02 写 `f"core={core}; 2A/P={w:.3f} m"`；或者增加 `observed_value` 和 `threshold` 两列。R-01 用未四舍五入的面积，或同时写出 `area_m2`。

---

### QA-06 — Medium — 业务逻辑在 src、scripts、notebook 三处重复，并存在四舍五入漂移

**位置**：
- `scripts/build_findings.py:86-89`：materiality 判定重写了一遍，阈值硬编码为 `1.0`，而且用的是四舍五入后的列：
  ```python
  legacy_r03 = screened["pre1990_overlap_m2"] > 1.0          # 已 round(2)
  material_r03 = legacy_r03 & (screened["pre1990_overlap_pct"] >= 1.0)   # 已 round(4)
  ```
- advisory 掩码写了 3 份：`screen.py:31-34`、`screen.py:65-68`、`build_findings.py:70-73`。
- reject rate 的计算写了 4 份：`screen.py:46`、`screen.py:102`、`report.py:168`、`build_findings.py`。
- 可视审核的汇总写了 2 份：`build_findings._visual_review_findings`（`:23-54`）和 `review_labels.summarise_review_labels`。
- `PLANTABLE_CLASSES`（`scripts/download_gisborne_data.py:43`）重复了 `RuleConfig.plantable_lcdb_classes`。
- `buffer(-15)` 和标题 `"erosion=pass"`（`build_findings.py:152,161`）把"所有分歧都是 AP-fail/erosion-pass"这一当前数据的性质硬编码进了代码。
- notebook 同样硬编码了 `> 1.0` 和 `>= 1.0`，没有 import 包。

**证据**（构造的边界案例：100×100 m 单元与 99.996 m² 的 PCL 重叠，占 0.99996%）：
```
status: candidate_review pct stored: 1.0 -> build_findings would count material: True
```
按规则引擎这是 advisory，但 `build_findings` 会把它算成 material。当前 Gisborne 数据里恰好 0 个不一致（`gisborne mismatch r04: 0 r03: 0`），所以这是潜在缺陷。这与项目在 R-01 上专门防范的"用四舍五入值做阈值判断"是同一类问题（`test_unrounded_area_controls_threshold_decision`）。

**影响**：阈值一改，findings、notebook 和实际状态就会悄悄不一致，README 里的数字也就不再可信。

**修复**：让 `evaluate_rules` 直接输出 `r03_material`、`r04_material`、`r03_any_overlap` 等布尔列（用原始值计算），`build_findings` 和 notebook 只做计数。抽出一个 `ets_screening.metrics` 模块，提供 `advisory_mask(results)`、`reject_rate(results)` 等函数给各处复用。下载脚本改为 `from ets_screening.rules import RuleConfig; RuleConfig().plantable_lcdb_classes`。

---

### QA-07 — Medium — `ets-screen` CLI 的错误处理与参数设计

**位置**：`src/ets_screening/screen.py:210-241`

**证据**（实际运行）：
```
$ ets-screen --candidates nope.gpkg ...            -> 完整 traceback ... pyogrio.errors.DataSourceError: nope.gpkg: No such file or directory   exit=1
$ ets-screen ... --reject-rate-threshold 0.1      -> 完整 traceback ... RejectRateExceeded: reject rate 58.3% exceeds 10.0% ...   exit=1
$ ets-screen --demo --reject-rate-threshold 5     -> 成功，manifest 写入 "reject_rate_threshold": 5.0
$ ets-screen --demo --reject-rate-threshold -1    -> RejectRateExceeded: ... exceeds -100.0%
$ ets-screen --demo --candidates /does/not/exist.gpkg  -> exit=0（输入被静默忽略）
$ ets-screen --version                            -> error: unrecognized arguments: --version
```
多图层 GPKG（第一个图层不是 candidates）：
```
UserWarning: More than one layer found in 'multi.gpkg': 'aaa_other' (default), 'candidates'. Specify layer parameter to avoid this warning.
... RejectRateExceeded: reject rate 100.0% exceeds 80.0%
```
这里只是碰巧被 reject gate 拦住了。阈值设为 1.0 时，会把错误的图层当作结果发布出去。CLI 本身没有 `--candidates-layer`；`path.gpkg/main.layer` 这种写法（`split_dataset_path`）在 `--help` 里也没有说明。

**影响**：预期内的输入错误和程序 bug 都返回 exit 1，调用方无法区分；用户看到的是一大段 traceback。阈值参数不检查范围，这与 ArcGIS 工具（`updateMessages` 限制在 0–1）的行为不一致。

**修复**：
```python
def _unit_interval(text: str) -> float:
    value = float(text)
    if not 0.0 <= value <= 1.0:
        raise argparse.ArgumentTypeError("must be between 0 and 1")
    return value

parser.add_argument("--reject-rate-threshold", type=_unit_interval, default=0.80)
parser.add_argument("--version", action="version", version=f"%(prog)s {version(...)}")
group = parser.add_mutually_exclusive_group(required=True)   # --demo 与 --candidates 互斥
...
try:
    manifest = run_screening(...)
except InputValidationError as err:
    parser.exit(2, f"ets-screen: input rejected: {err}\n")
except RejectRateExceeded as err:
    parser.exit(3, f"ets-screen: {err}\n")
except pyogrio.errors.DataSourceError as err:
    parser.exit(2, f"ets-screen: cannot open input: {err}\n")
```
另外，在 `read_layer` 里遇到多图层且未指定 layer 时直接报错，不要只发 warning（可用 `pyogrio.list_layers(container)`）。

---

### QA-08 — Medium — `ets-review-sample` 的健壮性问题

**位置**：`src/ets_screening/sample_review.py:44, 110-124`

```python
count = min(sample_size, len(candidates))      # 不校验负数
...
frame = gpd.read_file(args.input)              # 不用 read_layer / split_dataset_path，不校验列，不过滤 status
```

**证据**：
```
$ ets-review-sample demo_candidates.gpkg --sample-size -3  -> exit 0，review_sample_ids.csv 含 9 行（12 个要素中的 head(-3)）
$ ets-review-sample demo_out/quarantine.gpkg --sample-size 3 -> exit 0，抽到 G02,G06,G10（全是 quarantine/excluded），HTML 标题仍是 "Human review queue"
三次运行输出 review_queue.gpkg 的哈希：98b7da64…、d97f6e82…、5e7b631a…（互不相同；主流水线会调用 normalise_gpkg，这里没有）
$ ets-review-sample data/sample/demo_candidates.gpkg ...  -> 输入没有 status 列，弹窗显示 "Status: undefined"
```
`--sample-size 0` 会生成一个空地图，`map.fitBounds` 会在浏览器中报错。

**影响**：这是两个公开 console script 之一，却可能从被拒绝的单元里静默生成"人工审核队列"，而且输出不满足项目自己的字节稳定性承诺。

**修复**：`--sample-size` 用 `type=positive_int`；读取统一走 `read_layer(args.input, ("unit_id",), "review candidates")`；存在 `status` 列时过滤为 `candidate_review`，或者遇到非候选状态直接报错；写出后调用 `normalise_gpkg`；`write_review_bundle` 在样本为空时报错。

---

### QA-09 — Medium — 测试缺口和同义反复测试

**证据**：见第 0 节的覆盖率。具体缺口：
1. **CLI 入口 0 覆盖**：`screen.main`（`screen.py:210-241`）和 `sample_review.main`（`:110-124`），在 pytest 和 CI 中都没有被执行过。QA-07、QA-08 的问题因此都没被测出来。
2. **fail-loud 校验的负路径都没测**：`load.py` 的无 CRS、空图层、null/empty geometry、invalid geometry、`validate_candidates` 缺列、`read_layer` 缺列（`load.py:27,37,39,42-43,52,76`）。项目的核心卖点是"Input loading and fail-loud validation"，但只测了 WGS84、重复 ID、multipart 和重叠这四种。
3. **覆盖已有输出目录的分支没测**（`screen.py:184-187, 199`）。`reproduce.py` 每次运行和 ArcGIS 工具"原地刷新"的设计都依赖这条路径。
4. **`select_review_sample(sample_ids=...)` 没测**（`sample_review.py:39-43`），包括"ID 已不是候选"时抛 `ValueError` 的分支，而 `reproduce.py` 走的正是这条路径。
5. **没有任何非默认 `RuleConfig` 的测试**：本来能直接发现 QA-02。
6. **没有真实量级坐标或大规模的校验测试**：本来能发现 QA-01。所有规则测试都用坐标接近原点的小方块或 12 个 demo 要素。
7. **同义反复测试**：`tests/test_crs.py:test_same_shape_in_wgs84_would_destroy_area_threshold` 只断言 pyproj/shapely 的行为（WGS84 下面积 < 1），没有调用任何项目代码，删掉 `assert_nztm2000` 这个测试也照样通过。
8. `test_main_pipeline_propagates_linz_api_key` 把"密钥写入 HTML"固化成了期望行为（见 QA-10）。
9. CI（`.github/workflows/ci.yml`）不跑 lint/type check，不执行 notebook，不测 wheel 安装，只测 Python 3.12。
10. `pyproject.toml:34` 中 `addopts = "--basetemp=.pytest_tmp"`：pytest 每次启动会清空 basetemp，所以同一目录下的并发 pytest 会互相删掉临时文件；而且这个目录相对于 cwd，而不是 rootdir。

**修复**：新增 `tests/test_cli.py`：用 `subprocess.run([sys.executable, "-m", "ets_screening.screen", ...])` 或直接调用 `main()` 配合 `monkeypatch.setattr(sys, "argv", ...)`，断言退出码和 stderr。用 `@pytest.mark.parametrize` 把 `load.py` 的每条错误路径都覆盖到。写一个"先发布一次、再发布一次"的测试，断言旧的 managed 文件被替换、`review/review_labels.csv` 被保留。把 `test_same_shape_in_wgs84_...` 改为直接断言 `evaluate_rules(frame(4326), …)` 抛出 `InputValidationError`。CI 增加 `ruff check`、`pytest --cov --cov-fail-under=85`，以及 3.11/3.12/3.13 的矩阵。

---

### QA-10 — Medium — `review_map.html` 写入 API key，且嵌入内容未转义

**位置**：`src/ets_screening/sample_review.py:85-86, 97, 102`；`src/ets_screening/screen.py:147`

```python
if api_key:
    tile_url += f"?api={api_key}"
...
const features={json.dumps(geojson)};
... l.bindPopup('<b>'+f.properties.unit_id+'</b><br>Status: '+f.properties.status)
```

**证据**：
```
$ LINZ_BASEMAP_API_KEY="c01k-FAKE-SECRET" python scripts/reproduce.py
$ git status --short outputs/gisborne/review/
 M outputs/gisborne/review/review_map.html
$ git diff ... | grep -o "webp?api=[^']*"
webp?api=c01k-FAKE-SECRET
```
`outputs/gisborne/review/review_map.html` 在 `git ls-files` 里，是受版本控制的文件，所以密钥很容易被一起提交。另外，把 `unit_id` 设为 `G01</script><script>alert(document.domain)</script>` 后，这段字符串原样出现在 `<script>` 块中（`raw </script> inside features JSON: True`），可以提前闭合脚本标签；popup 用字符串拼接后交给 innerHTML，同样可以注入。

**影响**：密钥可能泄漏到 git 历史。输入数据中的属性值可以在审核人浏览器中执行脚本（这个 HTML 本来就是要分发给审核人的）。

**修复**：不要把密钥写进文件，改为在页面里从 `location.hash` 或 `prompt()` 读取，或者只在 `outputs/` 以外、已被 gitignore 的路径生成带 key 的版本。嵌入 JSON 时用 `json.dumps(geojson).replace("</", "<\\/")`。popup 用 `document.createElement` 加 `textContent` 构造。相应地修改 `test_main_pipeline_propagates_linz_api_key`，断言文件里**不包含**密钥。

---

### QA-11 — Medium — README 中的 multi-lobe 数字不可复现，且定义描述有误

**位置**：`README.md:54-61`；`notebooks/01_width_rule_comparison.ipynb`；`scripts/build_findings.py`

README 原文："In **181 of the 694** … requiring every part to exceed 500 m² gives 53, and 1,000 m² gives 23."

**证据**：`grep -rn "181\|lobe"` 在 `scripts/`、`src/`、`findings.json/csv`、notebook 中都没有找到对应的计算，只有 README 里有。我按 README 的描述自己重算：
```
disagreements 694 multi-part cores 181                 # 181 可复现
500  all>t: 16   >=2 parts >t: 53   all>=t: 16
1000 all>t: 1    >=2 parts >t: 23   all>=t: 1
```
53 和 23 对应的是"**至少两个**部分超过阈值"，不是 README 写的"**每个**部分都超过"（按后者只有 16 和 1）。

notebook 本身：用 nbclient 在当前环境执行后，6 个 code cell 的输出与提交版本**逐字一致**，这点很好。但它开头说 "recomputes each headline number in the project README"，实际上没有覆盖这组数字；它也不 import `ets_screening`，阈值是硬编码的；CI 不执行它；cell 缺少 `id`（nbformat 报 `MissingIDFieldWarning`，未来会升级为硬错误）。

**影响**：一个 headline 数字无法从提交的代码复现，而且它的定义描述是错的，这与"evidence-backed findings"的定位相冲突。

**修复**：在 `build_findings.py` 里加入 `width_multi_lobe_cores`、`width_multi_lobe_ge2_parts_gt_500m2` 等字段，写入 `findings.json`，notebook 从中读取；把 README 的描述改成"at least two parts exceed 500 m²"。CI 中增加 `jupyter nbconvert --execute --to notebook notebooks/*.ipynb`（或 `pytest --nbval-lax`）；对 notebook 执行一次 `nbformat.validator.normalize` 补上 cell id。

---

### QA-12 — Low — `reproduce.py` 的 argv 泄漏给 `ingest_review_labels`

**位置**：`scripts/reproduce.py:9-10, 38`；`scripts/ingest_review_labels.py:45`

```python
from ingest_review_labels import main as ingest_review_labels
...
if ingest_review_labels() != 0:        # 其内部 parser.parse_args() 读取的是 reproduce.py 的 sys.argv
```

**证据**：
```
$ python scripts/reproduce.py --help      # 耗时 23 s
... 先打印整个 findings JSON，再打印 ingest_review_labels 的帮助："Validate a completed human imagery review ..."
exit=0
$ git status --short   -> 8 个受版本控制的输出文件被改写（gpkg/png/pdf/html）
$ python scripts/reproduce.py --bogus
reproduce.py: error: unrecognized arguments: --bogus   exit=2（此时所有输出已写完）
```
帮助信息里的 `usage: reproduce.py [-h] [--review-dir REVIEW_DIR]` 也有误导性：`--review-dir` 只影响 ingest 这一步，`run_screening` 和 `build_findings` 仍写入硬编码的路径。

**修复**：把 `main()` 改为 `def main(argv: list[str] | None = None) -> int` 并调用 `parser.parse_args(argv)`，`reproduce.py` 里调用 `ingest_review_labels([])`；`reproduce.py` 自己加一个 argparse，尽早拒绝未知参数。更进一步，可以把 scripts 组织成包，用 `python -m` 调用，不要依赖 `sys.path[0]` 隐式导入同目录的模块。

---

### QA-13 — Low — 人工审核标签的校验可被绕过，日期汇总会算错

**位置**：`src/ets_screening/review_labels.py:68, 108, 115, 165-166`

**证据**：
```python
reviewer=" "（单个空格）, evidence_note=" "*25, review_date=["20260914","2026-09-15","2026-W37-1"]
-> ACCEPTED rows: 3 reviewer repr: ' '
-> review_date_first = 2026-09-15 | review_date_last = 20260914     # 字典序 min/max，结果错误
"Ai Nakamura" -> REJECTED: reviewer ['Ai Nakamura'] looks automated
```
原因：`labels.eq("")` 不会把空白当作空值；在 Python ≥3.11 中，`date.fromisoformat` 接受 `YYYYMMDD` 和 ISO 周日期，与报错信息所说的 "not ISO YYYY-MM-DD" 不符；`min`/`max` 是对字符串取的。"ai"/"model" 这类词会误伤真实姓名。

**影响**：这一步是项目明确要求必须由人完成的证据链关口，但"每列必填"和"至少 20 个字符"两条检查都能被空白字符满足。汇总里的日期范围也可能是错的。

**修复**：
```python
labels = labels.apply(lambda s: s.str.strip())
_require(labels["review_date"].str.fullmatch(r"\d{4}-\d{2}-\d{2}").all(), "...")
dates = pd.to_datetime(labels["review_date"], format="%Y-%m-%d")
("review_date_first", dates.min().date().isoformat())
```
另外建议把机器审核者的名单做成可配置的 deny-list，并提供显式的 override 参数（例如 `--allow-reviewer`），避免误伤真实姓名。

---

### QA-14 — Low — ArcGIS `.pyt` 的参数处理

**位置**：`arcgis/ets_screening.pyt:3, 81, 83-87`

```python
threshold = float(parameters[4].value or 0.80)
```

**问题与证据**：
- `updateMessages` 明确允许 `0 <= threshold <= 1`，但 `0.0 or 0.80` 会得到 `0.8`（`python3 -c "print(float(0.0 or 0.80))"` → `0.8`）。用户设成 0（"只要有任何拒绝就中止"）时，会被静默放宽到 80%。
- `execute` 没有捕获 `InputValidationError` 和 `RejectRateExceeded`，ArcGIS 里会显示 Python traceback，而不是 `arcpy.AddError` 输出的可读信息。
- `import os` 没有被使用（ruff F401）。
- 工具不提供 `study_label`、`data_note`、`review_sample_ids`、`RuleConfig` 参数，所以 ArcGIS 的产物标题永远是 "User-supplied screening run"。arcgis/README 已说明 review sample 的差异。
- 需确认 (Needs confirmation)：feature dataset 内的要素类（`x.gdb\dataset\fc`）会被 `split_dataset_path` 拆成 layer=`"dataset/fc"`，GDAL OpenFileGDB 可能无法按这个名字打开。

**修复**：
```python
value = parameters[4].value
threshold = 0.80 if value is None else float(value)
try:
    ...
except (InputValidationError, RejectRateExceeded) as err:
    arcpy.AddError(str(err))
    raise arcpy.ExecuteError(str(err))
```

---

### QA-15 — Low — `build_layout.py` 硬编码结果数字，文档计数漂移

**位置**：`arcgis/build_layout.py:52-77`；`README.md:209`

```python
"candidate_review": "Candidate for assessor review (2,687)",
"quarantine": "Quarantined - rule failure (2,801)",
"excluded": "Excluded - project conservation policy (224)",
SUBTITLE = "... 5,712 LCDB v6 ..."
INSET_NOTE = "... 22.5 m. One of 694 units (12.15%) ..."
```
**证据**：这些数字目前与 `outputs/gisborne/summary.csv`（2687/2801/224/5712/694）以及 `lcdb1000453434` 的 `width_area_perimeter_m=22.458` 一致，但它们是手工抄写的，数据或阈值一变就会悄悄过时。README 写的是 "54 tests"，而 `pytest` 实际是 55 passed。另外 `BLANK_TEMPLATE` 引用的是 Pro 安装目录下 routing services 的内部文件 `Resources\ArcToolBox\Services\routingservices\data\Blank.aprx`，需确认它是否属于稳定接口。

**修复**：在 `build_layout.py` 里用 `csv` 标准库读取 `summary.csv` 和 `width_method_comparison.csv` 生成标签（脚本只依赖 arcpy，不需要 pandas）；README 里不要写死测试数量。

---

### QA-16 — Low — 依赖管理与二进制产物跨环境漂移

**位置**：`pyproject.toml:12-20, 11`；`environment.yml`；`.github/workflows/ci.yml`

**证据**：
- 依赖只有下限、没有上限，也没有 lock 或 constraints 文件。CI 用 `pip install -e ".[dev]"`，从不使用 `environment.yml`，所以 conda 环境这条路径从未被验证。
- 在一个全新环境（pandas 3.0.6、GDAL 3.12.4、PROJ 9.5.1）中运行 `reproduce.py`：CSV 和 semantic hash **完全一致**（这是优点），但 `git status` 显示 8 个受版本控制的二进制文件被修改：
  ```
   M outputs/gisborne/candidates.gpkg / quarantine.gpkg / review/review_queue.gpkg
   M outputs/gisborne/figures/*.png (3)  / layout_map.pdf
   M outputs/gisborne/review/review_map.html     # 经纬度末位差异，例如 -37.62279803729452 vs ...4
  ```
  同一环境下连跑两次，这 8 个文件的 sha256 完全一致，所以 README 说的 "byte-stable within an environment" 成立。但提交这些文件时用的是哪个环境，没有任何记录。
- `python -m build` 输出警告：`project.license as a TOML table is deprecated … By 2027-Feb-18, you need to update your project`。
- 优点：最低版本矩阵（pandas 2.2.0、geopandas 1.0.0、shapely 2.0.0、numpy 1.26.4）测试通过，输出一致；Python 3.11 和 3.13 也都通过。

**修复**：新增 `requirements-lock.txt`（`uv pip compile`）或 `conda-lock.yml`，CI 按它安装；manifest 中记录依赖版本（见 QA-03）；CI 增加一个 `minimum-versions` job 和 Python 版本矩阵；`license = "MIT"` 加上 `license-files = ["LICENSE"]`；考虑不再跟踪 PNG、PDF、HTML 这类随环境变化的产物，或者在 README 里写明生成它们的环境。

---

### QA-17 — Low — 库级副作用与硬编码常量

**位置**：`src/ets_screening/report.py:11, 202-204`；`src/ets_screening/screen.py:15, 89-206, 111, 146`

**证据**：
```
before: module://matplotlib_inline.backend_inline
after import ets_screening.screen: Agg        # import 时把 Jupyter 的 inline backend 替换成了 Agg
```
- PDF 元数据 `"Author": "Feng Jiang"` 和固定日期 2026-09-12 会写进**任何用户**的运行结果。
- `sample_size=30` 在 `run_screening` 中写死，seed 和 sample size 都没有通过 API 或 CLI 暴露。
- 默认 `--output outputs` 时，staging 目录 `ets-screen-*` 建在 `output_dir.parent`，也就是 cwd。`.gitignore` 只忽略了 `outputs/ets-screen-*/`，进程被 kill 时会在仓库根目录留下未被忽略的临时目录；同时 `outputs/*.gpkg` 等 demo 产物也不在 gitignore 范围内。
- `run_screening` 有 9 个位置参数（ruff PLR0913/PLR0917），约 120 行代码同时负责校验、写文件、画图、生成审核包和原子发布。
- 整个包没有使用 `logging`，处理 5,712 个要素的约 20 秒里没有任何进度输出。
- `from .demo_data import build_demo_layers, write_demo_layers`：`write_demo_layers` 没有被使用（ruff F401）。

**修复**：把 `matplotlib.use("Agg")` 删掉，在绘图函数里用 `from matplotlib.figure import Figure; fig = Figure()` 这种面向对象写法，不依赖 pyplot 状态机；author 和日期改为参数（默认值取 `SOURCE_DATE_EPOCH`）；把 `run_screening` 拆成 `evaluate → write_stage(stage) → publish(stage, output_dir)` 三步，并把参数改成 keyword-only（`*,`）；staging 目录放在 `output_dir` 下的隐藏子目录，例如 `output_dir/.staging-*`；使用 `logging.getLogger(__name__)`。

---

## 2. 静态分析摘要

- **ruff 0.16.8**（默认规则集）：共 35 条，多数是 import 排序、`UP017`/`UP035` 这类风格问题。有意义的只有 F401 未使用导入：`screen.py:15`（`write_demo_layers`）、`arcgis/ets_screening.pyt:3`（`os`）、`tests/test_rules.py:1`（`pandas`）。扩展规则集（B、PL、S、TRY、PTH 等）在非测试代码中额外发现 `PLR0913`（`run_screening` 9 个参数）、`S310`（`urllib.urlopen` 协议未限制，但 URL 都是常量，风险低）和 `B905`，没有真正的 bug。
- **mypy 2.3.1**（`--ignore-missing-imports`）：3 个错误，都是 pandas-stubs 和 matplotlib 的类型签名噪声（`DataFrame.eq("")`、`Hashable + int`、`add_axes(list)`），不是实际缺陷。由于 geopandas 没有类型存根，大部分空间逻辑实际上是 `Any`，类型检查对这部分帮助有限。
- 代码里**没有静默吞掉异常的 `except`**。唯一的 `ignore_errors=True` 只用在清理 staging 目录（`screen.py:206`），这个用法合理。

## 3. 打包验证结果

- `python -m build`（从 `git archive` 得到的干净源码构建）成功生成 sdist 和 wheel。wheel **包含** `ets_screening/vendor/leaflet-1.9.4/{leaflet.js,leaflet.css,LICENSE}`。
- 在新 venv 中非 editable 安装 wheel，在仓库以外的目录执行 `ets-screen --demo --output out`（exit 0，产物齐全）和 `ets-review-sample out/candidates.gpkg`（exit 0），生成的 HTML 内联了 Leaflet。包本身**不依赖**任何仓库相对路径：`VENDOR` 通过 `__file__` 定位，`rules/` 和 `data/` 只由 `scripts/` 使用。
- 在 Python 3.13 下安装 wheel，把 sdist 里的 tests 单独运行：55 passed。
- 需要注意的是，`rules/rule_register.csv` **没有**打进包里，包也不读取它（见 QA-04）。

## 4. 观察到的优点

- **可复现性工程做得扎实**：GeoPackage 头部和时间戳归一化后，同一环境内重复运行的输出字节完全相同（已验证 8 个产物 sha256 一致）；CI 用 CSV diff 加 semantic hash 比对真实数据的输出；最低依赖版本下 CSV 也逐字节相同；notebook 重新执行后的输出与提交版本逐字一致。
- **fail-loud 的设计取向正确**：CRS 不做自动重投影；reject-rate gate 在写任何文件之前就中止；先在 staging 目录生成、再替换 managed 文件，不会留下半写的输出；没有静默吞异常的代码。
- **核心算法模块测试质量高**：`rules.py`、`geometry.py`、`arcgis_paths.py`、`io_utils.py` 都达到 100% 分支覆盖；测试用"逐维破坏一个有效样本"的方式确认每条规则都因为自己的原因失败，也覆盖了边界接触不算重叠、用未四舍五入的面积做阈值判断等细节；测试打乱顺序后仍全部通过，也不依赖仓库路径。
- **打包干净**：src layout，vendored Leaflet 正确声明为 package-data，wheel 离开仓库也能独立运行两个入口；`arcgis_paths` 把纯字符串逻辑从 arcpy 中剥离出来，所以能在 CI 里测试，这个设计值得肯定。
