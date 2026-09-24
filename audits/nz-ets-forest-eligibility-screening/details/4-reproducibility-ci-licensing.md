# 审计报告：可复现性、数据溯源、CI/DevOps 与许可（REP）

审计对象：`Kenchch/nz-ets-forest-eligibility-screening` @ `622e3b8`（浅克隆）
审计范围：可复现性 / 数据溯源 / CI 与 DevOps / 许可与署名
方法：把仓库 `cp -r` 到 scratchpad，所有运行都在副本里做，原仓库未改动（`git status` 为空）。测试用了 4 套环境：
- `venv312`：Python 3.12.3，依赖为最新版（geopandas 1.1.4、shapely 2.1.2/GEOS 3.13.1、pyogrio 0.13.0/GDAL 3.12.4、pyproj 3.8.0/PROJ 9.8.1、pandas 3.0.6、numpy 2.5.3、matplotlib 3.11.2）。它与 CI 日志里的版本基本一致，只有 pandas 不同（CI 为 3.0.5）。
- 共享 `venv`：Python 3.11.15，pyproj 3.7.2/PROJ 9.5.1，其余同上。
- `venv_low`：用 `uv pip install --resolution lowest-direct` 装的声明下限版本（geopandas 1.0.0、shapely 2.0.0/GEOS 3.11.1、pyogrio 0.10.0/GDAL 3.9.1、pyproj 3.7.0/PROJ 9.4.1、pandas 2.2.2、matplotlib 3.9.0）。numpy 需要手工降到 `<2` 才能运行。
- GitHub Actions：通过 GitHub API 读取了最近一次 CI 运行（run 35089350141）的完整日志。

外网限制：代理拒绝了对 `services*.arcgis.com`、`arcgis.mfe.govt.nz`、`tiles.arcgis.com`、`basemaps.linz.govt.nz`、`cdn.jsdelivr.net` 的 CONNECT（403），所以不能在线重跑 `download_gisborne_data.py` / `download_review_cards.py`，也查不到服务的 `maxRecordCount`/`copyrightText`。涉及这些的结论都标了 **需确认 (Needs confirmation)**。`registry.npmjs.org` 可以访问，已用它核对 vendored Leaflet。

---

## 基线结果：按 ci.yml 原样执行

```
$ pytest                         -> 55 passed in 8.50s
$ python scripts/verify_checksums.py -> 7 × ok
$ python scripts/reproduce.py    -> rc=0 (22 s)
$ git diff --exit-code -- "outputs/gisborne/*.csv" "outputs/gisborne/findings.json" "outputs/gisborne/review/*.csv"
diff exit=0
$ python scripts/semantic_hashes.py -> 3 × ok
```
CI 全部通过，但工作树里有 8 个已提交文件被改写了：

```
$ git diff --stat
 outputs/gisborne/candidates.gpkg                   | Bin 4239360 -> 4239360 bytes
 outputs/gisborne/figures/screening_overview.png    | Bin 853199 -> 867757 bytes
 .../gisborne/figures/width_disagreement_cases.png  | Bin 124081 -> 131996 bytes
 .../gisborne/figures/width_method_comparison.png   | Bin 67026 -> 72424 bytes
 outputs/gisborne/layout_map.pdf                    | Bin 150061 -> 150375 bytes
 outputs/gisborne/quarantine.gpkg                   | Bin 2527232 -> 2527232 bytes
 outputs/gisborne/review/review_map.html            |   2 +-
 outputs/gisborne/review/review_queue.gpkg          | Bin 118784 -> 118784 bytes
```
同一环境内重复运行 `reproduce.py` 两次，`diff -rq run1 run2` 完全一致：**同一环境内逐字节确定**。跨环境时 CSV/JSON 与 GPKG 语义哈希一致，二进制和 HTML 输出会漂移（见 REP-04）。

---

## 发现

### REP-01 · High · 上游原始数据未归档，processed 输入无法再推导出同样的 SHA-256
**位置**：`scripts/download_gisborne_data.py:26-41, 143-160, 241-264`；`data/checksums.sha256`；`.gitignore:7-8`（`data/raw/*`）

**问题**
1. 四个来源都是可变的实时 ArcGIS REST 服务（Living Atlas 的 LCDB 镜像、MfE、DOC、Stats NZ）。脚本没有固定服务版本、`editingInfo.lastEditDate`、`serviceItemId`，也没记录下载时间。raw 缓存被 `.gitignore` 排除，既没有 checksum，也没有归档。
2. **即使上游数据完全不变**，processed GPKG 的字节级 SHA-256 也重建不出来：GPKG 的物理结构（rtree 节点、触发器 DDL）随 GDAL 版本变化。所以 `checksums.sha256` 只能证明"这是当初提交的那几个文件"，证明不了"这些文件由该脚本从某份上游数据生成"。

**证据**（把同样的要素 `read_file` 后再 `to_file` + `normalise_gpkg`）：
```
GDAL 3.12.4: gisborne_boundary: committed 26f70188dde8 rewritten 8a5c03385839 bytes_equal=False sql_dump_equal=True
GDAL 3.12.4: gisborne_candidates: committed d44c2bee7378 rewritten 088acd17817f bytes_equal=False sql_dump_equal=False
   only committed: INSERT INTO "rtree_gisborne_candidates_geom_node" VALUES(19,X'0000002600...
GDAL 3.9.1:  gisborne_candidates: committed d44c2bee7378 rewritten ea74b544f3ae bytes_equal=False sql_dump_equal=False
   only committed: CREATE TRIGGER "rtree_gisborne_candidates_geom_update5" ...
   only rewritten: CREATE TRIGGER "rtree_gisborne_candidates_geom_update1" ...
```
脚本的缓存逻辑也不校验来源，raw 存在就直接用：
```python
if path.exists():
    print(f"{name}: reading cached {path}")
    return gpd.read_file(path)
```

**影响**：第三方按 README 的 "To refresh from the public services" 流程执行，`verify_checksums.py` 几乎一定失败，并且无法区分三种原因：上游数据变了、GDAL 版本不同、脚本有 bug。"pinned inputs" 的溯源链到 processed 文件就断了。

**修复**
- 把下载得到的 raw 响应（GeoJSON/GPKG）作为数据集发布到 Zenodo/OSF/GitHub Release，拿到 DOI 并在 manifest 里记录 raw 的 sha256。
- 给 processed 输入也做**语义哈希**（与 `semantic_hashes.py` 同一套逻辑，修复方式见 REP-05），并在 CI 里核对。字节级 sha256 只当"文件完整性"检查用。
- manifest 增加溯源字段（见 REP-08）：`retrieved_at`、各服务的 `currentVersion`/`editingInfo.lastEditDate`/`serviceItemId`、`where`、服务端 `objectIds` 数量、GDAL/GEOS/PROJ 版本。
- 最好改用 LRIS layer 123148 的正式导出（带版本和 DOI），替代 15 m 简化的镜像。

---

### REP-02 · Medium · ArcGIS 分批下载会静默截断：不核对返回数量、不检查 `exceededTransferLimit`、不重试
**位置**：`scripts/download_gisborne_data.py:70-81, 93-130`

**问题**：`_download_by_bbox` 先 `returnIdsOnly` 拿到全部 ID，再按 500 个一批用 `objectIds` 请求。如果某个服务的 `maxRecordCount` < 500，或服务端因大小限制截断，一批只会返回部分要素。代码不检查 `len(features) == len(ids)`，也不看 `exceededTransferLimit`。网络瞬断时也没有重试或退避，一次失败就整层重下（`timeout=180`）。

**证据**：离线 mock 了 `_request_json`，模拟 `maxRecordCount=200`：
```
FeatureServer: 500/1200
FeatureServer: 1000/1200
FeatureServer: 1200/1200
requested ids: 1200 returned features: 600 -> no exception raised
```
进度日志显示 "1200/1200"，实际只拿到了 600 条。

**影响**：候选面或排除层（DOC/LUCAS）会静默缺失要素，排除层缺失会直接导致误判为 candidate。现有 manifest 只记录处理后的数量，事后审计无法发现。真实服务的 `maxRecordCount` 因代理拦截未能查询，**需确认**（ArcGIS Online 托管图层常见默认值为 1000/2000，风险目前是潜在的）。

**修复**
```python
def _query_geojson(service, params, expected_ids=None, retries=4):
    for attempt in range(retries):
        try:
            payload = _request_json(f"{service}/query", {...}, post=True)
            break
        except (urllib.error.URLError, TimeoutError):
            if attempt == retries - 1: raise
            time.sleep(2 ** attempt)
    if payload.get("exceededTransferLimit") or payload.get("properties", {}).get("exceededTransferLimit"):
        raise RuntimeError(f"{service}: exceededTransferLimit; reduce batch_size")
    frame = gpd.GeoDataFrame.from_features(payload["features"], crs="EPSG:2193")
    if expected_ids is not None and len(frame) != len(expected_ids):
        raise RuntimeError(f"{service}: requested {len(expected_ids)} ids, got {len(frame)}")
    return frame
```
另外：先读取 `{service}?f=json` 的 `maxRecordCount`，令 `batch_size = min(500, maxRecordCount)`；把 `len(object_ids)` 写进 manifest。

---

### REP-03 · Medium · `-part-N` 编号取决于行顺序，`unit_id` 不是"稳定"标识
**位置**：`scripts/download_gisborne_data.py:218-222`；README "assigns stable `unit_id` values"

**问题**
```python
candidates = candidates.sort_values("LCDB_UID").reset_index(drop=True)   # 默认 quicksort，非稳定排序
part = candidates.groupby("LCDB_UID").cumcount() + 1
candidates.loc[total_parts > 1, "unit_id"] += "-part-" + part.astype(str)
```
同一 `LCDB_UID` 的多个部件按进入排序时的行序编号。这个行序来自服务端返回顺序加上 `clip`/`explode` 的输出顺序，与几何本身无关。

**证据**：对已提交的 candidates 做一次行置换（seeded shuffle），再逐字执行这段逻辑：
```
multipart source units: 70 parts: 376 | parts renumbered vs committed: 319
```
376 个部件中有 319 个会拿到不同的 `-part-N`。两套 pandas/numpy 版本得到相同的 mapping hash，所以字符串键的 tie 顺序目前跨版本一致；风险出在输入顺序（服务端或 GEOS 的输出顺序）。

**影响**：重新下载、或 GEOS 调整部件顺序后，`lcdbXXX-part-2` 可能指向另一块几何。`review_sample_ids.csv`、`advisory_candidates.csv`、`findings.json` 里的 `width_example_ids`（其中就有 `lcdb1000008518-part-1`）会悄悄指向不同的面。

**修复**：部件按几何派生键稳定排序，例如：
```python
rp = candidates.geometry.representative_point()
candidates = candidates.assign(_x=rp.x.round(3), _y=rp.y.round(3), _a=candidates.area.round(3))
candidates = candidates.sort_values(["LCDB_UID", "_y", "_x", "_a"], kind="mergesort")
```
或者直接用几何的 WKB 哈希前缀做部件后缀。同时加一条测试：打乱输入行序后，`unit_id → geometry` 的映射保持不变。

---

### REP-04 · Medium · CI 覆盖缺口：多种已提交产物会漂移却从不被检查，其中 `run_manifest.json` 是完全确定的文本也没被检查
**位置**：`.github/workflows/ci.yml:27-31`

**问题**：CI 的 `git diff` 只覆盖 9 个 CSV/JSON，外加 3 个 GPKG 的语义哈希。以下已提交产物**没有任何检查**：
```
outputs/gisborne/run_manifest.json          <- 确定性 JSON，却不在 diff 路径里
outputs/gisborne/review/review_map.html
outputs/gisborne/figures/*.png (3)
outputs/gisborne/layout_map.pdf, arcgis/layout_map.pdf
outputs/gisborne/review/cards/*.jpg (33 张, 8.6 MB)
outputs/gisborne/semantic_hashes.json（只作基准，没有被再生成）
notebooks/01_width_rule_comparison.ipynb（未执行）
```
另外 `git diff` 不会发现 reproduce 新生成但未提交的文件（untracked）。

**证据**
- 在 CI 同款环境运行后有 8 个文件被改写，CI 仍然全绿（见基线）。
- 已提交产物与本机的所有环境都对不上，只在未知的原始环境里生成过：
  - HTML：`const features=...` 中 1744 个坐标有 146 个末位不同，例如 `-37.62283437358143` 变成 `...144`。PROJ 9.5.1 的输出 ≠ PROJ 9.8.1 的输出（9.4.1 与 9.8.1 相同），已提交版本与这三者都不同。
  - PNG：`screening_overview.png` 像素完全相同，但文件字节不同（压缩实现不同）；`width_method_comparison.png` 有 31 个像素不同。
  - 这 31 个像素的来源：`report.py:111` 的 `comparison.sort_values("width_area_perimeter_m")` 用的是非稳定排序。5712 行里有 260 行宽度值重复，其中 2 组 tie 的 `methods_disagree` 颜色不同。tie 的顺序随 numpy/pandas 版本变化：
    ```
    venv312  numpy 2.5.3 pandas 3.0.6  order-hash 3a7535eeafdc67b7
    venv_low numpy 1.26.4 pandas 2.2.2 order-hash ca50cd10970af9af
    ```
- 各 GPKG 的 `iterdump()` 完全相同，只是 SQLite 页布局不同，所以用语义哈希核对 GPKG 的设计是正确的。

**影响**：README 首页的图、PDF、HTML 审阅地图和 review cards 都可能悄悄过期（例如 review sample 变了而 cards 没重画）。`run_manifest.json` 里的 `reject_rate`、阈值和 unresolved 规则如果漂移，CI 也发现不了。开发者本地运行后 `git add -A`，每次都会把几 MB 二进制噪声提交进历史（见 REP-12）。

**修复**（ci.yml 追加）：
```yaml
      - name: Verify deterministic evidence tables
        run: |
          git diff --exit-code -- "outputs/gisborne/*.csv" "outputs/gisborne/*.json" "outputs/gisborne/review/*.csv"
          test -z "$(git status --porcelain --untracked-files=all -- outputs)" || { git status --porcelain; exit 1; }
      - name: Verify review cards match pinned sample
        run: |
          diff <(tail -n +2 outputs/gisborne/review/review_sample_ids.csv) \
               <(ls outputs/gisborne/review/cards | grep -E '^[0-9]{2}_' | sed -E 's/^[0-9]{2}_//; s/\.jpg$//')
      - name: Execute notebook
        run: pip install nbclient ipykernel && python -m nbclient notebooks/01_width_rule_comparison.ipynb  # or pytest --nbval-lax
      - uses: actions/upload-artifact@<sha>   # 把 CI 生成的 PNG/PDF/HTML 作为 artifact 供对比
        with: { name: gisborne-figures, path: outputs/gisborne }
```
- HTML 做语义检查：解析 `features` JSON，坐标 `round(…, 7)` 后再哈希。
- 所有影响输出的排序都加 `kind="stable"`，并附加唯一次键（例如 `sort_values(["width_area_perimeter_m", "unit_id"])`）。
- PNG/PDF 要么不提交、改由 CI 或 Release 生成，要么用像素级容差比较（例如 `matplotlib.testing.compare.compare_images(tol=…)`）。

---

### REP-05 · Medium · `semantic_hashes.py` 有盲区：不包含列名、CRS、额外图层；浮点值未规范化
**位置**：`scripts/semantic_hashes.py:20-37`

**问题**：payload 只有按排序后列顺序排列的**属性值列表**和 `wkb_hex`，没有列名、没有 CRS、没有几何类型或 dtype 声明，且只读默认图层。浮点按 `repr` 精确比较，对 1-ULP 的差异很敏感。docstring 说它 "platform-tolerant"，实际只对 SQLite 物理布局宽容。

**证据**（对 `candidates.gpkg` 做篡改后重新计算）：
```
baseline           0f7ba28d65a7069e8e279bd3ad5201e02052b480773d4d0d20038b1a5cd8fe65
renamed column     0f7ba28d65a7...fe65   <- area_ha 改名为 area_hectares，哈希不变
CRS relabelled     0f7ba28d65a7...fe65   <- 把 CRS 标成 EPSG:4326（坐标不动），哈希不变
extra layer added  0f7ba28d65a7...fe65   <- 追加图层 zz_extra，哈希不变
bool->int dtype    e72ef2cb...           <- 能检测到
1-ULP float change 730d6f7e...           <- 仅改 1 ULP 就失败
```
作为对照：在 GEOS 3.11.1 与 3.13.1、pandas 2.2.2 与 3.0.6 下，三个 GPKG 的语义哈希都一致。目前没有观察到误报，但这是运气，设计上没有保证。

**影响**：改列名、把 CRS 标错（本项目的核心风险就是 CRS）、多出一个图层，这些输出 schema 回归都能通过 CI。反过来，底层几何库的正常 ULP 级变化会让 CI 误报。

**修复**
```python
def semantic_hash(path: Path, ndigits: int = 6) -> str:
    layers = sorted(name for name, _ in pyogrio.list_layers(path))
    h = sha256()
    for layer in layers:
        frame = gpd.read_file(path, layer=layer)
        header = {"layer": layer, "crs": frame.crs.to_epsg(),
                  "columns": {c: str(frame[c].dtype) for c in sorted(frame.columns)},
                  "geom_types": sorted(frame.geom_type.unique())}
        h.update(json.dumps(header, sort_keys=True).encode())
        frame = frame.sort_values("unit_id", kind="mergesort")
        for row in frame.itertuples(index=False):
            attrs = {c: _norm(getattr(row, c), ndigits) for c in sorted(frame.columns) if c != frame.geometry.name}
            geom = shapely.set_precision(row.geometry, 1e-3).normalize().wkb_hex  # 1 mm 网格
            h.update(json.dumps({"a": attrs, "g": geom}, sort_keys=True).encode())
    return h.hexdigest()
```
其中 `_norm` 对 float 做 `round(ndigits)`、NaN 转 `None`。同时断言 `unit_id` 唯一。

---

### REP-06 · Medium · 依赖全部浮动、没有锁文件；声明的下限版本装出来就是坏的；README 推荐的 conda 路径不在 CI 里
**位置**：`pyproject.toml:2, 10-24`；`environment.yml:5-13`；`ci.yml:17-20`

**问题**
1. 所有依赖都是 `>=`，没有 `uv.lock`、`requirements.lock` 或 `conda-lock.yml`。CI 每次装当时的最新版（日志：`pandas-3.0.5`，本机当前已是 3.0.6）。实际用过的版本不记录在任何产物里。
2. **声明的下限本身不可用**：
   ```
   $ uv pip install --resolution lowest-direct -e ".[dev]"   (Python 3.12)
   ModuleNotFoundError: No module named 'pkg_resources'   # shapely 2.0.0 没有 cp312 wheel，只能从 sdist 构建
   $ (Python 3.11) 解析结果：shapely==2.0.0 + numpy==2.4.6
   ImportError: numpy.core.multiarray failed to import      # pytest: 6 errors during collection
   ```
   手工加 `numpy<2` 以后，55 个测试通过，CSV/JSON 与语义哈希全部一致，说明核心结果对版本其实很稳健；但 GPKG、PNG、PDF 的字节仍然不同。
3. `environment.yml` 固定 `python=3.12`，`pyproject` 写的是 `>=3.11`。本地基线 venv 是 3.11，CI 只测 3.12。README 主推 `conda env create -f environment.yml`（conda-forge 构建的 GDAL/PROJ/GEOS），而 CI 用 pip wheel，**文档里的复现路径从未被 CI 验证**。
4. `setuptools>=75` 没有上限，而 `license = {text = "MIT"}` 已被弃用：
   ```
   SetuptoolsDeprecationWarning: `project.license` as a TOML table is deprecated
   By 2027-Feb-18, you need to update your project and remove deprecated calls
   ```

**影响**：以后 geopandas/shapely/pyogrio/PROJ 的版本变化可能改变输出甚至导致无法安装，而没有锁文件就没有"已知良好"的环境可以回退。下游用户按声明的下限安装会直接失败。

**修复**
```toml
[build-system]
requires = ["setuptools>=77,<81", "wheel"]
[project]
license = "MIT AND BSD-2-Clause"
license-files = ["LICENSE", "src/ets_screening/vendor/leaflet-1.9.4/LICENSE"]
dependencies = [
  "geopandas>=1.0", "shapely>=2.0.4", "numpy>=1.24",   # shapely 2.0.4 起兼容 numpy 2
  "pandas>=2.2", "pyogrio>=0.10", "pyproj>=3.7", "matplotlib>=3.9", "pillow>=11.0",
]
```
- 提交 `uv.lock`（或 `pip-compile --generate-hashes` 生成的 `requirements-lock.txt`）以及 `conda-lock.yml`。CI 默认用锁文件安装。
- CI 增加矩阵：`{locked, lowest-direct, latest} × {3.11, 3.12}`，另加一个 `conda` job（`mamba env create -f environment.yml`）。
- 把 `pip freeze` 或 `importlib.metadata` 的版本，以及 `pyogrio.__gdal_version_string__`、`shapely.geos_version_string`、`pyproj.proj_version_str` 写进 `run_manifest.json`。

---

### REP-07 · Low · CI 工作流加固：Action 用 tag 固定、Node 20 弃用、单一 OS 和 Python、无定时运行、无 lint
**位置**：`.github/workflows/ci.yml:3-5, 12-18`

**问题与证据**
- `actions/checkout@v4`、`actions/setup-python@v5` 按可变 tag 引用，没有固定 SHA。CI 日志：
  `##[warning]Node.js 20 is deprecated. The following actions target Node.js 20 but are being forced to run on Node.js 24: actions/checkout@v4, actions/setup-python@v5.`
- `checkout` 使用默认的 `persist-credentials: true`，token 在整个 job 期间留在 `.git/config`。`permissions: contents: read` 已经降低了风险，这一点做得好。
- 只有 `ubuntu-latest` + 3.12。作者主要在 Windows/ArcGIS Pro 上开发，提交 `f3679c6` 的说明写着 "tests ... passed on Windows and failed the first time CI ran them on ubuntu"，说明平台差异确实出过问题；但 CI 只能发现单向的问题。Windows 上 `pandas.to_csv` 默认 `lineterminator=os.linesep`，`write_text` 也会写 CRLF；仓库没有 `.gitattributes`，确定性检查的结果依赖每个开发者的 `core.autocrlf` 设置。
- `on: push` 加 `pull_request` 且没有分支过滤，同仓库的 PR 会跑两次。没有 `schedule`/`workflow_dispatch`，依赖浮动引起的漂移（REP-06）只会在下一次推送时才暴露。
- 没有 lint 或类型检查（`ruff`/`mypy`），也不执行 notebook（本次手动用 nbclient 执行成功，文本输出与提交版本一致）。

**修复**
```yaml
on:
  push: { branches: [main] }
  pull_request:
  schedule: [{ cron: "0 3 * * 1" }]
  workflow_dispatch:
jobs:
  test-and-reproduce:
    strategy:
      matrix: { os: [ubuntu-latest, windows-latest], python: ["3.11", "3.12"] }
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@<full-sha> # v5
        with: { persist-credentials: false }
      - uses: actions/setup-python@<full-sha> # v6
      - run: pipx run ruff check .
```
再配合 Dependabot（`package-ecosystem: github-actions`）自动更新已固定 SHA 的 action。

---

### REP-08 · Medium · `run_manifest.json` / `gisborne_manifest.json` 缺少关键溯源字段，并硬编码了伪时间戳
**位置**：`src/ets_screening/screen.py:151-167`；`scripts/download_gisborne_data.py:248-264`；`src/ets_screening/io_utils.py:9`；`src/ets_screening/report.py:203-204`；`scripts/reproduce.py:33-34`

**问题**：`run_manifest.json` 当前的键只有 `scope, crs, input_feature_type, feature_count, overlap_materiality, reject_rate, reject_rate_threshold, unresolved_rules`，**缺少**以下内容：
- git commit SHA 以及工作树是否 dirty；
- 包版本和 GDAL/GEOS/PROJ 版本；
- 输入文件的 sha256，或对 `gisborne_manifest.json` 的引用；
- 完整的 `RuleConfig`：`minimum_area_ha`、`width_threshold_m`、`plantable_lcdb_classes` 都没有写，只写了 overlap 两项；
- review 抽样的 `seed=20260912`、`sample_size=30`，以及样本是否来自 pinned 列表；
- 使用的 imagery 或 basemap 来源。

`gisborne_manifest.json` 有 URL、数量和 sha256，这点不错，但**缺少** `retrieved_at`、服务版本/`lastEditDate`、每层的 `where` 过滤、服务端 ID 数量（REP-02）、各数据集的**许可证与署名**、软件版本。`data_note="LCDB v6 / LUCAS v005 / DOC PCL"` 是在 `reproduce.py` 里手写的，不是从 manifest 读出来的。

在确定性方面，manifest 里没有机器路径，也没有运行时时间戳，这是优点。但所有 GPKG 的 `last_change` 都被写成 `FIXED_TIMESTAMP = "2026-09-12T00:00:00.000Z"`，PDF 的 `CreationDate/ModDate` 也写死为 `datetime(2026, 9, 12)`。无论哪天重新生成，文件都声称自己生成于 2026-09-12，这是误导性的溯源信息。

**影响**：拿到一份输出的审阅者无法回答"用哪个 commit、哪些库版本、哪份上游数据快照、哪些阈值生成的"，与项目 "auditable" 的定位不符。

**修复**
```python
import importlib.metadata as md, subprocess, pyogrio, shapely, pyproj
manifest["provenance"] = {
    "git_commit": subprocess.run(["git","rev-parse","HEAD"],capture_output=True,text=True).stdout.strip(),
    "git_dirty": bool(subprocess.run(["git","status","--porcelain"],capture_output=True,text=True).stdout),
    "packages": {p: md.version(p) for p in ["geopandas","shapely","pyogrio","pyproj","pandas","numpy","matplotlib"]},
    "gdal": pyogrio.__gdal_version_string__, "geos": shapely.geos_version_string, "proj": pyproj.proj_version_str,
    "inputs_sha256": {...}, "rule_config": dataclasses.asdict(cfg),  # frozenset 转为 sorted list
    "review_sample": {"seed": 20260912, "size": 30, "pinned": review_sample_ids is not None},
}
```
- 注意把 `packages` 放进一个单独的、不参与 diff 的文件（例如 `run_environment.json`），或者只写主版本号，以免影响跨环境的确定性检查。
- 固定时间戳改为 `SOURCE_DATE_EPOCH`（取 `git log -1 --format=%ct`），GPKG 和 PDF 都用它。

---

### REP-09 · Medium · Review cards 的影像来源、许可和分辨率都没有记录；实测有效分辨率约 10 m，而代码注释称约 1.32 m
**位置**：`scripts/download_review_cards.py:18-21, 65-67, 103-108`；`scripts/ingest_review_labels.py:32`；`data/README.md:13`

**问题**
1. 33 张 JPG（8.6 MB）以 MIT 仓库的一部分公开再分发，但**卡片上没有任何版权或许可署名**。caption 只有 `unit_id | lcdb_class | OVERVIEW Lx / DETAIL L10`。文档里也只有一句 "service attribution: LINZ" / "(credited to LINZ)"，没有许可证名称、链接和影像日期。
2. 代码注释说 "Level 10 ... (about 1.32 m per pixel)"，但那只是**瓦片网格**的分辨率。对 30 张卡片 detail 面板做列差分自相关，峰值出现在 7.5/15/22.7/30.2/37.8/45.3 px，周期约 7.56 px，7.56 × 1.3229 ≈ **10.0 m 的原生像元**。这更像 Sentinel-2 一类的 10 m 卫星产品，而非 LINZ 的航空正射影像。
3. 如果源头是 Copernicus Sentinel，署名应为 "Contains modified Copernicus Sentinel data 2024"。如果是 LINZ 的 CC BY 4.0 影像，则需要 "© LINZ CC BY 4.0" 加上修改说明。由于代理拦截，无法读取服务的 `copyrightText`，**需确认**。

**影响**：可能不满足来源许可的署名要求，属于许可合规风险。另外，README 让人工审阅者在这些卡片上判断 "riverbeds, coastal margins, roads and erosion scars"，10 m 像元对 1 ha 级别的单元判读力有限。审阅尚未开始，现在修正成本最低。

**修复**
- 下载脚本读取 `GDC_IMAGERY?f=json` 的 `copyrightText`、`description`、`serviceItemId` 并写入 `review/cards/SOURCE.json`（与每张卡的 sha256 一起），把 attribution 直接绘制在卡片底边。
- 在 `data/README.md` 写明该影像的许可证、链接、采集日期和 GSD。
- 如果许可不允许再分发，把 JPG 移出仓库，只保留生成脚本。
- 评估改用 LINZ Gisborne 0.x m 航空影像（CC BY 4.0，可通过 LINZ Basemaps 或 LDS 获取）。

---

### REP-10 · Medium · 数据许可与署名不完整：MIT 覆盖面有歧义、3 个来源没写许可、派生产物无署名、缺 CITATION.cff
**位置**：`LICENSE`；`README.md:288-291`；`data/README.md:7-14`；`src/ets_screening/report.py:189-196`；`pyproject.toml:11`

**问题与证据**
- GitHub 把整个仓库识别为 `"license":{"key":"mit"}`，但仓库里有 12 MB 派生自第三方数据的 processed GPKG，以及 outputs 下的 GPKG/CSV/PNG/PDF/JPG。README 只有一句概括性的 "Source data retain their publishers' licences"。
- 只有 LCDB 写明了许可（CC BY 4.0 + DOI）。**LUCAS（MfE）、DOC PCL、Stats NZ TA2026、GDC 影像都没有写许可证**。`rules/SOURCES.md` 也只给了 LCDB 的许可行。这些来源多半是 CC BY 4.0，但项目需要逐一核实并写明。
- CC BY 4.0 要求署名、许可链接和修改说明。PDF 页脚只有 `"... | EPSG:2193 | LCDB v6 / LUCAS v005 / DOC PCL"`，没有权利人、没有 "CC BY 4.0"、没有 "modified" 说明；三张 PNG 上完全没有来源署名。`review_map.html` 对 LINZ basemap 的署名是正确的（值得肯定）。
- 没有 `CITATION.cff`，也没有数据 DOI。
- wheel 元数据只有 `License: MIT`，但包里带了 BSD-2 的 Leaflet。wheel 里确实包含 `ets_screening/vendor/leaflet-1.9.4/LICENSE`，BSD-2 的声明保留要求是满足的；`license-files` 不包含它。

**修复**
- 新增 `DATA_LICENSE.md` 或 `NOTICE`，逐个数据集写明：权利人、许可证（附 URL）、版本/DOI、检索日期、所做修改（clip/make_valid/explode/简化镜像）。README 写成 "Code: MIT. Data derivatives in `data/processed` and `outputs/`: CC BY 4.0, see DATA_LICENSE.md"。
- 在 `plot_screening_overview`/`plot_layout_pdf` 里加署名行，例如：
  `"Data: LCDB v6.0 © Manaaki Whenua (CC BY 4.0); LUCAS LUM 2020 v005 © MfE (CC BY 4.0); PCL © DOC (CC BY 4.0); TA2026 © Stats NZ (CC BY 4.0). Modified."`
- 添加 `CITATION.cff`，并通过 Zenodo–GitHub 集成为版本快照（含数据）发 DOI。
- 可选：用 REUSE 规范（`.reuse/dep5` 或 `REUSE.toml`）按路径声明许可。

---

### REP-11 · Low · 环境里设置了 `LINZ_BASEMAP_API_KEY` 时，key 会被写进一个会提交的 HTML，且未转义
**位置**：`src/ets_screening/screen.py:147`；`src/ets_screening/sample_review.py:80-86, 99-100`；`tests/test_screen.py:41-43`

**问题**：`reproduce.py` 的输出目录就是被提交的 `outputs/gisborne/`。开发者本地设置了该变量时，key 会被写进 `review_map.html`，而 CI 不检查这个文件（REP-04）。key 被直接拼进 JS 单引号字符串，没有转义。

**证据**
```
$ LINZ_BASEMAP_API_KEY=c01SECRETDEMOKEY python scripts/reproduce.py
$ grep -o "webp?api=[A-Za-z0-9]*" outputs/gisborne/review/review_map.html
webp?api=c01SECRETDEMOKEY
$ git status --short outputs/gisborne/review/
 M outputs/gisborne/review/review_map.html
CI table check exit=0
```
另外，README 的表格写着 "Offline; no CDN or API key required"。Leaflet 确实是离线打包的，但瓦片仍需联网；LINZ Basemaps 不带 `api=` 能否访问，因代理拦截未能验证，**需确认**。

**影响**：个人 API key 会进入公开的 git 历史（LINZ key 属于浏览器端可见的 key，敏感性有限，但会被第三方滥用配额，也无法撤回历史）。

**修复**
- 提交版一律不写 key：`reproduce.py` 调用 `run_screening` 时显式传 `api_key=None`，或者 `write_review_bundle` 只在 `--embed-key` 显式参数时才嵌入。
- 运行时再注入，例如 `new URLSearchParams(location.hash.slice(1)).get('api')`。
- 必须嵌入时用 `json.dumps(tile_url)` 生成 JS 字符串。
- CI 增加 `! grep -rE "api=[A-Za-z0-9]{8,}" outputs/`。
- 修正 README 的 "no API key required" 表述。

---

### REP-12 · Low · 仓库卫生：约 89% 为二进制、历史膨胀快、缺 `.gitattributes`
**位置**：仓库根目录

**证据**
```
$ git ls-files -z | xargs -0 du -cb | tail -1          -> 35,981,194 total
$ git ls-files | grep -E "\.(gpkg|jpg|png|pdf)$" | xargs du -cb | tail -1 -> 31,971,832 total
GitHub API: "size": 32664 (KB)，仓库创建于 2026-09-12，4 天内
list_commits(path=outputs/gisborne/candidates.gpkg) -> 4 个提交（每次约 4 MB）
GitHub 语言识别: "language":"HTML"   <- 因为生成的 review_map.html 内嵌了 147 KB 的 leaflet.js
$ ls .gitattributes -> 不存在
```
`arcgis/layout_map.pdf`（1.9 MB，ArcGIS Pro 导出）与 `outputs/gisborne/layout_map.pdf`（150 KB，Matplotlib）的 sha256 不同（`ec0d74c8…` 与 `2876276f…`），**不是重复文件**。但同名容易混淆。

**影响**：每次重生成（尤其跨环境的字节漂移，见 REP-04）都会往历史里追加几 MB，clone 越来越慢；换行符行为依赖个人 git 配置；语言统计失真。

**修复**
```gitattributes
*.gpkg  binary
*.jpg   binary
*.png   binary
*.pdf   binary
*.csv   text eol=lf
*.json  text eol=lf
outputs/**/review_map.html linguist-generated=true
src/ets_screening/vendor/** linguist-vendored=true
```
- 大文件改用 Git LFS，或者只在 GitHub Release/Zenodo 上发布 GPKG、JPG 和 PDF，仓库里保留 manifest 和哈希。
- 把 `arcgis/layout_map.pdf` 改名为 `arcgis/layout_map_arcgispro.pdf`。
- 迁移到 LFS 后，可以用 `git lfs migrate import` 清理已有历史。

---

### REP-13 · Info · README 与实际行为有出入
**位置**：`README.md`

- "54 tests cover ..."：实际 `55 passed`（本地和 CI 日志一致）。
- "rerunning the pipeline on unchanged inputs produces no diff to commit"：只在**同一环境**内成立（已验证 run1 == run2）；在 CI 同款的 ubuntu 环境里会改写 8 个已提交文件（REP-04）。
- "Interactive review map | Offline; no CDN or API key required"：瓦片仍需联网，API key 情况**需确认**（REP-11）。
- "assigns stable `unit_id` values"：`-part-N` 取决于行顺序（REP-03）。

**修复**：更新文本；测试数量可以在 CI 里用 `pytest --collect-only -q | tail -1` 校验，或者干脆不写具体数字。

---

## 优点
- **同一环境内完全确定**：两次 `reproduce.py` 逐字节一致。`normalise_gpkg` 固定了 `last_change`、SQLite change counter 和 version 字段，并做了 `VACUUM`，所有 GPKG 的 `PRAGMA integrity_check` 都是 `ok`。PDF 固定了 `CreationDate`，避免了常见的时间戳漂移。
- **核心证据跨版本稳健**：在"声明下限"（GEOS 3.11/GDAL 3.9/pandas 2.2/numpy 1.26）与"最新"（GEOS 3.13/GDAL 3.12/pandas 3.0/numpy 2.5）之间，所有 CSV/JSON 逐字节一致，3 个 GPKG 语义哈希一致。notebook 重新执行后，文本输出与提交版本完全一致。用语义哈希而不是字节哈希来核对 GPKG 输出，这个设计是对的。
- **CI 基本卫生良好**：`permissions: contents: read`，端到端复现并做 diff 检查，输入有 checksum，review 样本 ID 已固定。发布前还有 80% reject 率的门槛（`RejectRateExceeded`）。
- **第三方组件处理规范**：vendored Leaflet 1.9.4 与 npm 官方包逐字节一致（`leaflet.js` 的 sha256 相同；css 和 LICENSE 只差 CRLF 与 LF）。BSD-2 LICENSE 同时保留在源码树和 wheel 中，HTML 内有 Leaflet 版权注释和正确的 LINZ basemap 署名。`rules/SOURCES.md` 记录了来源的核查日期，LCDB 的 DOI 和许可也写明了。
