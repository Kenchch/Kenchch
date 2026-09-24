# 审计报告：I/O 安全与健壮性 + 文档与输出完整性

- 对象：`Kenchch/nz-ets-forest-eligibility-screening` @ `622e3b8`
- 方法：只读审阅原仓库；所有写操作在副本 `scratchpad/sec/nz-ets-forest-eligibility-screening` 中进行（`PYTHONPATH` 指向副本 `src/`）。HTML 用 Playwright 驱动 `/opt/pw-browsers/chromium-1194` 无头加载（Playwright 装在独立的 `scratchpad/pwvenv`）。Leaflet 用 `npm pack leaflet@1.9.4` 做比对。外部服务（`basemaps.linz.govt.nz`、`arcgis.mfe.govt.nz` 等）被本环境的出站策略拦截（代理 CONNECT 403），所以下载脚本和外链不能在线验证，相关项已标为“需确认”。
- 基线：副本中 `pytest` 为 55 passed，`verify_checksums.py` 全部 ok，`reproduce.py` 耗时 22 s、exit 0，CSV 与 findings 均按字节复现，`semantic_hashes.py` 全部 ok。

## 汇总

| ID | 严重度 | 标题 |
|---|---|---|
| SEC-01 | High | `review_map.html` 存储型 XSS：任意属性可用 `</script>` 跳出，打开即执行；popup 经 `innerHTML` 注入 |
| SEC-02 | Medium | `LINZ_BASEMAP_API_KEY` 被写进受 git 跟踪的 `review_map.html`（CI 不检查），且未转义直接拼进 JS |
| SEC-03 | Medium | `run_screening` 会删除或覆盖用户所选输出目录里的同名文件，并 `rmtree` 整个 `figures/`（ArcGIS 工具同样如此） |
| SEC-04 | Medium | 评审标签校验有缺口：空白 reviewer 可通过；evidence_note 可用空格凑够长度；日期格式和范围没卡住；ANSI 编码直接崩溃；CSV 公式注入会传到汇总；ingest 会原地改写评审人的原始文件 |
| SEC-05 | Low | ArcGIS REST 分批下载不核对条数，截断时静默丢数据；raw 缓存不校验查询参数；没有重试；卡片文件名未做清理 |
| SEC-06 | Low | 参数处理：`reproduce.py --help` 会先跑完整条流水线再报错；`.pyt` 中阈值 0 被静默改成 0.80 |
| DOC-01 | High | 已提交的 `review_map.html` 抛出 JS TypeError，说明面板从不显示；整张地图没有免责声明和图例；README 的“Offline”说法有误导 |
| DOC-02 | Medium | 固定的 30 个评审样本用当前代码无法复现（0/30 重合）。它实际是旧算法（`DataFrame.sample`）在 2,520 个 clean 候选上抽的，167 个 advisory 候选的入样概率为 0 |
| DOC-03 | Medium | 评审卡片的黄色十字“at interior point”画错了位置：15/30 张落在多边形外，偏差中位 124 m。另有 21/30 张卡片的两个面板完全相同 |
| DOC-04 | Low | 多瓣统计（181 / 53 / 23）的阈值描述与实际算法不符，仓库里也没有任何代码或 notebook 能复现；“two separate lobes”实为 6 个碎片 |
| DOC-05 | Low | 过时或不准确的表述：“54 tests”（实为 55）；data/README 中 “12 invented geometries … only for fast unit tests”；“reproject to EPSG:2193” |
| DOC-06 | Low | 免责声明覆盖不全：CSV/JSON 输出、宽度图、评审卡片、HTML 都没有 “screening/triage only” 字样 |
| DOC-07 | Low | 机读性与一致性：`findings.csv` 嵌套值是 Python repr；`rule_results.csv` 中 R-07 名称丢了“>30% in each hectare”；`summary.csv` 的 3 与 manifest 的 4 口径不一致 |
| DOC-08 | Low | README 称“machine-generated reviewer names are rejected … cannot enter the evidence chain”，属过度宣称：正则可被轻易绕过，而且会误伤真人姓名 |
| NC-01 | 需确认 (Needs confirmation) | LINZ 无 key URL 是否真能用；各外链的有效性；data/README 称 “authoritative ArcGIS mirror” |

---

## SEC-01 — High — `review_map.html` 存储型 XSS

**位置**：`src/ets_screening/sample_review.py:97`（`const features={json.dumps(geojson)};`）、`:102`（`l.bindPopup('<b>'+f.properties.unit_id+'</b><br>Status: '+f.properties.status)`）

**问题**：
1. 整个样本 GeoDataFrame 的**全部列**原样进入 GeoJSON，再由 `json.dumps` 直接嵌进 `<script>`。`json.dumps` 不转义 `<`、`/`，所以任何属性里出现 `</script>` 都会提前闭合脚本块，后面的内容会被当作 HTML 或新的脚本执行。
2. popup 是字符串拼接后交给 Leaflet 的 `innerHTML`，所以 `unit_id` 或 `status` 里的 HTML 标签会被执行。

**威胁模型**：`ets-screen` CLI、`ets-review-sample` CLI 和 ArcGIS `.pyt` 都接受任意候选图层。`out = candidates.copy()` 会保留全部额外列，一直传到 `review_queue` 和 HTML。README:259 明确建议使用 “applicant-supplied stand boundaries”，而申请方相对评估员是利益相关的外部方。

**证据（在副本中实际运行）**：
```text
# 1) 恶意 lcdb_class = "</script><script>window.__xss2=1;document.body.setAttribute('data-pwned','script-breakout')</script>"
#    unit_id = "<img src=x onerror=...>", status = "<img src=y onerror=window.__xss3=1>"
$ python -m ets_screening.sample_review evil.gpkg --output out --sample-size 3
$ pwvenv/bin/python check.py out/review_map.html
 "on_load": {"x2": 1, "pwned": "script-breakout", ...}          # 打开页面即执行，无需交互
$ ... evil2.gpkg（只含 popup payload）
 "after_popup_clicks": {"x1": 1, "x3": 1}                         # 点 polygon 即执行

# 2) 端到端：完整流水线，候选文件带一个额外列 applicant_note
$ python -m ets_screening.screen --candidates applicant.gpkg --pre1990 data/sample/demo_pre1990.gpkg \
      --conservation data/sample/demo_conservation.gpkg --output pipe
$ grep -c "via-applicant_note" pipe/review/review_map.html   -> 1
 "on_load": {"pwned": "via-applicant_note", ...}
```

**影响**：评估员打开地图的那一刻，脚本就在本机浏览器里执行。它可以篡改显示的 status 或说明，伪造评审指引，或把页面里的全部候选数据发到外部（页面本来就要联网拉瓦片）。

**修复（已在副本中验证：两组 PoC 均不再触发，页面无 JS 错误，55 个测试仍通过）**：
```python
import urllib.parse
def js(value) -> str:
    # JSON 本身是合法 JS；转义 < > & 以及 U+2028/9，防止 </script> 跳出
    return (json.dumps(value).replace("<", "\\u003c").replace(">", "\\u003e")
            .replace("&", "\\u0026").replace("\u2028", "\\u2028").replace("\u2029", "\\u2029"))
...
const features={js(geojson)};
L.tileLayer({js(tile_url)}, {{attribution:{js(LINZ_ATTRIBUTION)},maxZoom:22}}).addTo(map);
onEachFeature:(f,l)=>{{const d=document.createElement('div');const b=document.createElement('b');
  b.textContent=String(f.properties.unit_id);
  d.append(b,document.createElement('br'),'Status: '+String(f.properties.status));l.bindPopup(d);}}
```
另外建议只导出必要字段，例如 `sample[["unit_id","lcdb_class","area_ha","status","geometry"]]`，并加一条 CSP：`<meta http-equiv="Content-Security-Policy" content="default-src 'none'; img-src https://basemaps.linz.govt.nz data:; style-src 'unsafe-inline'; script-src 'unsafe-inline'">`。

---

## SEC-02 — Medium — API key 写进受跟踪的 HTML，且未转义

**位置**：`src/ets_screening/sample_review.py:80-86, 99`；`src/ets_screening/screen.py:147`；`.github/workflows/ci.yml`（diff 检查只覆盖 `*.csv` 和 `findings.json`）

**问题**：设了 `LINZ_BASEMAP_API_KEY` 时，`reproduce.py` 会把 key 以明文 `?api=<key>` 写进 `outputs/gisborne/review/review_map.html`。这个文件受 git 跟踪、没有被 ignore，CI 也不检查它。`git add -A` 之后 key 就进了公开仓库。同时 key 未经转义直接拼进单引号 JS 字符串。

**证据**：
```text
$ LINZ_BASEMAP_API_KEY="c01k-my-personal-linz-key" python scripts/reproduce.py
$ git ls-files outputs/gisborne/review/review_map.html   -> outputs/gisborne/review/review_map.html
$ git check-ignore -v ... || echo "not ignored"            -> not ignored
$ grep -o "api=[^']*" outputs/gisborne/review/review_map.html  -> api=c01k-my-personal-linz-key
$ git diff --exit-code -- "outputs/gisborne/*.csv" "outputs/gisborne/findings.json" "outputs/gisborne/review/*.csv"
  -> CI evidence-table diff check: PASS (html not checked)
# JS 注入：LINZ_BASEMAP_API_KEY="k',{});window.__xss4=1;L.tileLayer('x"
 "on_load": {"x4": 1}
```
`tests/test_screen.py:41-46` 甚至把“key 出现在 HTML 中”写成了断言。

**影响**：个人 LINZ key 泄漏到公开仓库，可被滥用或导致被吊销。环境变量归操作者控制，所以 JS 注入本身风险较低，但说明拼接处没有任何转义。

**修复**：
- 已提交的产物里不写 key。页面运行时再从 `location.hash`（例如 `review_map.html#api=...`，hash 不会进入请求或日志）或 `localStorage` 读取，并用 `encodeURIComponent` 拼接：
  ```js
  const key = new URLSearchParams(location.hash.slice(1)).get('api');
  const url = 'https://basemaps.linz.govt.nz/v1/tiles/aerial/WebMercatorQuad/{z}/{x}/{y}.webp' + (key ? '?api=' + encodeURIComponent(key) : '');
  ```
- 如果坚持写入文件，就写到未跟踪的 `review_map.local.html`（加进 `.gitignore`），并用 `urllib.parse.quote` + `js()`（见 SEC-01）。
- CI 增加 `! grep -q "api=" outputs/gisborne/review/review_map.html`。

---

## SEC-03 — Medium — 输出目录中的同名内容被静默删除或覆盖

**位置**：`src/ets_screening/screen.py:170-187`（`managed_names` 中含 `"figures"`，执行 `shutil.rmtree(child)`）；`arcgis/ets_screening.pyt:49-55`（输出目录故意设为 `direction="Input"`，即“已存在的文件夹”）；`screen.py:215` CLI 默认 `--output outputs`

**问题**：`run_screening` 对用户指定的任意目录，删除 `candidates.gpkg`、`summary.csv`、`layout_map.pdf` 等同名文件，并对 `figures/` 整个目录做 `rmtree`，其中包括不是它生成的文件。没有标记文件、确认提示或 `--force`。ArcGIS 用户很常见的操作是把项目主目录选为输出目录。

**证据**：
```text
$ mkdir -p pipe/figures && echo "my thesis figures" > pipe/figures/important_notes.txt && echo precious > pipe/summary.csv
$ python -m ets_screening.screen --candidates applicant.gpkg ... --output pipe
--- after run
pipe/figures: screening_overview.png  width_method_comparison.png      # important_notes.txt 已被删除
pipe/summary.csv: metric,value / total_features,2 ...                  # 原内容被覆盖
```
另外，直接用 CLI 跑 `outputs/gisborne` 会删除 README 引用的 `figures/width_disagreement_cases.png`，这张图只由 `build_findings.py` 生成。

**影响**：用户数据被不可逆地删除。

**修复**：
```python
MARKER = ".ets_screening_output"
if output_dir.exists() and any(output_dir.iterdir()) and not (output_dir / MARKER).exists():
    raise FileExistsError(f"{output_dir} is not an ets-screening output folder; choose an empty folder")
...
for name in ("screening_overview.png", "width_method_comparison.png"):   # 只删自己生成的文件
    (output_dir / "figures" / name).unlink(missing_ok=True)
(output_dir / MARKER).touch()
```
同时提交 `outputs/gisborne/.ets_screening_output`。`.pyt` 的输出目录说明里也应写明“必须为空目录或本工具先前的输出目录”。

---

## SEC-04 — Medium — 评审标签（人工证据）校验有缺口

**位置**：`src/ets_screening/review_labels.py:60, 68-73, 106-122, 165-166, 170-171`；`scripts/ingest_review_labels.py:227-232`

**证据**（`load_review_labels(path, pinned_ids)`，脚本 `scratchpad/labels/t.py`）：
```text
ACCEPTED  whitespace_reviewer_and_note: reviewers=['   ']          # “named person” 可以是 3 个空格
ACCEPTED  filler_note                                               # evidence_note = "x"*20
ACCEPTED  date_basic_format: dates=['20260920']                     # README 声称只接受 ISO YYYY-MM-DD
ACCEPTED  date_iso_week:     dates=['2026-W38-7']
ACCEPTED  date_future:       dates=['2099-12-31']
ACCEPTED  date_pre_imagery:  dates=['1901-01-01']                   # 早于 2024 影像
ACCEPTED  formula_injection: reviewers=['=1+2']
CRASH     cp1252_excel: UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 ...
ACCEPTED  utf8_bom                                                  # BOM 处理正常（优点）
```
通过 `ingest_review_labels.py --review-dir rd` 走完整流程（reviewer 为 `=HYPERLINK("http://attacker.example/x","Jane Smith")`，evidence_note 为 `"   pasture visible   "` 用空格补齐）：
```text
exit=0
sha256 before 01f59a8f84beafac -> after e370366dd31ae76a      # 评审人原始文件被原地改写（去掉 BOM、重新排序）
review_summary.csv:
reviewer,"=HYPERLINK(""http://attacker.example/x"",""Jane Smith"")"   # 在 Excel 中打开即成为活公式
review_date_last,20260920                                            # 混合日期格式，按字符串取 min/max
```
Excel 用 “CSV (Comma delimited)”（cp1252）另存后，`ingest_review_labels.py` 直接抛 traceback（exit 1），而不是给出友好的 `review labels rejected`。含毛利语长音符的姓名（例如 `Hēmi Pōtae`）在 cp1252 下被替换成 `H?mi P?tae`，而且会被**接受**。

**影响**：这是全项目唯一的“人工证据”闸门。空白 reviewer 使 “signed by a named person” 形同虚设。公式注入可以传到 `review_summary.csv` 和 `findings.csv`（`visual_reviewer` 字段）。原始签名文件被工具改写，证据链上也就失去了原件。此外 `summarise_review_labels` 把 “Single-reviewer …” 写死了，多名评审时同样照写。

**修复**：
```python
try:
    labels = pd.read_csv(path, dtype=str, keep_default_na=False, encoding="utf-8-sig")
except UnicodeDecodeError as e:
    raise ReviewLabelError("save the label file as 'CSV UTF-8'") from e
labels = labels.apply(lambda c: c.str.strip())           # 先去掉首尾空白，再做空值和长度检查
ISO = re.compile(r"\d{4}-\d{2}-\d{2}")
FORMULA = re.compile(r"^[=+\-@\t\r]")
bad = labels.apply(lambda c: c.str.match(FORMULA)).any(axis=1)
_require(not bad.any(), f"cells may not start with = + - @ (CSV lines {[i+2 for i in labels.index[bad]]})")
for i, v in labels["review_date"].items():
    _require(bool(ISO.fullmatch(v)), ...)
    _require(date(2024, 1, 1) <= date.fromisoformat(v) <= date.today(), ...)
```
ingest 不应覆盖 `review_labels.csv`：可以写一份 `review_labels.normalised.csv`，或者只校验、不改写，并把原文件的 sha256 记入 `review_summary.csv`。`limitation` 文本应按 `len(reviewers)` 生成。

---

## SEC-05 — Low — 下载脚本的完整性与健壮性

**位置**：`scripts/download_gisborne_data.py:77, 114-130, 152-155`；`scripts/download_review_cards.py:55-59, 124`

**问题与证据**：
1. **分批截断时静默丢数据**：不核对每批返回条数是否等于请求的 `objectIds` 数，也不检查 `exceededTransferLimit`。我用本地假 ArcGIS 服务器（服务端每次最多返回 100 条，并标记 `exceededTransferLimit: true`）验证：
   ```text
   FeatureServer: 500/1000
   FeatureServer: 1000/1000
   objectIds advertised: 1000   features returned: 200   -> no error raised
   ```
   进度日志还显示 “1000/1000”，具有误导性。
2. `_cached_download` 发现 `data/raw/<name>.gpkg` 存在就直接复用，不校验 `where`、`fields`，也不看时间。修改 `CANDIDATE_SOURCE_CLASSES` 之后，旧缓存会被静默沿用。
3. 没有重试：卡片脚本约发 30×2×9=540 个瓦片请求，任何一个瞬时 5xx 都会让整个流程中止。
4. 卡片文件名 `f"{index:02d}_{row.unit_id}.jpg"` 未做清理。unit_id 含 `/` 时直接崩溃（实测 `FileNotFoundError: .../cards/01_../../ESCAPED_card.jpg`）。因为第一段总是 `NN_` 前缀目录且该目录不存在，实际无法路径穿越；但 Windows 非法字符（`:`、`?`）同样会导致崩溃。
5. 好的方面：只用 `https://`；`urllib` 默认校验 TLS；有显式 timeout（180 s 和 60 s）；不解压任何压缩包（不存在 zip-slip）；没有硬编码的 key 或 token。

**修复**：
```python
batch = _query_geojson(service, {...})
if len(batch) != len(ids):
    raise RuntimeError(f"{service}: requested {len(ids)} objectIds, received {len(batch)}")
```
在 `_query_geojson` 中检查 `payload.get("properties", {}).get("exceededTransferLimit")` 或 `payload.get("exceededTransferLimit")`。缓存旁放一个 `<name>.query.json`（包含 where、fields、bbox、下载时间），不匹配就重新下载，并把下载时间写进 `gisborne_manifest.json`。请求加简单的指数退避重试（3 次）。文件名用 `re.sub(r"[^A-Za-z0-9._-]", "_", unit_id)` 清理。

---

## SEC-06 — Low — 参数处理缺陷

**位置**：`scripts/reproduce.py:37-39` 与 `scripts/ingest_review_labels.py:218-220`；`arcgis/ets_screening.pyt:81`

**问题与证据**：
1. `reproduce.py` 自己没有 argparse，但它调用的 `ingest_review_labels.main()` 会解析 `sys.argv`。结果是 `python scripts/reproduce.py --help` 先花约 20 s 重写所有输出，最后才打印**另一个脚本**的帮助：
   ```text
   $ touch -d 2000-01-01 outputs/gisborne/summary.csv; python scripts/reproduce.py --help
   ...findings JSON printed...
   usage: reproduce.py [-h] [--review-dir REVIEW_DIR]
   Validate a completed human imagery review ...
   summary.csv mtime: 2000-01-01 -> 2026-09-23          # 输出已被重写
   $ python scripts/reproduce.py --dry-run
   reproduce.py: error: unrecognized arguments: --dry-run   # 同样是跑完整条流水线之后才报错
   ```
2. `.pyt` 中 `threshold = float(parameters[4].value or 0.80)`：用户填 `0`（任何拒绝都中止）时，`0 or 0.80` 得到 0.80，且没有任何提示。

**修复**：改为 `def main(argv: list[str] | None = None)` 加 `parser.parse_args(argv)`，`reproduce.py` 调用 `ingest_review_labels([])`，并给 `reproduce.py` 加上自己的 argparse。`.pyt` 改为 `threshold = 0.80 if parameters[4].value is None else float(parameters[4].value)`。

---

## DOC-01 — High — 评审地图的说明面板从不显示，且没有免责声明

**位置**：`src/ets_screening/sample_review.py:104-105`；已提交的 `outputs/gisborne/review/review_map.html:681`；`README.md:79, 196-198`；`tests/test_screen.py:99-107`

**问题**：
```js
L.control({position:'topright'}).onAdd=function(){...;return d;}.addTo(map);
```
按运算符优先级，`.addTo(map)` 被当作对函数对象本身的调用，于是抛出 TypeError。说明面板里有 “Human review queue / assign one label … / must be completed by a named person”，是整张地图上**唯一**的说明文字，它从不渲染。除此之外，页面没有图例、比例尺、`<meta name=viewport>`，也没有任何 “screening / not an eligibility determination” 字样。

**证据**（Chromium 无头加载**已提交**的文件）：
```text
"pageerrors": ["(intermediate value).addTo is not a function"],
"note": 0, "legend": 0, "mapAria": null,
"controls": ["leaflet-control-zoom ...", "leaflet-control-attribution ... LINZ CC BY 4.0 ..."],
"external_requests": ["basemaps.linz.govt.nz"], "n_tile_reqs": 18
$ grep -o "screening\|not an eligibility\|triage\|legend\|viewport" review_map.html   -> （无输出）
```
截图（`scratchpad/committed_map.png`）只有灰底和红色轮廓。README:79 写的是 “Interactive review map | **Offline**; no CDN or API key required”，但影像瓦片必须在线从 `basemaps.linz.govt.nz` 获取。在受限网络下，地图能打开，却没有影像，也就无法用来做影像判读。现有测试只做字符串断言（`"L.map(" in html`），不执行 JS，所以没能发现这个错误。

**影响**：README 宣称的主要评审工具实际上不可用：评审人看不到说明和标签词表，页面上也没有任何 “不是资格判定” 的提示。

**修复**：
- 代码修正（已在副本中验证，`"note": 1`、`"pageerrors": []`）：
  ```js
  const note=L.control({position:'topright'});note.onAdd=function(){...;return d;};note.addTo(map);
  ```
- 在面板首行加上 “Screening / triage only — not an ETS eligibility determination”；再加 `L.control.scale()`、图例（红框 = 待评审样本）、`<meta name="viewport" content="width=device-width,initial-scale=1">`，popup 中给出对应卡片 `cards/NN_<id>.jpg` 的链接。
- README:79 改为 “Leaflet bundled (no CDN); imagery tiles require network access to basemaps.linz.govt.nz”。
- 增加运行时冒烟测试（Playwright 可用则执行，否则 skip）：断言 `pageerror` 为空，且 `.note` 数量为 1。

---

## DOC-02 — Medium — 固定评审样本的来源不可复现，抽样框缺少 advisory 候选

**位置**：`outputs/gisborne/review/review_sample_ids.csv`；`README.md:77`（“Fixed-seed sample of 30 candidates”）；`arcgis/README.md:27-32`；`src/ets_screening/sample_review.py:44-56`

**问题**：当前代码的抽样算法是 SHA-256 哈希排序，用它和默认 seed 对当前的 2,687 个候选重抽，结果与已提交的 30 个 ID **重合 0 个**。即便只在 2,520 个 clean 候选上用当前算法重抽，重合也是 0。已提交的样本与 `DataFrame.sample(n=30, random_state=20260912)` 在 2,520 个 clean 候选上的结果**完全相同**。这说明样本来自一个已被删除的旧算法。仓库只有 1 个 commit，历史里无从核实。arcgis/README 把差异解释为 “committed sample predates the boundary-aware materiality rules”，但只换候选集不会得到 0/30 的重合，真正的原因是抽样算法换了。

**证据**：
```text
fresh == pinned: False overlap: 0 of 30
old-rule candidates 2520 overlap with pinned: 0          # 当前 hash 算法作用于旧候选集
old pandas.sample overlap 30                             # DataFrame.sample(n=30, random_state=20260912)
advisory candidates 167 of 2687 6.215%; advisory in pinned 0
```

**影响**：README 在 Finding 3 中承诺将来会发布 “visual agreement / false-positive rate”。按现在的样本，这个比率只能代表 clean 候选：167 个 advisory 候选的入样概率为 0，而恰恰是这批最可能因边界问题改变结论。文档把它称为 “Fixed-seed sample of 30 candidates”，读者会误以为是当前候选集的随机样本，而且用仓库代码无法复现。

**修复**：评审尚未开始，现在改代价最小。任选其一：
- (a) 用当前算法对当前候选重抽，重新渲染卡片；
- (b) 保留现样本，但在 README、`review_summary.csv` 和 `findings` 中写明抽样框与算法：“30 drawn with `DataFrame.sample(n=30, random_state=20260912)` from the 2,520 clean candidates of an earlier run; advisory candidates had zero inclusion probability”，同时更正 arcgis/README 的解释。

两种方案都应把抽样框 ID 列表的 sha256 写入 `review_summary.csv`。

---

## DOC-03 — Medium — 评审卡片的十字标记位置错误，双面板大多重复

**位置**：`scripts/download_review_cards.py:70-75`（mosaic 原点对齐到瓦片边界）、`:101-106`（十字固定画在面板像素 (384, 384)，caption 写的是 “DETAIL … at interior point”）、`:64-67`

**问题**：`representative_point` 在面板中的像素位置是 `256 + (cx mod 256)`，可落在 [256, 512) 的任何位置，而十字始终画在 (384, 384)。另外，小多边形在 `_choose_level` 下同样返回 L10，于是 “overview” 与 “detail” 两个面板完全相同。

**证据**（对已提交的 `review_queue.gpkg` 离线复算；卡片 `01_lcdb1000012117.jpg` 目视可见十字不在红框内）：
```text
cards where overview level == detail level (L10): 21 of 30
crosshair offset from interior point (m): min 15 median 124 max 218
crosshair falls outside the polygon on 15 of 30 cards
```

**影响**：README:78 把这些卡片称为 “Dual-panel imagery cards”，并作为人工判读的主要材料。一半卡片的“interior point”标记指向多边形外的地物，容易误导判读；70% 的卡片没有真正的概览上下文。卡片上也没有影像署名和 “screening only” 字样（见 DOC-06）。

**修复**：
```python
def panel(level):
    ...
    return mosaic, (centre_x - origin_x, centre_y - origin_y)
detail, (dx, dy) = panel(detail_level)
X = TILE_SIZE * 3 + dx
draw.line((X - 10, dy, X + 10, dy), ...); draw.line((X, dy - 10, X, dy + 10), ...)
overview_level = min(_choose_level(geometry.bounds), detail_level - 3)   # 保证概览比细节至少粗一级以上
```
更好的做法是拼 4×4 瓦片后以点为中心裁出 768×768。caption 中加入 “Imagery: Gisborne DC 2024 (credited to LINZ) | screening review aid – not an eligibility determination”。改完后应重新渲染，这同样要在评审开始前完成。

---

## DOC-04 — Low — 多瓣统计描述与算法不符，且无法复现

**位置**：`README.md:54-62, 241-242`；`arcgis/README.md:75-76`；`notebooks/01_width_rule_comparison.ipynb` cell 0（“recomputes each headline number in the project README”）

**证据**（我对 694 个分歧单元做 `buffer(-15)` 的复算）：
```text
multi-part cores 181 26.1%                         # 181 / 26.1% 正确
500  every part >t: 16   >=2 parts >t: 53
1000 every part >t: 1    >=2 parts >t: 23
lcdb1000453434: width_area_perimeter_m 22.458; core parts 6 [877.2, 205.6, 4246.6, 251.4, 14.7, 0.0]
```
README 写的是 “requiring **every** part to exceed 500 m² gives 53, and 1,000 m² gives 23”。按字面理解应为 16 和 1；53 和 23 实际对应“至少两个部分超过阈值”。这三个数在仓库的任何脚本、`findings.json` 和 notebook 里都算不出来，但 notebook 声称复算了 README 的每个主要数字。插图单元 `lcdb1000453434` 的核心有 6 个碎片，“two separate lobes”只有在 >500 m² 的口径下才成立。

**修复**：改成 “at least two core parts larger than 500 m² → 53 (1,000 m² → 23)”。在 `build_findings.py` 中输出 `width_multipart_core_units` 和 `width_multipart_core_units_ge2_parts_over_{500,1000}m2`，并加进 notebook。PDF 与 arcgis/README 改为 “six core fragments, two larger than 500 m²”。

---

## DOC-05 — Low — 过时或不准确的表述

| 位置 | 文档说法 | 实测 |
|---|---|---|
| `README.md:209` | “54 tests …” | `pytest` → `55 passed in 6.65s` |
| `data/README.md:30` | “`data/sample/` retains 12 invented NZTM geometries only for fast unit tests” | 3 个文件共 14 个几何（12 + 1 + 1）；`tests/` 中没有任何文件读 `data/sample`，测试用的是内存中的 `build_demo_layers()` |
| `data/README.md:21` | 下载流程会 “reproject to `EPSG:2193`” | 脚本不做重投影，而是请求时带 `outSR=2193`，再直接标注 `crs="EPSG:2193"`（`download_gisborne_data.py:87-90`） |

**修复**：改为 55（或删掉具体数字，避免反复过时）；data/README 改为 “12 candidate + 2 overlay fixtures used by `ets-screen --demo` / regenerated by `demo_data.py`”，并说明测试使用内存 fixture；把 “reproject” 改为 “requested in EPSG:2193 via `outSR`”。

---

## DOC-06 — Low — 免责声明没有覆盖全部面向用户的输出

**现状（逐一核对）**：

- **有**免责声明：
  - `README.md:3-7`
  - `run_manifest.json`（`"scope": "screening/triage only; not an eligibility determination"`）
  - `figures/screening_overview.png`（子标题）
  - `outputs/gisborne/layout_map.pdf` 与 `arcgis/layout_map.pdf`（均含 “SCREENING / TRIAGE ONLY - NOT AN ELIGIBILITY DETERMINATION”，后者为 420×297 mm，与 A3 描述相符）
  - `.pyt` 的 description
- **没有**免责声明：
  - `summary.csv`、`findings.csv`/`findings.json`、`rule_results.csv`、`advisory_candidates.csv`、`review_summary.csv`
  - `figures/width_method_comparison.png`、`figures/width_disagreement_cases.png`
  - 30 张卡片与 3 张 contact sheet
  - `review_map.html`（`<title>ETS review queue</title>`，容易被误读为官方 ETS 队列）

**影响**：CSV 和 JSON 最容易被单独转发或导入其他系统，一旦脱离 README，就看不出这些只是筛查结果。

**修复**：`summary.csv` 与 `findings.json` 加入 `scope` 行或键（直接复用 `run_manifest` 的文本）；status 列的取值已经是 `candidate_review`，这一点很好。图和卡片加上页脚文字；HTML 的 `<title>` 改为 “ETS screening review queue (triage only)”。

---

## DOC-07 — Low — 输出的机读性与命名一致性

**位置与证据**：
- `scripts/build_findings.py:189`：`findings.csv` 中嵌套值是 Python repr，不是 JSON，无法用 `json.loads` 解析：
  ```text
  failed_by_rule,"{'R-01': 1489, 'R-02': 1092, 'R-03': 691, 'R-04': 224, 'R-05': 1574}"
  width_example_ids,"['lcdb1000002347', 'lcdb1000004900', 'lcdb1000008518-part-1']"
  ```
- `src/ets_screening/rules.py:152-155`：`rule_results.csv` 中 R-07 的 `rule_name` 是 “Crown cover at maturity”，丢掉了 `rule_register.csv`、README 和 SOURCES.md 反复强调的 “more than 30% **in each hectare**”。R-01、R-02 的名称也与 register 不一致（“Area at least 1 ha” 与 “Area at least 1 hectare”；“30 m width erosion proxy” 与 “Average width at least 30 metres”）。
- `summary.csv` 为 `manual_rules_out_of_scope,3`，而 `run_manifest.json` 的 `unresolved_rules` 有 4 项（含 R-03），每行的 `manual_review_rule_ids` 也列了 4 项。

**修复**：写 CSV 时对非标量值做 `json.dumps(v)`；`rule_name` 直接从 `rules/rule_register.csv` 读取，保证单一来源；summary 中增加 `rules_requiring_manual_review,4` 并注明 R-03 属于 partial。

---

## DOC-08 — Low — README 过度宣称 “机器 reviewer 会被拒绝”

**位置**：`README.md:86-88, 218-219`；`src/ets_screening/review_labels.py:28-34`

**证据**：
```text
ACCEPTED  machine[Claude_Opus] / [GPT4o] / [OpenAI o3] / [Gemini2.5] / [Llama 3] / [Mistral Large] / [C1aude]
rejected  human[Ai Tanaka] / [Tom Bot] / [Kate Model]  -> "looks automated"
```
`\b` 词边界在 `_` 和数字处不起作用（例如 `Claude_Opus`、`GPT4o`），而 “Ai” 是常见的日文、中文名。

**影响**：README 称 “automated labels cannot enter the committed evidence chain”，这是做不到的：任何人都可以直接填 “Jane Smith”。与此同时，这个启发式会拒绝真实姓名。

**修复**：README 降级为 “a heuristic name filter catches obvious assistant/model names; it is not an authentication control”。真正的控制应当是：要求 reviewer 的 GPG 签名提交或 PR 审批，或在 `review_summary.csv` 中记录 attestation 和原文件 sha256。同时去掉 `ai` 这类高误伤词，或改成提示警告。

---

## NC-01 — 需确认 (Needs confirmation)

1. **LINZ 无 key 瓦片是否可用**：README:194-196 称 “otherwise writes a valid key-free URL”。据我所知，LINZ Basemaps 对瓦片请求要求 `?api=` key（开发者 key 为临时 key），无 key 的请求可能返回 4xx，这样地图在任何环境下都没有影像。本环境访问 `basemaps.linz.govt.nz` 被出站策略拦截（CONNECT 403），未能实测。建议作者用 `curl -I` 核实，并把结论写进 README。
2. **外链有效性**：`rules/SOURCES.md` 与 `rule_register.csv` 中的 legislation.govt.nz 链接（`/act/public/2002/40/en/latest/sections/DLM158592/`）、MPI `dmsdocument/71887`（称 “dated 10 May 2026”）、LRIS 123148、Stats NZ 123497 等，本环境无法访问，未核实。仓库内部相互引用一致，相对链接 16 条全部存在（脚本已核对）。
3. `data/README.md:10` 把 Living Atlas 镜像称为 “authoritative ArcGIS mirror”，而 README:115 称 “ArcGIS Living Atlas mirror … simplified at 15 m”。一个经过 15 m 简化的第三方镜像能否称为 “authoritative”，需要作者确认措辞。

---

## 已核实为正确的内容（供交叉参考）

- README 中的数字与已提交输出**全部一致**：5,712；302/78/275,035.5/283,698.7/679.3 ha；R-03 209/900；694（12.15%）且方向全部为 ap_fail/erosion_pass；181（26.1%）；4,223/0；36 个碎片；2,687 = 2,520 + 167；2,801；224；52.96%；0.0001/2.4853/36,801.4 ha；lcdb1000453434 的 2A/P = 22.458 m。
- 跨文件一致性：`findings.json` 与 `findings.csv` 33 个键全部一致；`summary.csv` 的 7 个指标与 findings 一致；`rule_results.csv` 共 45,696 行 = 5,712 × 8，各规则失败数与 `failed_by_rule` 一致；ID 集合在 `candidates.gpkg ∪ quarantine.gpkg`、`rule_results`、`width_method_comparison` 之间完全一致，且候选与隔离两组无交集；`advisory_candidates.csv` 167 行，是 candidates 的子集，且按 overlap 降序；30 个样本 ID 与模板、`review_queue.gpkg`、30 张卡片文件名、HTML 中 30 个 feature 完全一致，且全部仍在候选集中；`review_summary.csv` 为 pending 状态，与 findings 一致。

## 优点

- **供应链与传输**：`vendor/leaflet-1.9.4/leaflet.js` 与 npm `leaflet@1.9.4` 的 `dist/leaflet.js` 字节一致（sha256 `db49d009…e5641a`），CSS 和 LICENSE 仅有 CRLF 差异；HTML 不从任何 CDN 加载；所有外部 URL 都是 HTTPS；`urllib` 保持默认 TLS 校验并设置了显式 timeout；全仓库没有 `eval`、`exec`、`pickle`、`subprocess`、`shell=True`、`verify=False`，也没有解压操作和硬编码密钥（grep 已核实）。
- **可复现性扎实**：`verify_checksums.py` 全部通过；`reproduce.py` 在不同环境下 CSV 和 JSON 按字节复现，GeoPackage 语义哈希一致，同一环境下连续两次运行的 gpkg、HTML、PDF、PNG 也按字节稳定；输出先写入 staging 临时目录再替换，失败时不会留下半成品。
- **结论与输出诚实、自洽**：README 的每个主要数字都能从已提交的产物复核；pending 状态被如实记录，没有发布任何准确率；PDF、总览图和 `run_manifest.json` 都带有 “screening/triage only” 声明；SOURCES.md 对 R-01/R-02 的邻接例外、R-04/R-05 属项目政策等局限说得很坦诚。
- **标签 ingest 在结构层面是 fail-closed 的**：列名和顺序严格匹配；重复 ID、缺失 ID、多余 ID 与词表外标签都会被拒绝；UTF-8 BOM 能正确处理；拒绝时返回码非零且信息明确。
