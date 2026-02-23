# Sms01206 与 websms01206 逐项对照清单

以 Sms01206（OrderEntryNo / OrderEntryNo_i）为基准，逐项核对交互逻辑、接口调用、校验、数据关系与对话框在 websms01206 中的实现情况。

---

## 一、接口（RMI/API）对照

| Sms01206 接口 (OrderEntryNo_i) | websms01206 Controller | 说明 |
|--------------------------------|------------------------|------|
| getLabelString | ✅ api/getLabelString（已补全） | 画面/对话框标签取得；createForm  mounted 時取得 labelHash、ProductDialog 标题等で利用 |
| getSales | ❌ 未实现 | 旧系统也未在 OrderEntryNo 中调用 |
| getPattern | ❌ 未实现 | 同上 |
| getPrice | ✅ api/getPrice | |
| insert | ✅ api/insert | |
| update | ✅ api/update | |
| getProductDialogMaster | ✅ api/getProductDialogMaster | |
| getForwardDialogMaster | ✅ api/getForwardDialogMaster | |
| getRankDialogMaster | ✅ api/getRankDialogMaster | |
| getEntryNo | ✅ api/getEntryNo | |
| getOrderSub | ✅ api/getOrderSub | 一覧用；**参数需与旧系统一致**：key=entryNo, key1=customer, lang, corporate, type |
| getOrder | ✅ api/getOrder | |
| getVari | ✅ api/getVari | |
| getConsigns | ✅ api/getConsigns | |
| **getConsign** | ✅ api/getConsign（已补全） | 单件配送先取得；Controller/Service 透传 |
| getCharges | ✅ api/getCharges | |
| getChargeCustomers | ✅ api/getChargeCustomers | |
| getCustomerProducts | ✅ api/getCustomerProducts | |
| getCustomers | ✅ api/getCustomers | |
| getProducts | ✅ api/getProducts | |
| getUses | ✅ api/getUses | |
| getNpmEntry | ✅ api/getNpmEntry | |
| orderInsert | ✅ api/orderInsert | |
| orderUpdate | ✅ api/orderUpdate | |
| orderDelete | ✅ api/orderDelete | |
| getSupplys | ✅ api/getSupplys | |
| judgmentERP | ✅ api/judgmentERP | |
| moveR3 | ✅ api/moveR3 | |
| isHoliday | ✅ api/isHoliday | |
| getUnitPrice | ✅ api/getUnitPrice | |
| isUnitPriceMaster | ✅ api/isUnitPriceMaster | |
| getMakerMaster | ✅ api/getMakerMaster | |
| getPriceCodeMaster | ✅ api/getPriceCodeMaster | |
| updatePriceCode | ✅ api/updatePriceCode | |
| getExcelData | ✅ api/getExcelData | |
| getNpmLine | ✅ api/getNpmLine | |
| getKoOrder | ✅ api/getKoOrder | |
| isDeployPriceMaster | ✅ api/isDeployPriceMaster | |
| getDeployEntry | ✅ api/getDeployEntry | |
| getDistributeDeploy | ✅ api/getDistributeDeploy | |
| distributeDeploy | ✅ api/distributeDeploy | |
| updateSmsStatus | ✅ api/updateSmsStatus | |
| updateSmsStatusClear | ✅ api/updateSmsStatusClear | |
| getConporison | ✅ api/getConporison | |
| updateKoOrder | ✅ api/updateKoOrder | |
| getOyaOrder | ✅ api/getOyaOrder | |
| judgmentStopProduct | ✅ api/judgmentStopProduct | |
| getPSSPeriodProduct | ✅ api/getPSSPeriodProduct | |
| getQSeihinHash | ✅ api/getQSeihinHash（createForm 内调用） | |
| judgementCustomerConfidence | ✅ api/judgementCustomerConfidence | |
| reloadCustomerConfidence | ✅ api/reloadCustomerConfidence | |
| getProductPlant | ✅ api/getProductPlant | |
| getNpmProductDivision | ✅ api/getNpmProductDivision | |
| getActiveDate | ✅ api/getActiveDate | |
| getTransportList | ✅ api/getTransportList | |
| getOutInData | ✅ api/getOutInData | |
| getMultiRate | ✅ api/getMultiRate | |
| updateOutinData | ✅ api/updateOutinData | |
| getDefaultConsigns | ✅ api/getDefaultConsigns | |
| getForwadingListHtml | ✅ api/getForwadingListHtml | |
| setFactoryResults | ✅ api/setFactoryResults | |
| getMaxSalesUpdate | ✅ api/getMaxSalesUpdate | |
| judgementNNSN | ✅ api/judgementNNSN | |
| isDuplicationOrder | ✅ api/isDuplicationOrder | |
| isDuplicationOrder2 | ✅ api/isDuplicationOrder2 | |
| getCustomerForwardReadTime | ✅ api/getCustomerForwardReadTime | |
| getProductFromCustomerProduct | ✅ api/getProductFromCustomerProduct | |
| getInvoiceNoCount | ✅ api/getInvoiceNoCount | |
| getTurnoverDate | ✅ api/getTurnoverDate | |

---

## 二、校验与约束对照

| Sms01206 校验 | 触发时机 | websms01206 实现 | 备注 |
|---------------|----------|------------------|------|
| エントリーNo 必须 | 検索/追加/更新 | ✅ validateForm: entryNo 必填 | |
| 担当必须 | 追加/更新 | ✅ validateForm: chargeCode 必填 | |
| 得意先必须 | 追加/更新 | ✅ validateForm: customerCode 必填 | |
| 機種必须 | 追加/更新 | ✅ validateForm: productCode 必填 | |
| 顧客注文No 长度 20 桁 | 追加/更新前 | ✅ saveOrder 内 orderNo.length > 20 提示 | 旧系统 40 バイト表示、20 桁重複判定 |
| 顧客注文No 重複（rank=1） | 追加前 | ✅ isDuplicationOrder / isDuplicationOrder2 | |
| 明細 注文数 非负・数值 | 追加/更新 | ✅ validateForm: orderNumber 数値・>=0 | |
| 明細 出荷残 = 注文数 - 出荷済 | 输入时 | ✅ calcRemain(index) @input | |
| 日付 休日チェック | 明細日付 blur | ✅ onDetailDateBlur → isHoliday | |
| 日付 8 桁 YYYYMMDD | 明細日付 | ✅ onDetailDateBlur 内 replace(/\D/g,'').length===8 | |
| 納期/出荷/回答日 前後関係 | 明細日付 | ✅ validateDetailDatesForRow / onDetailDateBlur | 納期≤出荷希望日≤回答日 |
| 保管場所 スペース以外不可 | 明細 | ✅ onDetailStockBlur + validateForm | スペースのみ不可、maxlength=4 |
| インボイスNo 16 桁 | 明細 | ✅ validateForm + maxlength="16" | |
| コメント 40 字 | 明細 | - | 明細无 comment 输入时可不做 |
| 禁止字符 「'」「"」 | 表頭/明細 | ✅ validateForm checkForbiddenChars | entryNo/orderNo/note/明細 |
| NOTE 40 バイト | 表頭 | ✅ validateForm + maxlength="40" | |
| caseMark1～7 各 30 バイト | 表頭 | ✅ validateForm + maxlength="30" | |
| ランク=1 時 単価必须 | 追加/更新 | ⚠️ 未实现 | checkRank1UnitPriceProduct |
| 与信 超过时警告 | 得意先变更后 | ✅ onCustomerChange → judgementCustomerConfidence 表示 | |

---

## 三、组件与事件对照

| 组件 | Sms01206 行为 | websms01206 实现 |
|------|----------------|------------------|
| エントリーNo + 採番 | getEntryNo → 表示 | ✅ generateEntryNo |
| エントリーNo + 検索 | getOrderSub → 枝番; getOrder → 表示 | ✅ searchOrder（getOrder）; 一覧画面 getOrderSub |
| 枝番 0～9 | getOrder(entryNo, subNo) | ✅ onSubNoClick |
| 担当变更 | getChargeCustomers → 得意先列表 | ✅ onChargeChange |
| 得意先变更 | getCustomerProducts, getConsigns, getSupplys, getDefaultConsigns, judgementCustomerConfidence | ✅ onCustomerChange |
| 機種变更 | getPrice, judgmentStopProduct, getPSSPeriodProduct, judgmentERP, isUnitPriceMaster | ✅ onProductChange |
| 配送先ボタン | ConsignDialog → 选择后 setConsignLabel | ✅ openConsignDialog, confirmConsignDialog |
| ランクボタン | RankDecisionDialog → getRankDialogMaster | ✅ openRankDialog, selectRank |
| 販価区分变更 | getPrice → 単価/通貨 | ✅ onPriceCodeChange |
| 顧客機種 blur | getProductFromCustomerProduct → 機種コード | ✅ onCustomerProductBlur |
| 機展書No blur | getNpmEntry, getNpmProductDivision | ✅ onNpmEntryNoBlur |
| 明細 注文数/出荷済 | calcRemain | ✅ calcRemain |
| 明細 納期/出荷希望日 blur | 休日チェック | ✅ onDetailDateBlur |
| 明細 売上日 blur | getTurnoverDate | ✅ onDetailOnBoardDateBlur |
| 明細 プラント/保管場所 blur | isDeployPriceMaster 警告 | ✅ onDetailPlantBlur |
| 明細 インボイスNo blur | getInvoiceNoCount 表示 | ✅ onDetailInvoiceBlur |
| 追加/更新/削除/クリア | orderInsert/orderUpdate/orderDelete、确认后クリア | ✅ saveOrder, updateOrder, deleteOrder, clearForm |
| 出荷計画 | moveR3 | ✅ issueForwardPlan |
| 出荷依頼書 | getForwadingListHtml | ✅ openForwardListHtml |
| 出荷確定数 | setFactoryResults | ✅ confirmFactoryResults |
| Excel 出力 | getExcelData | ✅ exportExcel |

---

## 四、对话框对照

| 对话框 | Sms01206 | websms01206 |
|--------|----------|--------------|
| ConsignDialog | 配送先一覧选择・自由数据追加 | ✅ createForm 内 modal（showConsignDialog） |
| RankDecisionDialog | getRankDialogMaster 选择 | ✅ createForm 内 modal（showRankDialog） |
| ForwardRequestDialog | 出荷依頼表・Excel・出荷確定数 | ✅ createForm 内 modal（showForwardDialog） |
| ProductDialog | 機種・得意先検索 | ✅ 機種検索ボタン＋getProductDialogMaster 弹窗、行選択で機種セット |
| FactoryNumberDialog | 出荷済み数入力 | ✅ 已合并到 Forward 弹窗内「出荷確定数」 |
| OutinDialog | OUT-IN 連携 | ✅ 出荷依頼弹窗内「OUT-IN編集」→ getOutInData/updateOutinData 弹窗 |
| ReplyDialog | 回答区分选择 | ✅ 明細 回答区分 = getVari(REPLY) の a-select 下拉 |
| TaxDialog | 税区分选择 | ✅ 明細 税 = getVari(TAX) の a-select 下拉（未対応時は 0/1 固定） |
| DistributeDialog | 分割 | ✅ 明細「...」ボタン→分割値入力弹窗、OKで division 反映 |
| DeployDistributeDialog | 配分展開 | ✅ 明細「配分」ボタン→getDistributeDeploy 表示、配分実行で distributeDeploy |

---

## 五、一覧画面（list）与 getOrderSub 参数

| 项目 | Sms01206 | websms01206 |
|------|----------|--------------|
| getOrderSub 参数 | KEY=entryNo, KEY1=paraCustomer（或 ""）, LANG, CORPORATE, TYPE | 当前 POST body: entryNo, customer, product |
| 建议 | 与后端 Imart 一致：key, key1, lang, corporate, type；product 若后端支持可为 key2 | list 请求体增加 lang, corporate, type；key/key1 与 entryNo/customer 对应 |

---

## 六、已补全与待办

### 本次已做/建议补全

1. **list.jsp**：getOrderSub 请求体改为与旧系统一致：`key`, `key1`, `lang`, `corporate`, `type`（及可选 `key2`=product），便于后端兼容。
2. **createForm.jsp validateForm**：追加  
   - 顧客注文No 最大 20 文字（已有）；  
   - note 最大 40 字节（或 40 字符）；  
   - caseMark1～7 各最大 30 字符；  
   - 明細 invoiceNo 最大 16 字符；  
   - 禁止字符 `'` `"` 的检查或过滤。
3. **api/getConsign**：在 Controller/Service 中增加 getConsign 透传，供配送先单件取得（若 Imart 提供）。
4. **getLabelString**：若需与旧系统完全一致的多语言标签，可新增 api/getLabelString 透传；否则维持现有固定文案。

### 可选后续

- 明細 回答区分：getVari(TYPE=REPLY) 取得选项，改为下拉选择。  
- 明細 税区分：同上或固定选项。  
- 納期/出荷日/回答日 前后关系校验。  
- 保管場所 仅允许非空非全角空格。  
- ProductDialog / OutinDialog / 分割・配分弹窗 的 1:1 还原（业务复杂度高，可按需迭代）。

---

## 七、画面组件显示/禁用逻辑（Sms01206 → Web 已应用）

| 组件/逻辑 | Sms01206 来源 | Web 实现 |
|-----------|----------------|----------|
| **与信限度額ラベル** | `checkCustomerConfidence()` / `come1_.setVisible` | `showCreditLimit`：仅当 `isCcCorporate` 且得意先已选且无得意先错误时显示；与信対象外法人（corporate 不在 X,H,O,T,N,K）、得意先空、得意先校验错误、クリア后均不显示。`corporate` 由服务端 `corporate` 属性传入，默认 `X`。 |
| **isCcCorporate** | `isCcCorporate()` / `ccCorporates_` | `ccCorporates: ['X','H','O','T','N','K']`，`isCcCorporate()` 为 computed。 |
| **クリア时与信清除** | `clearButton___action` 内 `come1_.setVisible(false)`、`ccJudgeSw_ = OFF` | `doClearForm()` 内 `customerConfidenceLimit = ''`、`ccJudge = ''`；因得意先被清空，`showCreditLimit` 自动为 false。 |
| **得意先变更时与信** | 空时 `come1_.setVisible(false)` | `onCustomerChange` 中 `cc` 为空时已设 `customerConfidenceLimit = ''`、`ccJudge = ''`；`showCreditLimit` 依赖 `formData.customerCode` 与 `validationErrors.customerCode`。 |
| **detailLocked（明細・出荷確定ロック）** | `setCC8InputLock()` / `isRank1CC8()`：与信判定=8 且 ランク=1 | `detailLocked` computed；明細行输入・明細追加・出荷確定・分割/配分ボタン `:disabled="detailLocked"`。 |
| **採番ボタン** | `setCC8InputLock` 时 `collectButton_.setEnabled(false)` | `:disabled="!isNewEntry || detailLocked"`（与信8+ランク1 时也禁用採番）。 |
| **追加/更新/削除/クリア** | 各状态下的 setEnabled | `addButtonEnabled` / `updateButtonEnabled` / `deleteButtonEnabled` / `clearButtonEnabled` 已按新規/既存・校验状态控制。 |
| **出荷計画ボタン** | B エントリーのみ表示、二重防止 | `v-if="isBEntry"`、`issueForwardPlanLoading` 防止二重。 |
| **枝番栏** | getOrderSub 取得 orderHash_ 後のみ表示、存在する枝番のみ表示・当前ハイライト | `subNoList` を getOrderSub で取得。`v-if="subNoList.length > 0"` で栏表示。ボタンは `subNoList` のみ描画、`currentSubNo === item.subNo` で active、`subNoButtonsDisabled`（detailLocked 時）で不可。検索・採番・初期 entryNo で fetchSubNoList；クリア・削除後は subNoList=[]。 |
| **枝番 0～9** | 有 entryNo 且 getOrderSub 有数据时可切换 | 上記の通り。未取得時は枝番栏自体非表示。 |
| **ランクボタン** | 機種未选时禁用 | `:disabled="!formData.productCode"`。 |
| **製品在庫計画ボタン** | 枝番取得後のみ有効（検索データなし/クリア後は setEnabled(false)） | `:disabled="subNoList.length === 0"`。 |
| **機種検索ボタン** | 同上（検索データなし/クリア後は setEnabled(false)） | `:disabled="subNoList.length === 0"`。 |

未在 Web 实现的 Sms01206 组件逻辑（可按需追加）：

- **setFalseHead(false)**：表頭一括不可（参照画面・ウィンドウオープン時等）。Web 当前表頭始终可编辑；若需「先採番/検索后才可编辑表頭」可引入 `headLocked` 等状态。
- **会社間発注時 採番不可**：`paraDPCorporate_` 非空时 `collectButton_.setEnabled(false)`。Web 未传该参数，未实现。

以上为 Sms01206 与 websms01206 的逐项对照与还原要点。
