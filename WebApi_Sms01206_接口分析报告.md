# WebApi（Sms01206）接口分析报告：参数与约束（从 Sms01206 反推）

本报告**仅依据 Sms01206（OrderEntryNo_s / OrderEntryNo_i）源码**，反推 WebApi 各接口应传递的参数的具体信息与相关约束，不引用 SmsWeb。WebApi 指对 Sms01206 的 HTTP 封装；请求体 JSON 键与 `prm.get(KEY)` 等一致，响应 `data` 为 Sms01206 返回的 Hashtable 的 JSON 序列化。

---

## 一、通用约定

### 1.1 键名对应

| 请求体 JSON 键 | Sms01206 常量 | 说明 |
|----------------|---------------|------|
| key | KEY | 主键/主参数，类型因接口而异（String / String[] / Hashtable） |
| key1 | KEY1 | 副键/第二参数 |
| key2 | KEY2 | 第三参数 |
| key3 | KEY3 | 第四参数 |
| key6 | KEY6 | 处理判定等 |
| keys | KEYS | 项目名数组等 |
| outin | OUTIN | OUT-IN 用 Hashtable |
| corporate | CORPORATE | 法人コード |
| type | TYPE | 種別（如 "order"） |
| lang | LANG | 言語（如 "ja"） |

### 1.2 响应 data

- 成功时 HTTP 响应体：`{ "error": false, "data": { "ans": ..., "ans1": ... } }`。
- `data` 的键与 Sms01206 返回的 Hashtable 一致：`ans`（ANS）、`ans1`（ANS1）、`ans2`（ANS2）等。
- 类型：由 Sms01206 的 `ansHash.put(ANS, value)` 决定（String、Boolean、String[]、String[][]、Hashtable、Vector 等），JSON 序列化后为 string/boolean/array/object。

### 1.3 约束与 null 处理

- 源码中 `(String)prm.get(KEY)` 等：若未传或为 null，得到 `null`，后续可能 NPE 或按 `isEmpty` 判断（`isEmpty` 为 `s==null || s.length()==0`）。
- **必填**：在方法内被直接使用且无 null 分支的键，视为必填；否则视为可选。
- **类型**：以源码中 cast 为准（String、String[]、Hashtable、Vector 等）；数组下标、长度约束从源码归纳。

---

## 二、orderInsert

### 2.1 接口与路径

- **路径**: `POST {baseUrl}/sms01206/orderInsert`

### 2.2 请求参数（从 Sms01206 反推）

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Object (Hashtable) | 是 | 订单头+明细 | 见下方「key（orderHash）结构」。键为 "H0"～"H43"（头）与 "D1","D2",…（明细）；值头为 String，明细值为 Object（明细行 Hashtable）。**不可为 null**，否则 NPE。 |
| **outin** | Object (Hashtable) | 否 | OUT-IN 画面用 | key：明细号（detailNo，String）；value：String[]（oiData）。可为 null；若为 null，`oiTbl.get(detailNo)` 为 null 则不向 hash 放 OUTIN。 |
| **key6** | String | 否 | 処理判定（syoriHantei） | 仅日志用，可空。 |
| **key1** | String | 否 | 会社間発注用 法人（dpCorporate） | 非空时写入 `d_deploy_corporate`；空或 null 不设置。 |
| **corporate** | String | 是 | 法人コード | 写入 `d_corporate_code`，用于 insertOrder。 |
| **type** | String | 是 | 種別（如 "order"） | 传给 SMSServerIfc.TYPE。 |
| **lang** | String | 是 | 言語（如 "ja"） | 用于 `isInsertEntry(checkEntry, lang, type)` 等。 |

### 2.3 key（orderHash）结构（Sms01206 头・明细定义）

- **头部分**：键名为 `"H" + 头索引`（0～43），即 **H0, H1, …, H43**。值为 String。  
  索引与项目对应关系（`headItems_[i]`，i=0..43）：

| 索引 | 键名 | 项目（DB/逻辑） |
|------|------|------------------|
| 0 | H0 | d_charge_code |
| 1 | H1 | d_customer_code |
| 2 | H2 | d_npm_entry_no |
| 3 | H3 | d_product_code |
| 4 | H4 | d_use_code |
| 5 | H5 | d_handle_code |
| 6 | H6 | d_route_code |
| 7 | H7 | d_order_no |
| 8 | H8 | d_note |
| 9 | H9 | d_voucher_code |
| 10 | H10 | d_attachment_code |
| 11 | H11 | d_delivery_code1 |
| 12 | H12 | d_survey_report_copy |
| 13 | H13 | d_specification_copy |
| 14 | H14 | d_consign_code |
| 15 | H15 | d_consign_address |
| 16 | H16 | d_consign_section |
| 17 | H17 | d_consign_tel |
| 18 | H18 | d_carton_code |
| 19～25 | H19～H25 | d_case_mark1～7 |
| 26 | H26 | d_entry_no（エントリーNo） |
| 27 | H27 | d_customer_name |
| 28 | H28 | d_consign_name |
| 29 | H29 | d_unit_price |
| 30 | H30 | d_currency_code |
| 31 | H31 | d_decision_rank |
| 32 | H32 | d_customer_product |
| 33 | H33 | d_product_name |
| 34 | H34 | d_user_code |
| 35 | H35 | d_user_name |
| 36 | H36 | d_supply_code |
| 37 | H37 | d_supply_name |
| 38 | H38 | d_price_code |
| 39 | H39 | d_maker_code |
| 40 | H40 | d_maker_name |
| 41 | H41 | d_charge_name_english |
| 42 | H42 | d_use_name_english |
| 43 | H43 | d_entry_sub_no（枝番） |

- **约束**：头中 **H26（entry_no）、H43（sub_no）** 及明细中的 **detailNo** 会参与 `isInsertEntry` 双重检查；**entry_no** 会经 `getCutEntryNo` 处理（截断/规范化）。  
- **明细部分**：键名为 **"D" + 明细序号**（如 D1, D2, …），值为 **Hashtable**。该 Hashtable 的键为字符串 **"0"～"19"**（对应明细项索引 0～TBL_MAX_DETAIL-1），值为 String。索引与项目对应（`detailItems_[i]`）：

| 索引 | 键 | 项目 | 约束/说明 |
|------|-----|------|------------|
| 0 | "0" | d_detail_no | 明细番号 |
| 1 | "1" | d_order_number | 注文数；**=0 的行不插入** |
| 2 | "2" | d_factory_number | 出荷済数（插入时固定 0） |
| 3 | "3" | d_remain_number | 出荷残数（插入时=注文数） |
| 4 | "4" | （空） | 未使用 |
| 5 | "5" | d_delivery_date | 納期 |
| 6 | "6" | d_forward_date | 乙仲出荷日 |
| 7 | "7" | d_reply_date | 回答日 |
| 8 | "8" | d_reply_code | 回答コード |
| 9 | "9" | d_forward2_date |  |
| 10 | "10" | d_on_board_date |  |
| 11 | "11" | d_plant_code |  |
| 12 | "12" | d_stock_location |  |
| 13～19 | "13"～"19" | d_tax_flg, d_invoice_no, d_collect_month, d_comment_note, d_carrier_out_date, d_carrier_in_date, d_inspect_date |  |
| 20～35 | "20"～"35" | 会社間・OUT-IN 等拡張項目（d_package_number, d_status, d_delivery_priority, d_r3_entry_no, d_deploy_corporate 等） | 部分仅在 iDetail 分支中设置 |

- **日期格式**：索引 17/18（d_carrier_out_date, d_carrier_in_date）若非空且非 `Table.NULL`，期望 **yyyyMMddHHmm**（12 位），会经 `SimpleDateFormat("yyyyMMddHHmm")` 格式化。

### 2.4 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 插入件数（数字字符串）。成功 ≥1；某条失败时可为负数。 |
| ans1 | String | 仅在某条明细插入失败时存在，为失败明细号（detailNo）。 |

---

## 三、orderUpdate

### 3.1 路径

- `POST {baseUrl}/sms01206/orderUpdate`

### 3.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Object (Hashtable) | 是 | 更新用头+明细 | 结构与 orderInsert 的 key 类似：头键 H0～H43，明细键 D1,D2,…；值 String 或 Hashtable（明细行）。用于 updateHash。 |
| **key1** | Object (Hashtable) | 是 | 既存データ（subHash） | 含枝番・既存明细等；键含 `headItems_[SUB_NO_HEAD][1]+SUB_NO_HEAD`（如 "B43"）、`headItems_[ENTRY_NO_HEAD][1]+ENTRY_NO_HEAD`（如 "H26"）等，用于取得 entryNo、subNo、product、customer、price、handle 等。 |
| **outin** | Object (Hashtable) | 否 | OUT-IN 更新用 | 同 orderInsert；可为 null。 |
| **key2** | String | 否 | mente2（出荷済みデータ更新可否） | "1" 时仅 SMS DB 更新等；影响是否调用 R3/SCP。 |
| **corporate** | String | 是 | 法人コード |  |
| **type** | String | 是 | 種別 |  |
| **lang** | String | 是 | 言語 |  |

### 3.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 更新件数。 |
| ans1 | String | 成功时为 `""`。 |

---

## 四、orderDelete

### 4.1 路径

- `POST {baseUrl}/sms01206/orderDelete`

### 4.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | エントリーNo（entryNo） | 用于 deleteOrder 的 KEY（String[] entryNo = new String[]{ key }）。 |
| **corporate** | String | 是 | 法人コード |  |
| **type** | String | 是 | 種別 |  |

### 4.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 削除件数。 |

---

## 五、getOrder

### 5.1 路径

- `POST {baseUrl}/sms01206/getOrder`

### 5.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | エントリーNo（entryNo） | selectOrder 的 WHERE：d_entry_no = key。 |
| **key1** | String | 否 | サブNo（entrySubNo） | 默认建议 "0"。WHERE：d_entry_sub_no = key1。 |
| **lang** | String | 是 | 言語 |  |
| **corporate** | String | 是 | 法人コード |  |
| **type** | String | 是 | 種別 |  |

### 5.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Object (Hashtable) | 头键 H0～H43、明细键 D1,D2,…（值 Hashtable，键 "0"～"19" 等）。 |

---

## 六、getOrderSub

### 6.1 路径

- `POST {baseUrl}/sms01206/getOrderSub`

### 6.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | エントリーNo | WHERE d_entry_no = key。 |
| **key1** | String | 否 | 得意先コード（customer） | 非空时过滤 d_customer_code = key1。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 |  | 过滤 d_corporate_code。 |
| **type** | String | 是 |  |  |

### 6.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Object | 序号 → Hashtable（"0"=subNo, "1"=productCode 等）。 |
| ans1 | Array (String[]) | OUT-IN 用 outinCustomer（长度 13）。 |

---

## 七、getEntryNo

### 7.1 路径

- `POST {baseUrl}/sms01206/getEntryNo`

### 7.2 请求参数

- Sms01206 侧 **无参数**（RMI 为无参）。WebApi 若需统一请求体，可传空对象或 `lang`/`corporate`（由 WebApi 层自用）。

### 7.3 响应

- Sms01206 返回 **String**（エントリー番号）。若 WebApi 统一为 `data` 对象，则应为 `data.ans` = 该字符串。

---

## 八、getPrice

### 8.1 路径

- `POST {baseUrl}/sms01206/getPrice`

### 8.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 検索キー | **key[0]** = 得意先コード、**key[2]** = 機種コード（源码 keys[2]=getKeys[0], keys[1]=getKeys[2]）。长度至少 3。 |
| **key2** | String | 否 | ERP 用フラグ | "ERP" 时走 ERP 検索分支。 |
| **lang** | String | 是 |  |  |
| **type** | String | 是 |  |  |
| **corporate** | String | 是 |  | 与 key 一起组成 getMaster(PRICE) 的 keys。 |

### 8.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | 长度 14+1；含得意先仕様書№、他社向販売、販価、通貨、相手機種名等（索引 1～13 等）。 |

---

## 九、getCharges

### 9.1 路径

- `POST {baseUrl}/sms01206/getCharges`

### 9.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 否 | 担当者コードのあいまい検索 | 源码中 keys[1] = key + "%"，可空。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 |  |  |

### 9.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 担当者コード・名称等（通常 [0]=null 行、[1]=空行、[2] 以降=数据）。 |

---

## 十、getCustomers / getProducts / getConsigns / getSupplys / getUses / getCustomerProducts

以下 6 个接口均为マスター検索，格式统一为路径、请求参数、响应 data。

### 10.1 getCustomers

- **路径**：`POST {baseUrl}/sms01206/getCustomers`
- **请求参数**：**key**（String，検索キー；内部拼为 key+"%"）、**lang**（String）、**corporate**（String）。keys[0]=corporate, keys[1]=key+"%"；TYPE=CUSTOMER_LIST_1。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 3 列：得意先コード、得意先名、担当者コード。

### 10.2 getProducts

- **路径**：`POST {baseUrl}/sms01206/getProducts`
- **请求参数**：**key**（String，機種検索キー；首字符 "%" 时用 PRODUCT_LIST_3 否则 PRODUCT）、**key1**（String，可选；"ERP" 时在通常检索 0 件时走 ERP 检索）、**lang**、**type**、**corporate**。機種末尾 "'" 时强制 ERP。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：機種コード、機種名。

### 10.3 getConsigns

- **路径**：`POST {baseUrl}/sms01206/getConsigns`
- **请求参数**：**key**（String，得意先コード customer）、**lang**、**corporate**。keys[0]=corporate, keys[1]=customer；TYPE=CUSTOMER_CONSIGN。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 5 列：発送先コード、得意先１、住所１、部署、ＴＥＬ。

### 10.4 getSupplys

- **路径**：`POST {baseUrl}/sms01206/getSupplys`
- **请求参数**：**key**（String，得意先コード customer）、**lang**、**corporate**。keys[0]=corporate, keys[1]=customer；TYPE=CUSTOMER_SUPPLY。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：納入先コード、得意先１。

### 10.5 getUses

- **路径**：`POST {baseUrl}/sms01206/getUses`
- **请求参数**：**key**（String，用途検索キー；内部拼为 key+"%"）、**lang**（String）。keys[0]=key+"%"；TYPE=USE_LIST_1；未使用 corporate。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：用途コード、用途名。

### 10.6 getCustomerProducts

- **路径**：`POST {baseUrl}/sms01206/getCustomerProducts`
- **请求参数**：**key**（String，得意先コード）、**lang**、**type**、**corporate**。keys[0]=corporate, keys[1]=key；TYPE=CUSTOMER_PRODUCT。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：機種コード、機種名（s[2]）。

---

## 十一、getVari

### 11.1 路径

- `POST {baseUrl}/sms01206/getVari`

### 11.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **lang** | String | 是 | 言語 |  |
| **type** | String | 否 | 区分種別 | HANDLE→HANDLE_DIVISION、ROUTE→ROUTE_DIVISION、VOUCHER→VOUCHER_DIVISION、ATTACHMENT→ATTACHMENT_DIVISION、DELIVERY→DELIVERY_DIVISION1、CARTON→CARTON_DIVISION、CURRENCY、REPLY→REPLY_DIVISION。不传则 getMaster 仅 LANG。 |

### 11.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 第 1 行为空行 [ "", "" ]，其后每行 2 列：コード、名称（s[2], s[3]）。 |

---

## 十二、getNpmEntry

### 12.1 路径

- `POST {baseUrl}/sms01206/getNpmEntry`

### 12.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 機展書コード（code） | selectNewProductName 的 KEY。 |
| **lang** | String | 否 |  | hash 未使用；若下游需要可传。 |
| **type** | String | 否 |  | hash 未使用。 |

### 12.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 機展書名（selectNewProductName 返回的 ANS 经 toValue）。 |

---

## 十三、getMakerMaster / getPriceCodeMaster

### 13.1 getMakerMaster

- **路径**：`POST {baseUrl}/sms01206/getMakerMaster`
- **请求参数**：**lang**（String）、**corporate**（String）。无 key；内部 keys[0]=corporate，TYPE=SECOND_USER_LIST（メーカー＝第二ユーザーリスト）。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：メーカーコード、名称（经 keyChange 排序）。

### 13.2 getPriceCodeMaster

- **路径**：`POST {baseUrl}/sms01206/getPriceCodeMaster`
- **请求参数**：**lang**（String）、**corporate**（String）。无 key；内部 keys[0]=corporate，TYPE=PRICE_DIVISION（販価決定区分）。
- **响应 data**：**ans** = String[][]，第 1 行空行，其后每行 2 列：販価決定区分コード、名称（经 sortVari 排序）。

---

## 十四、getDefaultConsigns

### 14.1 路径

- `POST {baseUrl}/sms01206/getDefaultConsigns`

### 14.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 得意先コード（customer） | keys[1]=customer。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 | 法人コード | keys[0]=corporate；TYPE=CUSTOMER_CONDITION。 |

### 14.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | 长度 5；得意先条件による初期値（扱い区分/配送先/納入先等）。 |

---

## 十五、judgmentERP

### 15.1 路径

- `POST {baseUrl}/sms01206/judgmentERP`

### 15.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | ERP 連携機種判定用キー（機種コード等） | 传 remoteObject_.judgmentERP(prmHash)。 |
| **lang** | String | 否 |  | prmHash 未使用。 |
| **corporate** | String | 是 | 法人コード | 传 prmHash。 |

### 15.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | ERP 連携機種判定結果 |
| ans1 | Boolean | FLAG（null 时视为 false） |

---

## 十六、isHoliday

### 16.1 路径

- `POST {baseUrl}/sms01206/isHoliday`

### 16.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 日付（DATE） | 传给 judgmentHoliday；用于休日判定。 |
| **corporate** | String | 是 |  |  |

### 16.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | "H"=休日、"W"=工作日。 |

---

## 十七、isUnitPriceMaster

### 17.1 路径

- `POST {baseUrl}/sms01206/isUnitPriceMaster`

### 17.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 販価マスター検索キー | 与 getPrice 同様、corporate/機種/得意先等；用于 getMaster(PRICE)。 |
| **key2** | String | 否 | "ERP" 时走 ERP 検索 |  |
| **lang** | String | 是 |  |  |
| **type** | String | 是 |  |  |
| **corporate** | String | 是 |  |  |

### 17.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | 機種・販価マスター存在有無。 |

---

## 十八、getTransportList

### 18.1 路径

- `POST {baseUrl}/sms01206/getTransportList`

### 18.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 否 | 法人別輸送手段用 keys | 传 getMaster(tbl)；TYPE=TRANSPORT_LIST。可为 null。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 否 |  | tbl 仅放 KEY、LANG、TYPE，未使用 corporate。 |

### 18.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 第 1 行空行，其后每行 4 列（輸送手段マスター）。 |

---

## 十九、isDuplicationOrder

### 19.1 路径

- `POST {baseUrl}/sms01206/isDuplicationOrder`

### 19.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key1** | String | 否 | OrderNo 重複チェック用 |  |
| **key2** | String | 否 |  |  |
| **key3** | String | 否 |  |  |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 |  |  |

### 19.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | 重複有無 |
| ans1 | Boolean |  |

---

## 二十、getTurnoverDate

### 20.1 路径

- `POST {baseUrl}/sms01206/getTurnoverDate`

### 20.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 売上日取得用キー | 至少 3 元素；源码 key[0], key[1], key[2] 使用。 |
| **lang** | String | 否 |  |  |
| **corporate** | String | 否 |  |  |
| **type** | String | 否 |  |  |

### 20.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | 长度 2；売上日等（data[0], data[1]）。 |

---

## 二十一、getExcelData

### 21.1 路径

- `POST {baseUrl}/sms01206/getExcelData`

### 21.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **type** | String | 是 | 種別 |  |
| **lang** | String | 是 |  |  |
| **key** | Array (String[]) | 是 | **key[0]**=entry_no、**key[1]**=entry_sub_no | WHERE d_entry_no = key[0] AND d_entry_sub_no = key[1]。 |
| **keys** | Array (String[]) | 是 | 取得項目名の配列 | 如 d_entry_no, d_customer_name, d_product_code 等；用于 select 的 VALUES 与结果列顺序。 |

### 21.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (Vector of String[]) | 每行为 keys 顺序的 String[]。 |

---

## 二十二、updatePriceCode

### 22.1 路径

- `POST {baseUrl}/sms01206/updatePriceCode`

### 22.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | key[0]=entry_no, key[1]=entry_sub_no, key[2]=detail_no, key[3]=price_code（販価＝０で R3 開放時、販価決定区分 2→3 更新） |  |
| **corporate** | String | 是 | 法人コード |  |
| **type** | String | 是 | 種別 | updateSms 用。 |

### 22.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 更新結果コード等。 |

---

## 二十三、judgementCustomerConfidence

### 23.1 路径

- `POST {baseUrl}/sms01206/judgementCustomerConfidence`

### 23.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 否 | 得意先与信判定用（customer 等） | 传 judgmentCustomerConfidence(prmHash)。 |
| **key1** | Array (String[][]) | 否 | 追加 VALUES；length=0 时响应带 outValue（ans1） |  |
| **lang** | String | 否 |  | prmHash 未使用。 |
| **corporate** | String | 是 | 法人コード |  |

### 23.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 与信判定結果（data）。 |
| ans1 | Array (String[]) | key1 长度为 0 时返回 VALUES（outValue）；否则不设。 |

---

## 二十四、getMultiRate / getActiveDate

### 24.1 getMultiRate

- **路径**：`POST {baseUrl}/sms01206/getMultiRate`
- **请求参数**：**lang**（String）、**key**（Array String[]，长度至少 5）。key[0]=toCurrency, key[1]=currency, key[2]=yyyymm, key[4]=rateType（"M" 时按会計年度/期）；key[3] 由内部从 key[2] 算出。未使用 corporate。
- **响应 data**：**ans** = String（換算レート，默认 "1"；异常时保持 "1"）。

### 24.2 getActiveDate

- **路径**：`POST {baseUrl}/sms01206/getActiveDate`
- **请求参数**：**key**（Array String[]，通常 key[0]=日付 yyyymmdd、key[1]=オフセット等；DMM 时 key[0]/key[1] 使用）、**key1**（String，可选；"ON" 时走 DMM 稼動日除いて遡る）、**corporate**（String）。
- **响应 data**：**ans** = String（指定日から手番遡った稼動日）。

---

## 二十五、getChargeCustomers

### 25.1 路径

- `POST {baseUrl}/sms01206/getChargeCustomers`

### 25.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 担当者コード（chargeCode） | keys[1]=key；TYPE=CHARGE_CUSTOMER。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 | 法人コード | keys[0]=corporate。 |

### 25.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 第 1 行空行，其后每行 3 列：得意先コード、得意先名称、担当者コード。 |

---

## 二十六、getCustomerForwardReadTime

### 26.1 路径

- `POST {baseUrl}/sms01206/getCustomerForwardReadTime`

### 26.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **corporate** | String | 是 | 法人コード（wcorp） | 从 cPath_  CSV 筛选 corp.equals(wcorp) 的行。 |

### 26.3 响应 data

- 方法直接返回 Hashtable（**非** ansHash.put(ANS,...)）。即 **data** = 该 Hashtable；键为 `cust+"/"+corp2+"/"+plant+"/"+stock`，值为 String[6]（corp, cust, corp2, plant, stock, num）。若 WebApi 统一包装为 `{ "ans": ... }`，则 ans = 该 Hashtable。

---

## 二十七、getConporison

### 27.1 路径

- `POST {baseUrl}/sms01206/getConporison`

### 27.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 機種マスター検索キー（keys） | getMaster 用；Utility.getKeyString(keys) 取结果行；s[7]=製品区分。 |
| **lang** | String | 是 |  | TYPE=PRODUCT。 |
| **corporate** | String | 否 |  | 源码未使用 corporate。 |

### 27.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 製品区分（s[7] 经 toValue）。 |

---

## 二十八、getRankDialogMaster

### 28.1 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| **key** | String | 否 | ランク検索用 |
| **key1** | String | 否 |  |
| **lang** | String | 是 |  |
| **corporate** | String | 是 |  |

### 28.2 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 4 列：桁、ナンバー、値、見出し。 |

---

## 二十九、moveR3

### 29.1 路径

- `POST {baseUrl}/sms01206/moveR3`

### 29.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **lang** | String | 是 | 言語 |  |
| **type** | String | 是 | 種別 |  |
| **corporate** | String | 是 | 法人コード |  |
| **key1** | Array (Vector) | 是 | R3 移行対象エントリのリスト | 各要素は String[]：entry[0]=entry_no, entry[1]=entry_sub_no, entry[2]=detail_no。getActualDetail(entry, lang, corporate, type) で全項目取得し、moveOrderToR3 に渡す。 |

### 29.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 処理結果（成功時は corporate、エラー時は ERROR 等）。 |
| ans1 | String | メッセージ等（value）。 |

---

## 三十、getNpmLine

### 30.1 路径

- `POST {baseUrl}/sms01206/getNpmLine`

### 30.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 機展書コード（code） | selectNewProductName の KEY。取得した NPMLine00 の specification を返す。 |

### 30.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 機展書コメント情報（specification）。 |

---

## 三十一、getKoOrder

### 31.1 路径

- `POST {baseUrl}/sms01206/getKoOrder`

### 31.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 親エントリ指定 | oya[0]=package_entry_no, oya[1]=package_entry_sub_no, oya[2]=package_detail_no。WHERE 条件に使用。 |
| **key1** | Array (String[]) | 是 | 取得項目名の配列（koItems） | selectOrder の VALUES と結果行の列順に使用。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 |  |  |
| **type** | String | 是 |  |  |

### 31.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (Vector of String[]) | 同梱品子オーダー行の配列。各行は key1 の項目順の String[]。 |

---

## 三十二、isDeployPriceMaster

### 32.1 路径

- `POST {baseUrl}/sms01206/isDeployPriceMaster`

### 32.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 継続プラント販価マスター検索キー | 長さ至少 5。key[0]～key[4] がログで使用。judgmentDeployPriceMaster(tbl) に渡す。 |

### 32.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | 継続プラント販価マスター存在有無。例外時は false。 |
| ans1 | Array (String[][]) | 取得した VALUES（val）。 |

---

## 三十三、getDeployEntry

### 33.1 路径

- `POST {baseUrl}/sms01206/getDeployEntry`

### 33.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Object (Hashtable) | 是 | キーごとの検索条件 | 各 key に対応する value は String[]（dt）。各 dt で getDeployEntry(tbl) を呼び、結果の String[][] を outHash に key で格納。 |
| **type** | String | 是 | 種別 | SMSServerIfc.TYPE に渡す。 |

### 33.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Object (Hashtable) | key → String[][]（各キーに対する getDeployEntry の結果）。 |
| ans1 | String | 総件数（cnt の文字列）。 |

---

## 三十四、getDistributeDeploy

### 34.1 路径

- `POST {baseUrl}/sms01206/getDistributeDeploy`

### 34.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 派生元指定 | moto[0]=entry_no, moto[1]=entry_sub_no, moto[2]=detail_no（派生元）。内部で getOrder(prmHash) を呼ぶため、lang/corporate/type も prmHash に設定される。 |
| **key2** | String | 是 | テーブル表示最大件数（tblMaxCnt） | 追加データの最終枝番と比較し、last > tblMaxCnt1 の場合は beforeHash を空にする。 |
| **lang** | String | 否 | getOrder 用 |  |
| **corporate** | String | 否 | getOrder 用 |  |
| **type** | String | 否 | getOrder 用 |  |

### 34.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Object (Hashtable) | getOrder の結果（beforeHash）。追加時の次 detail_no が last 以下でない場合は空。 |
| ans1 | String | 次に挿入する detail_no（last の次）。上記で空の場合は未設定。 |

---

## 三十五、distributeDeploy

### 35.1 路径

- `POST {baseUrl}/sms01206/distributeDeploy`

### 35.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 派生元 | moto[0]=entry_no, moto[1]=sub_no, moto[2]=detail_no, moto[3]=法人。 |
| **key1** | Object (Hashtable) | 是 | 追加するオーダーデータ（beforeHash） | orderInsert の key として渡す。 |
| **key2** | String | 否 | status | 非空时调用 moveR3。 |
| **key3** | String | 否 | next（新 detail_no） |  |
| **key4** | String | 否 | dpCorporate（会社間用法人） | orderInsert の key1 として渡す。 |
| **corporate** | String | 否 | orderInsert では moto[3] を使用 |  |
| **type** | String | 是 |  |  |
| **lang** | String | 是 |  |  |

### 35.3 响应 data

- orderInsert の戻り値をそのまま返す。ans（件数）、ans1（失敗時は明细号等）。

---

## 三十六、updateSmsStatus

### 36.1 路径

- `POST {baseUrl}/sms01206/updateSmsStatus`

### 36.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **type** | String | 是 | 種別 |  |
| **key1** | Array (Vector) | 是 | 更新対象エントリのリスト | 各要素は String[]：entry[0]=entry_no, entry[1]=entry_sub_no, entry[2]=detail_no。d_status=ST_Q、d_reply_code="1" で updateSms。 |

### 36.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 更新件数（okCnt）。 |

---

## 三十七、updateSmsStatusClear

### 37.1 路径

- `POST {baseUrl}/sms01206/updateSmsStatusClear`

### 37.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **type** | String | 是 | 種別 |  |
| **key1** | Array (Vector) | 是 | 同梱品子エントリのリスト | 各要素は String[]（ko）。KO_ENTORY_NO, KO_ENTORY_SUB_NO, KO_DETAIL_NO 等を使用。d_reply_code="0", d_status=Table.NULL で updateSms。 |

### 37.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 更新件数（cnt1）。 |
| ans1 | String | 成功時は ""。 |

---

## 三十八、updateKoOrder

### 38.1 路径

- `POST {baseUrl}/sms01206/updateKoOrder`

### 38.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **corporate** | String | 是 | 法人コード |  |
| **type** | String | 是 | 種別 |  |
| **lang** | String | 是 | 言語 |  |
| **key1** | Array (Vector) | 是 | 同梱品子オーダーリスト（koOrders） | 各要素は String[]（ko）。koItems_ のインデックスで KO_ENTORY_NO, KO_ENTORY_SUB_NO, KO_DETAIL_NO, KO_STATUS, KO_FACTORY, KO_REMAIN 等を参照。 |
| **key2** | String | 否 | mente2（出荷済みデータ更新可否） | "1" の時は SMS のみ更新等。 |

### 38.3 响应 data

- updateSmsKoOrder の戻り値。ans = kekkaTbl（Hashtable：entry キー → 更新結果メッセージ）。

---

## 三十九、getOyaOrder

### 39.1 路径

- `POST {baseUrl}/sms01206/getOyaOrder`

### 39.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 親エントリ指定 | oya[0], oya[1], oya[2]。WHERE d_package_entry_no AND d_package_entry_sub_no AND d_package_detail_no。 |
| **lang** | String | 是 |  |  |
| **corporate** | String | 是 |  |  |
| **type** | String | 是 |  |  |

### 39.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | 親オーダーの packegeItems_ 項目順の配列（長さ packegeItems_.length）。 |

---

## 四十、judgmentStopProduct

### 40.1 路径

- `POST {baseUrl}/sms01206/judgmentStopProduct`

### 40.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 廃機種判定用キー（機種コード等） |  |
| **corporate** | String | 是 | 法人コード |  |

### 40.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 判定結果（hantei）。 |
| ans2 | String | kinshi（2017/02/21 追加）。 |

---

## 四十一、getPSSPeriodProduct

### 41.1 路径

- `POST {baseUrl}/sms01206/getPSSPeriodProduct`

### 41.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 機種コード | getPSSPeriodProduct(prmHash) の KEY。 |
| **corporate** | String | 是 | 法人コード |  |

### 41.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | 長さ 6 のデータ。 |

---

## 四十二、getQSeihinHash

### 42.1 路径

- `POST {baseUrl}/sms01206/getQSeihinHash`

### 42.2 请求参数

- Sms01206 側で **引数なし**（Hashtable を渡しても使用しない）。RMI は `getQSeihinHash()`。WebApi では空オブジェクトでよい。

### 42.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Object (Hashtable) | ビジネスユニットから製品区分を取得した qSeihinHash_（キー・値は実装依存）。 |

---

## 四十三、reloadCustomerConfidence

### 43.1 路径

- `POST {baseUrl}/sms01206/reloadCustomerConfidence`

### 43.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 得意先与信再取得用キー |  |
| **corporate** | String | 是 | 法人コード |  |

### 43.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 再取得した判定結果（data）。 |

---

## 四十四、getProductPlant

### 44.1 路径

- `POST {baseUrl}/sms01206/getProductPlant`

### 44.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 機種コード等 | getProductPlant(prmHash) の KEY。 |
| **corporate** | String | 是 | 法人コード |  |

### 44.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[][]) | 機種別自動設定プラント一覧。 |

---

## 四十五、getNpmProductDivision

### 45.1 路径

- `POST {baseUrl}/sms01206/getNpmProductDivision`

### 45.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | 機展書コード（code） | selectNewProductName の KEY。 |
| **key1** | Array (String[]) | 是 | 取得項目名の配列（items） | NPMLine00 から取得する項目。 |
| **key2** | Array (String[]) | 否 | 格納用 data 配列 | 長さは key1.length 以上。 |
| **lang** | String | 否 |  |  |
| **type** | String | 否 |  |  |

### 45.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | key1 の項目順の値配列。 |

---

## 四十六、getOutInData

### 46.1 路径

- `POST {baseUrl}/sms01206/getOutInData`

### 46.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | 検索キー | getOutInData(tbl) の KEY。 |

### 46.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Array (String[]) | OUT-IN 連携オーダー情報。 |

---

## 四十七、updateOutinData

### 47.1 路径

- `POST {baseUrl}/sms01206/updateOutinData`

### 47.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **outin** | Object (Hashtable) | 否 | OUT-IN 更新用。key=detailNo, value=oiData 等 | size>0 の時 updateSmsOi、否则 updateSms。 |
| **type** | String | 是 | 種別 |  |
| **key** | Array (String[][]) | 是 | 更新項目 | values[i][0]=項目名、values[i][1]=値。OrderLine00 に set して更新。 |

### 47.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 更新件数（okCnt）。 |

---

## 四十八、getForwadingListHtml

### 48.1 路径

- `POST {baseUrl}/sms01206/getForwadingListHtml`

### 48.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key1** | String | 是 | エントリーNo（entryNo） | ファイル名の一部（entryNo_time.html）。 |
| **key2** | String | 是 | 時刻（yyyyMMddHHmmsss） | ファイル名の一部。 |
| **key3** | String | 是 | JSP 起動 URL | 出荷依頼表 HTML 生成用。 |

### 48.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 生成した HTML の URL（workURL）。 |

---

## 四十九、setFactoryResults

### 49.1 路径

- `POST {baseUrl}/sms01206/setFactoryResults`

### 49.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | val[0]=EntryNo, val[1]=EntrySubNo, val[2]=detailNo, val[3]=数量 | Pipe で "63 " + val[0] + " " + val[1] + " " + val[2] + " " + val[3] を送信。 |

### 49.3 响应 data

- 実装では **new Hashtable()** を返しており、**ans 等は設定されない**（空の data）。

---

## 五十、getMaxSalesUpdate

### 50.1 路径

- `POST {baseUrl}/sms01206/getMaxSalesUpdate`

### 50.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | String | 是 | エントリーNo（val） | SQL: select max(sales_update) from order_tbl where entry_no = 'val'。 |

### 50.3 响应 data

- **executeQuery の戻り値をそのまま返す**。通常は ans に Vector 等が入る（クエリ結果の形式に依存）。

---

## 五十一、judgementNNSN

### 51.1 路径

- `POST {baseUrl}/sms01206/judgementNNSN`

### 51.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **corporate** | String | 是 | 法人コード（corp） | NNSN 判定用。isItemAri(NNSN, corp) で true/false。 |

### 51.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | NNSN 該当可否。例外時は空 Hashtable。 |

---

## 五十二、isDuplicationOrder2

### 52.1 路径

- `POST {baseUrl}/sms01206/isDuplicationOrder2`

### 52.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **corporate** | String | 是 | 法人コード |  |
| **key1** | String | 是 | エントリーNo（除外する entry_no） | SQL で entry_no != key1 で検索。 |
| **key2** | String | 是 | 機種コード（product） | 同一 product_code、order_number!=0、decision_rank='1' の重複チェック。 |

### 52.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Boolean | 重複あり true / なし false。例外時は空 Hashtable。 |

---

## 五十三、getProductFromCustomerProduct

### 53.1 路径

- `POST {baseUrl}/sms01206/getProductFromCustomerProduct`

### 53.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **key** | Array (String[]) | 是 | val[0]=corporate_code, val[1]=customer_code, val[2]=customer_product | price_master から max(product_code) を取得。 |

### 53.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | String | 取得した product_code（pro[0]）。例外時は空 Hashtable。 |

---

## 五十四、getInvoiceNoCount

### 54.1 路径

- `POST {baseUrl}/sms01206/getInvoiceNoCount`

### 54.2 请求参数

| 参数名 | 类型 | 必填 | 说明 | 约束与用法 |
|--------|------|------|------|------------|
| **corporate** | String | 是 | 法人コード |  |
| **key1** | String | 是 | インボイスNo（invoiceNo） |  |
| **key2** | String | 是 | 除外するエントリーNo（entryNo） | SQL: entry_no != key2, remain_number != 0。 |

### 54.3 响应 data

| 键 | 类型 | 说明 |
|----|------|------|
| ans | Number (Integer) | 同一 invoice_no の該当件数。例外時は空 Hashtable。 |

---

## 五十五、Sms01206 源码位置

| 说明 | 路径 |
|------|------|
| RMI 接口与常量 | Sms01206/OrderEntryNo_i.java |
| RMI 实现（头・明细定义、各方法 prm/ansHash） | Sms01206/OrderEntryNo_s.java |

---

*本报告全部由 Sms01206 源码反推，未使用 SmsWeb。具体类型与边界约束以 OrderEntryNo_s 内各方法的 cast 与分支为准。*
