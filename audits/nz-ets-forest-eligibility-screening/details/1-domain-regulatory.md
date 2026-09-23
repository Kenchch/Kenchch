# 领域 / 法规忠实度审计：nz-ets-forest-eligibility-screening @ 622e3b8

审计角度：规则实现是否忠实反映《Climate Change Response Act 2002》(CCRA) s 4 以及 MPI / Te Uru Rākau 指南；工具有没有说多或说少。

## 方法与证据范围说明

- 通读了 `rules/rule_register.csv`、`rules/SOURCES.md`、`src/ets_screening/{rules,screen,geometry,load,report}.py`、`scripts/{download_gisborne_data,build_findings,reproduce}.py`、`README.md`、`data/README.md`、`arcgis/ets_screening.pyt`、`tests/test_rules.py`。
- 用共享 venv **只读**加载了 `data/processed/*.gpkg` 和 `outputs/gisborne/*`，另外在 scratchpad 里构造了合成几何，直接调用 `evaluate_rules` 复现问题。仓库里没有写入任何东西。
- **联网核对受限**：WebFetch 访问 `www.legislation.govt.nz`、`www.mpi.govt.nz`、`www.simpsongrierson.com` 时都返回 `EGRESS_BLOCKED`，curl 被代理以 403 拒绝。所以下文的法条原文来自仓库 `rules/SOURCES.md` 的逐字引文（结构上与我所知的 s 4 一致），MPI 指南和 2025 修正法的内容来自 **WebSearch 返回的摘要**。这些页面我没能直接打开，凡是只靠搜索摘要支撑的结论，都标了"需确认"或注明"搜索摘要"。搜索中出现的 URL：
  - MPI LUC 限制：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/about-forestry-in-the-emissions-trading-scheme-ets/ets-land-use-capability-class-restrictions
  - MPI 不受限土地：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/about-forestry-in-the-emissions-trading-scheme-ets/ets-land-use-capability-class-restrictions/post-1989-forest-land-that-is-not-restricted-from-entering-the-ets
  - MPI 新闻：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/news-and-changes-to-the-ets/limits-to-restrict-farm-to-forest-conversions
  - Climate Change Response (Emissions Trading Scheme—Forestry Conversion) Amendment Act 2025：https://www.legislation.govt.nz/act/public/2025/52/en/latest/
  - MPI post-1989 / pre-1990 页面：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/about-forestry-in-the-emissions-trading-scheme-ets/post-1989-forest-land ，…/pre-1990-forest-land
  - MPI 映射土地合格性：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/mapping-and-managing-forest-land-in-the-ets/making-sure-mapped-land-is-eligible-for-the-emissions-trading-scheme
  - MPI 森林地定义：https://www.mpi.govt.nz/forestry/forestry-in-the-emissions-trading-scheme/about-forestry-in-the-emissions-trading-scheme-ets/how-forest-land-is-defined-in-the-ets
  - MfE LUCAS 2020 LUM 技术报告（类别定义）：https://environment.govt.nz/assets/publications/land/Establishing-New-Zealands-LUCAS-2020-Land-Use-Map.pdf

---

## 发现汇总

| ID | 严重度 | 标题 |
|---|---|---|
| DOM-01 | High | R-03 的 pre-1990 证据只用 LUCAS 72（人工林），漏掉 71 Natural Forest（1989 年天然林是法定硬性排除） |
| DOM-02 | High | 相对 1% 重要性阈值让约 1,006 ha 已测绘的 pre-1990 人工林留在 `candidate_review` 中，且 R-03 记为 `passed=True` |
| DOM-03 | High | 2025 年 10 月 31 日生效的 LUC 1–6 外来林注册限制未列入规则登记册，而候选地块全部是农地 |
| DOM-04 | Medium | R-01/R-02 按 LCDB 单元孤立判定：307 个仅因面积失败的可种植单元与邻接单元合计 ≥1 ha |
| DOM-05 | Medium | R-02 实际起决定作用的是向下偏的 2A/P，而不是文档所称的"主判据"erosion；恰好 30 m 宽也判失败 |
| DOM-06 | Medium | R-04 整单元"excluded"：65 个单元（3,237 ha）仅因 90.9 ha 河岸 marginal strip 被整块排除 |
| DOM-07 | Medium | `rule_results.csv` 夸大可测性，并丢失了判定依据（testable=True、R-03 advisory 记为 passed、R-02 observed 与 passed 矛盾） |
| DOM-08 | Low | R-05 白名单里 "Bare or Lightly Vegetated Surfaces" 在真实数据中命中 0 个；类别词表没有校验 |
| DOM-09 | Low | 适用范围和其他法定门槛缺失：post-1989 定义 para (b)、可注册权益、已建立的 post-1989 林 |
| DOM-10 | Low | 术语和引用精度：R-07 "at maturity"、R-02 把 (c)(i)/(c)(ii) 混为一谈、R-04 法律依据表述不一致 |
| DOM-11 | Low | 15 m 简化同样影响 R-01 的阈值附近判定，但限制说明里只提到窄形状 |

---

## DOM-01 [High] R-03 的 pre-1990 证据只用 LUCAS 72（人工林），漏掉 71 Natural Forest

**位置**：`scripts/download_gisborne_data.py:200-206`、`:226-232`；`rules/rule_register.csv:4`；`rules/SOURCES.md:140-145`；`data/README.md:11`

**问题**：post-1989 forest land 的主要途径 para (a)(i) 要求土地在 "was not forest land on 31 December 1989"。1989 年是任何类型森林（天然林或人工林）的土地都不满足 (a)(i)。1989 年是天然（本土）林、2007 年仍是天然林的土地，还有更严格的后果：它不是 pre-1990 forest land（para (a)(i)(C) 要求 2007 年以外来树种为主），也不能通过 (a)(ii)（1990–2007 年间毁林）或 (a)(iii)（针对 pre-1990 forest land）进入 post-1989。也就是说，这类土地**永远不能**注册为 post-1989 forest land。工具的证据层只下载了 1989 年和 2007 年都是 72 类的多边形：

```python
# download_gisborne_data.py:205
where="LUCID_1989 LIKE '72%' AND LUCID_2007 LIKE '72%'",
```

**证据**：
- 读取 `data/processed/gisborne_pre1990_evidence.gpkg`，结果是 `LUCID_1989 {'72 - Planted Forest - Pre 1990': 1933}`、`LUCID_2007 {'72 - Planted Forest - Pre 1990': 1933}`。证据层中没有 71 类。
- MfE LUCAS 2020 LUM 技术报告（搜索摘要）的定义："Pre-1990 natural forest is defined as self-sown indigenous or exotic trees which occur on land which was forest land at 1990 … also includes wilding pines"。
- MPI 页面（搜索摘要）："Land that was native forest on 31 December 1989 and remained native forest on 31 December 2007 is not pre-1990 forest land … cannot be registered as post-1989 forest land"。
- 同一过滤条件还漏掉 71→72（1989 年天然林、2007 年已改种人工林）的情况。按定义这是 pre-1990 forest land。

**影响**：凡是 2023 年 LCDB 为草地或灌丛、但 LUCAS 1989/2007 为天然林的单元（2008 年后清除的天然林、manuka/kanuka、LUCAS 71 中的野生松，以及边界错配处），都会以 R-03 `passed=True` 进入 `candidate_review`。这是一种**法定硬性排除**被系统性漏检的情况，属于"说多"（假阳性），方向与项目自称的保守原则相反。仓库里没有 71 类数据，代理又阻止了查询，所以影响的具体单元数**需确认**。另外，规则名 "Not materially mapped as pre-1990 forest land" 和 README 中 "R-03 post-1989 rather than mapped pre-1990" 给读者的印象是覆盖了全部 1989 年森林。

**修复**：下载 1989 年为 71 或 72 的全部多边形，并按法律途径分级：

```python
lucas = _cached_download("lucas_v005_bbox", LUCAS_URL, bbox,
    "OBJECTID,LUCID_1989,LUCID_2007,LUCID_2020",
    where="LUCID_1989 LIKE '71%' OR LUCID_1989 LIKE '72%'")
c89 = lucas["LUCID_1989"].str[:2]; c07 = lucas["LUCID_2007"].str[:2]
lucas["evidence_class"] = np.select(
    [c89.eq("71") & c07.eq("71"),          # 1989 与 2007 均为天然林：post-1989 硬性排除
     c07.eq("72")],                        # 1989 为林地且 2007 为人工林：pre-1990 forest land 证据
    ["natural_1989_2007_never_post1989", "pre1990_planted"],
    default="forest_1989_deforested_by_2007")  # 可能走 (a)(ii) 途径：仅作 advisory
```

R-03 对前两类判 material，第三类记为 advisory（`R-03-a-ii-route`）。同步更新 rule_register R-03 的 implementation 列，以及 SOURCES.md:140-145 的说明。

---

## DOM-02 [High] 相对 1% 阈值让约 1,006 ha pre-1990 人工林留在 `candidate_review`，且 R-03 记为 `passed=True`

**位置**：`src/ets_screening/rules.py:98-112`、`:140-151`；`rules/rule_register.csv:4-5`；`README.md:32-38`

**问题**：重要性判定用的是 `overlap > 1 m² AND overlap/unit ≥ 1%`。LCDB 单元最大可达 36,801 ha，1% 的相对阈值相当于数百公顷，而森林地的法定最小面积只有 1 ha。所以一个单元内部即使含有**本身就满足 1 ha 门槛**的 pre-1990 人工林，只要占比不到 1%，整单元仍判 R-03 通过。

**证据**（对已提交输出只读计算）：
```
candidates with pre1990 overlap >=1ha: 43   sum ha 990.9
total pre1990 overlap ha in candidates 1006.0
lcdb1000035378-part-11  16345.5 ha  pre1990_overlap 146.9 ha (0.8985%)  -> candidate_review
lcdb1000035680-part-70  17728.4 ha  pre1990_overlap 145.9 ha (0.8229%)  -> candidate_review
可种植且未被 excluded 的单元中，pre-1990 重叠面积：candidate_review 1006.0 ha vs quarantine 2004.0 ha
candidates with conservation overlap >=1ha: 22   sum ha 660.1   (最大 297.2 ha，lcdb1000035680-part-118)
```
也就是说，可种植单元内已测绘的 pre-1990 人工林中，约 33% 落在"候选"状态。合成用例复现如下：一个 400 ha 单元内含 3 ha 的 pre-1990 林。
```
unit_id  area_ha  pre1990_overlap_m2  pct   r03_no_pre1990_overlap  advisory_rule_ids  status
big400ha 400.0    30000.0             0.75  True                    R-03-low-overlap   candidate_review
rule_results: big400ha,R-03,...,testable_from_open_data=True,passed=True,observed=True
```

**影响**：README 的 Finding 1 把 1% 规则描述成对"几平方米的边界错配"的修正。但在大单元上，它会把几十到上百公顷**真实的** pre-1990 森林证据降级为 advisory，而审计表 `rule_results.csv` 对这些单元写的是 R-03 `passed=True`。这是"说多"：审计链会被读成"R-03 已通过"。

**修复**：在相对阈值之外加一个与法定最小面积挂钩的绝对阈值，并且不再对 advisory 情况写 `passed=True`：
```python
# RuleConfig
absolute_material_overlap_m2: float = 10_000.0   # = CCRA s4 forest land 最小 1 ha
pre1990_material = (pre1990_area > cfg.minimum_overlap_area_m2) & (
    (pre1990_ratio * 100 >= cfg.minimum_overlap_pct)
    | (pre1990_area >= cfg.absolute_material_overlap_m2))
# long table: advisory -> passed = pd.NA, observed = f"{m2:.0f} m2 / {pct:.3f}%"
```
更好的做法是像 README 自己建议的那样：用 `geometry.difference(overlay_union)` 把重叠部分切出去单独处置，剩余部分重新跑 R-01/R-02。R-04 同样适用。

---

## DOM-03 [High] 2025 年 10 月 31 日生效的 LUC 1–6 外来林注册限制未列入规则登记册

**位置**：`rules/rule_register.csv`（没有对应行）；`rules/SOURCES.md:171-173`（只是一句"notes LUC-class restrictions"）；`README.md:90-101` 规则表；`src/ets_screening/rules.py:127`（`manual_review_rule_ids`）；`src/ets_screening/screen.py:165`（`unresolved_rules`）

**问题**：根据 Climate Change Response (Emissions Trading Scheme—Forestry Conversion) Amendment Act 2025（2025 No 52），自 **2025 年 10 月 31 日**起，LUC 1–6 类农地转为外来林后注册为 post-1989 forest land 受到限制。以下内容来自搜索摘要：每个农场可注册的外来林不超过其 LUC 1–6 土地的 25%；LUC 6 另有年度抽签许可（2026 年为 7,500 ha）；豁免包括 LUC 7–8、exempt Māori land、2025 年 10 月 31 日已是 forest land 的土地、区域或地区规划中划定的高或极高侵蚀易发地、Crown afforestation、未测绘和非耕作土地；本土林不受限。具体条文编号**需确认**。

工具的候选集**全部**是 LCDB 草地或灌丛：
```
Gorse and/or Broom 345 units / 2,243 ha; High Producing Exotic Grassland 1,346 / 281,439 ha;
Low Producing Grassland 971 / 21,092 ha; Mixed Exotic Shrubland 25 / 415 ha; total 305,189 ha
```
这正是这项法律针对的"farm-to-exotic-forest conversion"。SOURCES.md 已经注意到这项限制，但它既不在规则登记册里，也不在人工复核列表（`R-03|R-06|R-07|R-08`）或 README 的决策边界表里。

**影响**：对 2025 年 10 月 31 日以后的新造林（候选地块都属于这种情况），这是一道实际的注册门槛。输出没有提示这一点，会让读者高估 2,687 个候选的注册可行性，属于"说多"。而且这道门槛**可以用开放数据部分检验**：NZLRI LUC（LRIS）、Tairāwhiti 规划中的侵蚀易发叠加层，MPI 本身也发布了 LUC 测绘方法和 property-scale LUC shapefile schema（dmsdocument 70756，搜索摘要）。

**修复**：
1. rule_register 新增一行：`R-09,Exotic registration restriction on LUC 1–6 land,"CCRA as amended by 2025 No 52 (restricted land, 25% farm allowance, LUC 6 permits, exemptions)",<MPI LUC 页面>,partial,"NZLRI LUC overlay: luc_1_6_pct, luc_6_pct; regional erosion-prone overlay","Flag (not quarantine) R-09-exotic-restricted; indigenous species, exemptions and farm-level 25% allowance need manual evidence"`。
2. `evaluate_rules` 增加 `luc` 叠加层（复用 `_overlap_metrics`），输出 `luc_1_6_pct`、`erosion_prone_pct`，并把 R-09 加入 `manual_review_rule_ids` 和 `unresolved_rules`。
3. README 决策边界表补一行。在注明时点时说明 LCDB 2023/24 无法证明某地块"2025 年 10 月 31 日已是 forest land"。

---

## DOM-04 [Medium] R-01/R-02 按 LCDB 单元孤立判定，有 307 个可种植单元仅因面积失败，而与邻接单元合计 ≥1 ha

**位置**：`src/ets_screening/rules.py:88-91`；`rules/rule_register.csv:2-3`（已声明未测邻接）；`rules/SOURCES.md:255-265`；`scripts/download_gisborne_data.py:195-199`（森林类别根本没有下载）

**问题**：CCRA 的 1 ha 和平均宽度检验针对的是**森林（树冠覆盖）区域**，不是土地覆盖制图单元。LCDB 在 "High Producing Exotic Grassland" 和 "Gorse and/or Broom" 之间画的类别边界，与一片新造林的范围没有法律上的关系。para (c)(ii) 的 contiguity 例外和 MPI 的 15 m 规则（搜索摘要："small areas of trees (less than 1 hectare) more than 15 metres from adjacent forest" 才被排除）都要求把相邻区域一起考虑。仓库已诚实声明"adjacency is not tested"，但既没有量化影响，也没有下载现存森林类别，(c)(ii) 与现有林相接的情形因此完全无从评估。

**证据**：
```
plantable units failing R-01: 811
plantable units whose only failures are R-01 (and possibly R-02): 716
... with at least one other plantable unit within 15 m: 314   touching: 284
... union >= 1 ha: 307
with candidate_review neighbour within 15 m: 274
例：lcdb1000417456 Gorse and/or Broom 0.9987 ha，紧邻 16,345 ha 的 candidate_review 单元 → quarantine
```
合成用例：两个相接的 0.6 ha 草地单元，两者都是 `R-01 → quarantine`，但合计 1.2 ha。

**影响**：在仅因 R-01（±R-02）被隔离的单元中，约 43%（307/716）只是制图单元切分造成的，属于"说少"（假隔离），而且会稀释人工复核队列。方向保守，所以评为 Medium。

**修复**：先把可种植单元聚合成"潜在造林块"，再在块上检验 R-01/R-02，保留单元级结果用于审计：
```python
plant = out[out["r05_lcdb_proxy_pass"]]
merged = plant.geometry.buffer(7.5).union_all()            # 15 m 间隙容差
blocks = gpd.GeoDataFrame(geometry=list(getattr(merged, "geoms", [merged])), crs=2193)
blocks["block_id"] = [f"B{i:05d}" for i in range(len(blocks))]
j = gpd.sjoin(plant[["unit_id", "geometry"]], blocks, predicate="intersects")
block_area = j.assign(a=plant.set_index("unit_id").loc[j.unit_id].area.values).groupby("block_id").a.sum()
out["block_id"] = out.unit_id.map(j.set_index("unit_id").block_id)
out["r01_block_pass"] = out.block_id.map(block_area).fillna(0) >= 10_000
# R-01 单元失败但块通过 -> advisory "R-01-block-pass"，不整单元 quarantine
```
另外，把 LCDB 的 Exotic Forest、Forest - Harvested、Manuka and/or Kanuka、Indigenous Forest 作为"adjacent forest"上下文层下载，用于标记 (c)(ii) 或 15 m 路径。

---

## DOM-05 [Medium] R-02 实际起决定作用的是向下偏的 2A/P，而不是文档所称的"主判据"erosion；恰好 30 m 宽也判失败

**位置**：`src/ets_screening/geometry.py:190`、`:202`、`:211-215`；`src/ets_screening/rules.py:91`；`README.md:48-52`、`:95`

**问题**：
1. `r02_width_proxy_pass = width_core_pass & ~methods_disagree`，等价于 `core_pass AND ap_pass`。已提交的 findings 显示 `ap_pass_erosion_fail = 0`，所以在真实数据上 **R-02 通过 ⇔ 2A/P ≥ 30 m**，erosion 从未单独决定过结果。README 称 erosion "retained as the primary narrow-strip diagnostic"，这个说法有误导性。
2. 2A/P 对平均宽度有系统性低估：对 W×L 的矩形，2A/P = WL/(W+L) < W。400 m 长的带状地需要约 32.4 m 真宽才能通过；100 m 长则需要约 42.9 m。
3. 法条只排除 "average width of less than 30 metres"，即 ≥30 m 不应被排除。但 `buffer(-15)` 对恰好 30 m 宽的形状返回空，判失败。

**证据**（合成用例，直接调用 `evaluate_rules`）：
```
strip30 (30×400 m, 1.2 ha): width_ap_m 27.907, core False           -> quarantine
strip32 (32×400 m):         width_ap_m 29.630, core True, disagree  -> quarantine
strip35 (35×400 m):         width_ap_m 32.184                       -> candidate_review
```
真实数据：
```
plantable, >=1 ha units failing R-02: 72  (全部 erosion core 存在，只因 2A/P<30 失败)
  equivalent-rectangle width (P/4 - sqrt(P²/16 - A)) >= 30 m: 31
R-02-only quarantines: 62, 其中等效矩形宽度 >=30 m: 28
```
等效矩形宽度只是我用来演示的近似，不是 MPI 的中心线法。

**影响**：属于"说少"（假隔离），方向保守；但文档对判据的描述与实际不符，审计者会误以为 erosion 是决定因素。

**修复**：
```python
def width_equivalent_rectangle(g):             # 对矩形精确，偏差远小于 2A/P
    A, P = g.area, g.length
    d = P * P / 16.0 - A
    return P / 4.0 - math.sqrt(d) if d >= 0 else math.sqrt(A)
def width_erosion(g, half_width_m=15.0, tol=1e-3):
    return not g.buffer(-(half_width_m - tol)).is_empty   # 法条 "less than 30 m" 才排除
```
更进一步可以实现近似 MPI 方法：用 skeleton 求最长路径作中心线，每 20 m 做垂线求平均宽度。在 README 中写明哪个判据是 binding 的。

---

## DOM-06 [Medium] R-04 整单元 "excluded"：65 个单元（3,237 ha）仅因 90.9 ha 河岸 marginal strip 被整块排除

**位置**：`src/ets_screening/rules.py:102-107`、`:130`；`rules/rule_register.csv:5`；`data/README.md:12`；`rules/SOURCES.md:249-253`

**问题**：DOC 图层 410 个要素中有 236 个是 `MARGINAL_STRIP`（232 个 `S24_3_FIXED_MARGINAL_STRIP`），也就是沿水体的窄条 Crown 保护地。R-04 只要重叠 ≥1% 就把**整个** LCDB 单元标为 `excluded`，而且这个状态会覆盖 quarantine。法律上：MPI（搜索摘要）"Forest on Crown land is ineligible if you are not party to a Crown conservation contract"。因此排除 DOC 那一部分对私人申请人是有依据的，但依据不延伸到相邻的私有地。与此同时，`data/README.md:12` 写的是 "conservation tenure is not a statutory ETS eligibility rule"，低估了"可注册权益"（ownership / forestry right / lease / Crown conservation contract）这层法律基础。

**证据**：
```
excluded: 224, plantable: 125
plantable excluded solely by marginal strips: 65 units, 3,237.1 ha, strip overlap 90.9 ha
plantable excluded with overlap <50%: 85, non-DOC remainder 3,788.0 ha
plantable excluded whose only failure is R-04 and remainder >=1 ha: 51
例：lcdb1000035680-part-96  1,531.0 ha, DOC overlap 1.46% -> excluded
合成：400 ha 农地 + 沿边 20 m 宽 DOC 条（恰好 1%）-> status excluded
```

**影响**：约 3,788 ha 非 DOC 的私有剩余地被移出机会队列（"说少"）。同时，"excluded" 这个状态名暗示有法定依据，对剩余部分则属于"说多"。

**修复**：用 `geom.difference(doc_union)` 切出 DOC 部分，只把切出部分标为 `excluded`，剩余部分以新的子单元 ID 重新跑 R-01/R-02。rule_register R-04 的 source_clause 改为引用 MPI 的 Crown land / Crown conservation contract 要求，并与 `data/README.md:12` 的表述保持一致。

---

## DOM-07 [Medium] `rule_results.csv` 夸大可测性，并丢失了判定依据

**位置**：`src/ets_screening/rules.py:133-151`（`"testable_from_open_data": True`、`"observed": row[observed_col]`）

**问题和证据**：
- 规则登记册中 R-02 为 `proxy`、R-03 为 `partial`、R-05 为 `proxy`，但审计表对 R-01 到 R-05 一律写 `testable_from_open_data=True`（全量检查：`R-01..R-05 [True]`）。
- R-02 的 `observed` 列写的是 `width_core_pass`，不是宽度值：`passed=False, observed=True` 共 694 行，读者看到的是自相矛盾的记录。
- R-03 和 R-04 的 `observed` 就是 pass 布尔值本身（`passed` 与 `observed` 完全一致），没有记录重叠面积或百分比；有 advisory 重叠的单元写 `passed=True`（见 DOM-02）。

**影响**：`rule_results.csv` 是面向审计者的逐条证据链，它的措辞比 register 和 README 更"确定"，属于说多。

**修复**：`testable_from_open_data` 直接从 `rule_register.csv` 读取。`observed` 按规则记录实际量：R-01 记 `area_ha`，R-02 记 `f"2A/P={w:.1f} m; core={core}"`，R-03/R-04 记 `f"{m2:.0f} m2 ({pct:.3f}%)"`。有 advisory 时，`passed` 写 `pd.NA` 或新增 `outcome` 列（`pass/fail/advisory/manual`）。

---

## DOM-08 [Low] R-05 白名单中的类别在真实数据中命中 0 个，类别词表没有校验

**位置**：`src/ets_screening/rules.py:20-30`；`scripts/download_gisborne_data.py:43-66`、`:195-199`

**问题和证据**：`data/processed/gisborne_candidates.gpkg` 的类别计数中**没有** "Bare or Lightly Vegetated Surfaces"（5,712 个单元里是 0），但它在默认 allow-list 里。镜像服务的类别名（如 "Gravel and Rock"、"Sand and Gravel"、"Not land"）看起来与 LRIS LCDB 的规范类名（我所知为 "Gravel or Rock"、"Sand or Gravel"）不同，这一点**需确认**：LRIS 无法访问。SOURCES.md:232 称"classifies units by `Name_2023`"，默认两者一致。LCDB v6 的 "Landslide" 类在 Tairāwhiti 侵蚀区很常见（搜索摘要显示 v6 新测绘了约 2,800 ha 滑坡，具体类名**需确认**），它既不在候选名单也不在对照名单里，会被 `where Name_2023 IN (...)` 静默丢弃。

**影响**：R-05 的代理映射有一个空转条目，任何词表漂移都不会报错。影响有限，因为 R-05 本身只是项目代理，但这削弱了"R-05 is tested rather than made tautological"的说法。

**修复**：下载阶段用 `returnDistinctValues=true&outFields=Name_2023,Class_2023` 获取词表；`RuleConfig` 中任何 allow-list 名称不在词表里就 `raise`；以 `Class_2023` 整数代码而不是名称做映射；把类别到 R-05 的映射表写进 rule_register 或 SOURCES。

---

## DOM-09 [Low] 适用范围和其他法定门槛缺失

**位置**：`rules/rule_register.csv`；`README.md:3-7`、`:11-13`；`src/ets_screening/rules.py:127`；`src/ets_screening/screen.py:48`、`:165`

**问题**：
1. post-1989 forest land 定义 para (b)（"is not area 1 (approved) land … or P90 offsetting land"，SOURCES.md:108-109 已逐字引用），以及土地是否已注册，在 register 和人工复核列表里都没有出现。
2. 可注册权益（landowner、registered forestry right / lease、Crown conservation contract）只在 README 的免责说明里提了一句，没有规则 ID，所以不会出现在逐单元的 `manual_review_rule_ids` 中。
3. 按法条，只有 "forest land" 才能是 post-1989 forest land。R-05 只选非森林覆盖，因此输出的是**潜在造林机会**，不是"post-1989 forest-land cases"（README:12）。已经存在但尚未注册的 post-1989 林（再生 manuka/kanuka、1990 年后种植的人工林），这正是 post-1989 注册的主要来源之一，却被完全排除在输入之外。README 没有说明这一点。

**修复**：register 增加 R-10（para (b) / 已注册状态，manual）和 R-11（registrable interest，manual），并加入 `manual_review_rule_ids` 和 `unresolved_rules`，同时让 `manual_rules_out_of_scope` 从 register 计数，不再硬编码为 "3"。README:12 改为 "prioritising potential **afforestation** sites that could become post-1989 forest land"，并在 Limitations 里注明已建立的 post-1989 林不在筛查范围内。

---

## DOM-10 [Low] 术语和引用精度

**位置**：`src/ets_screening/rules.py:154`；`README.md:6`；`rules/rule_register.csv:3`

- `("R-07", "Crown cover at maturity")`：法条是 "has, or is likely to have, tree crown cover … of more than 30% in each hectare"，没有 "at maturity"。"at maturity" 属于 forest species 的 5 m 高度检验。README:6 的 "canopy cover and height at maturity" 同样把两者混在一起。建议改为 "Crown cover >30% in each hectare (has or likely to have)"。
- register R-02 的 source_clause 写的是 "para (c)(i)-(ii) (… unless contiguous with …)"，但法条中 contiguity 例外只附在 (c)(ii) 上，(c)(i) shelter belt 没有这个例外。MPI 的通俗页面对 shelter belt 也写了 "unless they join onto other forest land"。建议分开引用，并注明两种来源的差异。
- R-04 的法律依据表述不一致，见 DOM-06。

---

## DOM-11 [Low] 15 m 简化同样影响 R-01 阈值附近的判定，但限制说明里只提窄形状

**位置**：`README.md:248-254`；`data/processed/gisborne_manifest.json`（`lcdb_geometry_note`）

**证据**：可种植单元中，面积在 [0.95, 1.05) ha 的有 92 个，在 [0.9, 1.1) ha 的有 177 个，中位周长约 523 m。平均边界位移 1 m 就会带来约 523 m²（约 5%）的面积变化，所以在 15 m 简化容差下，这些单元的 R-01 判定不可靠。README 只说简化 "can affect narrow-feature diagnostics"。

**修复**：在 Limitations 里补充 R-01 阈值附近的敏感性。对 |area − 1 ha| < 10% 的单元加 advisory `R-01-near-threshold`，直到改用 LRIS 原始几何重跑。

---

## 本领域观察到的优点

- **免责定位清晰且贯穿各处**："screening/triage only; not an eligibility determination" 出现在 README 开头、`run_manifest.json`、PDF 版面、ArcGIS 工具描述和概览图中。R-04/R-05 被明确标为项目策略而非法定规则，量化结果也没有冒充合格性认定。
- **法源工作扎实**：SOURCES.md 逐字引用 s 4 的四个定义，注明版本日期、失效锚点（LMS282060）和"未并入修正"的警示。正确指出 crown cover 是 "in each hectare" 而不是整体平均、(c)(ii) 有 contiguity 例外，也正确描述了 MPI 中心线 20 m 间距的方法和 15 m 路径。
- **LCDB 没有 1990 时点的问题处理得当**：用 LUCAS 1989/2007 作为 31 Dec 1989 / 31 Dec 2007 两个法定日期的代理，并明确说明 LUCAS 不能证明负债清偿、exempt land 等状态，R-03 也始终保留在人工复核列表中。
- **阈值方向基本正确，保守失败都进入人工复核**：R-01 用未取整面积 `>= 1 ha`（与 "at least 1 hectare" 一致），用测试证明 0.99996 ha 会失败；只接触边界不算重叠；宽度方法分歧会升级为隔离而不是被静默放行。
