# OrderInsert Mock 填充后仍触发 R3 分支原因分析

## 以 OrderEntryNo_240522 为基准的深入分析

---

## 一、问题现象

即使 Mock 已填充 D01.10（売上日）和 D01.19（検収日），以下日志仍出现：

```
call orderInsert
 orderInsert  syoriHantei 0= [insertButton]
insert= [26227ABZ][0][1]
getTurnoverMonth= [202603]
isInsertEntry= [26227ABZ01]
CALL getCMIReplyDate [X][X080][__SMS_TABLE_NULL__][TMDFY03DA2]
```

---

## 二、OrderEntryNo_240522 与 OrderEntryNo_s 的 data 映射关系

### 2.1 明细字段常量对照（detailItems_）

| index | 常量名 | 含义 | D01 key |
|-------|--------|------|---------|
| 5 | DELIVERY_DETAIL | 納期 | "5" |
| 6 | FORWARD_DETAIL | 出荷希望日 | "6" |
| 7 | REPLY_DETAIL | 回答日 | "7" |
| 9 | FORWARD2_DETAIL | 乙仲出荷日 | "9" |
| **10** | **ON_BOARD_DETAIL** | **売上日** | **"10"** |
| **19** | **INSPECT_DETAIL** | **検収日** | **"19"** |

R3 分支触发条件（SMSServer2_s.insertOrder 约 334–416 行）：

- `isJapan(ccode)`
- `handle_code == "1"`
- **`isNull2(d_on_board_date)`** 或 **`isNull2(d_inspect_date)`**

`isNull2` 定义（约 4659–4664 行）：

```java
protected boolean isNull2(String val){
    if(null == val) return true;
    if(val.length() == 0) return true;
    if("__SMS_TABLE_NULL__".equals(val)) return true;  // 关键：空占位符也被视为 null
    return false;
}
```

---

## 三、日志含义说明

### 3.1 getTurnoverMonth / getCMIReplyDate ≠ 一定走 R3 阻塞路径

| 日志 | 所在位置 | 含义 |
|------|----------|------|
| `getTurnoverMonth= [202603]` | OrderEntryNo_s 约 1566 行 | 计算売上年月，基于 date2（納期/出荷希望日/回答日/乙仲出荷日/売上日） |
| `CALL getCMIReplyDate` | OrderEntryNo_s 约 1594–1597 行 | NSCM 会社間対応，回答日为空时尝试补回回答日 |
| `isInsertEntry=` | OrderEntryNo_s 约 1579 行 | 防止重复登録的检查 |

这些日志在每次 orderInsert 都会输出，与是否进入 SMSServer2.insertOrder 的 R3 路径（getInspectDate）无直接对应关系。  
真正会长时间阻塞的是：`getInspectDate → getTurnoverDate → getCustomerReadTime → R3Server.getERPMasterCustomers → Pipe`。

### 3.2 判断是否走了 R3 阻塞路径

应检查是否出现 SMSServer2 侧的日志，例如：

- `YasuCheck210320-1` / `YasuCheck210320-2` / `YasuCheck210320-3`
- `insertOrder 売上日自動設定` / `insertOrder 検収日自動設定`

若同时有 2.5 分钟左右阻塞，则可确认走了 R3 路径。

---

## 四、Mock 填充与请求路径

### 4.1 Mock 逻辑（WebSms01206ServiceMock.applyOrderInsertDummyAvoidR3）

- 条件：`corporate == "X"` 且 `key.B5 == "1"`（扱い区分）
- 对每个 D01、D02…：若納期（detail."5"）为 8 位，且 detail."10" / detail."19" 非 8 位，则用納期填充 detail."10" 和 detail."19"（约 42–44 行）

### 4.2 请求路径

```
前端 → SmsWeb /api/orderInsert
  → WebSms01206ServiceImpl.orderInsert (Mock 在此修改 input)
  → passThrough → ImartApiClient.post("orderInsert", input)
  → HTTP POST 到 SmsWebApi
  → Sms01206Controller.orderInsert → convertJsonToHashtable
  → RMI OrderEntryNo_s.orderInsert → remoteObject_.insertOrder(hash)
  → SMSServer2.insertOrder
```

若请求不经过 SmsWeb，而是直接访问 SmsWebApi，则 Mock 不会执行，R3 分支仍会被触发。

---

## 五、可能导致 Mock 填充无效的原因

### 5.1 请求未经过 SmsWeb（最优先排查）

- 前端 API baseUrl 直接指向 SmsWebApi
- 测试时直接调用 `/sms01206/orderInsert` 而非 SmsWeb 的 `/websms01206/maintenance/api/orderInsert`

**建议**：确认前端使用的 orderInsert 接口 URL 是否经过 SmsWeb。

### 5.2 Mock 条件不满足

- `corporate` 必须为 `"X"`（区分大小写）
- `key.B5` 必须为 `"1"`
- 納期（detail."5"）必须为 8 位数字

任一不满足时，Mock 不会修改任何明细。

### 5.3 `__SMS_TABLE_NULL__` 的语义

`__SMS_TABLE_NULL__`（Table.NULL）在 `isNull2` 中按“空”处理。若 `d_on_board_date` 或 `d_inspect_date` 被存为该字符串，R3 分支仍会触发。  
Mock 应写入的是纳期日期（如 `"20260302"`），而不是 `Table.NULL`。需确认 Mock 填充后的 JSON 中 D01."10"/"19" 为正常 8 位日期字符串。

### 5.4 OrderEntryNo_s 中的迭代与 key 类型

OrderEntryNo_s 中（约 1479–1486 行）：

```java
Enumeration key1s = new EnumerationSorter(...).sort(detailHash.keys());
while (key1s.hasMoreElements()) {
    String key1 = (String)key1s.nextElement();  // 要求 key 为 String
    int iDetail = new Integer(key1).intValue();
    if (iDetail < TBL_MAX_DETAIL){
        String detail = (String)detailHash.get(key1);
        ...
        line[0].set(detailItems_[iDetail], detail);
```

若 JSON 解析后 D01 的 key 变成 Integer，`(String)key1s.nextElement()` 会抛出 ClassCastException，index 10/19 可能未被正确设置。  
SmsWebApi 的 `convertMapToHashtable` 对 JSON 对象使用 `Map<String, Object>`，key 应为 String；若中间环节改成 Integer key，则可能出错。需要确认 SmsWebApi 解析后的 D01 结构。

### 5.5 前端 buildKey 与 D01 结构

`sms01206-order-key.js` 的 buildKey 已包含 `"10"` 和 `"19"`：

```javascript
'10': d.onBoardDate || '',
'19': d.inspectDate || ''
```

未填时为空字符串。Mock 会在满足条件时将其替换为纳期。

---

## 六、OrderEntryNo_240522 的参照点

### 6.1 参数结构

OrderEntryNo_240522 使用 `beforeHash_` 作为 `prmHash.put(OrderEntryNo_i.KEY, beforeHash_)`，结构与 Web buildKey 一致：H 系列 + B 系列 + D01/D02…，Dxx 内部为 `"1"`–`"19"` 等 key。

### 6.2 売上日・検収日的来源差异

- **Swing（OrderEntryNo_240522）**：`setOnBoardDate` 等在日期 blur 时调用 `getTurnoverDate`，売上日・検収日由 UI 或 R3 计算后写入明细，插入前可能已具备值。
- **Web**：若未在 blur 时调用 getTurnoverDate，D01."10"/"19" 可能为空，依赖 Mock 在 orderInsert 前填充。

因此，需确认 Web 端的“納期/出荷希望日 blur 时是否同步更新売上日・検収日”，以及 Mock 是否必然在此前执行。

---

## 七、验证建议

1. **确认请求是否经过 SmsWeb**
   - 检查前端 baseUrl / orderInsert 的完整 URL
   - 在 `WebSms01206ServiceImpl.orderInsert` 入口增加日志，确认该方法是否被调用

2. **确认 Mock 修改后的 payload**
   - 在 Mock 执行后、passThrough 前，打印 `input`（含 key.D01）
   - 确认 D01."10"、D01."19" 已被填充为 8 位日期（如 `"20260302"`）

3. **确认 SmsWebApi 收到的 JSON**
   - 在 Sms01206Controller.orderInsert 中打印 `input` 字符串
   - 确认 JSON 中 `key.D01["10"]`、`key.D01["19"]` 为 8 位日期

4. **确认 SMSServer2 侧的 R3 路径**
   - 检查是否出现 `YasuCheck210320-*`、`insertOrder 売上日自動設定`、`insertOrder 検収日自動設定`
   - 若出现，再检查传入的 `line[0]` 中 `d_on_board_date`、`d_inspect_date` 的值

---

## 八、结论

- `getTurnoverMonth` 与 `getCMIReplyDate` 的日志**不代表**一定进入 R3 阻塞路径；实际阻塞在 SMSServer2.insertOrder 的 `getInspectDate` 调用链。
- 若 Mock 正确执行且请求走 SmsWeb，应能避免因 `d_on_board_date` / `d_inspect_date` 为空而进入 R3 路径。
- 优先排查：**请求是否绕过了 SmsWeb**；其次确认 Mock 条件、JSON 解析与 key 类型，以及 `__SMS_TABLE_NULL__` 的混用。
