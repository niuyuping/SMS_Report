# OrderEntryNo_240522 与 WebSms 导出 Excel 页面对比

---

## 【导出 Excel 按钮所在的“页面”是否一致】

**结论：不完全一致。** 同一业务“出荷依頼”对话框，但 240522 是 Java AWT 对话框、WebSms 是 Web 模态框，**项目与布局有对应也有差异**。下表仅对比“导出 Excel 按钮所在的那一页”（即出荷依頼对话框）的构成。

| 项目 | OrderEntryNo_240522（ForwardRequestDialog） | WebSms（forwardDialog.jsp） | 是否一致 |
|------|---------------------------------------------|-----------------------------|----------|
| **画面标题** | 「出荷依頼表 出力」（midashi_，label 5121） | 「出荷依頼書」 | 不一致（文言不同） |
| **印刷種別** | 新規印刷 / 修正印刷 / 削除印刷（3 个 ToggleButton） | 新規印刷 / 修正印刷 / 削除印刷（radio） | 一致 |
| **備考** | 5 行 TextArea，空白区切り最大 5 項目 | 備考 textarea，同上 | 一致 |
| **利益センター** | 右側 **リスト**（list1_：14 行×3 列、事業所コード等）で 1 行選択 | **セレクト**（dropdown）で 1 件選択 | 不一致（UI 形態不同） |
| **修正内容（試作＋修正印刷時）** | 「修正内容を選択して下さい。」＋ 販価/数量/販売モデル/その他（4 个 ToggleButton） | 同上ラベル＋4 个 checkbox | 一致（表現のみ checkbox） |
| **確認/発行** | 按钮「**確認**」→ 出荷依頼書発行（URL 表示） | 按钮「**発行**」→ 同様に発行 | 不一致（240522 は「確認」、WebSms は「発行」） |
| **Excel 按钮** | 按钮「to excel」（label 2950 で「Excel出力」に変更可） | 按钮「**EXCEL出力**」 | 一致（文言はほぼ同じ） |
| **閉じる** | 按钮「閉じる」 | 按钮「閉じる」 | 一致 |
| **エントリーNo・枝番・明細の表示** | ダイアログ内には**表示なし**（親画面で選択済み） | **あり**：エントリーNo、枝番、顧客注文No、最終更新日、稼働日、OUT-IN、当該枝番の注文数/出荷済数/出荷残数/ユーザ納期/出荷希望日/回答日 | 不一致（WebSms のみ参照・明細表示あり） |
| **OUT-IN** | ダイアログ内に**なし** | **あり**（OUT-IN 一覧表示＋「OUT-IN編集」） | 不一致（WebSms のみ） |
| **出荷確定** | ダイアログ内に**なし** | **あり**（出荷確定数入力＋「出荷確定」按钮） | 不一致（WebSms のみ） |

- **一致している点**：印刷種別 3 種、備考、修正理由 4 種、Excel 按钮の存在と役割、閉じる。  
- **不一致な点**：标题文言、確認 vs 発行、利益センターがリスト vs セレクト、240522 にはない「参照・明細」「OUT-IN」「出荷確定」が WebSms にある。

因此：**“导出 Excel 按钮所在页面”在业务上是对应的（同一出荷依頼对话框），但画面项目与 UI 并非完全一致**；若需与 240522 完全一致，需在 WebSms 侧调整标题/按钮名、利益センターをリスト形式にする、および参照・OUT-IN・出荷確定の有無を揃える必要がある。

---

## 一、导出入口与触发方式

| 项目 | OrderEntryNo_240522 | WebSms |
|------|---------------------|--------|
| 入口位置 | 出荷依頼对话框内，通过 `forwardDialog_.isExcel()` 判断是否执行 Excel 输出 | 出荷依頼对话框内独立按钮「EXCEL出力」 |
| 触发时机 | 用户在同一对话框中选择“Excel”后，点击执行（与出荷依頼書発行同一流程分支：`else { if(forwardDialog_.isExcel()) toExcel(...); }`） | 用户直接点击「EXCEL出力」按钮 |
| 实现位置 | `OrderEntryNo_240522.java` 约 4192–4200 行 | `sms01206-dialog-forward.js` 的 `exportExcel(vm)`，跳转至 `report/getExcel?…` |

---

## 二、“Excel 页面”形态差异（核心区别）

### 240522 的 Excel 流程（Applet + 浏览器）

1. **在 Applet 内执行 `toExcel(key, fdata, riekiCenterItem, sRiyuu)`**
   - 用 `getExcelData(key)` 取得数据，按 `editExcel1`～`editExcel5`、必要时 `editDoukonExcel` 拼出整份 **TSV 文本**。
   - 将 TSV 写入 **系统剪贴板**（`StringSelection` → `Clipboard.setContents`）。
   - 用 `getAppletContext().showDocument(new URL(whttp), "_blank")` 打开新窗口。
   - 打开的 URL：`http://{server}/{EXCEL_FOLDER}/{PROGRAM_NAME}/{HTML_NAME}`  
     即：`http://{server}/excel/Sms01206/Sms01206_01.html`。

2. **“Excel 页面” = 该 HTML 页面**
   - 文件：`Sms01206/excel/Sms01206_01.html`。
   - 内容：仅 `onLoad="replacepage()"`，在 `replacepage()` 中执行：
     - `window.location.replace(urlurl);`
     - `urlurl = surl[0] + "Sms01206/Sms01206_01.xls"`
   - 即：**页面加载后立即重定向到服务器上的 `.xls` 地址**（`surl` 来自 `excel_server_url.js`）。
   - **数据**：TSV 已在客户端剪贴板中，未通过该 HTML 或 URL 参数传给服务器；.xls 多为模板或另途生成，用户可能需自行粘贴剪贴板内容。

### WebSms 的 Excel 流程（纯 Web）

1. **用户点击「EXCEL出力」**
   - JS 拼出查询参数：`entryNo`, `subNo`, `fdata0`～`fdata5`, `riekiCenterItem`, `sRiyuu` 等。
   - 直接：`window.location.href = contextPath + '/websms01206/maintenance/report/getExcel?' + params`。

2. **服务端 `report/getExcel`**
   - Controller：`WebSms01206Controller.downloadForwardExcelFull(...)`。
   - 调用 `webSms01206Service.buildForwardExcelTsv(entryNo, subNo, fdata, riekiCenterItem, sRiyuu)` 生成与 240522 一致的 **整份 TSV 文本**。
   - 将 TSV 交给视图：`ForwardExcelTsvDownloadView`。
   - 视图中：把 TSV 按行/列解析，用 POI 生成 **.xlsx**，设置 `Content-Disposition: attachment`，**直接以附件下载**。

3. **“页面”形态**
   - **没有单独的“Excel 页面”**：不打开中间 HTML，不写剪贴板，不重定向到 .xls。
   - 浏览器只是发起一次 GET（带参数），然后收到一个 **.xlsx 文件下载**（文件名如 `conyyyyMMddHHmmss.xlsx`）。

---

## 三、对比小结（页面与交互）

| 对比项 | OrderEntryNo_240522 | WebSms |
|--------|---------------------|--------|
| 是否有可见“Excel 页面” | 有：先打开 `Sms01206_01.html`，再立即跳转到 `Sms01206_01.xls` | 无：仅触发文件下载，无中间页 |
| 剪贴板 | 使用：TSV 写入剪贴板 | 不使用 |
| 新窗口 | 使用：`showDocument(..., "_blank")` 打开 HTML | 使用：当前窗口跳转到 getExcel，下载后仍可停留在原页或由浏览器处理 |
| 文件格式 | 重定向到 `.xls`（由 `excel_server_url.js` 等决定） | 直接下载 `.xlsx`（POI 生成） |
| 数据到文件的途径 | 客户端拼 TSV → 剪贴板；服务器侧 .xls 可能为模板或另途生成 | 服务端拼 TSV → 服务端转 .xlsx → 响应体下载 |
| 试作・修正理由 | 试作且修正印刷时需选修正理由，否则不执行 toExcel（对话框逻辑） | 试作且修正印刷时未选修正理由则禁用「EXCEL出力」并提示（`forwardDialogExcelDisabled()`） |

---

## 四、导出内容与列定义（一致部分）

- **列定义**：WebSms 的 `ForwardExcelTsvBuilder.EXCEL_ITEMS` 与 240522 的 `excelItems_` 一致（约 44 项，含 `d_delivery_date`～`d_comment_note` 表部分、同梱用等）。
- **结构**：均为 上部（editExcel1）＋ 表（editExcel2）＋ 下部（editExcel3/4）＋ 最下部（editExcel5）＋ 同梱时（editDoukonExcel），TSV 格式与 240522 的 toExcel 输出兼容。
- **参数**：entryNo / subNo / fdata / riekiCenterItem / sRiyuu 在 WebSms 中通过 URL 参数传递，与 240522 的 toExcel 参数对应。

---

## 五、WebSms 额外提供的导出方式

- **`report/forward.xlsx`**：仅表形式（getExcelData 的订单数据），带 DataType 列，无上部/下部/同梱。与 240522 的“整份版式”不同，为简化版 Excel 导出。

---

## 六、结论（页面与体验差异）

1. **240522**：导出 Excel 会先**打开一个 HTML 页面**（`Sms01206_01.html`），该页**立即重定向到 .xls**；TSV 在**剪贴板**，依赖 Applet 与可能存在的服务器 .xls 或用户手动粘贴。
2. **WebSms**：**没有**对应的“Excel 页面”，点击「EXCEL出力」即**直接下载 .xlsx 文件**，无剪贴板、无中间 HTML、无重定向，导出内容与 240522 的 toExcel 整份版式一致。

若需在 WebSms 中“再现”240522 的“先打开一个再跳转”的页面行为，可在前端先打开一个空白或说明页再触发 `report/getExcel` 的下载（或新窗口打开 getExcel），但从功能与数据上，当前 WebSms 已实现与 240522 相同的 Excel 导出内容与参数逻辑。
