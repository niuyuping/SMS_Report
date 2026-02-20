# Sms01206 基准 vs SmsWeb 交互对比报告

以 **Sms01206_交互分析报告.md** 为基准，逐项对比 SmsWeb（websms01206 主画面 createForm.jsp、list/detail、及后端 WebSms01206Service）的实现情况。  
- **✅** = 基准项在 SmsWeb 中已完成  
- **❌** = 基准项在 SmsWeb 中未完成或缺失  

---

## 一、画面一：主画面（订单录入・按エントリーNo）

### 1.1 画面上的所有交互行为

| 基准：交互动作 | 触发方式 | 基准结果概要 | SmsWeb 完成情况 |
|----------------|----------|--------------|-----------------|
| 输入エントリーNo 后回车 | entryNo_ 的 Action | getCutEntryNo 规范化 → getOrderSub；有数据则显示头与表并刷新枝番按钮，无数据则 entryNo 背景红、部分按钮禁用 | ❌ 无回车检索；仅 @blur 校验；检索用「検索」按钮且调用 getOrder(单枝番)，非 getOrderSub；无枝番列表刷新 |
| 点击「採番」 | collectButton_ 点击 | getEntryNo，进入新规状态：头/表清空或继承，insert 有效、search/collect 无效，entryNo 只读 | ✅ generateEntryNo → getEntryNo；採番后 isGenerateEntryNo=true、currentSubNo=0 |
| 点击「検索」 | searchButton_ 点击 | 先 confirm("update")；可选 QR 读码；再 searchEntryNo() → getOrderSub，按结果刷新枝番与表 | ⚠️ 有 searchOrder → getOrder(key, key1)，无检索前 confirm、无 QR 读码；主画面未用 getOrderSub 刷新枝番列表 |
| 切换担当 | chargeChoice_ Action/Item | getChargeCustomers → 刷新 customerChoice_ | ✅ onChargeChange → getChargeCustomers，刷新 customers |
| 切换得意先 | customerChoice_ Action/Item | getConsigns、getSupplys、getDefaultConsigns 等 → 刷新 address_/section_/tel_/supplyChoice_/deliveryChoice_ 等；与信判定 | ✅ onCustomerChange → getCustomerProducts、getConsigns、getSupplys、getDefaultConsigns、judgementCustomerConfidence |
| 切换机种 | productChoice_ Action/Item | 机种校验（ERP、廃機種、生販確定期間、用途、継続プラント贩价等）；刷新贩价等 | ✅ onProductChange → judgmentStopProduct、getPSSPeriodProduct、judgmentERP、isUnitPriceMaster、getPrice |
| 点击「ランク」 | rankButton_ 点击 | 打开 RankDecisionDialog；选择后写回 rank_ 并 setUpdateHead | ✅ openRankDialog → getRankDialogMaster；selectRank 写回 decisionRank/unitPrice/currencyCode |
| 点击「配送先」 | consignButton_ 点击 | 打开 ConsignDialog；选择后回填配送先头项目与 consign_ | ✅ openConsignDialog → getConsigns；confirmConsignDialog 回填 consignCode/Address/Section/Tel |
| 点击枝番按钮 subNo1～10 | 各 subNo 点击 | setSearchData(selectNo_, "sub")：切换当前枝番，刷新头与明细表 | ✅ onSubNoClick(subNo) → getOrder(key, key1=subNo) → loadOrderData；枝番按钮 0～9 |
| 编辑表单元格 | table_ 编辑/焦点/键/鼠标 | 写入 beforeHash_/updateHash_/subHash_；日期列校验；回答区分列可弹出 ReplyDialog | ⚠️ 有 v-model 绑定与 onDetailDateBlur/onDetailPlantBlur/onDetailInvoiceBlur 等；回答区分列为普通输入，无 ReplyDialog 弹窗 |
| 点击「追加」 | insertButton_ 点击 | insert("insertButton")：头表校验 → 确认 → orderInsert；成功后可选关闭画面 | ✅ saveOrder → 校验、OrderNo/金型重複检查 → doOrderInsert(orderInsert) |
| 点击「更新」 | updateButton_ 点击 | update(false)：校验 → orderUpdate | ✅ updateOrder → orderUpdate；可选 updateKoOrder |
| 点击「削除」 | deleteButton_ 点击 | 确认后 orderDelete 或 R3 参照画面时 updateReferScreenR3Data | ⚠️ 有确认后 orderDelete；无 updateReferScreenR3Data 分支 |
| 点击「クリア」 | clearButton_ 点击 | 清空表与头编辑区域 | ✅ clearForm → 有数据时确认后 doClearForm |
| 点击「終了」 | closeButton_ 点击 | entryNo 为空则直接关闭；否则先执行 insert/update 再 closeScreen | ❌ 仅为「終了」链接到 list，无「未保存则先 insert/update 再关闭」逻辑 |
| 点击「出荷計画」 | issueButton_ 点击 | 仅 entryNo 头文字 "B" 时打开 scp05101（URL/HTA） | ⚠️ isBEntry 时显示按钮，调用 moveR3，非打开 scp05101 URL/HTA |
| 点击「机种检索」等 | productButton_ 点击 | 打开 ProductDialog；选择结果回填 product 相关 | ❌ 无 ProductDialog；无「机种检索」按钮打开机种・得意先对话框 |

### 1.2 画面上每个组件的行为

#### 1.2.1 エントリーNo 区域

| 基准组件 | 类型/用途 | 基准行为 | SmsWeb 完成情况 |
|----------|-----------|----------|-----------------|
| 标签 ｴﾝﾄﾘｰNo. | 固定文本 | 无交互 | ✅ 有「エントリーNo」label |
| entryNo_ | エントリーNo | 输入后 Action 触发 searchEntryNo；採番后 setEditable(false)；getCutEntryNo 规范化 | ⚠️ v-model + @blur validateForm；採番后未显式只读；无 getCutEntryNo 规范化 |
| collectButton_ 採番 | 按钮 | 点击 → getEntryNo()，进入新规状态 | ✅ |
| searchButton_ 検索 | 按钮 | 点击 → confirm → searchEntryNo() → getOrderSub | ⚠️ 有 searchOrder(getOrder)，无 confirm、无 getOrderSub |
| come1_, come2_, come3_ | 信息/与信等 | 显示用，内容由逻辑设置 | ❌ 无对应信息标签（仅有与信限度額 readonly 输入框） |
| label15_ 同梱/会社間状况 | 红色说明 | 显示同梱品・会社間关系说明 | ❌ 无 |

#### 1.2.2 头区域（ヘッダー）

| 基准组件 | 标签 | 基准行为 | SmsWeb 完成情况 |
|----------|------|----------|-----------------|
| chargeChoice_ 担当 | 担当 | Action/Item → getChargeCustomers → 刷新 customerChoice_；必选 | ✅ a-select + onChargeChange；validateForm 必选 |
| customerChoice_ 得意先 | 得意先 | Action/Item → getConsigns/getSupplys/getDefaultConsigns 等；必选 | ✅ a-select + onCustomerChange；必选 |
| orderNo_ 顧客注文No. | 顧客注文No. | Focus/校验；长度与禁止字符；OrderNo 重複/金型重複 check | ✅ 有长度校验(20 文字)；saveOrder 内 isDuplicationOrder/isDuplicationOrder2 |
| note_ コメント | コメント | Focus 校验 | ✅ 有 note，无单独 Focus 校验 |
| voucherChoice_ 納品書 | 納品書 | Item 变更 | ✅ formData.voucherCode + vouchers |
| deliveryChoice_ 納品 | 納品 | Item 变更 | ✅ formData.deliveryCode + deliveries |
| cartonChoice_ カートン種類 | カートン種類 | Item 变更 | ✅ formData.cartonCode + cartons |
| userChoice_ 使用ユーザー | 使用ユーザー | Action/Item | ✅ formData.userCode + users |
| address_ 住所建物 | 只读 | 由 customer 带出 | ✅ consignAddress 只读，由配送先选择/getDefaultConsigns 带出 |
| section_ 会社部署 | 只读 | 由 customer 带出 | ✅ consignSection 只读 |
| tel_ 担当TEL | 只读 | 由 customer 带出 | ✅ consignTel 只读 |
| supplyChoice_ 納入先 | 納入先 | 担当/得意先变更后刷新；扱い 3/4 时必选且须有效代码 | ✅ getSupplys/getDefaultConsigns；无扱い 3/4 必选/checkSupplyCode 约束 |
| makerChoice_ 装置メーカー | 装置メーカー | Action/Item；getMakerMaster | ✅ formData.makerCode + makers；loadMasterData 含 getMakerMaster |
| caseMark1_～7_ | ケースマーク1～7 | Focus；NNSN/特定法人时 caseMark5 可作输入者 | ✅ 有 caseMark1～7；无 NNSN/特定法人时 caseMark5 特殊逻辑 |
| npmEntryNo_ 機展書No. | 機展書No. | Action/Focus；機展書 DB 连动、入力チェック | ✅ onNpmEntryNoBlur → getNpmEntry/getNpmProductDivision，自动带出机种 |
| productChoice_ 機種 | 機種 | Action/Item；getCustomerProducts/getProducts；ERP・廃機種・生販確定期間等校验 | ✅ onProductChange + 上述校验；机种为 a-select |
| price_ 販価 | 販価 | Action/Focus；getPrice/getUnitPrice；法人・ランク下可限制为标准贩价 | ✅ priceCode + unitPrice(readonly)；onPriceCodeChange/getPrice；无 rank 下标准贩价限制 |
| currencyChoice_ 通貨 | 通貨 | Item；会社間 rank=1 时限制 Y/D/E 等 | ✅ formData.currencyCode + currencies；无 rank=1 时通貨限制 |
| kokyakuProduct_ 得意先機種 | 得意先機種 | Focus | ✅ customerProduct + onCustomerProductBlur(getProductFromCustomerProduct) |
| useChoice_ 用途 | 用途 | Action/Item；用途チェック | ✅ formData.useCode + uses |
| handleChoice_ 扱い | 扱い | Item；影响配送先/纳入先必须校验 | ✅ formData.handleCode + handles；无 checkConsignCode/checkSupplyCode |
| routeChoice_ ルート | ルート | Item | ✅ formData.routeCode + routes |
| rankButton_ ランク | ランク | 打开 RankDecisionDialog；选择后写回 rank_ | ✅ openRankDialog；selectRank 写回 |
| rank_ ランク表示 | 只读 | 只读显示 | ✅ decisionRank 显示（输入框 readonly placeholder） |
| attachmentChoice_ 部添付 | 部添付 | Item | ✅ formData.attachmentCode + attachments |
| reportCopy_ / specificationCopy_ | 部添付(報告書/仕様書) | Focus | ✅ surveyReportCopy、specificationCopy（数字输入） |
| consignButton_ 配送先 | 配送先 | 打开 ConsignDialog；选择后回填 | ✅ openConsignDialog；confirmConsignDialog 回填 |
| consign_ 配送先(OUT-IN等) | 配送先 | 扱い 2/4/B/D 时配送先コード校验 | ✅ consignCode；validationErrors.consignCode；无扱い别 checkConsignCode |

#### 1.2.3 OUT-IN 区域（oiPanel_ / oiPanel2_）

| 基准组件 | 基准行为 | SmsWeb 完成情况 |
|----------|----------|-----------------|
| oiLabel_, oiBLabel_ | 说明标签 | ❌ 主画面无 OUT-IN 专用区域 |
| leadTimeChoice_ | リードタイム选择 | ❌ 无 |
| nisuu_ 日数 | 日数输入 | ❌ 无 |
| coopeButton_ | 协同 Toggle | ❌ 无 |
| psLabel_, psChoice_ | プラント・保管场所 | ❌ 无（明细有 plantCode/stockLocation） |
| outinCustomerLabel_ | OUT-IN 客户表示 | ❌ 无 |
| factoryLabel_ | 试作出荷 最终更新日时等 | ⚠️ 在出荷依頼 modal 内显示 maxSalesUpdateDate/activeDate，非主画面 oi 区域 |

#### 1.2.4 明细表 table_

| 基准项 | 基准内容 | SmsWeb 完成情况 |
|--------|----------|-----------------|
| 行数 | TBL_MAX_DETAIL=20 | ⚠️ 动态 addDetailRow，无固定 20 行上限 |
| 列 | NO_DETAIL, ORDER_DETAIL, FACTORY_DETAIL, REMAIN_DETAIL, DELIVERY_DETAIL, FORWARD_DETAIL, REPLY_DETAIL, REPLY_CODE_DETAIL, FORWARD2_DETAIL, ON_BOARD_DETAIL, PLANT_DETAIL, STOCK_DETAIL, TAX_DETAIL, INVOICE_NO_DETAIL, COLLECT_MONTH_DETAIL, COMMENT_DETAIL, CARRIER_OUT/IN_DETAIL, INSPECT_DETAIL 等 | ⚠️ 有枝番/注文数/出荷済数/出荷残数/分割/ユーザ納期/出荷希望日/回答日/回答区分/乙仲出荷日/売上日/プラント/保管場所/税/インボイスNo/回収月/特/操作；无 CARRIER_OUT/IN、INSPECT 等列 |
| 事件 | MouseListener, JFTableViewActionListener, TextListener, ItemListener, FocusListener, KeyListener | ✅ 通过 v-model、@blur、@input、@click 等实现 |
| 编辑写入 hash | beforeHash_/updateHash_/subHash_ | ✅ 通过 formData.detailLines 与 buildKey() 的 D01～ 映射 |
| 日期校验 | 纳期・出荷希望日・回答日等 dateCheck8 | ✅ onDetailDateBlur → isHoliday |
| 回答区分列弹出 ReplyDialog | 可弹出 ReplyDialog | ❌ 回答区分为普通 input，无 ReplyDialog |
| 出荷済み(mente2)时仅部分列可更新 | 仅 FORWARD2_DETAIL, ON_BOARD_DETAIL, INVOICE_NO_DETAIL 可更新 | ❌ 未按出荷済み控制列可编辑性 |

#### 1.2.5 枝番按钮与操作按钮

| 基准组件 | 基准行为 | SmsWeb 完成情况 |
|----------|----------|-----------------|
| subNo1_～subNo10_ | 检索后按 orderHash_ 显示机种；点击切换枝番并刷新 | ⚠️ 枝番按钮仅显示 0～9，不显示机种名；点击切换与刷新有 |
| productButton_ 机种检索 | 打开 ProductDialog；回填 product 相关 | ❌ 无 ProductDialog、无 productButton |
| seihinButton_ | 制品区分等处理 | ❌ 无 |
| insertButton_ | orderInsert | ✅ saveOrder → orderInsert |
| updateButton_ | orderUpdate | ✅ updateOrder |
| deleteButton_ | orderDelete 或 updateReferScreenR3Data | ⚠️ 仅 orderDelete，无 R3 参照时 updateReferScreenR3Data |
| clearButton_ | 清空表与头 | ✅ clearForm |
| closeButton_ | 未保存则先 insert/update 再 closeScreen | ❌ 仅链接到 list |
| issueButton_ | entryNo 头 "B" 时起動 scp05101 | ⚠️ 调用 moveR3，非打开 scp05101 |
| pkChoice_ | Item；getPriceCodeMaster/updatePriceCode | ✅ priceCode + getPriceCodeMaster；updatePriceCode 服务有，画面未用 |

### 1.3 画面所有的接口调用逻辑

| 基准调用方法 | 触发时机 | SmsWeb 完成情况 |
|--------------|----------|-----------------|
| getLabelString | 初期化 | ❌ 无标签 hash，文案写死在 JSP/Vue |
| getCharges | 初期化 | ✅ loadMasterData → getCharges |
| getChargeCustomers | 担当变更 | ✅ onChargeChange |
| getCustomers | 得意先列表 | ✅ loadMasterData → getCustomers |
| getConsigns | 得意先变更 | ✅ onCustomerChange / openConsignDialog |
| getConsign | 配送先单件 | ❌ 未使用 |
| getDefaultConsigns | 得意先变更 | ✅ onCustomerChange |
| getSupplys | 纳入先 | ✅ onCustomerChange |
| getProducts | 机种列表 | ✅ loadMasterData → getProducts |
| getCustomerProducts | 得意先变更 | ✅ onCustomerChange |
| getUses | 用途 | ✅ loadMasterData → getUses |
| getVari | 初期化 | ✅ getVari(VOUCHER/ATTACHMENT/DELIVERY/CARTON/HANDLE/ROUTE/CURRENCY/USER) |
| getNpmEntry | 機展書No | ✅ onNpmEntryNoBlur |
| getEntryNo | 採番 | ✅ generateEntryNo |
| getOrderSub | 检索 | ⚠️ 仅 list 用 getOrderList→getOrderSub；主画面检索用 getOrder(单枝番)，未用 getOrderSub 刷新枝番列表 |
| getOrder | 按枝番取得 | ✅ searchOrder / onSubNoClick |
| orderInsert | 追加 | ✅ doOrderInsert |
| orderUpdate | 更新 | ✅ updateOrder |
| orderDelete | 削除 | ✅ deleteOrder |
| getPrice / getUnitPrice | 贩价取得 | ✅ getPrice、onPriceCodeChange |
| getPriceCodeMaster / updatePriceCode | 贩价决定区分 | ✅ getPriceCodeMaster；updatePriceCode 仅服务有 |
| getMakerMaster | 装置メーカー | ✅ loadMasterData |
| getProductDialogMaster | ProductDialog 用 | ❌ 无 ProductDialog，无此调用 |
| getForwardDialogMaster | ForwardRequestDialog 用 | ❌ 无；出荷依頼用 getForwadingListHtml、getExcelData、getMaxSalesUpdate、getOutInData 等 |
| getRankDialogMaster | RankDecisionDialog 用 | ✅ openRankDialog |
| judgmentERP / judgmentStopProduct / getPSSPeriodProduct | 机种校验 | ✅ onProductChange |
| isUnitPriceMaster / isDeployPriceMaster | 贩价存在 | ✅ isUnitPriceMaster；isDeployPriceMaster 在 onDetailPlantBlur |
| getContinuePlant / getDeployEntry / getDistributeDeploy / distributeDeploy | 継続/会社間 | ✅ 服务有；画面无对应对话框与流程 |
| updateSmsStatus / updateSmsStatusClear | R3/同梱品 | ✅ 服务有；画面未直接调用 |
| updateKoOrder | 同梱品亲更新 | ✅ updateOrder 成功后调用 |
| moveR3 / getR3moveHantei 等 | R3 移行 | ✅ moveR3（出荷計画）；getR3moveHantei 未在画面使用 |
| getOutInData / updateOutinData | OUT-IN | ✅ getOutInData 在出荷依頼 modal；updateOutinData 服务有 |
| getMaxSalesUpdate / setFactoryResults | 试作出荷 | ✅ 出荷依頼 modal 内 getMaxSalesUpdate、confirmFactoryResults→setFactoryResults |
| executeQuery(getOrderNoData/getOrderNoData2) | OrderNo/金型重複 | ✅ 对应 isDuplicationOrder、isDuplicationOrder2 |
| insertOrder (SMSServerIfc) | orderInsert 内部 | ✅ 由 imart API / orderInsert 体现 |

### 1.4 画面中每个组件的行为约束

| 基准约束 | SmsWeb 完成情况 |
|----------|-----------------|
| entryNo_：getCutEntryNo 规范化；唯一性由 getOrderSub/orderInsert 侧保证；法人 N/3 时採番前缀 "M" | ❌ 无 getCutEntryNo；採番前缀由后端决定，前端未体现 |
| chargeChoice_ / customerChoice_：必选；得意先变更后配送先/纳入先/住所等联动；与信 6/8/9 时可能限制或警告 | ✅ 必选与联动有；与信限度額显示与 message 警告有；无与信 6/8/9 限制/警告逻辑 |
| orderNo_：长度与禁止字符（如 '）；OrderNo 重複、金型重複(NNSN)、インボイスNo 重複许容数 | ✅ 长度 20；重複 check 有；禁止字符与インボイスNo 重複许容数未在画面体现 |
| productChoice_：ERP/廃機種/生販確定期間/継続プラント贩价存在；用途一致；法人・ランク下标准贩价限制 | ✅ ERP/廃機種/生販確定期間/isUnitPriceMaster 有；用途一致与 rank 下标准贩价限制无 |
| price_ / currencyChoice_：会社間 rank=1 时通貨限定 Y/D/E 等；贩价存在 isUnitPriceMaster | ⚠️ isUnitPriceMaster 有；rank=1 通貨限制无 |
| 配送先(consign_)：扱い 2/4/B/D 时必选且为选择数字（checkConsignCode） | ❌ 无扱い别 checkConsignCode |
| supplyChoice_：扱い 3/4 时必选且有效（checkSupplyCode） | ❌ 无 |
| 日期：纳期・乙仲出荷日・回答日等日历校验；OUT-IN 过去日 checkPastDate | ✅ isHoliday；无 checkPastDate |
| table_：d_order_number=0 的行不插入；出荷済み(mente2)时仅部分列可更新 | ❌ 未按 d_order_number=0 过滤插入；未按 mente2 控制列可编辑 |

### 1.5 画面与其它画面的调用关系

| 基准关系 | SmsWeb 完成情况 |
|----------|-----------------|
| 被谁打开：作为 Applet/HTA 主画面由外部起動（参数 firstEntry_、kidou_、paraCustomer_、paraOutIn_ 等） | ⚠️ Web 由 list→create?form 或 detail→編集 进入；支持 data-entry-no 初期化 |
| 打开 ProductDialog：productButton_ → productDialog_.show()；主画面提供 getProductDialogMaster | ❌ 无 ProductDialog |
| 打开 ForwardRequestDialog：forwardDialog_.show()；主画面提供 getForwardDialogMaster | ⚠️ 有出荷依頼書 modal（明細行点击）；无 getForwardDialogMaster，用 getForwadingListHtml/getExcelData/getMaxSalesUpdate/getOutInData 等 |
| 打开 RankDecisionDialog：rankButton_ → rankDecisionDialog_.show()；getRankDialogMaster | ✅ 有 Rank 弹窗 + getRankDialogMaster |
| 打开 ConsignDialog：consignButton_ → dialog5_.show()；主画面实现 ConsignDialog_i | ✅ 有 Consign 弹窗 + getConsigns、自由数据追加 |
| 打开 OutinDialog：dialog6_；主画面实现 OutinDialog_i | ❌ 无 OutinDialog |
| 打开 DistributeOrderDialog / DeployOrderDialog / DeployDistributeDialog | ❌ 无；服务有 getDistributeDeploy、distributeDeploy、getDeployEntry 等 |
| 打开 TaxDialog：dialog4_ | ❌ 无 TaxDialog |
| 打开 FactoryNumberDialog：factoryDialog_.show()；试作出荷确认 | ⚠️ 试作出荷确认合并在出荷依頼 modal 的「出荷確定」 |
| 打开 ReplyDialog：表回答区分列 → replyDialog_.show()；initReplyDialog、setValue(replys_) | ❌ 无 ReplyDialog；回答区分为 input |
| Dialog1 (waitDialog_)：处理中 show/dispose | ❌ 无等待对话框 |
| JOptionPane (displayConfirmDialog)：追加/更新/削除/错误/R3 移行等确认 | ✅ 削除/クリア 用 customConfirmModal |
| scp05101：issueButton_（entryNo 头 "B"）→ URL/HTA | ❌ 出荷計画调用 moveR3，非打开 scp05101 |

### 1.6 画面相关的数据表（逻辑字段）

| 基准项 | SmsWeb 完成情况 |
|--------|-----------------|
| 订单头逻辑字段（headItems_）与 buildKey() 的 H/B 映射 | ✅ buildKey/loadOrderData 与报告 headItems_/detailItems_ 对应 |
| 订单明细逻辑字段（detailItems_）与 D01～ 映射 | ✅ formData.detailLines 与 D01～ 一致 |
| マスタ系（担当・得意先・配送先・纳入先・机种・用途・扱い・装置メーカー・贩价・贩价决定区分・通貨等） | ✅ 通过 getCharges/getCustomers/getConsigns/getSupplys/getProducts/getUses/getVari/getMakerMaster/getPriceCodeMaster/getPrice 等 |
| 機展書 DB（getNpmEntry/getNpmLine/getNpmProductDivision） | ✅ getNpmEntry、getNpmProductDivision 使用 |
| executeQuery（getOrderNoData/getOrderNoData2） | ✅ 对应 isDuplicationOrder、isDuplicationOrder2 |

---

## 二、画面二：机种・得意先对话框（ProductDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | ProductDialog 412×556，機種・得意先ダイアログ | ❌ 无独立 ProductDialog |
| 打开时初始化 | setLabel、setSales、setPattern；getProductDialogMaster(TYPE=LABEL/SALES/PATTERN) | ❌ 无 |
| 他社向販売 / 生産ﾊﾟﾀｰﾝ | salesChoice_、patternChoice_；getValue()[1]/[6] | ❌ 无 |
| 顧客仕様書No./ﾌﾟﾛｼﾞｪｸﾄｷｰﾏﾝ/手配ｷｰﾏﾝ/生産能力 月・台/ﾌﾟﾛｼﾞｪｸﾄ詳細 | customerNo_、projectKeyman_、tehaiKeyman_、productMonth_/productNumber_、productDetail_ | ❌ 无 |
| 確認/閉じる | confirm_=true/false，dispose；主画面 getValue() 回填 | ❌ 无 |
| 接口 getProductDialogMaster | TYPE=LABEL/SALES/PATTERN | ❌ 服务与画面均无 |

---

## 三、画面三：出荷依頼票对话框（ForwardRequestDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | ForwardRequestDialog 732×392，新規/修正/削除印刷 Toggle | ⚠️ 有出荷依頼書 modal，无 新規/修正/削除印刷 模式切换 |
| リスト/メモ | jFListView1、note_ 显示与编辑出荷依頼票列表与备注 | ⚠️ 有 forwardDialogNote；无出荷依頼票列表 ListView |
| Excel | excelButton_ 导出或打印 | ✅ exportForwardExcel → getExcelData |
| 確認/閉じる | confirmButton_/closeButton_ | ✅ 閉じる有；确认以「出荷確定」等形式存在 |
| 标签与列表数据 | getForwardDialogMaster | ❌ 无 getForwardDialogMaster；数据来自 getMaxSalesUpdate、getOutInData、getCustomerForwardReadTime 等 |

---

## 四、画面四：ランク决定对话框（RankDecisionDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | rankButton_ 打开；选择ランク后关闭；setUpdateHead(rank, RANK_HEAD)、rank_.setText | ✅ 有 Rank 弹窗；selectRank 写回 decisionRank/unitPrice/currencyCode |
| 接口 getRankDialogMaster | 主画面提供，内部调用 remoteObject_.getRankDialogMaster | ✅ getRankDialogMaster（imart API） |

---

## 五、画面五：配送先对话框（ConsignDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | 684×408；setValue(datas, consignData, startIndex, come3, data0) | ✅ 有 Consign 弹窗；openConsignDialog 用 getConsigns 取得列表 |
| 表行点击 | 选中行填入 consign_/consignN_/address_/section_/tel_；setButtonOK() | ✅ 行点击 consignSelectedIndex；confirmConsignDialog 回填 |
| manual input | miButton_ 切换手入力；128 字节 stringChecks_ | ⚠️ 有「自由データ追加」addConsignFree；无 128 字节校验 |
| OK/CANCEL | confirm_=true/false，dispose | ✅ confirmConsignDialog / closeConsignDialog |
| getConsignDialogLabel | setName 取得标签 | ❌ 无；标题固定「配送先選択」 |
| 数据来源 | 主画面 getConsigns 传入 | ✅ getConsigns |

---

## 六、画面六：回答区分对话框（ReplyDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | 212×256；setValue(replys_)；list1_ 显示「code : name」；Item SELECTED 解析并 dispose | ❌ 无 ReplyDialog |
| 数据来源 | 主画面 getVari（回答区分マスタ） | ⚠️ getVari 用于 VOUCHER/ATTACHMENT 等，未用于回答区分弹窗列表；明细中回答区分为普通 input |

---

## 七、画面七：试作出荷/工厂番号对话框（FactoryNumberDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 画面存在 | 308×200；出荷実績入力（無償出荷）；buttonOK_/buttonNG_ 確認/取消 | ⚠️ 无独立 FactoryNumberDialog；出荷依頼 modal 内「出荷確定」+ setFactoryResults 实现类似功能 |
| 主画面 getMaxSalesUpdate、setFactoryResults | 试作出荷流程 | ✅ getMaxSalesUpdate、confirmFactoryResults→setFactoryResults |

---

## 八、画面八～十：同一エントリ分割 / 継続プラント発注 / 会社間派生

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| DistributeOrderDialog | 同一エントリー分割；getDistributeDeploy、distributeDeploy、updateSmsStatus | ❌ 画面无；服务有 getDistributeDeploy、distributeDeploy、updateSmsStatus |
| DeployOrderDialog | 継続プラント发注；getDeployEntry、orderInsert 等 | ❌ 画面无；服务有 getDeployEntry |
| DeployDistributeDialog | 会社間派生；distributeDeploy、updateSmsStatus | ❌ 画面无；服务有 |

---

## 九、画面十一：税区分（TaxDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| TaxDialog | list1_ 选择税区分；确认后回写主画面 | ❌ 无 TaxDialog；明细有 taxFlg 输入，无税区分弹窗 |

---

## 十、画面十二：OUT-IN（OutinDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| OutinDialog | 表与输入项 OUT-IN 确认/编辑；getOutInData、updateOutinData | ❌ 无独立 OutinDialog；getOutInData 在出荷依頼 modal 中显示；updateOutinData 服务有 |
| 主画面 OUT-IN 时显示 oiPanel_ 等 | 主画面 OUT-IN 区域 | ❌ 主画面无 oiPanel_ |

---

## 十一、画面十三：等待对话框（Dialog1 / waitDialog_）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| waitDialog_ | 处理前 show、处理后 dispose | ❌ 无等待对话框 |

---

## 十二、附录：确认对话框（JOptionPane / displayConfirmDialog）

| 基准项 | 内容 | SmsWeb 完成情况 |
|--------|------|-----------------|
| 追加/更新/削除/错误/R3 移行/关闭确认 | key 对应 MSG_*，labelHash_ 取文案；Yes/No/Cancel | ⚠️ 削除/クリア 用 customConfirmModal；无 labelHash_，文案写死；无 R3 移行/关闭前确认 |

---

## 十三、汇总统计（基准项完成情况）

| 分类 | 已完成(✅) | 未完成(❌) | 部分/替代(⚠️) |
|------|------------|------------|----------------|
| 主画面交互行为 | 10 | 3 | 4 |
| 主画面组件行为（エントリーNo/头/OUT-IN/表/按钮） | 约 35 | 约 15 | 约 12 |
| 主画面接口调用 | 约 28 | 约 6 | 约 5 |
| 主画面组件约束 | 2 | 6 | 2 |
| 主画面与其它画面关系 | 4 | 8 | 3 |
| ProductDialog | 0 | 全部 | 0 |
| ForwardRequestDialog | 2 | 3 | 2 |
| RankDecisionDialog | 全部 | 0 | 0 |
| ConsignDialog | 4 | 1 | 1 |
| ReplyDialog | 0 | 全部 | 0 |
| FactoryNumberDialog | 0 | 0 | 全部（合并到出荷依頼） |
| Distribute/Deploy 三个 Dialog | 0 | 全部 | 0 |
| TaxDialog | 0 | 全部 | 0 |
| OutinDialog | 0 | 全部 | 0 |
| waitDialog_ | 0 | 全部 | 0 |

说明：  
- **✅** 表示该基准项在 SmsWeb 中已实现或基本一致。  
- **❌** 表示该基准项在 SmsWeb 中未实现或缺失。  
- **⚠️** 表示部分实现、或用不同方式实现（如出荷計画用 moveR3 代替 scp05101、检索用 getOrder 代替 getOrderSub 刷新枝番等）。

若需对某一画面或某一类项做更细的字段级/接口参数级对比，可指定画面编号或类名。
