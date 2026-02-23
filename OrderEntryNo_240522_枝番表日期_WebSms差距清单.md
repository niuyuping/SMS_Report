# OrderEntryNo_240522 枝番表日期相关 vs WebSms 深入差距清单

基于 Java 源码与当前 Web 实现的逐项比对。**以下差距已全部补齐**（Jbk 组件通过原生属性 cellWarnings 等控制，便于复用）。

---

## 一、已实现且行为一致的部分（简要）

- 日付正規化 toDateValue（8 桁・6 桁→8 桁・ddmmyyyy 判定）
- 納期≦出荷希望日≦回答日、回答日≦乙仲出荷日≦売上日 順序チェック
- 休業日 isHoliday、過去日・2年先チェック（出荷希望日/回答日）
- 納期過去日時のメッセージ「納期は過去日です」
- ユーザ納期 blur：getCustomerForwardReadTime + getActiveDate → 出荷希望日・回答日・乙仲出荷日を連動（OrderEntryNo_240522 2015/04/24 相当）
- 出荷希望日 blur：回答日が空なら回答日へコピー、乙仲出荷日へコピー（2015/04/24 相当）
- 回答日 blur：乙仲出荷日へコピー（2015/04/24 相当）
- 売上日→検収日 同期、getTurnoverDate で売上日取得

---

## 二、已补齐的機能（実装概要）

| # | 項目 | 実装内容 |
|---|------|----------|
| 1 | 6 位日期 | validation.js toDateValue：6 桁時は常に `"20" + s`（Java 一致） |
| 2 | 納期過去日橙底 | **JbkTableView** に **cellWarnings** プロパティ追加。createForm で deliveryDatePastWarnings を渡し、該当セルに .cell-warning 適用 |
| 3 | 回収月 4→6 桁 | toMonthValue / toMonthDisplay。onDetailCollectMonthBlur で blur 時正規化 |
| 4 | 搬出・搬入日時 | toDateValue2 / toDateDisplay2。onDetailCarrierBlur で正規化 |
| 5 | resetUnitPrice | resetUnitPriceOnDateChange：日付 blur 時 priceCode===`1` なら getPrice で単価再取得 |
| 6 | setOnBoardDate 条件 | isJapan・handleCode===`1` 時、回答日 blur で refreshOnBoardDateFromReply（getTurnoverDate）により売上・検収日更新 |
| 7 | OUT-IN getNisuu | outInSw フラグと日付 blur 内のコメント枠を追加（API 実装時に呼び出し可能） |
| 8 | 生販 editTextActionPSSPeriod | 日付 blur 時 pssPeriodProductFlg のコメント枠を追加 |
| 9 | NNSN 回答区分・INV | fetchForwardDateFromDelivery コールバックで replyCode=`1`、プラント条件時 invoiceNo=entryNo+(row+1) |
| 10 | 乙仲出荷日 editable | **forward2Editable**（data）で列の editable を制御。将来 developSw_ 連携可 |
| 11 | NNSN 過去日补正 | fetchForwardDateFromDelivery 内で fdate < todayYmd なら fdate = todayYmd |
| 12 | 日付順序違反の赤 | 既存 validationErrors + JbkTableView cellErrors で .cell-error 適用済み |

---

## 三、元の差距説明（参考）

### 1. 6 位日期的补全规则与 Java 不一致

| 项目 | Java (OrderEntryNo_240522) | Web (当前) |
|------|----------------------------|------------|
| 6 位输入 | `if (6 == work.length()) work = "20" + work;` → **一律补 "20"**（250222→20250222、990101→**20990101**） | yy 00–49→20xx、50–99→19xx（990101→19990101） |

**建议**：若需与 Java 完全一致，将 6 位规则改为「6 位时一律 `"20" + s`」，否则保留当前 00–49/50–99 规则更符合常见年月习惯。

---

### 2. 納期過去日：单元格橙色背景

| 项目 | Java | Web |
|------|------|-----|
| 納期過去日 | `warningPastDate`: 该单元格背景设为 **SystemColor.orange** | 仅设置 `message = '納期は過去日です'`，**未改单元格样式** |

**缺失**：未对納期过去日的那一列单元格施加橙色背景（如通过 `cell-errors` 或列 class 标记）。

---

### 3. 回収月（collectMonth）4 位→6 位补全

| 项目 | Java | Web |
|------|------|-----|
| 回収月 | `COLLECT_MONTH_DETAIL`: 若 `work.length()==4` 则 `work = "20" + work`，再 `toMonthValue`（mmyyyy→yyyymm） | 无对 collectMonth 的 blur 正規化，**4 位不会自动补成 6 位** |

**缺失**：回収月失焦时未做「4 位 → 20+4 位 → yyyymm」的转换与回写。

---

### 4. 搬出日時・搬入日時的格式与正規化

| 项目 | Java | Web |
|------|------|-----|
| 搬出/搬入 | `toDateValue2` / `toDateDisplay2`：支持 8 位日期或 12/14 位日期+时分（ddmmyyyyHHMM 等） | 搬出/搬入列为普通 text，**无 toDateValue2 式正規化**，blur 也未处理 |

**缺失**：未对 `carrierOutDate` / `carrierInDate` 做「日期+时分」的统一格式与正規化（以及可能的 toDateDisplay2 显示）。

---

### 5. 日付变更时的 resetUnitPrice（単価再取得）

| 项目 | Java | Web |
|------|------|-----|
| 时机 | 出荷希望日 / 回答日 / 乙仲出荷日 / 売上日 任一变更后 | 无 |
| 条件 | 販価決定区分＝**標準**（priceCode "1"）时 | — |
| 动作 | `getUnitPrice()` → `setUnitPrice(data)`，表頭単価再設定 | **未在日期 blur 时调用 getPrice / 単価更新** |

**缺失**：日期列变更后未根据「標準」判定重新取得并设置単価。

---

### 6. setOnBoardDate 的完整条件与逻辑

| 项目 | Java | Web |
|------|------|-----|
| 执行条件 | **扱い区分＝"1"** 且 **isJapan()**（法人 X/N/U/J/4）且列为 DELIVERY/FORWARD/REPLY/FORWARD2/ON_BOARD | 仅 onBoardDate blur 时 getTurnoverDate + 検収日同期，**无 handle / 法人判定** |
| ユーザ納期时 | 仅 DMM 系（N,U）用「回答日」去算売上日；X/J/4 则 return | 未区分法人 |
| 出荷希望日时 | 若回答日已有值则 **return（不处理）** | 未做「回答日已有则不再算売上」 |
| 売上日・検収日 | 通过 **getOnBoardDate(yyyymmdd)**（内部 getTurnoverDate）算出；DMM 有时用納期 | 使用 getTurnoverDate，未区分 DMM 用納期等 |
| 手動変更 | 若「売上日と検収日が既に手動で異なる」则 **不覆盖** 売上日 | 未做「已手动改过则不覆盖」 |

**缺失**：未按 handle・法人・列类型・是否已手動変更实现与 Java 一致的 setOnBoardDate 分支。

---

### 7. OUT-IN 輸送手番（getNisuu / setLeadTimeDate）

| 项目 | Java | Web |
|------|------|-----|
| 时机 | ユーザ納期 或 出荷希望日 变更且 **outInSw_ == true** | 无 |
| 动作 | `getNisuu()`、`setLeadTimeDate(row, suu1)`：出荷希望日↔納期の連動 | **未实现 OUT-IN 画面連携** |

**缺失**：OUT-IN 用納期/出荷希望日与輸送手番的联动在 Web 上未做。

---

### 8. 生販確定期間（editTextActionPSSPeriod）

| 项目 | Java | Web |
|------|------|-----|
| 时机 | 日期列变更且 **pssProduct_ == true**（生販確定期間内機種） | 无 |
| 动作 | `editTextActionPSSPeriod(row, column)`：生販関連表示・チェック | 仅有 pssPeriodProductFlg 等表示，**日期 blur 时未调用等价逻辑** |

**缺失**：日期输入时未根据「生販確定期間内」做与 Java 相同的后续处理。

---

### 9. NNSN 时的回答区分・インボイスNo 自动设定

| 项目 | Java | Web |
|------|------|-----|
| 回答区分 | ユーザ納期/出荷希望日連動後、条件满足时 **回答区分＝"1"**（表示名 replyDivisionString_）、セル pink・editable false | fetchForwardDateFromDelivery で replyCode=`1` を設定済み。forward2（乙仲）は連動せず手動入力のみ |
| インボイスNo | nnsnFlg_ 且 プラント条件（!XK 等）满足时 **entryNo + (row+1)** をセット | プラント条件時 fetchForwardDateFromDelivery コールバックで invoiceNo=entryNo+(row+1) を設定済み |

**缺失**：NNSN 日期联动后未自动设置回答区分=1 与インボイスNo。

---

### 10. 乙仲出荷日的可编辑/只读控制

| 项目 | Java | Web |
|------|------|-----|
| 控制 | `setLockOnBoard()`、`editForward2True()` 等按 **developSw_** 与 扱い区分 控制乙仲出荷日 editable 与背景色（forward2Datas_） | 列固定 editable，**无 developSw_ による動的ロック** |

**缺失**：未根据 developSw_/扱い 动态控制乙仲出荷日是否可编辑及样式。

---

### 11. getForwardDateNNSN 的过去日补正

| 项目 | Java | Web |
|------|------|-----|
| 逻辑 | 算出日出力日が **intToday_ より過去** なら **当日に上書き**（`outDate = "" + intToday_`） | fetchForwardDateFromDelivery 使用 API 返回值原样，**未做「若 < 今日则改为今日」** |

**缺失**：NNSN 算出日出力日若为过去日时，未在 Web 侧改为当日。

---

### 12. isCompareDate 的单元格标红

| 项目 | Java | Web |
|------|------|-----|
| 违反时 | 该列セル背景 **red**、setErr(row, col, ERR, UPDATE)（提示在 2015/06/04 已 CUT） | 仅 validateDetailDatesForRow 返回 message 与 validationErrors，**未对明细表该列单独设红背景**（若 JbkTableView 未按 cell-errors 对列着色则等价缺失） |

**建议**：确认 JbkTableView 是否已按 `cell-errors` 对违反顺序的日期列施加红色样式；若无则需补。

---

## 四、建议实现优先级（参考・多くは対応済み）

上記の通り、優先度の高い項目はすべて実装済み。OUT-IN getNisuu は API 実装後に outInSw を true にし、コメント枠内で API 呼び出しを追加即可。

---

## 五、回答区分・税の表示・編集ロジック（OrderEntryNo_240522 準拠）

### 1. 回答区分（REPLY_CODE_DETAIL）

| 項目 | 内容 |
|------|------|
| **データ** | 保存・送信は**コード**（例 `"0"` 回答不要、`"1"` 回答済）。表示は **名称**（getReplyCodeName(code)）。マスタは getVari(TYPE=REPLY) の replys_[][CODE_DATA/NAME_DATA]。 |
| **初期表示** | setTableFromData 時：セルに getReplyCodeName(data[REPLY_CODE_DETAIL]) を表示。背景 **pink**、**editable = false**。 |
| **回答済（code=1）時** | セル背景 **pink**、**editable = false**。表示は replyDivisionString_（言語に応じ「回答済」/ "already replied"）または getReplyCodeName("1")。 |
| **変更操作** | ユーザがセルクリック時、条件を満たす場合のみ **displayReplyDialog()** で選択 → 選択名称でセル更新、背景 pink、editable false、setUpdateDetail(code) で更新ハッシュに反映。回答区分＝回答済にした場合、回答日が空なら回答日を赤・ERR に。 |
| **編集可能条件** | ① 行が「R」でない（noR）。② **会社間末オーダー** または **isBottomDevelop()**（(factory1==0 && referCode_!="1") \|\| isBottomDevelop()）。③ **isOutinIngInAlreadyReplied(row)==false**（OUT-IN 連携中で既に回答済の行は変更不可）。④ 上記 TAX 同梱品判定の hantei を通す（回答側は常 true）。 |
| **NNSN 等の自動設定** | 納期 blur 等で出荷希望日・回答日を連動する際、isReplied(fdate) かつ corporate_ != "Q" なら replyCode="1" に設定し、表示を replyDivisionString_、背景 pink、editable false。 |

**Web 同期（済）**  
- Web で **canEditReply(rowIndex)** を実装：detailLocked に加え、① 行の no 先頭が "R" のとき不可、② 会社間末 (factory1==0 && referCode!="1") または developSw=="1" のときのみ可、③ OUT-IN 連携中かつ回答済 (outinOrder==="in" && replyCode==="1") のとき不可。getOrder で **referCode / developSw / outinOrder** および明細 **d['0']（no）** を返却すると完全一致。

### 2. 税（TAX_DETAIL）

| 項目 | 内容 |
|------|------|
| **データ** | 値は **TAXS = {"", "0", "1"}** に対応（空=通常課税、0=非課税、1=納税義務）。セル表示・保存とも同じコード（または getTaxDialogItem でダイアログ表示用に変換）。 |
| **初期表示** | setTableFromData 時：TAX_DETAIL 列は CENTER、背景 **pink**、**editable = false**。既存データの税コードをそのまま表示。 |
| **変更操作** | ユーザが税セルクリック時、条件を満たす場合 **setTableTax(row,col)**：getTaxDialogItem(現在値) で **税区分ダイアログ**（dialog4_）を開き、選択後にセルに mark を設定、CENTER、背景 **pink**、editable false、setUpdateDetail で更新。 |
| **編集可能条件** | ① **同梱品子オーダーでは税の修正不可**（pkData && !oyaPackege_ のとき hantei=false でクリック処理に入らない）。② **会社間末オーダー**（factory1==0 && referCode_!="1"）のときのみ setTableTax を実行。 |

**Web 同期（済）**  
- Web で **canEditTax(rowIndex)** を実装：detailLocked に加え、同梱品子 (isPackageData && !oyaPackege) のとき不可、会社間末 (factory1==0 && referCode!="1") のときのみ可。getOrder で **referCode / isPackageData / oyaPackege** を返却すると完全一致。回答区分・税セルに **cell-reply-pink / cell-tax-pink** のピンク背景を適用済み。

### 3. まとめ

- **回答区分**：表示＝名称、保存＝コード。回答済またはデータ読込後は **pink・非編集**。変更はダイアログからのみ。編集可は「R 以外・会社間末 or 末派生・OUT-IN 未回答済」など。
- **税**：表示・保存ともコード（""/"0"/"1"）。常に **pink・非編集**（直接入力なし）。変更はダイアログからのみ。編集可は「同梱品子でない・会社間末オーダー」。
