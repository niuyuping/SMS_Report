# websms01206 与 Sms01206 全面对比与差距分析

> **说明**：项目中未找到「Sms02106」模块，本报告以 **Sms01206**（桌面版 OrderEntryNo/OrderEntryNo_i）为基准，对 websms01206 进行逐画面、逐功能、逐交互的全面对比，并分析画面间调用关系与数据流，判断是否能实现与 Sms01206 完全一致的功能、用户体验与操作逻辑。

---

## 一、基准说明与画面/URL 对照

### 1.1 基准与对象

| 项目 | 说明 |
|------|------|
| 基准系统 | **Sms01206**（桌面 Java Applet：OrderEntryNo / OrderEntryNo_i RMI） |
| 对比对象 | **websms01206**（Spring MVC + JSP + Vue3 + Imart WebApi 透传） |
| 目标 | 用户在使用 websms01206 时的体验和所有交互逻辑与 Sms01206 保持完全一致 |

### 1.2 画面与 URL 对照

| 画面 | Sms01206 | websms01206 URL | 对应关系 |
|------|----------|-----------------|----------|
| 一覧 | 主画面一覧/検索 | `GET /websms01206/maintenance/list` | ✅ 有 |
| 新規 | 採番→入力 | `GET /websms01206/maintenance/create?form` | ✅ 有 |
| 編集 | 枝番切替→検索 | `GET /websms01206/maintenance/{entryNo}/update?form` | ✅ 有 |
| 詳細 | 只读表示 | `GET /websms01206/maintenance/{entryNo}` | ✅ 有 |
| 登録確認 | 確認→登録実行 | `GET /websms01206/maintenance/createConfirm` | ✅ 有 |
| 登録実行 | orderInsert 调用 | `POST /websms01206/maintenance/create`（兼容用）；实际由 JS 调用 `api/orderInsert` | ✅ 已实现 |

---

## 二、websms01206 画面间调用关系与数据流

### 2.1 画面迁移关系

```
list（一覧）
  ├─ 検索 → POST api/getOrderSub（条件: key, key1, key2, lang, corporate, type）
  ├─ 新規登録 → GET create?form
  ├─ 詳細 → GET {entryNo}（服务端 getOrder 注入 modelJson）
  └─ 編集 → GET {entryNo}/update?form

create?form / {entryNo}/update?form（新規/編集）
  ├─ 採番 → api/getEntryNo
  ├─ 検索/枝番 → api/getOrder（key=entryNo, key1=subNo）
  ├─ 各类下拉/弹窗 → api/getCharges, getCustomers, getConsigns, getRankDialogMaster, getProductDialogMaster, getForwadingListHtml, setFactoryResults, getOutInData, updateOutinData, moveR3 等
  ├─ 追加/更新/削除 → api/orderInsert, orderUpdate, orderDelete
  ├─ 确认 → 将 formData 写入 sessionStorage，跳转 createConfirm
  └─ 終了 → 返回 list

createConfirm（登録確認）
  ├─ 数据来源：sessionStorage 的 formData，或 GET 参数 entryNo + api/getOrder
  ├─ 登録実行 → POST api/orderInsert（key 由 buildKeyFromFormData 构建），成功 → location 到 list
  ├─ 修正 → GET create?form
  └─ 戻る → list

detail（詳細）
  ├─ 数据来源：服务端 getOrder 注入 modelJson，前端解析 model.ans 与 D01～Dxx
  ├─ 編集 → GET {entryNo}/update?form
  └─ 一覧に戻る → list
```

### 2.2 数据流与一致性结论

- **一覧 ↔ 新規/編集/詳細**：通过 entryNo 与 getOrder/getOrderSub 与 Sms01206 一致；list 的 getOrderSub 参数已按 key/key1/key2/lang/corporate/type 与旧系统对齐。
- **新規/編集 → 確認 → 登録**：formData 经 sessionStorage 或 getOrder 到 createConfirm，再经 `buildKeyFromFormData` 调用 api/orderInsert，与 Sms01206 的「確認→orderInsert」流程一致。
- **结论**：在现有设计下，**画面间调用关系与数据流可以支持与 Sms01206 一致的业务流程**；差异主要来自单画面内的校验、约束与个别未实现功能（见下）。

---

## 三、功能与需求差距清单（按画面与类别）

### 3.1 接口（API）层面

| Sms01206 (OrderEntryNo_i) | websms01206 | 说明 |
|---------------------------|-------------|------|
| getSales | ❌ 未实现 | 他社向販売検索（旧系统也未在 OrderEntryNo 中调用） |
| getPattern | ❌ 未实现 | 生産パターン検索（同上） |
| 其余 RMI 方法 | ✅ 已透传 | getLabelString、getPrice、orderInsert、getOrderSub、getConsign、getOrder、getVari、orderUpdate、orderDelete、getRankDialogMaster、getProductDialogMaster、getForwadingListHtml、setFactoryResults、getOutInData、updateOutinData、moveR3 等均已通过 api/* 透传 |

### 3.2 校验与业务约束（未实现或需加强）

| 项目 | Sms01206 行为 | websms01206 现状 | 建议 |
|------|----------------|------------------|------|
| **ランク=1 時 単価必须（機種・販価マスター）** | 機種変更時・追加/更新前 `checkRank1UnitPriceProduct()`：ERP 機種且 rank=1 时，若 `!isUnitPriceMaster()` 则报错并阻止 | ✅ 已在 validateForm 中实现：rank=1 且 erp 機種时要求 isUnitPriceMaster，否则 productCode 错误；rank=1 时単価必填 | 保持；可选在 selectRank 选择 rank=1 后即时调用 isUnitPriceMaster 并提示 |
| **販価区分=1（rank=1）时的标准単価チェック** | `isStandardUnitPrice(product)`：rank=1 时检查 `checkStandardUnitPrice`（priceCode=「3」個別販価不可），异常时 pkChoice 标红 | ❌ **未实现** | 在 getPrice/ランク选择后、以及 validateForm 中增加：rank=1 且 priceCode 为個別（如 "3"）时报错并标红販価欄 |
| **得意先 与信管理判定=8 且 rank=1 时的明細ロック** | `setCC8InputLock()`：根据 **cCustomer_**（与信管理判定，由 judgeCustomerConfidence 设置）与 rank=1 锁定表编辑、回収按钮 | ⚠️ **逻辑错误**：当前使用 `customerCode === '8'`（得意先コード） | 应使用「与信管理判定=8」：在 onCustomerChange 时从 judgementCustomerConfidence 的返回值中取得判定码（若 Imart 返回判定码），保存为 ccJudge；detailLocked 改为 `ccJudge === '8' && formData.decisionRank === '1'` |
| 明細 コメント 40 字 | 旧系统有明細コメント长度限制 | 明細无 comment 字段 | 若后端有该字段，则表头/明細一并校验 |
| 保管場所 スペースのみ不可 | 已有 onDetailStockBlur + validateForm | ✅ 已实现 | 保持 |
| 納期≤出荷希望日≤回答日 | validateDetailDatesForRow | ✅ 已实现 | 保持 |

### 3.3 一覧画面（list.jsp）

| 项目 | Sms01206 | websms01206 | 建议 |
|------|----------|-------------|------|
| getOrderSub 请求参数 | KEY=entryNo, KEY1=得意先(paraCustomer), LANG, CORPORATE, TYPE | 已使用 key, key1, key2, lang, corporate, type | ✅ 已一致 |
| 响应行映射 | 后端可能返回数组或对象（d_entry_no 等） | 已实现 mapOrderRow 兼容数组与多种键名 | ✅ 已一致 |
| **分页** | 旧系统多为前端/后端分页 | goToPage(p) 仅设 currentPage 并再次 loadOrders()，**未向 API 传 page/offset**；若后端不返回分页则 total 用 length | 若后端不支持分页：应在前端对 orders 做 slice 显示 `(currentPage-1)*pageSize` ～ `currentPage*pageSize`；若后端支持分页：请求体增加 page、pageSize 或 offset |

### 3.4 新規・編集表单（createForm.jsp）

#### 3.4.1 表头项目与明細

- 担当/得意先/機種/顧客注文No/コメント/納品書/納品/カートン/ユーザー/配送先/納入先/装置メーカー/case mark1～7/機展書No/ランク/販価/単価/通貨/顧客機種/用途/扱い/販売ルート等：**已存在且与旧系统对应**。
- 明細：注文数/出荷済/出荷残/分割/納期/出荷希望日/回答日/回答区分/乙仲出荷日/売上日/プラント/保管場所/税/インボイスNo/回収月/特/操作：**均已存在**；出荷残由 calcRemain 计算，与旧系统一致。

#### 3.4.2 未实现或需对齐的交互

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| ランク=1 時 単価必须 | 機種変更・追加/更新前校验 | ✅ validateForm 已实现 |
| ランク=1 時 標準販価チェック（個別販価不可） | isStandardUnitPrice / checkStandardUnitPrice，販価欄标红 | ❌ 未实现 |
| 得意先=8 且 rank=1 明細ロック | 与信管理判定=8（cCustomer_） | ⚠️ 当前用 customerCode===’8’，应改为与信判定码 |
| 廃止機種警告 | judgmentStopProduct 弹窗 | ✅ 已有 alert |
| 生販確定期間警告 | getPSSPeriodProduct 后画面表示 | ✅ 已有 pssPeriodProductFlg |
| 与信超过警告 | judgementCustomerConfidence 表示 | ✅ 已有 customerConfidenceLimit 与 message |
| **製品在庫計画ボタン** | 有按钮且可联动（seihinButton_） | **仅有按钮，未绑定 @click/API/画面** | 需确认旧系统该按钮调用的接口或画面并补全 |
| 付属品/検査結果/仕様書 | 有下拉或入力 | 当前为 label + 入力/下拉，需确认与 getVari 等后端键一致 |
| 確認ダイアログ文案 | displayConfirmDialog 等日英文案（getLabelString） | customConfirmModal 为固定日文 | 若需多语言可接 getLabelString 或与旧系统文案一致 |

### 3.5 详细画面（detail.jsp）

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| 表头 H26/H0/H1/H27/B3/B33/B2/H7/H8/伝票・添付・納入・カートン/荷受先・納入先/住所・部署・電話/メーカー・ユーザー/扱い・ルート・用途/単価・通貨・販価/case mark1～7 | 已按相同键显示 | ✅ 一致 |
| 明細 D01～Dxx 的键（'1','2','5'～'13'） | 与 getOrder 约定一致 | 已做 '0'/'1'、'4'/'5' 等兼容 | ✅ 一致 |
| 編集/一覧に戻る | 有 | 已有 | ✅ |

### 3.6 登録確認画面（createConfirm.jsp）

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| 数据来源 | 新規提交前或 session 数据 | sessionStorage 或 getOrder(entryNo) | ✅ 已有 |
| 登録実行 | 提交到 orderInsert | doRegister() 用 buildKeyFromFormData 调用 api/orderInsert，成功跳 list | ✅ 已实现 |
| 修正/戻る | 戻る到 create?form / list | 已有 | ✅ |

### 3.7 对话框与弹窗

| 对话框 | Sms01206 | websms01206 |
|--------|----------|-------------|
| ConsignDialog | 配送先一覧选择・自由数据追加 | ✅ 已有 modal + addConsignFree |
| RankDecisionDialog | getRankDialogMaster 选择 | ✅ openRankDialog / selectRank |
| ForwardRequestDialog | 出荷依頼表・Excel・出荷確定数 | ✅ 已合并到出荷依頼弹窗 |
| ProductDialog | 機種・得意先検索 | ✅ 機種検索 + getProductDialogMaster |
| FactoryNumberDialog | 出荷済み数入力 | ✅ 已合并到 Forward 弹窗「出荷確定数」 |
| OutinDialog | getOutInData / updateOutinData | ✅ 出荷依頼内「OUT-IN編集」 |
| ReplyDialog / TaxDialog | 回答区分・税区分选择 | ✅ 明細 a-select（getVari REPLY/TAX） |
| DistributeDialog / DeployDistributeDialog | 分割・配分展開 | ✅ 已有 |

---

## 四、Controller / 业务流程

| 项目 | 说明 |
|------|------|
| POST create（登録実行） | 确认画面已通过 JS 调用 api/orderInsert 并 redirect list；form POST 仅做兼容 redirect，逻辑正确 |
| detail 的 getOrder | corporate: 'X'，与编辑一致 |
| createConfirm 的 getOrder | 从 sessionStorage 或 entryNo + api/getOrder 取数，corporate 与编辑一致即可 |

---

## 五、总结：为与 Sms01206 完全一致仍需完成的事项（已补全情况）

### 5.1 必须修复/实现（✅ 已实现）

1. **得意先 与信管理判定=8 且 rank=1 的明細ロック** ✅  
   - createForm.jsp：新增 `ccJudge`，在 onCustomerChange 与 loadOrderData 中从 judgementCustomerConfidence 的 `ans`（及 `ans1`）解析判定码并保存；detailLocked 改为 `ccJudge === '8' && decisionRank === '1'`。

2. **ランク=1 时的标准単価チェック（個別販価不可）** ✅  
   - validateForm 中增加 rank=1 且 priceCode==='3' 时的错误；onPriceCodeChange / selectRank 中即时设置 validationErrors.priceCode 并标红販価欄。

3. **一覧分页** ✅  
   - list.jsp：新增 computed `displayedOrders`，对 `orders` 做 slice 显示当前页；goToPage 仅切换 currentPage，不再重复请求；totalPages 由 total 或 orders.length 计算。

### 5.2 建议确认后补全（✅ 已实现）

4. **製品在庫計画按钮** ✅  
   - Sms01206：seihinButton_ → getPlantStock/getTablePlantSloc → kidouSms01228(product, plants)，起動 stemapki.jsp?program_nm=sms01228&product=...&plant=...&stock_location=...。  
   - websms01206：为「製品在庫計画」绑定 @click="openSeihinStockPlan"；从 formData.productCode 与第一条有プラント/保管場所的明細取得 plant/stockLocation，若 `seihinPlanBaseUrl` 或 contextPath+'/stemapki.jsp' 存在则新窗口打开同参数 URL，否则提示 URL 未設定。

5. **付属品/検査結果/仕様書** ✅（确认无需改代码）  
   - 付属品 = attachmentCode（getVari ATTACHMENT）、検査結果・仕様書 = surveyReportCopy / specificationCopy（部添付）。表头与 buildKey 已一致，无需变更。

6. **明細コメント 40字** ✅  
   - createForm：emptyDetail 与 buildKey 的 Dxx 增加 `comment` / `'16'`；明細表增加「コメント」列，maxlength=40；validateForm 校验 40 字。createConfirm 的 formDataToAns / buildKeyFromFormData 同步含 '16': d.comment。

### 5.3 可选

- getSales / getPattern：旧系统也未在 OrderEntryNo 中调用，可按需追加。
- 確認ダイアログ多语言：通过 getLabelString 与旧系统文案统一。

---

## 六、结论：能否完全实现与 Sms01206 一致

- **画面与流程**：websms01206 的画面划分、URL、画面间跳转与数据流（list → create/update/detail/createConfirm → orderInsert → list）与 Sms01206 对应关系清晰，**可以**在现有架构下实现与 Sms01206 一致的主流程。
- **接口**：除 getSales/getPattern 外，所需 RMI 已通过 api 透传，**可以**支撑与旧系统一致的业务操作。
- **差距**：主要在于少数校验与约束（与信判定=8 的 detailLocked、rank=1 标准単価チェック）、一覧分页方式、以及「製品在庫計画」按钮与付属品/検査結果/仕様書的最终绑定。完成上述「必须」与「建议确认后补全」项后，**用户体验与操作逻辑可与 Sms01206 基本完全一致**；可选项按需实施即可进一步对齐细节与多语言。
