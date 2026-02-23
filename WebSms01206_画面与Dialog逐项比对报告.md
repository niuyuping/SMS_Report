# WebSms01206 与 Sms01206 画面・Dialog 逐项比对报告

> 从**样式、组件交互逻辑、接口调用、数据逻辑**四方面，逐一比对每个画面和 Dialog，列出 Web 版缺失或需补全项。基准：Sms01206（OrderEntryNo / OrderEntryNo_i）。

---

## 一、主画面

### 1.1 一覧（list.jsp）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 一覧検索条件・結果テーブル・ボタン配置 | 已用 sms01206-common.css、form-panel、sms-table | 无缺失；可依实际 UI 再微调 |
| **组件交互** | 検索・リセット・新規登録・詳細/編集リンク・分页 | 検索→getOrderSub；分页为前端 slice（displayedOrders） | 若后端支持分页，未传 page/offset；当前为前端分页 |
| **接口** | getOrderSub（key, key1, key2, lang, corporate, type） | api/getOrderSub，参数一致 | 无缺失 |
| **数据逻辑** | 行映射 d_entry_no, d_customer_*, d_product_*, d_order_no, d_delivery_date, d_unit_price 等 | mapOrderRow 兼容数组与多种键名（entryNo, H26, d_entry_no 等） | 无缺失 |

---

### 1.2 新規/編集（createForm.jsp + websms01206-createForm.js）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 三列表頭・明細テーブル・枝番栏・ボタン（採番/検索/追加/確認/更新/削除/クリア/出荷計画/終了）・与信表示 | 三列 main-form-cols、order-detail-table、枝番 subnumber-bar、製品在庫計画 btn-seihin | 已对齐；label 未统一用 getLabelString（见下） |
| **组件交互** | 採番→getEntryNo；検索→getOrderSub+getOrder；枝番切替→getOrder；確認→sessionStorage→createConfirm；追加/更新/削除/クリア/出荷計画/終了 | 均已实现；確認・修正回带数据已实现 | 无缺失 |
| **接口** | getEntryNo, getOrderSub, getOrder, getCharges, getCustomers, getProducts, getVari(VOUCHER/ATTACHMENT/DELIVERY/CARTON/HANDLE/ROUTE/CURRENCY/USER/REPLY/TAX), getPriceCodeMaster, getMakerMaster, getLabelString(OrderEntryNo), getChargeCustomers, getCustomerProducts, getConsigns, getSupplys, getDefaultConsigns, judgementCustomerConfidence, reloadCustomerConfidence, judgmentStopProduct, getPSSPeriodProduct, judgmentERP, isUnitPriceMaster, getPrice, orderInsert, orderUpdate, orderDelete, getProductPlant, isDuplicationOrder, isDuplicationOrder2, getKoOrder, getOyaOrder, getCustomerForwardReadTime, getMaxSalesUpdate, getActiveDate, getOutInData, getForwadingListHtml, setFactoryResults, getExcelData, getTurnoverDate, getRankDialogMaster, moveR3, getProductDialogMaster, getOutInData, updateOutinData, getDistributeDeploy, distributeDeploy, getNpmEntry, getNpmProductDivision, getInvoiceNoCount, isDeployPriceMaster 等 | 均已透传调用 | **缺失**：getSales / getPattern 未在 Controller/Service 暴露；ProductDialog 内未使用 TYPE=SALES/PATTERN（见 Dialog 节） |
| **数据逻辑** | buildKey：H0～H26, B2～B43, D01～Dxx（'1'注文数,'2'出荷済,'5'～'16' 等） | buildKey 中 Dxx 含 '1','5'～'16'，**不含 '2'(factoryNumber)/'3'(remainNumber)**；orderInsert 通常新规时由后端计算出荷済/出荷残，一般无问题 | 若后端 orderInsert 要求明細带 '2'/'3' 则需在 buildKey 中补全；否则无缺失 |
| **确认对话框文案** | displayConfirmDialog(key)：MSG_DELETE, MSG_CLEAR, MSG_CHECK 等来自 labelHash_.get(key)（getLabelString） | 削除確認「削除してよろしいですか。」、クリア確認「更新データがあります。画面クリアしてもいいですか。」为**固定日文** | **缺失**：未用 labelHash（getLabelString）驱动確認文案，多语言/与旧系统一致需改为 labelHash['MSG_DELETE'] 等 |
| **画面ラベル** | 各ラベル（担当・得意先・機種・出荷計画・製品在庫計画等）由 labelHash_.get("5274") 等取得 | getLabelString(OrderEntryNo, keys) 已调用，labelHash 已注入；**部分 JSP 仍为硬编码**（如按钮文字） | 可选：表頭/按钮文字统一用 labelHash 以与旧系统/多语言一致 |

---

### 1.3 詳細（detail.jsp）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 表頭 H26/H0/H1/H27/B3/B33 等・明細テーブル・編集/一覧に戻る | form-row + span、detail-panel + sms-table、btn-row | 无缺失 |
| **组件交互** | 編集→{entryNo}/update?form；一覧に戻る→list | 已实现 | 无缺失 |
| **接口** | 服务端 getOrder 注入 modelJson | 服务端 detail 中 getOrder(key=entryNo, key1='0', ...) 注入 modelJson | 无缺失 |
| **数据逻辑** | model.ans 与 D01～Dxx 键一致；明細列与 keys_/labelHash 一致 | model 解析兼容 data.ans / ans；detailLines 使用 '1'～'13' | **缺失**：明細表只显示 明細No・注文数・工場番号・納期・出荷依頼日・回答日・回答区分・乙仲出荷日・売上日・プラント・保管場所・税区分；**未显示 インボイスNo・回収月・特・コメント** | 需在 detail 明細表增加上述 4 列并绑定 row['14'], row['15'], tokushi, row['16'] |

---

### 1.4 登録確認（createConfirm.jsp）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 確認表示・登録実行/修正/戻る | 同 common 风格・按钮一致 | 无缺失 |
| **组件交互** | 数据来自 sessionStorage 或 entryNo+getOrder；登録実行→orderInsert→list；修正→sessionStorage→create?form；戻る→list | 已实现；修正时 submitPayload 写回 sessionStorage，createForm mounted 恢复 | 无缺失 |
| **接口** | getOrder（entryNo 时）、orderInsert | 已使用 | 无缺失 |
| **数据逻辑** | buildKeyFromFormData 与 createForm buildKey 同构 | buildKeyFromFormData 的 Dxx 含 '1','5'～'16'，**不含 '2'(出荷済)/'3'(出荷残)**；与 createForm 一致 | 若后端 orderInsert 要求明細 '2'/'3' 则需补全；否则无缺失 |
| **orderAnsToModel** | 明細 Dxx 含 factoryNumber, remainNumber 等 | formDataToAns 的 Dxx 含 '2','14','15','16'；orderAnsToModel 的 detailLines 未包含 factoryNumber/remainNumber/invoiceNo/collectMonth/tokushi/comment 的完整映射 | 显示用 orderAnsToModel 已足够；submitPayload 来自 formData，buildKeyFromFormData 已含 invoiceNo/collectMonth/comment（'14','15','16'），**tokushi 若存在需在 buildKeyFromFormData 的 Dxx 中增加对应键**（若旧系统有） |

---

## 二、Dialog / Modal

### 2.1 ProductDialog（機種検索）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 機種コード・機種名一覧・選択・閉じる | modal-panel、consign-table、機種コード/機種名 2 列 | 无缺失 |
| **组件交互** | 打开时 getProductDialogMaster(COMPONENT, TYPE, …)；TYPE=LABEL 取标题；TYPE 默认機種一覧；另有 **TYPE=SALES（他社向販売）・TYPE=PATTERN（生産パターン）** タブ/検索 | 仅调用 getProductDialogMaster(lang, corporate) 取機種一覧；**无 SALES/PATTERN タブ或検索** | **缺失**：ProductDialog 未实现「他社向販売」「生産パターン」检索（getSales/getPattern 或 getProductDialogMaster( type: 'SALES'/'PATTERN')）；若业务需要则需补 API 与 UI |
| **接口** | getProductDialogMaster(COMPONENT, TYPE, KEYS…)；内部 getSales/getPattern 对应 TYPE=SALES/PATTERN | api/getProductDialogMaster 已存在；**未传 type: 'SALES'/'PATTERN'**；且 Controller 未暴露 getSales/getPattern | 见上；可选补全 |
| **数据逻辑** | 列表为 code/name 或数组 [code, name] | convertToOptions / productDialogList；selectProductDialogRow 回填 formData.productCode | 无缺失 |

---

### 2.2 ConsignDialog（配送先選択）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | エントリーNo・顧客注文No 参照・自由データ追加・一覧選択・OK/CANCEL | 已有；标题「配送先選択」硬编码 | 可选：标题用 getLabelString("ConsignDialog", keys) |
| **组件交互** | 打开时 getConsigns(customerCode)；自由データ追加→一覧に反映；行选择→OK 确定 | getConsigns；addConsignFree；confirmConsignDialog | 无缺失 |
| **接口** | getConsigns | api/getConsigns | 无缺失 |
| **数据逻辑** | consignData_ 与表頭荷受先・住所・部署・TEL 同步 | consignSelectedIndex→consignList[].code/name/address/section/tel 回填 formData.consign* | 无缺失 |

---

### 2.3 RankDecisionDialog（ランク決定）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 情報ランク・ランク・CRD/RE/SDM/ASR 列・確認/閉じる | rankListWithLabels、rankDialogStaticLabels（1～9）・確認/閉じる | 无缺失；标题「RankDecisionDialog」可改为 getLabelString |
| **组件交互** | getRankDialogMaster(type:'RANK', customerCode, productCode)；选择行→確認でランク確定 | getRankDialogMaster；selectRank 回填 decisionRank、unitPrice 等 | 无缺失 |
| **接口** | getRankDialogMaster | api/getRankDialogMaster | 无缺失 |
| **数据逻辑** | ランク・単価・通貨等回填表頭 | selectRank 内 getPrice、formData.decisionRank/unitPrice/currencyCode 等 | 无缺失 |

---

### 2.4 ForwardRequestDialog（出荷依頼書）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | エントリーNo・枝番・顧客注文No・最終更新日・稼働日・OUT-IN・明細（注文数/出荷済/出荷残/納期/出荷希望日/回答日）・備考・出荷確定数・発行/EXCEL/閉じる | forward-ref・forward-detail-box・forward-note・forward-factory-row・発行/EXCEL出力/閉じる | 无缺失 |
| **组件交互** | getCustomerForwardReadTime, getMaxSalesUpdate, getActiveDate, getOutInData；OUT-IN編集→OutinDialog；出荷確定→setFactoryResults；発行→getForwadingListHtml；EXCEL→getExcelData | 均已实现 | 无缺失 |
| **接口** | getForwadingListHtml, getExcelData, setFactoryResults, getOutInData, getCustomerForwardReadTime, getMaxSalesUpdate, getActiveDate | 均已调用 | 无缺失 |
| **数据逻辑** | 明細行 index 与 key/key1/key2 对应 | forwardDialogRowIndex、forwardDialogNote、forwardDialogFactoryQty、confirmFactoryResults、exportExcel | 无缺失 |

---

### 2.5 OutinDialog（OUT-IN 連携）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | エントリーNo/枝番/明細・OUT-IN データ表示・OK/閉じる | エントリーNo/枝番/明細行・outinEditData(JSON 表示)・OK/閉じる | 无缺失；旧系统若为表形式可考虑表格化展示 |
| **组件交互** | getOutInData(key=[entryNo, subNo, detailNo])；更新→updateOutinData | getOutInData；confirmOutinDialog→updateOutinData | 无缺失 |
| **接口** | getOutInData, updateOutinData | 已使用 | 无缺失 |
| **数据逻辑** | key2 为 OUT-IN 用パラメータ | key: [en, currentSubNo, detailNo], key2: outinEditData | 无缺失 |

---

### 2.6 DistributeDialog（分割）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 分割入力（例 "=>"）・OK/CANCEL | distributeDivisionValue・OK/CANCEL | 无缺失 |
| **组件交互** | 输入分割値→OK で明細 division 更新 | confirmDistributeDialog 将 distributeDivisionValue 写回 detail.division | 无缺失 |
| **接口** | 仅前端更新明細；无单独 API | 无 | 无缺失 |
| **数据逻辑** | division 写入当前明細行 | 已实现 | 无缺失 |

---

### 2.7 DeployDistributeDialog（配分展開）

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | 明細行情報・配分データ一覧・配分実行/閉じる | deployDistributeRowIndex・deployDistributeList（項目・値）・配分実行/閉じる | 无缺失 |
| **组件交互** | getDistributeDeploy→表示→distributeDeploy 実行 | getDistributeDeploy；confirmDeployDistributeDialog→distributeDeploy | 无缺失 |
| **接口** | getDistributeDeploy, distributeDeploy | 已使用 | 无缺失 |
| **数据逻辑** | key=[entryNo, subNo, detailNo] | 已一致 | 无缺失 |

---

### 2.8 ConfirmModal / AlertModal / WaitDialog

| 维度 | Sms01206 | WebSms01206 | 缺失/差异 |
|------|----------|-------------|-----------|
| **样式** | ComformDialog：YES/NO；Dialog1：しばらくお待ち下さい。 | confirmModal：確認/YES/NO；alertModal：OK；waitDialog：しばらくお待ち下さい。 | 无缺失 |
| **组件交互** | displayConfirmDialog(key)→labelHash_.get(key)；displayConfirmDialog2 同様 | 固定文案「削除してよろしいですか。」「更新データがあります…」；**未使用 labelHash** | **缺失**：確認文案应改为 labelHash['MSG_DELETE']、labelHash['MSG_CLEAR'] 等（需与 getLabelString 返回键一致） |
| **接口** | getLabelString | getLabelString(OrderEntryNo) 已调用，labelHash 已存在 | 仅文案未用 labelHash 驱动 |
| **数据逻辑** | 回调执行削除/クリア等 | confirmYesCallback 已支持 | 无缺失 |

---

## 三、接口层面汇总

| 项目 | Sms01206 | WebSms01206 | 说明 |
|------|----------|-------------|------|
| getSales | OrderEntryNo_i 定义；ProductDialog TYPE=SALES 时使用 | Controller/Service **未暴露** | 他社向販売検索；可选实现 |
| getPattern | 同上；TYPE=PATTERN | Controller/Service **未暴露** | 生産パターン検索；可选实现 |
| 其余 RMI | 已通过 WebSms01206Controller api/* 透传 | 已实现 | 无缺失 |

---

## 四、数据逻辑汇总

| 项目 | 说明 |
|------|------|
| buildKey Dxx | createForm/createConfirm 的 Dxx 未含 '2'(出荷済)/'3'(出荷残)；新规登録一般由后端计算，若后端要求则补全 |
| createConfirm buildKeyFromFormData | 已含 '14'(invoiceNo), '15'(collectMonth), '16'(comment)；若旧系统有「特」则需增加对应键 |
| detail 明細列 | **缺少 インボイスNo・回収月・特・コメント** 四列；需在 detail.jsp 表头与 tbody 中增加并绑定 row['14'], row['15'], tokushi, row['16'] |
| orderAnsToModel（createConfirm） | 明細行若需显示 factoryNumber/remainNumber/invoiceNo/collectMonth/tokushi/comment，可在 orderAnsToModel 的 detailLines 中补全映射（当前 createConfirm 显示用 model 已足够，主要缺 detail 画面） |

---

## 五、样式与多语言

| 项目 | 说明 |
|------|------|
| 共通 CSS | sms01206-common.css、sms01206-createForm.css 已用；一覧・詳細・確認は common のみ |
| 画面・Dialog 标题/按钮 | 多数为日文硬编码；与 Sms01206 完全一致或多语言需：getLabelString 按 component（OrderEntryNo, ConsignDialog, OutinDialog, DeployDistributeDialog 等）与 keys 取文案，并替换 JSP/JS 中的固定文字 |
| 待機ダイアログ | 标题「しばらくお待ち下さい。」与 Sms01206 labelHash_.get("4667") 一致时可改为 labelHash['4667'] |

---

## 六、结论与建议

1. **必须补全**
   - **detail.jsp**：明細表增加 **インボイスNo・回収月・特・コメント** 四列，数据绑定 row['14'], row['15'], tokushi, row['16']（若 model 中明細无 tokushi 键则与 getOrder 约定一致即可）。

2. **建议补全**
   - **確認ダイアログ**：削除/クリア 等确认文案改为使用 **labelHash**（getLabelString 返回的键，如 MSG_DELETE, MSG_CLEAR）以与旧系统一致并支持多语言。
   - **一覧分页**：若后端 getOrderSub 支持 page/offset，则在请求中传入并用后端 total；否则维持当前前端 slice 分页即可。

3. **可选补全**
   - **getSales / getPattern**：若需 ProductDialog 内「他社向販売」「生産パターン」检索，则 Controller/Service 暴露 getSales/getPattern（或 getProductDialogMaster 的 type=SALES/PATTERN），并在 ProductDialog 增加タブ/検索与对应接口调用。
   - **画面・Dialog 全ラベル**：表頭・按钮・Dialog 标题等统一由 getLabelString 按 component/keys 取得并替换硬编码。
   - **createConfirm buildKeyFromFormData**：若后端 orderInsert 要求明細 '2'/'3'，则 Dxx 中补 '2','3'；若有「特」则补对应键。

完成上述必须项并视需要完成建议项后，样式、组件交互、接口与数据逻辑可与 Sms01206 对齐到可接受范围；可选项用于进一步一致性与多语言。

---

## 七、补全实施履历（已全部对应）

以下缺失项均已按报告实施补全：

| 类别 | 内容 | 实施概要 |
|------|------|----------|
| **必须** | detail 明細 4 列 | detail.jsp 明細表增加 インボイスNo・回収月・特・コメント；detailLines 增加 '14','15','16','17' 映射 |
| **建议** | 確認ダイアログ labelHash | createForm.js 削除/クリア確認文案改为 labelHash['MSG_DELETE']/['MSG_CLEAR']（含 875/7458）；confirmModal 标题用 labelHash['confirmModalTitle']；waitDialog 标题用 labelHash['4667'] |
| **建议** | 一覧分页 | list.jsp 的 getOrderSub 请求体增加 page, pageSize；goToPage 时调用 loadOrders()；loadOrders 不再强制 currentPage=1 |
| **可选** | getSales/getPattern | WebSms01206Service/Impl 新增 getSales、getPattern（仮データ空一覧）；Controller 暴露 api/getSales、api/getPattern；ProductDialog 增加 機種検索/他社向販売/生産パターン タブ与 loadProductDialogByType(type) |
| **可选** | 画面・Dialog ラベル | getLabelString 仮データ扩充（consignDialog, outinDialog, distributeDialog, deployDistributeDialog, RankDecisionDialog, MSG_*., 4667, confirmModalTitle, closeButton, okButton 等）；各 Dialog 标题・ボタン改为 (labelHash && labelHash['xxx']) \|\| '既定文言' |
| **可选** | createConfirm buildKey Dxx | buildKeyFromFormData 的 Dxx 增加 '2'(factoryNumber), '3'(remainNumber), '17'(tokushi)；createForm buildKey 的 Dxx 同步增加 '2','3','17' |

getLabelString 为假数据（WebSms01206ServiceImpl 内 Map 返回），如需与実機一致可改为 passThrough("getLabelString", input)。
