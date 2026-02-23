# websms01206 createForm 与 OrderEntryNo_180608時点の本番 还原说明

本页以 **OrderEntryNo_180608時点の本番.java**（2018年6月8日时点本番）为基准，对 createForm 进行布局与标签的 100% 还原。

---

## 一、已实施的布局与标签调整（与 180608 一致）

### 1. 主表单三列结构（左7 / 中7 / 右7）

| 区域 | 180608 本番 | createForm 现状 |
|------|-------------|------------------|
| **左列** | 担当、顧客注文No.、コメント、納品書、納品、カートン種類、使用ユーザー（7 行） | ✅ 一致。已移除第 8 行「機種コード」只读框（180608 中機種仅在下段） |
| **中列** | 得意先、配送先（按钮+输入）、住所建物、会社部署、担当TEL、納入先コード、装置メーカー | ✅ 一致。「納入先」标签已改为「納入先コード」 |
| **右列** | ケースマーク1～7 | ✅ 一致。标签已由「case mark1」等改为「ケースマーク1」～「ケースマーク7」 |

### 2. 下段区块

- **左**：機展書No.、機種（+機種検索）、ランク（按钮+输入）、販価（販価区分+単価+通貨）  
- **右**：顧客機種（+製品在庫計画）、用途、扱い区分、販売ルート；添付品、検査結果（部添付）、仕様書（部添付）  

标签「付属品」已改为「添付品」（与 180608 jFMultiLineLabel10 一致）。

### 3. 枝番行（180608 本番との一致）

- **180608**：jFPanel5 内 subNo1_～subNo10_ は **setVisible(false) が初期値**。エントリーNo を検索して getOrderSub で orderHash_ を取得した後に setVisible(true) で表示。  
- **Web**：**枝番バーは formData.entryNo があるときのみ表示**（`v-if="formData.entryNo"`）。エントリーNo 未入力時は枝番行を非表示にして 180608 と同一挙動に。

### 4. 明细表行数・明細追加（180608 本番との一致）

- **180608**：`table_.setRowCount(30)` で **明細テーブルは初期 30 行**。`table_.addRow()` で行追加も可能。  
- **Web**：  
  - **初期・クリア・削除後**：明細を **30 行**で初期化（mounted / doClearForm / 削除成功時）。  
  - **注文読込時**：API の明細が 30 未満の場合は空行を足して **30 行に揃える**。  
  - **「明細追加」ボタン**：そのまま利用（180608 の addRow に相当）。  

### 5. 明细表列

- 180608 本番：20 列（枝番～INSPECT）、2 行表头（2 行目枝番为蓝色等）。  
- 当前 Web：枝番、注文数、出荷済数、出荷残数、分割、ユーザ納期、出荷希望日、回答日、回答区分、乙仲出荷日、売上日、プラント、保管場所、税、インボイスNo、回収月、特、コメント、操作。  
- 逻辑列与 180608 的 TBL_MAX_DETAIL=20 对应；若后端支持 CARRIER_OUT/CARRIER_IN/INSPECT 等列，可再扩展列。

---

## 二、交互与数据逻辑（已与 180608 对齐）

- **担当**：下拉展开时 getCharges；选择变更时 getChargeCustomers 刷新得意先。  
- **得意先**：变更时 getCustomerProducts / getConsigns / getSupplys / getDefaultConsigns / judgementCustomerConfidence 等。  
- **配送先**：按钮打开弹窗，getConsigns 选择后回填。  
- **機種**：变更时 getPrice、judgmentStopProduct、getPSSPeriodProduct、judgmentERP、isUnitPriceMaster 等。  
- **ランク**：按钮打开 getRankDialogMaster 选择。  
- **販価区分**：变更时 getPrice 更新単価/通貨。  
- **枝番**：0～9 切换，getOrder(entryNo, subNo) 加载数据。  
- **明细**：注文数/出荷済入力时出荷残计算；日付 blur 休日チェック；保存前 validateForm（担当・得意先・機種必填等）。  
- **追加/更新/削除/クリア**：orderInsert / orderUpdate / orderDelete 及清空逻辑。

以上行为与 180608 及 Sms01206 对照清单一致。

---

## 三、可选后续增强（非 180608 必须）

1. **明细表列**：若 WebAPI 返回乙仲出荷・検収等 20 列，可增加列与 180608 的 setColumnCount(20) 完全一致。  
2. **表头两行**：180608 为 setHeaderRows(2)，第二行枝番等为蓝色；若需像素级一致可增加第二行表头。  
3. **标签多语言**：180608 通过 labelHash_（getLabelString）从服务器取 5249/6542 等；Web 已支持 getLabelString，若需与生产环境文案完全一致可统一走同一套 key。

---

## 四、文件修改清单

- `createForm.jsp`  
  - 主表单注释改为「OrderEntryNo_180608時点の本番 完全一致」。  
  - 左列删除「機種コード」只读行。  
  - 「納入先」→「納入先コード」。  
  - 「case mark1」～「case mark7」→「ケースマーク1」～「ケースマーク7」。  
  - 下段「付属品」→「添付品」；下段与明細注释改为 180608 本番描述。  
  - **枝番行**：`v-if="formData.entryNo"` でエントリーNo があるときのみ表示（180608 の setVisible(false) 初期・検索後に表示に合わせる）。  
  - **明細テーブル**：初期・クリア・削除後に **30 行**で初期化（mounted / doClearForm / 削除成功時）。loadOrderData で取得明細が 30 未満の場合は空行で 30 行に揃える。「明細追加」ボタンは 180608 の addRow 相当で維持。

---

以上已完成以 OrderEntryNo_180608時点の本番 为基准的布局与标签还原；功能与交互已按同一版本逻辑实现。
