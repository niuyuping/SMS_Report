# OrderEntryNo_240522 機種・ランク・販価 关联关系分析

以 **OrderEntryNo_240522.java** 为基准，整理 **機種（product）**、**ランク（rank）**、**販価（単価・販価決定区分）** 相关组件的关联关系，并对照 Web createForm 的实现差异。

---

## 一、Java 侧组件与常量

| 概念 | 控件/变量 | 说明 |
|------|-----------|------|
| 機種 | `productChoice_` | 机种下拉（代码 + 名称） |
| | `kokyakuProduct_` | 顾客机种名（由 getPrice 回填） |
| | `productButton_` | 机种明细对话框按钮 |
| ランク | `rank_` | ランク显示/输入 |
| | `rankButton_` | 打开 RankDecisionDialog |
| 販価・単価 | `price_` | **表头単価**（显示/输入；受販価区分控制是否可编辑） |
| | `pkChoice_` | **販価決定区分**（標準=1、個別=3、無償出荷=8 等；决定 price_ 的取得方式与可编辑性） |
| | `priceCodes_` | 販価区分 Master（code↔名称） |

---

## 二、数据流与 API

### 2.1 機種 → 販価（表头単価・区分・通貨・顾客机种）

- **入口**：`productChoice___action_actionPerformed` / `productChoice___item_itemStateChanged` 中，在唯一确定机种后（`prmStr.length == 2` 或 item 选择）调用：
  - `setPriceMasterItem(new String(), product)`  
    → 内部若非 78T 机种则 `setPkChoice()` + `setPrice(customer, product)`。
- **setPrice(customer, product)**（约 6204–6322 行）：
  - 参数：`key = [customer, product]`（customer 空时从 `customerChoice_` 取）。
  - 调用 **`remoteObject_.getPrice(prmHash)`**，用回传数组设置：
    - 用途（prmStr[9,10]）— 注释中部分被 cut。
    - **販価**：`prmStr[11]` 非空 → `price_.setText`、`setPriceCodeName("1")`（標準）；空 → `setPriceCodeName("3")`（個別）。
    - 通貨：`prmStr[12]`。
    - 顧客機種：`prmStr[13]` → `kokyakuProduct_.setText`。
  - 特定条件下自动设定 rank=1/6：NNSN 或法人 R/L/M/S/V/Q/3 时 `checkWhenRank1("1", product)`；金型时 `checkWhenRank1("6", product)`。

即：**机种变更后，用「得意先 + 机种」从 getPrice 取得并设置表头単価、販価区分（1 或 3）、通貨、顾客机种**。

### 2.2 販価（単価）再取得：日付变更时

- **resetUnitPrice(int column)**（约 11375–11389 行）：
  - 仅在枝番表列为 `FORWARD_DETAIL` / `REPLY_DETAIL` / `ON_BOARD_DETAIL` / `FORWARD2_DETAIL` 时处理。
  - 若 **販価決定区分＝標準**（`getPriceCode(pkChoice_) == "1"`）则：
    - `getUnitPrice()`：参数为法人・机种・得意先・有効日等；
    - `setUnitPrice(data)`：更新 `price_` 与通貨。
- **getUnitPrice** 使用 `productChoice_` / `customerChoice_` 的 code 及 `getEffectiveDay()`，调用 **`remoteObject_.getUnitPrice(prmHash)`**。

即：**日期列变更且販価＝標準时，按当前机种・得意先・有効日再取得単価并回写**。

### 2.3 ランク 与 機種・販価 的联动

- **选 rank 的入口**：`rankButton___action_actionPerformed`（约 8748–8777 行）  
  → `displayRankDecisionDialog()`（用 customer + product 打开）  
  → 用户选择 `work`：
  - 若 `work == "1"`：先 `isKitenProduct(product,"1")`，非机展则 **`checkWhenRank1(work, product)`**；
  - 否则：`rank_.setText(work)`、`setUpdateHead(work, RANK_HEAD)`，并 `clearErrHash(PRICE_CODE_HEAD)`、`resetTablePlantSloc(false)`、`setCC8InputLock()`。

- **checkWhenRank1(rank, product)**（约 16283–16309 行）  
  仅当 rank 为 "1" 时执行一串校验，全部通过才 `rank_.setText(rank)`：
  1. `checkCurrencyDeployOrder(rank)` — 通貨・展開関連；
  2. **checkRank1UnitPrice()** — 表头「販価マスター存在」；
  3. **isStandardUnitPriceRank(product)** — 機種＋販価区分（個別不可）；
  4. `isCC8NotRank1()` — 与信管理判定=8 时不可 rank=1；
  5. `isNotRank1Outin()` / `isChangeRankOi()` — OUT-IN 等。

- **checkRank1UnitPrice()**（约 15773–15797 行）  
  - 条件：`mente_=="0"` 且 `erpProduct_`（ERP 连携机种）。  
  - 若 **!isUnitPriceMaster()**（法人・机种・得意先 在単価マスタ不存在）→ 报错「ランク=1 に設定出来ません」。

- **checkRank1UnitPriceProduct()**（约 15800–15825 行）  
  - 在 **机种变更后** 调用（productChoice action/item 内）。  
  - 条件：`rank_.getText()=="1"` 且 `erpProduct_`。  
  - 若 **!isUnitPriceMaster()** → 报错「ランク=1 ではこの機種は設定出来ません」并 **productChoice 红底**、`setErr(PRODUCT_HEAD, ERR)`。

- **isStandardUnitPriceRank(product)**（约 15874–15890 行）  
  - 若 **checkStandardUnitPrice(product)** 为 true → 报错「個別販価の為 ランク=1 に設定できません」。  
  - **checkStandardUnitPrice(product)**（约 15856–15872）：  
    - 条件：`mente_=="0"`，且非「標準単価チェック除外」（`judgeExceptStandardUnitPriceOrder(product)` 为 false）；  
    - 若 **販価決定区分＝個別**（`getPriceCode(pkChoice_) == "3"`）→ 视为错误（不可设 rank=1）。

- **isStandardUnitPrice(product)**（约 15828–15853 行）  
  - 在 **机种变更后** 调用（productChoice action/item 内），当 **rank==1** 时：  
    - 若 `checkStandardUnitPrice(product)` 为 true → **pkChoice 红底**、`setErr(PRICE_CODE_HEAD, ERR)`、弹窗「個別販価には設定できません」；  
    - 否则清除 PRICE_CODE_HEAD 错误。

即：**rank=1 与「表头単価マスタ存在」以及「販価区分≠個別」强绑定**；机种变更时若已是 rank=1，会同时校验「単価マスタ」和「不可個別」。

---

## 三、关联关系小结（Java）

| 方向 | 关联内容 |
|------|----------|
| **機種 → 販価** | 机种确定后 `setPriceMasterItem` → `setPrice` → **getPrice(customer, product)** 设置表头単価、販価区分(1/3)、通貨、顾客机种。 |
| **機種 → ランク** | 机种变更后若当前 rank=1：**checkRank1UnitPriceProduct()**（ERP 机种且无単価マスタ则报错+机种红）、**isStandardUnitPrice(product)**（個別则販価区分红+报错）。setPrice 内 NNSN/金型等会 **checkWhenRank1("1"/"6", product)** 自动设 rank。 |
| **ランク → 販価** | 选 rank=1 时 **checkWhenRank1** 内：**checkRank1UnitPrice()**（単価マスタ必须存在）、**isStandardUnitPriceRank(product)**（不可個別）。rank 从 1 改为其他时 **clearErrHash(PRICE_CODE_HEAD)**。 |
| **販価 → 機種** | 仅通过 getPrice(key 含 product)、getUnitPrice(isUnitPriceMaster) 的参数体现；无「販価变更导致机种清空」逻辑。 |
| **日付 → 販価** | 枝番表 納期/回収/搬出搬入等日付列 blur 时 **resetUnitPrice(column)**：販価＝標準(1) 时 **getUnitPrice() → setUnitPrice()** 再取得表头単価。 |

---

## 四、Web createForm 对照

### 4.1 已实现且与 Java 同构的部分

- **機種 → 販価**  
  - `onProductChange()` 内：用 `[customerCode, priceCode||'1', productCode]` 调 **getPrice**，回填 `unitPrice`、`currencyCode`、`customerProduct`；并根据 row[11] 有无自动设 `priceCode`（有→'1'、无→'3'），与 Java setPriceCodeName 一致。
- **日付 → 販価**  
  - `resetUnitPriceOnDateChange()`：在枝番日付 blur 时若 **priceCode==='1'** 则用 getPrice 再取得单价并更新表头，与 Java **resetUnitPrice** 行为一致。
- **ランク=1 且 個別販価**  
  - `validateForm` 及 販価区分变更处：若 `decisionRank === '1' && priceCode === '3'` 则报错「ランク=1 の場合は個別販価は設定できません」，与 Java **isStandardUnitPriceRank** / **checkStandardUnitPrice** 的提交时校验一致。

### 4.2 与 Java 的差异或可补齐点（已按下列项补全）

| 项目 | Java（OrderEntryNo_240522） | Web（当前 createForm） |
|------|-----------------------------|-------------------------|
| 机种变更后 rank=1 + ERP 机种 + 无単価マスタ | **checkRank1UnitPriceProduct()**：弹窗报错并将 **productChoice 红底**、setErr(PRODUCT_HEAD) | **已补全**：getPrice 回调内若 decisionRank==='1' 且 erpProductFlag && !isUnitPriceMasterFlag 则 validationErrors.productCode、showAlert「ランク=1 ではこの機種は設定出来ません」；機種 jbk-combo-box 绑定 :error |
| 机种变更后 rank=1 + 販価=個別 | **isStandardUnitPrice(product)**：**pkChoice 红底**、setErr(PRICE_CODE_HEAD)、弹窗「個別販価には設定できません」 | **已补全**：同上 getPrice 回调内若 priceCode==='3' 则 validationErrors.priceCode、showAlert「個別販価には設定できません」 |
| 选择 rank=1 时的一串校验 | **checkWhenRank1**：checkRank1UnitPrice、isStandardUnitPriceRank 等 | **已补全**：Rank 对话框 selectRank 时若 item.rank==='1' 先校验 priceCode!=='3' 与 isUnitPriceMaster，不通过则不写入 rank、不关闭对话框 |
| getPrice 回传后自动设販価区分 | 有単価 → setPriceCodeName("1")，无 → setPriceCodeName("3") | **已补全**：getPrice 回调内 row[11] 非空则 priceCode='1'，否则 priceCode='3' |

---

## 五、建议实现顺序（Web 侧）

1. **机种变更时**  
   - 若当前 **decisionRank === '1'**：  
     - 在已有 isUnitPriceMaster + judgmentERP 的基础上，若 **erpProductFlag && !isUnitPriceMasterFlag** 则弹窗「ランク=1 ではこの機種は設定出来ません」并将机种控件标错（与 Java checkRank1UnitPriceProduct 一致）。  
     - 若当前 **priceCode === '3'**（個別），则把販価区分标错并弹窗「個別販価には設定できません」（与 Java isStandardUnitPrice 一致）。
2. **选择 rank=1 时**（Rank 对话框确认时）：  
   - 调用与 Java 等价的 **checkRank1UnitPrice**（isUnitPriceMaster）、**isStandardUnitPriceRank**（checkStandardUnitPrice）；若存在 checkCurrencyDeployOrder / isCC8NotRank1 / isNotRank1Outin / isChangeRankOi 的 API 或逻辑，可一并在此执行，不通过则不写入 rank=1。
3. **getPrice 回传**：若后端能返回「応設販価区分（1/3）」或等价信息，在 onProductChange 的 getPrice 回调里设置 `formData.priceCode`，与 Java setPriceCodeName 一致。

以上为以 OrderEntryNo_240522 为基准的機種・ランク・販価 关联分析与 Web 差距要点。

---

## 六、价格（販価）与单价（単価）组件的关联与交互逻辑

本节在機種・ランク・販価 基础上，**单独展开「販価」相关组件中「价格（販価決定区分）」与「单价（表头単価・枝番単価）」的关联与交互**。

### 6.1 组件区分（Java）

| 组件 | 变量/常量 | 含义 |
|------|-----------|------|
| **表头単価** | `price_` | 表头显示的「単価」；可由 getPrice/getUnitPrice 回写，或用户在「個別」等时可编辑。 |
| **販価決定区分** | `pkChoice_` | 表头「販価決定区分」下拉：標準(1)、個別(3)、無償出荷(8)、報復(2)、ERP標準(E) 等；决定表头単価是否可编辑、是否从 API 再取得。 |
| **販価区分 Master** | `priceCodes_` | getPriceCodeMaster 取得，用于 getPriceCode(item) / setPriceCodeName(code) 的 code↔名称 映射。 |
| **枝番単価・区分** | 明细数据 `KO_PRICE_CODE` / `KO_UNIT_PRICE` | 子/枝番数据中的販価区分与単価；提交时按 isUnitPriceNotZero0(priceCode, price) 校验（修理品+無償出荷时 0 可）。 |
| **表头常量** | `PRICE_HEAD`(29)、`PRICE_CODE_HEAD`(38) | 表头更新・错误标红时使用。 |

- **getPrice**：参数为 [customer, product]，用于**机种确定后**一次取得「表头単価、応設販価区分(1/3)、通貨、顾客机种」并回写 `price_` / `pkChoice_`。
- **getUnitPrice**：参数为法人・机种・得意先・有効日等，用于**標準(1) 或 ERP標準(E) 时**按有効日再取得「単価・通貨」并回写 `price_`；日付列变更时的 resetUnitPrice、販価区分切到標準时的 resetUnitPriceSection 都会用到。

即：**「价格」在这里主要指販価決定区分（pkChoice_）；「单价」指表头 price_ 及枝番上的単価**；二者通过「区分决定単価是否可编辑、是否从 API 取得」紧密联动。

### 6.2 販価決定区分 → 表头単価・通貨（pkChoice 变更时）

- **入口**：`pkChoice___item_itemStateChanged`（约 12059–12083 行）。
  - 先 `clearErrHash(PRICE_CODE_HEAD)`、`pkChoice_.setBackground(window)`；
  - 非「会社間末オーダー」时调用 **`resetUnitPriceSection(code)`**（code = getPriceCode(pkChoice_.getSelectedItem())）；
  - 再 **isStandardUnitPrice(product)**（rank=1 时個別不可）、**checkCurrencyDeployOrder(rank)**。

- **resetUnitPriceSection(code)**（约 12086–12111 行）：
  - **code = "1"（標準）**：`getUnitPrice()` → `setUnitPrice(data)` 更新 `price_` 与通貨；若 setUnitPrice 返回 false（単価マスタ无数据等）则把 code 改为 "3"。
  - **code = "E"（ERP標準）**：先置 code="1"，再 `getUnitPrice("ERP")` → `setUnitPrice(data)`，失败同样 fallback 到 "3"。
  - 然后 **setPriceCodeName(code)**（刷新 pkChoice 显示）、**setUpdateHead(code, PRICE_CODE_HEAD)**、**resetPriceCurrency(code)**。

- **resetPriceCurrency(code)**（约 12126–12147 行）：
  - **code = "1" 或 "2"**：`price_.setEnabled(false)`、`currencyChoice_.setEnabled(false)`（表头単価・通貨不可编辑）。
  - **code = "8"（無償出荷）**：`price_.setText("0")`、`price_.setEnabled(false)`、通貨不可编辑、`setUpdateHead("0", PRICE_HEAD)`。
  - **其他**（含 "3" 個別）：`price_.setEnabled(true)`、通貨按 setCurrencyChoiceFlg 控制。
  - 最后 `price_` / `currencyChoice_` 背景置为 window。

即：**販価区分变更会驱动「表头単価」的取值方式（標準/ERP 时从 getUnitPrice 再取）和可编辑性（標準・報復・無償出荷时不可编辑；個別时可编辑）**。

### 6.3 表头単価输入校验（price_ 的 focus/action）

- **price___focus_focusLost** / **price___action_actionPerformed**（约 8721–8725、19123–19127 行）  
  → 均调用 **inputCheckPrice()**（约 19130–19146 行）：
  - 用 **setNumeric(key)** 格式化 `price_.getText()`，写回 `price_` 与 **setUpdateHead(..., PRICE_HEAD)**；
  - 若 **!isUnitPriceNotZero()** 则 **clearStatusQDataAll()**（販価=0 时状态清除）；
  - 异常时 **setErr(PRICE_HEAD, ERR)**、**price_.setBackground(red)**。

- **isUnitPriceNotZero()**（约 11071–11080 行）：  
  - 取 `priceCode = getPriceCode(pkChoice_.getText().trim())`、`price = price_.getText().trim()`；  
  - 调用 **isUnitPriceNotZero0(priceCode, price)**（约 14636–14659 行）：  
    - 若 price 非空且数值≠0 → true；  
    - 若 price 为空或 0：仅当 **priceCode = "8"（無償出荷）且 !erpFlag_** 时为 true（修理品时無償 0 不可）；否则 false。

即：**表头単価的输入与「販価決定区分」联动校验**：非無償时単価不可为 0；無償(8) 时可为 0，但修理品(erpFlag) 时仍不可 0。

### 6.4 更新/追加时的単価マスタ检查与区分切替

- **checkUnitPriceMaster(msg, tbl1, buttonKbn)**（约 11541–11577 行）：  
  - 若 **!isUnitPriceMaster()**（法人・机种・得意先 在単価マスタ不存在）：
    - **productChoice_.setBackground(orange)**；
    - 若当前 **getPriceCode(pkChoice_) == "1"（標準）** 则 **setPriceCodeName("3")**、**setUpdateHead("3", PRICE_CODE_HEAD)**（自动切到個別）；
    - 弹窗提示（更新时带同梱等文案）。
  - 插入/更新前会调用，用于「標準」却无マスタ时自动改为個別并提示。

- **displayUnitPriceZeroMessage()**（约 12177 起）：  
  - 販価=0 时的提示逻辑（修理品+無償出荷时 R3 移行不采用等），与 **isUnitPriceNotZero0** 规则一致。

### 6.5 枝番単価与表头的关系

- 提交/校验时，枝番行数据中有 **KO_PRICE_CODE**、**KO_UNIT_PRICE**（OrderEntryNo_i）；  
  - 与表头一致：表头 `price_` / `pkChoice_` 决定订单级「単価・販価区分」，枝番行带出或继承同一套（out[Y_PRICE_A] = setNumeric(price_.getText()) 等）。
  - 枝番行也使用 **isUnitPriceNotZero0(data[KO_PRICE_CODE], data[KO_UNIT_PRICE])** 做「販価区分＋単価」的 0 校验（约 14783），规则与表头一致（無償 8 且非 erpFlag 时可为 0）。

### 6.6 小结：价格（販価）与单价（単価）的交互链

| 触发 | 动作 |
|------|------|
| **販価区分变更** | resetUnitPriceSection → 標準/ERP 时 getUnitPrice → setUnitPrice 更新 price_；resetPriceCurrency 控制 price_/通貨的 enabled 与 8 时清 0。 |
| **表头単価 blur/Enter** | inputCheckPrice → 格式化、setUpdateHead、isUnitPriceNotZero 否则 clearStatusQDataAll；异常则 PRICE_HEAD 红底。 |
| **日付列变更** | resetUnitPrice：僅在販価＝標準(1) 时 getUnitPrice → setUnitPrice（见 §2.2）。 |
| **机种确定** | setPrice → getPrice 回写 price_、setPriceCodeName(1/3)、通貨（见 §2.1）。 |
| **更新/追加前** | checkUnitPriceMaster：无単価マスタ则 productChoice 橙底、標準→個別切替、弹窗。 |

### 6.7 Web 侧对应要点（已补全）

- **表头**：`formData.unitPrice`、`formData.priceCode`；**已补全** onPriceCodeChange：無償(8) 时 unitPrice='0'；getPrice 回调中標準(1) 且单价空时 fallback priceCode='3'。表头単価为只读，由販価区分控制逻辑与 Java resetPriceCurrency 一致。
- **表头単価校验**：**已补全** sms01206-validation.js 中 **isUnitPriceNotZero0(priceCode, price, erpFlag)**，validateForm 中调用；無償(8) 且非修理品时単価 0 可，否则 0 不可。
- **枝番**：明细行与表头一致；提交时 validateForm 对表头 unitPrice 做 isUnitPriceNotZero0 校验。枝番行若有 KO_PRICE_CODE/KO_UNIT_PRICE 可复用同一校验。

---

## 七、表头与表格的联动（OrderEntryNo_240522）

| 表头/条件 | 表格列 | 联动内容 |
|-----------|--------|----------|
| **扱い区分（handleCode）** | 乙仲出荷日（forward2Date） | 扱いが 2/4/B/D（海外売等）のときのみ乙仲列編集可・通常色；それ以外は **disabledColumns** で列全体灰・編集不可。切替時に既存乙仲出荷日をクリア。JbkTableView 原生 prop **disabledColumns** で制御。 |
| **ランク・機種** | プラント・保管場所 | ランク変更時・機種確定時 **getProductPlant** で機種別プラントマスタを取得し、注文数ありでプラント空の行にデフォルトを自動設定（resetTablePlantSloc / editTablePlantSloc 相当）。同一エントリー内でプラント・保管場所は同一必須（validateSamePlantSloc）。 |

---

## 八、プラント・保管場所 の来源与补全

### 8.1 Java 侧来源

- **getPlantStock()**：表头 header row1 の PLANT_DETAIL/STOCK_DETAIL を優先取得；空なら **getTablePlantSloc()** で表体の「先頭の注文ありかつプラントあり」の行から取得。
- **getProductPlant(product)**：**remoteObject_.getProductPlant(prmHash)**（KEY=product, CORPORATE）で機種別プラントマスタを取得。戻りは String[][] で M_PLANT, M_SLOC, M_JUNI(ランク) 等。productPlant_ に product を key にキャッシュし、得意先 or "1" で **getProductPlantTbl()** から [plant, stock] を取得。
- **resetTablePlantSloc(judge)**（ランク変更時）：getHeadPlantSloc で表头を取得；表头が空なら getProductPlantTbl → isProductPlantRank で **editTablePlantSloc(dt, judge)** で全行（注文ありかつプラント空 or judge 時）に dt を設定。表头に値がある場合は **editFirstLinePlantStock()** で表头に getProductPlantTbl の値を表示。
- **editPlantArea(row)**：プラント/保管場所セル編集時、表头 or getTablePlantSloc(row) or getProductPlantTbl からデフォルトを取得し **editOneLinePlantArea(dt, row)**。
- **同一エントリー内**：**isSamePlantSloc(inRow)** / **checkSamePlantSloc** で、表内「先頭の注文あり・プラントあり」行を基準に、全注文行のプラント・保管場所が一致することをチェック。

### 8.2 Web 侧补全

- **applyDefaultPlantStockFromProduct()**：機種変更・getPrice 成功後およびランク選択確定後に呼ぶ。**getProductPlant** API（key=productCode, key1=customerCode）で取得し、戻り先頭の [plantCode, stockLocation] を、注文数が入っているがプラントが空の行に一括設定。OrderEntryNo_240522 の resetTablePlantSloc / editTablePlantSloc に相当。
- **validateSamePlantSloc()**：プラント/保管場所 blur 時（onDetailPlantBlur 内）で実行。表内「先頭の注文あり・プラントあり」行を基準に、全注文行の plantCode/stockLocation が一致するか検証し、不一致行に **validationErrors['detail_'+row+'_plantCode'/_stockLocation]** を設定（JbkTableView の cellErrors で赤表示）。
- 表头に「デフォルトプラント/保管場所」の独立入力は未実装。Java の表头 row1 表示は、getProductPlant 取得値を editFirstLinePlantStock で表头に書き込む仕様のため、Web では「機種・ランクに応じたデフォルトを空行に自動填入」で代替。
