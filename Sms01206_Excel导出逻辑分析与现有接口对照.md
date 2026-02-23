# Sms01206 Excel 导出逻辑分析与现有接口对照

## 一、Sms01206（Java）Excel 导出逻辑

### 1. 入口与参数

- **触发**：出荷依頼 Dialog（ForwardRequestDialog）选择「EXCEL出力」后调用 `toExcel(key, fdata, riekiCenterItem, sRiyuu)`。
- **参数**：
  - `key`：`[entryNo, subNo]`（枝番表当前选中的エントリーNo、枝番）。
  - `fdata`：Dialog 内输入的 **配布先 / 商品企画 / 販売 / 品証 / 生産管理 / 試作 / 他** 等 7 项（`excelUnder1_` 对应标签的输入值），来自 `forwardDialog_.getValue1()` 的 `ans`。
  - `riekiCenterItem`：利益センター，来自 Dialog 的 `ans1`。
  - `sRiyuu`：理由（Vector），来自 Dialog 的 `ans2`。

### 2. 数据取得

- **getExcelData(key)**  
  - 客户端将 `KEY=key`、`KEYS=excelItems_`、`LANG`、`TYPE` 传给 RMI。  
  - 服务端（OrderEntryNo_s）按 `d_entry_no`、`d_entry_sub_no` 查询该枝番下所有明细，按 **excelItems_** 顺序组装成 `Vector<String[]>` 返回（每行一个 String[]，与 excelItems_ 列顺序一致）。  
- **excelItems_**：约 47 项，包含 d_entry_no, d_entry_sub_no, d_detail_no, d_charge_code, …, d_delivery_date～d_comment_note（表部分）, d_case_mark1～7, d_consign_*, d_carrier_*, d_note, d_package_number, d_currency_code, d_inspect_date, d_handle_code, d_npm_entry_no 等。

### 3. 排版与输出（客户端）

整份 Excel 内容在客户端拼成 **一段タブ区切りテキスト**（printData），再：

- 复制到系统剪贴板（StringSelection → Clipboard）；
- 用浏览器打开固定 URL：`http://{serverNM}/excel/Sms01206/Sms01206_01.html`（新窗口）。

即：**Java 端不生成 .xlsx 文件**，而是生成 TSV 文本 + 剪贴板 + 固定 HTML 页面（由该 HTML 负责粘贴并呈现/另存为 Excel）。

排版由 5 块 + 同梱块 组成：

| 块 | 方法 | 内容概要 |
|----|------|----------|
| **上部** | editExcel1 → editExcelOutData1 | 1 行头信息：エントリー+枝番、ランク、通貨・単価、担当・得意先・仕様・荷受・機種・注文No・伝票・添付・用途・扱い、以及 **fdata（配布先等 7 项）**、**sRiyuu（理由）**。配送先等用 consignData_ 补足。 |
| **表** | editExcel2 | exVector 每行从 **exStart_**（d_delivery_date）到 **exEnd_**（d_comment_note），最多 10 行；回答区分用 getReplyCodeName 转名称；不足 10 行补空行。 |
| **下部** | editExcel3（一般）或 editExcel4（試作） | **editExcel3**：連絡事項・配送先、ケースマーク・住所、搬出/搬入日時、コメント、case_mark4～7 + **fdata[1]～[6]**。**editExcel4**：機展書 No、npm 関連、配送先等（getExcelDataNpm、getNpmData、getNpmProductDivision 等）。 |
| **最下部** | editExcel5 | 一般：**excelUnder1_** 固定标签（配布先/商品企画/販売/品証/生産管理/試作/他）。試作：**excelUnder2_**（作番、利益センター、試作台数、材料見積、組立見積、倉入単価）、**excelUnder3_**（承認・設計部署長等）。 |
| **同梱** | editDoukonExcel | 同梱品时（d_package_number 非空）：同梱親子関係をたどり、**editExcelDoukonDetail** で同梱子明细を追加。 |

此外：

- 機展機種（NP- 开头）时用 **getNpmData** 补足 fdata 中的コメント。
- **judgeShisakuProduct** 为試作時走 editExcel4 / editExcel5 試作用分支。

### 4. 小结（Java）

- **数据**：与 excelItems_ 一致的明细数据，来自 **getExcelData** 一个 RMI。
- **版式**：多块固定版式（上部 / 表 / 下部 / 最下部 / 同梱），依赖 **fdata、riekiCenterItem、sRiyuu** 及同梱・機展・試作等逻辑，在**客户端**拼接为 TSV。
- **输出**：TSV 文本 → 剪贴板 + 打开固定 HTML，**不生成 .xlsx**。

---

## 二、现有 Web 导出接口

### 1. 接口与数据流

- **api/getExcelData**（POST）  
  - 入参：type, lang, key（如 [entryNo, "0"]）, keys（列 key 列表，与 excelItems_ 一致）。  
  - 行为：透传 RMI `getExcelData`，返回 ans（List，每行与 keys 顺序一致）。  
  - **与 Sms01206 使用的 getExcelData 为同一数据源。**

- **report/forward.xlsx**（GET，参数 entryNo）  
  - 调用 `getExcelData(key=[entryNo,"0"], keys=EXCEL_KEYS)` 取数。  
  - 将 ans 转为 `excelRows`（每行 Map，含 d_data_type + EXCEL_KEYS 各列），`excelColumns` 为 EXCEL_KEYS 对应表头。  
  - 交给 **ExportExcelDownloadView** 生成 **.xlsx** 下载（单 Sheet「出荷依頼」、表头 + 数据行）。

- **ExportExcelDownloadView**  
  - 从 model 取 `excelRows`、`excelColumns`、`fileName`，用 POI 生成 **.xlsx**（表头 + 数据行，autoSizeColumn）。

### 2. 前端

- 出荷依頼 Dialog 的「EXCEL出力」：跳转 `report/getExcel?entryNo=xxx`（若实际路由为 `report/forward.xlsx`，需统一 URL 或做重定向）。

---

## 三、能否用现有接口实现「与 Sms01206 同样的」导出？

### 1. 仅就「明细数据导出为 Excel」——能

- **数据**：现有接口已调用与 Sms01206 相同的 **getExcelData**，列与 **excelItems_** 一致（Web 的 EXCEL_KEYS 与 excelItems_ 对应）。
- **格式**：当前是「单表 + 表头 + 多行数据」的 .xlsx，等价于 Sms01206 的 **表部分（editExcel2）** 的表格化；列范围相当于 exStart_～exEnd_ 并含前后必要项目。
- **结论**：**用现有 getExcelData + report/forward.xlsx + ExportExcelDownloadView，已经能实现「与 Sms01206 表部分一致的明细 Excel 导出」。**

### 2. 若要实现 Sms01206「完整」导出（上部・下部・最下部・同梱・試作/機展）——不能直接实现，需扩展

差异要点：

| 项目 | Sms01206（Java） | 现有 Web 接口 |
|------|------------------|----------------|
| 数据 | getExcelData 一份 | 同一 getExcelData ✅ |
| 上部块 | editExcel1：头行 + fdata + sRiyuu | 无 fdata/riekiCenterItem/sRiyuu，未实现上部块 ❌ |
| 表块 | editExcel2：最多 10 行、回答区分名称化、空行补齐 | 有表结构，可做成一致 ✅ |
| 下部块 | editExcel3/4：配送先・ケースマーク・搬出搬入・fdata[1]～[6]、試作/npm | 无 ❌ |
| 最下部块 | editExcel5：配布先/商品企画/… 或 作番/利益センター/… | 无 ❌ |
| 同梱块 | editDoukonExcel | 无 ❌ |
| 输出形式 | TSV + 剪贴板 + 固定 HTML | .xlsx 单表下载 |

要实现「完整一致」需要至少满足：

1. **取得 Dialog 侧输入**：  
   - 出荷依頼 Dialog 的 **fdata**（配布先等 7 项）、**riekiCenterItem**、**sRiyuu** 在导出时能传到后端或前端导出逻辑。
2. **服务端或前端实现排版逻辑**：  
   - 等价实现 editExcel1～5、editDoukonExcel（或由后端提供「已按 Sms01206 规则拼好的多段数据/TSV」）。
3. **同梱・試作・機展**：  
   - 同梱：需同梱子数据（可考虑 getExcelData 多 key 或单独接口）。  
   - 試作/機展：getNpmData、getExcelDataNpm、getNpmProductDivision 等若仅在 RMI 存在，需在 Web 侧暴露并在此处调用。
4. **输出形式二选一**：  
   - 要么继续「TSV + 固定 HTML」方式（与 Java 一致）；  
   - 要么用 .xlsx 多区域/多 Sheet 复刻上部・表・下部・最下部・同梱（实现量大）。

---

## 四、结论与建议

- **仅需「与 Sms01206 相同的明细表 Excel 下载」**：  
  **可以**，现有 **getExcelData + report/forward.xlsx + ExportExcelDownloadView** 已满足；只需确认前端的 report/getExcel 是否指向 report/forward.xlsx（或改为统一使用 report/forward.xlsx）。

- **需要「与 Sms01206 完全一致的整份 Excel（上部・表・下部・最下部・同梱・試作/機展）」**：  
  **不能**仅靠现有接口实现。需要：  
  1）在导出流程中传入 fdata、riekiCenterItem、sRiyuu；  
  2）实现 editExcel1～5、editDoukonExcel 的等价逻辑（或后端提供等价 TSV/结构化数据）；  
  3）同梱・試作・機展用到的 API 在 Web 可用；  
  4）选择 TSV+HTML 或 .xlsx 多块版式并实现对应输出。
