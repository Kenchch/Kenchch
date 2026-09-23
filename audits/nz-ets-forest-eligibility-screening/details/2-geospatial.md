# 地理空间与算法正确性审计：nz-ets-forest-eligibility-screening @ 622e3b8

审计角度：CRS、几何有效性、宽度算法、叠加/面积占比、邻接/切分、性能。
环境：geopandas 1.1.4 / shapely 2.1.2 (GEOS 3.13.1) / pandas 3.0.6 / pyproj 3.7.2。原仓库未做任何修改；所有运行都在副本 `scratchpad/geo/repo` 上完成（基线 55 个测试全部通过，运行时用了 `--basetemp`）。复现脚本在 `scratchpad/geo_scratch/s01…s23_*.py`。

## 总览

| ID | 严重度 | 位置 | 一句话 |
|---|---|---|---|
| GEO-01 | High | `src/ets_screening/rules.py:98-107` | 叠加"实质性"只看相对比例 (≥1%)，结果随尺度变化：进入候选队列的单元里含多达 297 ha DOC 保护地 / 147 ha pre-1990 林地；反过来，小单元会因 <15 m 宽的边界碎片被排除 |
| GEO-02 | Medium | `geometry.py:23`, `rules.py:91` | R-02 实际上就等于 `2A/P ≥ 30`（5,712 个单元全部一致），erosion 从未决定过结果；2A/P 系统性偏低，直的 33 m×304 m 条带也会被隔离 |
| GEO-03 | Medium | `load.py:60-61` | 重叠校验用绝对容差 1e-6 m²，比浮点噪声 (±2e-5 m²) 还小；真实数据的随机半数子集有 11/30 被误判为"重叠"而拒绝 |
| GEO-04 | Medium | `rules.py:88` | R-01 按单个 LCDB 单元判定，没有合并相邻的可种植单元：104 个"只因 R-01 失败"的单元与 ≥1 ha 的可种植连片共边 |
| GEO-05 | Medium | `screen.py:101`, `report.py:122` | `run_screening` 用默认 30 m 重算宽度对比，忽略 `config.width_threshold_m`，发布的 CSV/summary/图与 GPKG 互相矛盾 |
| GEO-06 | Medium（需确认 Needs confirmation） | `scripts/download_gisborne_data.py:205,229-232` | pre-1990 证据只取 LUCAS 72/72，漏掉 1989 年的天然林 (71) |
| GEO-07 | Low | README L41-60, notebook cell 7-8 | Finding 2 的解释有误："方向一致"是 2A/P 的数学性质；42% 的分歧单元是凸多边形，并不是分叉或哑铃形 |
| GEO-08 | Low | `scripts/build_findings.py:86-90`, notebook cell 3 | 用四舍五入后的发布列重新套阈值，得出的结论与 `rules.py` 用未取整值做的判定矛盾 |
| GEO-09 | Low | `download_gisborne_data.py:218-222` | `-part-N` 编号取决于部件顺序，与几何无关（376 个 ID 受影响） |
| GEO-10 | Low | `rules.py:82-86`, `load.py:53` | 唯一性按原始值检查，join 却用 `astype(str)`；`[1, "1"]` 会直接崩溃 |
| GEO-11 | Low | `load.py:28`, `tests/test_crs.py` | 复合 CRS（NZTM + NZVD2016）被误拒；CRS 的真实入口路径没有测试覆盖 |

---

## GEO-01 [High] 叠加实质性只用相对阈值，随尺度变化：大单元里公顷级保护地/pre-1990 林地被当作"碎片"放行，小单元却因亚 15 m 碎片被排除

**位置**：`src/ets_screening/rules.py:98-107`；`rules/rule_register.csv` R-03/R-04；README Finding 1（L27-36）；notebook cell 4（"This is a mapping-generalisation artefact, not a land-status finding"）

**问题**：
```python
pre1990_material = (
    (pre1990_area > config.minimum_overlap_area_m2)
    & (pre1990_ratio * 100 >= config.minimum_overlap_pct)
)
```
阈值是"候选单元面积的 1%"。LCDB 单元面积从 0.0001 ha 到 36,801 ha 不等（`findings.json`），所以 1% 对应的绝对面积相差 8 个数量级。README 把 <1% 的重叠统一称为 "sliver"（边界泛化碎片），但真实数据并不支持这个说法。

**证据**（`s06_material.py`、`s20_finding1.py`、`s15_sliver.py`，真实 Gisborne 数据）：
```
pre1990_overlap_m2 max m2 1468588.41  candidates with overlap >= 1 ha: 43  >=5 ha: 17  sum over those (ha): 990.9
conservation_overlap_m2 max m2 2972336.33  candidates with overlap >= 1 ha: 22  >=5 ha: 11  sum over those (ha): 660.1
                      unit_id     area_ha  conservation_overlap_m2  conservation_overlap_pct                  advisory_rule_ids
1922  lcdb1000035680-part-118  36801.4156               2972336.33                    0.8077  R-03-low-overlap|R-04-low-overlap
                      unit_id     area_ha  pre1990_overlap_m2  pre1990_overlap_pct                  advisory_rule_ids
1733   lcdb1000035378-part-11  16345.5133          1468588.41               0.8985  R-03-low-overlap|R-04-low-overlap
```
```
R-04 low-overlap (advisory) units: 78  total intersection ha: 679.3
 units with >=1 ha of DOC overlap: 25  their intersection ha: 668.6  statuses: {'candidate_review': 22, 'quarantine': 3}
 low-overlap units whose overlap contains a >=30 m wide core (not a boundary sliver): 22  core-bearing intersection ha: 654.5
```
README 称为 sliver 的 679.3 ha 里，668.6 ha（98.4%）集中在 25 个单元中，每个单元的重叠都 ≥1 ha；其中 654.5 ha 的重叠区内部有 ≥30 m 宽的核心。这些是成块的 DOC 保护地，不是边界碎片。`lcdb1000035680-part-118` 状态为 `candidate_review`，但里面有 297 ha 公共保护地。

反方向也存在（重叠区经 `buffer(-half)` 后为空，说明它处处窄于 2×half）：
```
R-04 excluded: n=224; overlap narrower than 15.0 m everywhere (pure boundary sliver): 20
R-03 material: n=691; overlap narrower than 15.0 m everywhere (pure boundary sliver): 73
# examples whose overlap is <10 m wide everywhere (buffer(-5) empty):
             unit_id  area_ha    status  overlap_m2   pct
80           lcdb1000004902   1.7761  excluded       882.1  4.97
```
LCDB 镜像本身按 15 m 做了简化（manifest `lcdb_geometry_note`），所以处处窄于 15 m 的重叠正落在位置误差范围内，却足以把 1.78 ha 的单元整个排除。

**影响**：候选队列里有 48 个单元（22 个涉保护地、43 个涉 pre-1990，部分重叠）各含 ≥1 ha 的冲突用地，仍被标为可审查候选；1 ha 本身就是 forest land 的最小面积。README Finding 1 的核心论断（"<1% = 泛化伪影"）被数据证伪。同时，小单元会被纯碎片误排除/误隔离。

**修复**：加一个绝对面积下限，并在算面积前先用形态学开运算去掉亚容差碎片，把两个量都记录下来。更好的做法是裁剪候选几何，而不是对整个单元下结论：
```python
@dataclass(frozen=True)
class RuleConfig:
    ...
    absolute_material_overlap_m2: float = 10_000.0   # 1 ha = forest-land minimum
    sliver_half_width_m: float = 7.5                  # half the 15 m LCDB generalisation

# in _overlap_metrics
inter = geometry.intersection(target)
if sliver_half_width_m > 0:
    inter = inter.buffer(-sliver_half_width_m).buffer(sliver_half_width_m).intersection(geometry)
areas.append(float(inter.area))

# in evaluate_rules
pre1990_material = (pre1990_area > cfg.minimum_overlap_area_m2) & (
    (pre1990_ratio * 100 >= cfg.minimum_overlap_pct)
    | (pre1990_area >= cfg.absolute_material_overlap_m2)
)
# Preferable: publish the residual geometry candidate.difference(overlay_union) and screen that.
```

---

## GEO-02 [Medium] R-02 实际等于 `2A/P ≥ 30`，erosion 从未起作用；2A/P 相对 MPI 平均宽度系统性偏低，直条带会被误隔离

**位置**：`src/ets_screening/geometry.py:23`（`return 2.0 * geometry.area / geometry.length`）、`rules.py:91`、README L48-53（"The erosion test is retained as the primary narrow-strip diagnostic"）

**问题**：
1. `r02 = core_pass & ~(ap_pass != core_pass)` 化简后就是 `core_pass & ap_pass`。对凸多边形，A ≤ r·P（r 为内切圆半径），因此 2A/P ≤ 2r，也就是 `ap_pass ⇒ core_pass`；对一般多边形只多出凹角扇形项，实际中同样成立。结果是 R-02 **完全由 2A/P 决定**，erosion 只是"装饰"。
2. 2A/P = A/(P/2) 用半周长代替 MPI 的中心线长度。对 w×L 的矩形，2A/P = wL/(w+L) < w；对紧凑形状最多低估一半。偏差最大的恰好是 30-34 m 宽的直条带（防护林带、河岸带），而宽度规则针对的正是这类形状。

**证据**（`s01_real.py`、`s03_convex.py`、`s16b_transect.py`）：
```
R02 == ap_pass for all rows: True          # 5,712 / 5,712
ap_pass & ~core_pass: 0
ALL: max ratio 2A/P / inscribed diameter = 0.9127
```
```
            unit_id  area_ha  width_ap_m  width_core_pass  transect_avg_20m  w_equiv_rect failed_rule_ids     status
   strip 33 x 304 m   1.0032      29.769             True              33.0          33.0            R-02 quarantine
   strip 31 x 400 m   1.2400      28.770             True              31.0          31.0            R-02 quarantine
dumbbell (demo G04)   2.0800      29.714             True              37.1          31.1            R-02 quarantine
```
（`transect_avg_20m` 是沿长轴每 20 m 做一次垂直截线得到的平均宽度，对这些轴对齐形状近似 MPI 方法。）平均宽度正好 33 m、31 m 的直条带，唯一失败原因是 R-02。真实数据中有 62 个"只因 R-02 失败"的单元，其中 28 个的等效矩形宽度 ≥30 m，而且都有 erosion 核心（`s02_width.py`）。

另外，demo 里的哑铃形 G04 按 20 m 截线平均约 37 m，可能**通过** MPI 的平均宽度标准。README 说哑铃形会"still failing MPI's formal centre-line average"，这一点需确认 (Needs confirmation)，要对照 MPI 指南中"最长路径中心线"的具体做法。

**影响**：R-02 的隔离结果（1,092 个）里混有平均宽度明显超过 30 m 的直条带；把 erosion 称作"主判据"具有误导性。隔离不等于拒绝，所以定为 Medium。

**修复**：用中心线法（骨架最长路径加 20 m 垂直截线）实现 MPI 近似；过渡期可以先把 2A/P 换成等效矩形宽度（对条带是精确的），再加上 erosion 作为必要条件：
```python
def width_equivalent_rectangle(geometry: BaseGeometry) -> float:
    """Width of the rectangle with identical area and perimeter (exact for strips)."""
    if geometry is None or geometry.is_empty or geometry.length <= 0:
        return math.nan
    A, P = geometry.area, geometry.length
    disc = (P / 4.0) ** 2 - A
    return math.sqrt(A) if disc <= 0 else P / 4.0 - math.sqrt(disc)

# rules.py: make the combination explicit and honest
out["r02_width_proxy_pass"] = out["width_core_pass"] & (out["width_eqrect_m"] >= cfg.width_threshold_m)
```
README 也应改写：说明 R-02 的判定等价于哪个量。

---

## GEO-03 [Medium] `validate_candidates` 的重叠检测容差比浮点噪声还小，会拒绝本来不重叠的真实数据

**位置**：`src/ets_screening/load.py:60-61`
```python
overlap_area = float(frame.geometry.area.sum() - frame.geometry.union_all().area)
if overlap_area > 1e-6:
```
**问题**：面积之和与 `union_all()` 面积之差受浮点和 OverlayNG 噪声影响。在 NZTM 坐标和几千个多边形的规模下，这个噪声约 1e-5 m²，符号随机。1e-6 的绝对阈值等于在拿噪声做判断。已提交的全量数据能通过，只是因为残差碰巧为负（-1.955e-05）。

**证据**（`s07b/s07c_overlap_tol.py`）：
```
random 50% subsets rejected: 11 / 30
random_state 25 n 2856 true pairwise overlap area sum: 0.0
InputValidationError: candidate polygons overlap by 0.000 m2; submitted mapping must not overlap
```
**影响**：用户对同一份数据取子区域或子集时（包括通过 ArcGIS 工具），流程有约 1/3 概率 fail-loud，而且报错信息"overlap by 0.000 m2"让人无从下手。反过来，真实的小面积重叠则可能被负噪声抵消。

**修复**：改成成对精确检测（已验证：真实子集返回 0，测试夹具能检出包含关系；耗时 1.89 s，`union_all` 需要 4.78 s）：
```python
def _pairwise_overlaps(frame, rel_tol=1e-9):
    geoms = frame.geometry.values
    left, right = frame.sindex.query(geoms, predicate="intersects")
    keep = left < right
    left, right = left[keep], right[keep]
    inter = shapely.area(shapely.intersection(geoms[left], geoms[right]))
    tol = np.maximum(1e-6, rel_tol * np.minimum(shapely.area(geoms[left]), shapely.area(geoms[right])))
    bad = inter > tol
    return float(inter[bad].sum()), list(zip(frame.index[left[bad]], frame.index[right[bad]]))[:5]

overlap_area, pairs = _pairwise_overlaps(frame)
if pairs:
    raise InputValidationError(f"candidate polygons overlap by {overlap_area:.3f} m2 at rows {pairs}")
```

---

## GEO-04 [Medium] R-01 按单个 LCDB 单元判定，相邻可种植单元不合并

**位置**：`src/ets_screening/rules.py:88`；数据准备 `download_gisborne_data.py:214-224`（explode 后各部件独立）；notebook cell 14 已部分承认"fragments … are not re-aggregated"

**问题**：法定的 "area of land of at least 1 hectare" 指的是林地面积，不是土地覆盖制图单元。相邻的 "Low Producing Grassland" 和 "Gorse and/or Broom" 可以作为同一块造林地，但每个单元都被单独拿去比 1 ha。规则登记表里写的 "adjacency is not tested" 指的是 MPI 的 15 m 邻接林地规则，和这里说的是两回事。

**证据**（`s05_adjacency.py`，只在"干净或仅 R-01 失败"的可种植单元之间建共边连通图）：
```
R-01-only fails in cluster>=1 ha of clean/R-01-only plantable units: 104
sum own area of the 104 (ha): 76.7
       unit_id                      lcdb_class  area_ha  cluster_ha
lcdb1000417456              Gorse and/or Broom   0.9987    45348.59
neighbours of lcdb1000417456 : [{'unit_id': 'lcdb1000035378-part-11', ..., 'area_ha': 16345.5133, 'failed_rule_ids': ''}]
```
0.9987 ha 的荆豆地与一个 16,345 ha 的干净候选单元共边，却因 R-01 被隔离。

**影响**：327 个"仅 R-01 失败"的单元中，104 个（76.7 ha）属于误隔离；另外也要注意，district 边界和 explode 产生的碎片都会遇到同样的问题。

**修复**（已在真实数据上跑通）：
```python
plantable = out[out["r05_lcdb_proxy_pass"] & out["r04_no_conservation_overlap"]]
blocks = gpd.GeoDataFrame(geometry=[plantable.union_all()], crs=out.crs).explode(index_parts=False, ignore_index=True)
blocks["block_area_ha"] = blocks.area / 10_000
pts = plantable[["unit_id"]].set_geometry(plantable.representative_point(), crs=out.crs)
hit = gpd.sjoin(pts, blocks, predicate="within")
out["contiguous_plantable_ha"] = out["unit_id"].map(hit.set_index("unit_id")["block_area_ha"])
out["r01_area_pass"] = (raw_area_ha >= cfg.minimum_area_ha) | (out["contiguous_plantable_ha"] >= cfg.minimum_area_ha)
```
（至少应该把 `contiguous_plantable_ha` 作为 advisory 字段输出。）

---

## GEO-05 [Medium] `run_screening` 忽略 `config.width_threshold_m`，发布的宽度对比与判定相互矛盾

**位置**：`src/ets_screening/screen.py:101`（`comparison = compare_width_methods(results)`，使用默认 30 m）、`report.py:122`（`axhline(30.0)` 写死）、`screen.py:158-161`（manifest 只记录 overlap 阈值，不记录宽度和面积阈值）

**证据**（`s12_config.py`，`RuleConfig(width_threshold_m=40.0)`）：
```
G11 (40 m x 300 m) in candidates/quarantine gpkg: width_ap_pass=False width_core_pass=False disagree=False r02=False status=quarantine
G11 in width_method_comparison.csv: {'width_area_perimeter_m': 35.294, 'area_perimeter_pass': True, 'erosion_core_pass': True, 'methods_disagree': False}
```
同一单元在 GPKG 里两种方法都判失败，在审计 CSV 里却都判通过；`summary.csv` 的 `width_method_disagreements` 和图也按 30 m 计算；`run_manifest.json` 无法说明这次运行用了 40 m。另外宽度被重复计算了一次（`compare_width_methods` 单次 4.4 s）。

**修复**：
```python
cfg = config or RuleConfig()
results, audit = evaluate_rules(candidates, pre1990, conservation, cfg)
comparison = results[["unit_id", "width_ap_m", "width_ap_pass", "width_core_pass", "width_methods_disagree"]].rename(
    columns={"width_ap_m": "width_area_perimeter_m", "width_ap_pass": "area_perimeter_pass",
             "width_core_pass": "erosion_core_pass", "width_methods_disagree": "methods_disagree"})
manifest["thresholds"] = dataclasses.asdict(cfg) | {"plantable_lcdb_classes": sorted(cfg.plantable_lcdb_classes)}
plot_width_comparison(comparison, path, threshold_m=cfg.width_threshold_m)
```

---

## GEO-06 [Medium，需确认 Needs confirmation] pre-1990 证据只取 LUCAS 72/72，漏掉 1989 年天然林 (71)

**位置**：`scripts/download_gisborne_data.py:205`（`where="LUCID_1989 LIKE '72%' AND LUCID_2007 LIKE '72%'"`）、`:229-232`

**问题**：`rules/SOURCES.md` 引用的 s 4(1) 规定，post-1989 forest land 必须满足以下之一：(i) 1989-12-31 不是 forest land；(ii) 1989 年是 forest land，但在 1990–2007 年间被毁林；(iii) pre-1990 forest land（2007 年以外来树种为主）在 2008 年后毁林并已清偿。1989 年为天然林 (LUCAS 71)、2007 年仍为天然林的土地，这三条都不满足；但它在 2023 年可能已成为草地或荆豆地，进入候选层。当前过滤只保留 72/72（pre-1990 人工林），这类土地不会被 R-03 标记。

**证据**：无法联网到 MfE 服务量化（`curl: (56) CONNECT tunnel failed, response 403`），已提交的 `gisborne_pre1990_evidence.gpkg` 中只有 72/72 记录（1,933 个）。LUCAS 代码语义（71 = Natural Forest）和法条适用需要领域专家确认。

**影响**：R-03 的自动证据层可能系统性遗漏一类不合格土地（R-03 仍列为人工复核项，这在一定程度上缓解了问题）。

**修复**：
```python
where=("(LUCID_1989 LIKE '71%' OR LUCID_1989 LIKE '72%') AND "
       "(LUCID_2007 LIKE '71%' OR LUCID_2007 LIKE '72%')"),
# keep LUCID_1989/LUCID_2007 in the output so R-03 can report evidence type per overlap
```

---

## GEO-07 [Low] README Finding 2 的形状解释与数值伪影

**位置**：README L41-60；notebook cell 7-8；`geometry.py:35`

**问题与证据**（`s02_width.py`、`s03_convex.py`、`s09b.py`）：
- "All 694 run in the same direction" 是 2A/P ≤ 内切圆直径（见 GEO-02）的必然结果，不是实证发现。
- README 把分歧归因于 "Long branching, dumbbell and highly concave polygons"，但：
```
convex (solidity>=0.99) disagreements: 293          # 42% of 694
  area_ha quantiles {0.0: 0.11, 0.25: 0.198, 0.5: 0.276, 0.75: 0.359, 1.0: 1.39}
  vertex count quantiles {0.0: 3.0, 0.5: 4.0, 1.0: 30.0}
  2A/P / inscribed diam ratio quantiles {0.0: 0.5, 0.5: 0.545, 1.0: 0.913}
```
这 293 个主要是小三角形和四边形，本来就不满足 R-01；它们之所以"分歧"，只是因为 2A/P 对紧凑形状约等于内切圆直径的一半。
- README 里的 "181 / 53 / 23" 可以复现（`core split into >=2 parts: 181`，>500 m²: 53，>1000 m²: 23），但仓库里没有任何代码生成这组数字（`build_findings.py`/notebook 都没有）。
- erosion 的数值边界：宽正好 30.000 m 的条带判失败，实际生效阈值是 30.0015 m（GEOS buffer 简化容差）；真实数据中有 11 个单元的 "core" 面积 <1 m²（其中 2 个 <0.01 m²），属于数值碎片。181 里有 8 个仅由 <1 m² 的碎片构成"分裂"。

**修复**：改写 README 的归因；把 split-core 统计写进 `build_findings.py`；核心判定改为有面积下限：
```python
def width_erosion(geometry, half_width_m=15.0, min_core_m2=1.0):
    ...
    return geometry.buffer(-half_width_m).area > min_core_m2
```

---

## GEO-08 [Low] 发布列经过四舍五入，下游用它重套阈值，结论与实际判定矛盾

**位置**：`rules.py:94-97`（`.round(2)` / `.round(4)`）；`scripts/build_findings.py:86-90`；notebook cell 3

**证据**（`s11_rounding.py`）：
```
  unit_id  pre1990_overlap_m2  pre1990_overlap_pct  r03_no_pre1990_overlap advisory_rule_ids            status
0       P               100.0               1.0000                    True  R-03-low-overlap  candidate_review
1       Q                 1.0               0.0001                    True  R-03-low-overlap  candidate_review
build_findings view  -> legacy: [True, False]  material: [True, False]  advisory(low): [False, False]
rules.py decision    -> material: [False, False]  advisory: [True, True]
```
P 的发布记录显示 "100 m² / 1.0000%"，按文档规则应属实质性重叠，状态却是 candidate_review。项目已经为 R-01 修过同类问题（`test_unrounded_area_controls_threshold_decision`），R-03/R-04 却没有同步处理。已提交的真实数据中目前没有触发这种情况（224 = 224，691 = 691）。

**修复**：把判定布尔值作为列发布（`r03_material`、`r04_material`、`r03_advisory`），`build_findings.py` 和 notebook 直接读这些列，不再用取整后的数值重新判定。

---

## GEO-09 [Low] `-part-N` 标识符取决于部件顺序，不取决于几何

**位置**：`scripts/download_gisborne_data.py:218-222`

**证据**（`s14_partids.py`，逐字复用该段逻辑）：
```
geometries equal: True
run1 area by unit_id: {'lcdb1-part-1': 90000.0, 'lcdb1-part-2': 2500.0}
run2 area by unit_id: {'lcdb1-part-1': 2500.0, 'lcdb1-part-2': 90000.0}
candidates with -part- ids: 376 of 5712
pinned review sample ids with -part-: []
```
**影响**：重新下载、升级 GEOS 或服务端部件顺序变化后，同一个 ID 可能指向另一块几何，与 README 所说的 "stable `unit_id`" 不符。目前固定的 30 个复核样本里没有 `-part-` ID，所以影响有限。

**修复**：
```python
rp = candidates.geometry.representative_point()
candidates = (candidates.assign(_neg_area=-candidates.area.round(3), _x=rp.x.round(3), _y=rp.y.round(3))
              .sort_values(["LCDB_UID", "_neg_area", "_x", "_y"], kind="mergesort")
              .drop(columns=["_neg_area", "_x", "_y"]).reset_index(drop=True))
```

---

## GEO-10 [Low] `unit_id` 唯一性按原始值检查，join 却按字符串

**位置**：`load.py:53`（`frame["unit_id"].duplicated()`）、`rules.py:82-86`（`.astype(str).map(...)`）

**证据**（`s10_index.py`）：
```
--- mixed-type unit_id [1, "1"]
CRASH: InvalidIndexError Reindexing only valid with uniquely valued Index objects
```
（重复的 DataFrame index `[0, 0]` 已测试，结果正确，不是问题。）

**修复**：`compare_width_methods` 的结果与 `out` 行序一致，直接按位置赋值即可：
```python
widths = compare_width_methods(out, config.width_threshold_m)
out["width_ap_m"] = widths["width_area_perimeter_m"].to_numpy()
...
# and in validate_candidates:
if frame["unit_id"].isna().any() or frame["unit_id"].astype(str).duplicated().any(): ...
```

---

## GEO-11 [Low] 复合 CRS 被误拒；CRS 的真实入口路径没有测试覆盖

**位置**：`load.py:28`；`tests/test_crs.py`

**证据**（`s08_crs.py`、`s18_compound.py`、`s19_crs_paths.py`）：CRS 守卫**没有误放行**（ESRI WKT 被接受；单位为英尺、中央经线错误的都被拒绝），但：
```
read back crs: NZGD2000 / New Zealand Transverse Mercator 2000 + NZVD2016 height | to_epsg(): None | is_compound: True
InputValidationError: candidates uses COMPD_CS["NZGD2000 / New Zealand Transverse Mercator 2000 + NZVD2016 height", ...
compound 2193+7839 (NZVD2016)            to_epsg()=None   units=metre      -> rejected
```
（另外，与 NZTM 等价的 proj4 定义 `+proj=tmerc +lon_0=173 +k=0.9996 +x_0=1600000 +y_0=10000000 +ellps=GRS80 +towgs84=0,…` 同样得到 `to_epsg()=None`，被拒绝。）
带 Z 值和垂直基准的 ArcGIS/GPKG 数据（EPSG:2193+7839）会被拒绝。`evaluate_rules` 对 overlay 层 CRS 错误、`crs=None`、`read_layer` 读到错误 CRS 这几种情况，代码行为都正确（实测报 `InputValidationError`），但没有一个测试覆盖；`test_crs.py` 只直接调用 `assert_nztm2000(frame(4326))`。

**修复**：
```python
crs = frame.crs
if crs.is_compound:
    crs = crs.sub_crs_list[0]          # horizontal component
if crs.to_epsg() != EXPECTED_EPSG: ...
```
并补充测试：`evaluate_rules(c, pre_in_4326, empty)`、`crs=None`、`read_layer(<gpkg in 4326>)`、复合 CRS。

---

## 性能（Info，不单列）
真实数据 5,712 个单元运行 `evaluate_rules` 需 12.3 s。其中 `validate_candidates`（`union_all`）5.1 s，两次叠加分别 0.42 s 和 1.98 s；宽度计算在 `run_screening` 里又重复一次（4.4 s，改成矢量化只需 2.4 s）。都基于 sindex，没有 O(n²) 循环；全国规模的瓶颈会是 `union_all` 校验，GEO-03 的修复同时解决了这一点。

## 做得好的地方
- `_overlap_metrics` 先用 sindex 选出相交要素，局部 `union_all()` 后再求交，避免了 overlay 自身重叠造成的重复计算；用真实面积而不是 `intersects`，仅边界接触时面积为 0（有测试）；实测重复 DataFrame index 下对齐也正确。
- CRS 策略是 fail-loud、不自动重投影，实测没有误放行（英尺单位、错误中央经线都被拒绝）；R-01 用未取整面积判定（有专门测试）。
- 候选层入口拒绝 null/empty/invalid/multipart/重叠几何；数据准备阶段做了 `make_valid` 和按边界裁剪；隔离输出保留规则 ID、重叠面积和比例，不静默删除几何。
- 宽度两种代理方法并列输出、分歧升级到人工复核的思路是对的（问题在于 2A/P 本身的偏差和对分歧成因的解释，见 GEO-02/07）。
