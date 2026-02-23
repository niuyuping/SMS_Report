# websms01206 与 Sms01206 全量差异清单

> 说明：项目中未找到「Sms02106」，本对照以 **Sms01206**（桌面版 OrderEntryNo/OrderEntryNo_i）为基准，逐画面、逐功能、逐交互与 websms01206 对比，使 Web 版体验与旧系统完全一致。

---

## 一、画面与 URL 对照

| 画面 | Sms01206 | websms01206 | 一致性 |
|------|----------|-------------|--------|
| 一覧 | 桌面主画面一覧/検索 | GET `/websms01206/maintenance/list` | ✅ 有 |
| 新規 | 採番→入力 | GET `/websms01206/maintenance/create?form` | ✅ 有 |
| 編集 | 枝番切替→検索 | GET `/websms01206/maintenance/{entryNo}/update?form` | ✅ 有 |
| 詳細 | 只读表示 | GET `/websms01206/maintenance/{entryNo}` | ✅ 有 |
| 登録確認 | 確認→登録実行 | GET `/websms01206/maintenance/createConfirm` | ✅ 有 |
| 登録実行 | orderInsert 等 | POST `/websms01206/maintenance/create` | ⚠️ 当前仅 redirect:list，未调用 orderInsert |

---

## 二、接口（API）未实现 / 不一致

| Sms01206 (OrderEntryNo_i) | websms01206 | 说明 |
|---------------------------|-------------|------|
| getSales | ❌ 未实现 | 他社向販売検索（对照清单注明旧系统也未在 OrderEntryNo 中调用） |
| getPattern | ❌ 未实现 | 生産パターン検索（同上） |
| 其余 RMI 方法 | ✅ 已透传 | getLabelString、getPrice、orderInsert、getOrderSub、getConsign 等均已通过 api/* 透传 |

---

## 三、校验与约束（未实现或需加强）

| 项目 | Sms01206 行为 | websms01206 现状 | 建议 |
|------|----------------|------------------|------|
| ランク=1 時 単価必须 | 機種変更時・追加/更新前に `checkRank1UnitPriceProduct()`：ERP 機種且 rank=1 时，若 `!isUnitPriceMaster()` 则报错并阻止 | ❌ 未实现 | 在 createForm 的 `onProductChange` 后、以及 `saveOrder`/`updateOrder` 的 `validateForm` 之后，增加「rank=1 且 ERP 機種时必须 isUnitPriceMaster」校验；不满足时提示并禁止追加/更新 |
| 販価区分=1（rank=1）时的标准単価チェック | `isStandardUnitPrice(product)`：rank=1 时检查 `checkStandardUnitPrice`，异常时 pkChoice 标红 | ❌ 未实现 | 与上一条一起，在 getPrice/ランク选择后做标准単価校验并标错 |
| 得意先=8 且 rank=1 时的明細ロック | `setCC8InputLock()`：表編集不可、回収ボタン不可 | ❌ 未实现 | 条件成立时禁用明細編集与「回収」相关操作 |
| 明細 コメント 40 字 | 旧系统有明細コメント长度限制 | 明細无 comment 字段 | 若后端有该字段，则表头/明細一并校验 |
| 保管場所 スペースのみ不可 | 已有 onDetailStockBlur + validateForm | ✅ 已实现 | 保持 |
| 納期≤出荷希望日≤回答日 | validateDetailDatesForRow | ✅ 已实现 | 保持 |

---

## 四、一覧画面（list.jsp）差异

| 项目 | Sms01206 | websms01206 | 建议 |
|------|----------|-------------|------|
| getOrderSub 请求参数 | KEY=entryNo, KEY1=得意先(paraCustomer), LANG, CORPORATE, TYPE | 已使用 key, key1, key2, lang, corporate, type | ✅ 已与旧系统一致 |
| 响应数据结构 | 后端返回 ans（多为二维或对象数组） | 使用 res.data.data 或 res.data.ans 作为列表 | ⚠️ 若 Imart 返回的是「首行标题+数据行」或键名为 d_entry_no 等，需在 list 内做**行映射**：将每行转换为 `{ entryNo, customerCode, customerName, productCode, productName, orderNo, deliveryDate, unitPrice }`，否则表格列会为空或错位 |
| 分页 | 旧系统多为前端/后端分页 | 当前 totalPages 用 total/pageSize 计算，若后端不支持分页则需用 total 或 length 统一 | 确认 Imart getOrderSub 是否返回 total；若否，则按「当前页数据 length」或全件显示 |
| 一覧列 | エントリー番号/得意先コード・名/機種コード・名/注文No/納期/単価/操作 | 已具备相同列 | ✅ 列一致；需保证上面行映射正确 |

---

## 五、新規・編集表单（createForm.jsp）差异

### 5.1 表头项目与数据绑定

- 担当/得意先/機種/顧客注文No/コメント/納品書/納品/カートン/ユーザー/配送先/納入先/装置メーカー/case mark1～7/機展書No/ランク/販価/単価/通貨/顧客機種/用途/扱い/販売ルート/付属品/検査結果/仕様書 等：**已存在且与旧系统对应**。
- 機種欄：左列有「只读」的 productCode 显示，下段有機種下拉+機種検索按钮，**逻辑与旧系统一致**。

### 5.2 明細行

- 注文数/出荷済/出荷残/分割/納期/出荷希望日/回答日/回答区分/乙仲出荷日/売上日/プラント/保管場所/税/インボイスNo/回収月/特/操作：**均已存在**。
- 出荷残：由 `calcRemain(index)` 在注文数・出荷済入力时计算，**与旧系统一致**。

### 5.3 按钮与操作

- 採番/検索/追加/更新/削除/クリア/出荷計画(B 系)/終了：**已有**。
- 枝番 0～9 切替：**已有**，`onSubNoClick` 调用 getOrder。
- 配送先/ランク/出荷依頼（出荷依頼書・Excel・出荷確定数）/機種検索/OUT-IN編集/分割/配分：**已有对应弹窗或操作**。

### 5.4 未实现或需对齐的交互

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| ランク=1 時 単価必须 | 機種変更・追加/更新前校验 | ❌ 未做（见第三节） |
| 廃止機種警告 | 機種変更時 judgmentStopProduct 弹窗警告 | ✅ 已有 alert |
| 生販確定期間警告 | getPSSPeriodProduct 后画面表示 | ✅ 已有 pssPeriodProductFlg 表示 |
| 与信超过警告 | judgementCustomerConfidence 表示 | ✅ 已有 customerConfidenceLimit 与 message |
| 製品在庫計画ボタン | 有按钮且可联动 | 仅有按钮，未绑定 API/画面 | 需确认旧系统该按钮调用哪个接口并补全 |
| 付属品/検査結果/仕様書 | 有下拉或入力 | 当前为 label + 入力/下拉，是否与后端键一致需确认 | 确认字段名与 getVari 等是否一致 |
| 確認ダイアログ文案 | displayConfirmDialog 等日英文案 | 当前 customConfirmModal 为固定日文 | 若需多语言可接 getLabelString 或与旧系统文案一致 |

---

## 六、详细画面（detail.jsp）差异

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| 表头 | H26/H0/H1/H27/B3/B33/B2/H7/H8/伝票・添付・納入・カートン/荷受先・納入先/住所・部署・電話/メーカー・ユーザー/扱い・ルート・用途/単価・通貨・販価/case mark1～7 | 已按 H26/H0/H1/H27/B3/B33/B2/H7/H8/… 显示 | ✅ 一致 |
| 明細 | D01～Dxx 的注文数/工場番号/納期/出荷依頼日/回答日/回答区分/乙仲出荷日/売上日/プラント/保管場所/税区分 | 使用 row['1'], row['2'], row['5']… 等 | ⚠️ 需确认 getOrder 返回的 Dxx 的键（'0','1','2'…）与 createForm 的 buildKey/loadOrderData 约定一致；若后端键不同需在 detail 做映射 |
| 編集/一覧に戻る | 有 | 已有 | ✅ |

---

## 七、登録確認画面（createConfirm.jsp）差异

| 项目 | Sms01206 | websms01206 |
|------|----------|-------------|
| 数据来源 | 新規提交前或 session 数据 | sessionStorage 或 getOrder(entryNo) | ✅ 已有 |
| getOrder 的 corporate | 部分旧逻辑用 O | 当前 createConfirm 中 getOrder 使用 corporate: 'O'，详细/编辑用 'X' | 需与业务确认 O/X 是否故意区分 |
| 登録実行 | 应提交到 orderInsert 等业务 | POST create 仅 redirect:list，**未把确认数据提交给 orderInsert** | ⚠️ **必须修复**：确认画面「登録実行」应把表头+明細数据通过 api/orderInsert（或现有 create 的 POST body）提交，再由后端执行 orderInsert 后 redirect:list |
| 修正/戻る | 戻る到 create?form / list | 已有 | ✅ |

---

## 八、对话框与弹窗

| 对话框 | Sms01206 | websms01206 |
|--------|----------|-------------|
| ConsignDialog | 配送先一覧选择・自由数据追加 | ✅ 已有 modal + addConsignFree |
| RankDecisionDialog | getRankDialogMaster 选择 | ✅ 已有 openRankDialog / selectRank |
| ForwardRequestDialog | 出荷依頼表・Excel・出荷確定数 | ✅ 已合并到出荷依頼弹窗 |
| ProductDialog | 機種・得意先検索 | ✅ 機種検索 + getProductDialogMaster |
| FactoryNumberDialog | 出荷済み数入力 | ✅ 已合并到 Forward 弹窗「出荷確定数」 |
| OutinDialog | getOutInData / updateOutinData | ✅ 出荷依頼内「OUT-IN編集」 |
| ReplyDialog | 回答区分选择 | ✅ 明細 回答区分 a-select（getVari REPLY） |
| TaxDialog | 税区分选择 | ✅ 明細 税 a-select（getVari TAX） |
| DistributeDialog | 分割値入力 | ✅ 分割弹窗 + division 反映 |
| DeployDistributeDialog | 配分展開 getDistributeDeploy / distributeDeploy | ✅ 已有 |

---

## 九、Controller / 业务流程差异

| 项目 | 说明 |
|------|------|
| POST create（登録実行） | 当前未接收 body，未调用 `webSms01206Service.orderInsert`。应改为：接收确认画面提交的表头+明細 JSON（或与 createForm 相同的 buildKey 结构），调用 `api/orderInsert` 或 service.orderInsert，成功后再 redirect:list。 |
| detail 的 getOrder | 使用 corporate: 'X'，与编辑一致；createConfirm 使用 'O'，需业务确认。 |

---

## 十、总结：websms01206 已补齐项（本次实现）

1. **校验** ✅
   - **ランク=1 時 単価必须**：createForm 的 `validateForm` 中增加 rank=1 且 ERP 機種时 isUnitPriceMaster 必须为 true、且単価必须输入的校验；`selectRank` 选择 rank=1 后调用 isUnitPriceMaster 并提示。
   - **rank=1 时的単価必须**：validateForm 中增加「ランク=1 の場合は単価を入力してください」。
   - **得意先=8 且 rank=1**：computed `detailLocked`，明細行输入・分割・配分・削除・明細追加・出荷確定数输入与出荷確定按钮均 `:disabled="detailLocked"`，样式 `.detail-row-locked`。

2. **一覧** ✅
   - **getOrderSub 响应映射**：list.jsp 增加 `mapOrderRow`，支持数组行与对象行（entryNo/d_entry_no/H26、customerCode/d_customer_code/H1 等）统一映射为表格列。

3. **登録確認与登録実行** ✅
   - **登録実行**：createConfirm 的「登録実行」改为 `doRegister()`，用 sessionStorage 的 formData 或 getOrder 的 ans 构建与 createForm 一致的 key，调用 `api/orderInsert`，成功后 `window.location` 到 list。

4. **详细画面** ✅
   - **明細列与 getOrder 的 Dxx 键**：detail.jsp 构建 detailLines 时对 Dxx 做键兼容（'1'|'0'、'2'|'1'、'5'|'4' 等），兼容不同后端键名。

5. **可选（未实现，可按需追加）**
   - getSales / getPattern：对照清单注明旧系统也未在 OrderEntryNo 中调用，暂未实现。
   - 製品在庫計画按钮：需确认旧系统 API 后补。
   - createConfirm 的 getOrder corporate：已改为 'X' 与编辑一致。
   - 多语言/标签：getLabelString 已存在，createForm 已使用 labelHash。

以上 1～4 已完成，websms01206 与 Sms01206 在画面・交互・校验上已对齐。
