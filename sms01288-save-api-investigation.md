# Sms01288 save 接口调查文档

## 一、概述

本文档详细记录 `/sms01288/save` API 的调用流程、数据转换逻辑，以及 RMI 调用前的完整参数结构。调查范围从 HTTP 请求入口到 `Sms01288_s.save(Hashtable)` 调用前为止。

---

## 二、调用链路

```
HTTP POST /sms01288/save (JSON body)
    ↓
Sms01288Controller.save(@Body String input)
    ↓
SmsWebApiUtil.convertJsonToHashtable(input, targetKeys)
    ↓
remoteObject_.save(prm)  [Sms01288_s.save]
```

**可选路径（经 SmsWeb 转发）**：

```
Web 前端 / Postman
    ↓
WebSms01288Controller.save(@RequestBody SaveRequestDto)
    ↓
WebSms01288ServiceImpl.save() → normalizeSaveRequest()
    ↓
SmsApiClient.postForMap() → HTTP POST
    ↓
SmsWebApi (Sms01288Controller.save)
```

---

## 三、相关源码位置

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| API 入口 | `SmsWebApi/.../Sms01288Controller.java` | `@Path(PATH_SAVE)`，接收 JSON |
| JSON→Hashtable | `SmsWebApi/.../SmsWebApiUtil.java` | `convertJsonToHashtable(input, targetKeys)` |
| RMI 实现 | `java applet/sms01288/Sms01288_s.java` | `save(Hashtable prmHash)` |
| RMI 接口 | `java applet/sms01288/Sms01288_i.java` | 常量定义、方法签名 |
| 请求规范化 | `SmsWeb/.../WebSms01288ServiceImpl.java` | `normalizeSaveRequest()` |
| 请求 DTO | `SmsWeb/.../dto/SaveRequestDto.java` | 字段定义 |

---

## 四、API 请求格式

### 4.1 HTTP 请求

```
method:      POST
uri:         /sms01288/save
contentType: application/json
```

### 4.2 请求体 JSON 结构（示例）

```json
{
  "corporate": "X",
  "key4": "26220AAT",
  "type": "order",
  "lang": "ja",
  "key2": null,
  "key": {
    "key1": {
      "26220AAU": [
        ["d_product_code", "70RSCCD020"],
        ["d_product_name", "SC5000-CD-0002"],
        ["d_price_code", 8],
        ["d_unit_price", 0.0],
        ["d_currency_code", "Y"],
        ["d_order_number", 1],
        ["d_factory_number", 0],
        ["d_remain_number", 1],
        ["d_comment_note", "__SMS_TABLE_NULL__"],
        ["d_package_entry_no", "26220AAT"],
        ["d_package_entry_sub_no", 0],
        ["d_package_detail_no", 1]
      ]
    },
    "key2": {
      "26220AAT": [
        ["d_comment_note", "__SMS_TABLE_NULL__"],
        ["d_package_number", 1]
      ]
    },
    "key3": {},
    "key5": {},
    "key6": {
      "26220AAT": "",
      "": ""
    },
    "key8": []
  },
  "key7": [],
  "keys": [
    ["d_supply_code", ""],
    ["d_turnover_month", 202603],
    ["d_turnover_day", 17]
  ]
}
```

### 4.3 顶层字段说明

| 字段 | 类型 | 说明 | RMI 对应 |
|------|------|------|----------|
| corporate | String | 公司代码 | CORPORATE |
| key4 | String | エントリーNo（主键） | KEY4 |
| type | String | 类型（如 order） | TYPE |
| lang | String | 语言（如 ja） | LANG |
| key2 | String | 价格选项（"" 或 "1"） | KEY2 |
| key | Object | 主表单数据（见下） | KEY |
| key5 | Object | R3 移行用 Hash | KEY5 |
| key7 | Array | R3 解除用 Vector | KEY7 |
| keys | Array | Panel3 更新向量 | KEYS |

---

## 五、SmsWebApiUtil 转换逻辑

### 5.1 Vector 目标键

`Sms01288Controller.save()` 中指定：

```java
Set<String> targetKeys = new HashSet<>();
targetKeys.add(Sms01288_i.KEYS);   // "keys"
targetKeys.add(Sms01288_i.KEY7);   // "key7"
```

仅 `keys` 和 `key7` 会被转换为 `Vector`，其余数组保持为 `String[]` 或 `String[][]`。

### 5.2 转换规则

| JSON 类型 | 条件 | 转换结果 |
|-----------|------|----------|
| null | - | `""` |
| String/Boolean/Number | - | 原样 |
| `[]`（空数组） | 在 targetKeys 中 | `Vector`（空） |
| `[[a,b],[c,d]]` | 在 targetKeys 中 | `Vector<String[]>` |
| `[[a,b],[c,d]]` | 不在 targetKeys 中 | `String[][]` |
| `[a,b,c]` | 不在 targetKeys 中 | `String[]` |
| `{}` | - | `Hashtable`（递归） |

### 5.3 keys 的转换

- 输入：`[["d_supply_code",""],["d_turnover_month",202603],["d_turnover_day",17]]`
- 输出：`Vector<String[]>`，元素为 `["d_supply_code",""]`、`["d_turnover_month","202603"]`、`["d_turnover_day","17"]`
- 数值会通过 `String.valueOf()` 转为字符串

### 5.4 key7 的转换

- 输入：`[]`（JSON 数组）→ 输出：空 `Vector`
- 输入：`"[]"`（JSON 字符串）→ 输出：`"[]"`（String），**不会**转为 Vector，可能导致 RMI 侧类型错误

### 5.5 key 内 key1/key2/key3 的转换

- 不在 targetKeys 中，嵌套数组 `[[k,v],[k,v],...]` 转为 `String[][]`
- 例如 `"26220AAU": [[...],[...]]` → `Hashtable` 中值为 `String[][]`

### 5.6 key8 的转换

- 输入：`[]`（JSON 数组）
- 输出：`String[]`（空数组），因为 key8 不在 targetKeys 中
- 注意：RMI 侧期望 `Vector`，此处存在类型差异（见后文）

### 5.7 null 处理

- `key2: null` → `""`
- 其他 null 值 → `""`

---

## 六、WebSms01288ServiceImpl 规范化

当请求经 SmsWeb 转发时，会先执行 `normalizeSaveRequest()`：

| 条件 | 处理 |
|------|------|
| key2 == null | 设为 `""` |
| key5 == null | 设为 `{}` |
| key7 == null | 设为 `[]` |
| keys == null | 设为 `[]` |
| key == null | 新建含 key1~key8 的空结构 |
| key.key1/key2/key3/key6 == null | 设为 `{}` |
| key.key8 == null | 设为 `[]` |
| key.key6 含 `""` 键 | 移除 `key6[""]` |

---

## 七、RMI 调用前的完整参数（prm）

以下为上述示例请求经 `convertJsonToHashtable` 后的 RMI 参数结构（含类型与示例值）。

```
remoteClass: Sms01288_s
method:      save
signature:   Sms01288_s.save(Hashtable)

params: prm [Hashtable<String, Object>]
```

### 7.1 顶层参数

| 键 | 类型 | 示例值 |
|----|------|--------|
| corporate | String | `"X"` |
| key4 | String | `"26220AAT"` |
| type | String | `"order"` |
| lang | String | `"ja"` |
| key2 | String | `""` |
| keys | Vector | 见下 |
| key7 | Vector | `[]`（空） |
| key | Hashtable | 见下 |

### 7.2 keys (Vector<String[]>)

| 索引 | 值 | 类型 |
|------|-----|------|
| 0 | `["d_supply_code", ""]` | String[] |
| 1 | `["d_turnover_month", "202603"]` | String[] |
| 2 | `["d_turnover_day", "17"]` | String[] |

### 7.3 key (Hashtable) 内结构

| 子键 | 类型 | 说明 |
|------|------|------|
| key1 | Hashtable | insertHash |
| key2 | Hashtable | updateHash |
| key3 | Hashtable | deleteHash |
| key6 | Hashtable | statusTbl |
| key8 | String[] 或 String | changeVector（类型见下） |

### 7.4 key.key1 (insertHash)

结构：`Hashtable<String, String[][]>`

| 主键 | 值 | 类型 |
|------|-----|------|
| 26220AAU | `[["d_product_code","70RSCCD020"], ["d_product_name","SC5000-CD-0002"], ...]` | String[][] |

每行格式：`[列名, 值]`

### 7.5 key.key2 (updateHash)

结构：`Hashtable<String, String[][]>`

| 主键 | 值 | 类型 |
|------|-----|------|
| 26220AAT | `[["d_comment_note","__SMS_TABLE_NULL__"], ["d_package_number","1"]]` | String[][] |

### 7.6 key.key3 (deleteHash)

结构：`Hashtable<String, String[][]>`，示例中为空 `{}`

### 7.7 key.key6 (statusTbl)

结构：`Hashtable<String, String>`

| 主键 | 值 |
|------|-----|
| 26220AAT | `""` |

若经 SmsWeb 转发并执行 `normalizeSaveRequest`，`key6[""]` 会被移除；直接调用 SmsWebApi 时不会过滤。

### 7.8 key.key8 (changeVector)

- JSON：`[]`
- SmsWebApiUtil 输出：`String[]`（空）或 `"[]"`（若为字符串）
- RMI 期望：`Vector`（entryNo 列表）

---

## 八、Sms01288_s.save 内部使用方式

```java
Vector p3UpVector   = (Vector)prmHash.get(KEYS);      // keys
Hashtable prm2      = (Hashtable)prmHash.get(KEY);   // key
Hashtable insertHash = (Hashtable)prm2.get(KEY1);     // key1
Hashtable updateHash = (Hashtable)prm2.get(KEY2);     // key2
Hashtable deleteHash = (Hashtable)prm2.get(KEY3);     // key3
Hashtable statusHash = (Hashtable)prm2.get(KEY6);    // key6
Vector changeVector  = (Vector)prm2.get(KEY8);        // key8
String entry         = (String)prmHash.get(KEY4);     // key4
Hashtable r3Hash     = (Hashtable)prmHash.get(KEY5);  // key5
Vector r3Vector      = (Vector)prmHash.get(KEY7);     // key7
```

在 `update()` 和 `insert()` 中：

```java
Vector vector = (Vector)tbl.get(key);  // tbl 为 insertHash/updateHash/deleteHash
for (int i = 0; i < vector.size(); i++) {
    String[] data = (String[])vector.elementAt(i);
    line[0].set(data[0], data[1]);
}
```

即 RMI 期望：`Hashtable<String, Vector<String[]>>`。

---

## 九、类型差异与潜在问题

### 9.1 key1、key2、key3

| 项目 | SmsWebApiUtil 输出 | Sms01288_s 期望 |
|------|--------------------|-----------------|
| 值类型 | `String[][]` | `Vector<String[]>` |

若直接传入 `String[][]`，在 `(Vector)tbl.get(key)` 处会触发 `ClassCastException`。Java Applet 侧通过 `getInsertHash`/`getUpdateHash`/`getDeleteHash` 生成的是 `Vector`。

### 9.2 key8

| 项目 | SmsWebApiUtil 输出 | Sms01288_s 期望 |
|------|--------------------|-----------------|
| 类型 | `String[]` 或 `"[]"` | `Vector` |

同样存在类型不匹配风险。

### 9.3 建议

1. 在 `Sms01288Controller.save()` 的 `targetKeys` 中增加 `key1`、`key2`、`key3`、`key8`，使这些嵌套数组也转为 `Vector`；或
2. 在 `SmsWebApiUtil` 中，对 `key` 内 `key1`、`key2`、`key3`、`key8` 的数组做特殊处理，显式转换为 `Vector`。

---

## 十、完整输入输出示例（按日志格式）

### 输入（Request）

```
method:      POST
uri:         /sms01288/save
contentType: application/json
body.raw:    {"corporate":"X","key4":"26220AAT","type":"order","lang":"ja","key2":null,"key":{"key1":{"26220AAU":[["d_product_code","70RSCCD020"],["d_product_name","SC5000-CD-0002"],["d_price_code",8],["d_unit_price",0.0],["d_currency_code","Y"],["d_order_number",1],["d_factory_number",0],["d_remain_number",1],["d_comment_note","__SMS_TABLE_NULL__"],["d_package_entry_no","26220AAT"],["d_package_entry_sub_no",0],["d_package_detail_no",1]]},"key2":{"26220AAT":[["d_comment_note","__SMS_TABLE_NULL__"],["d_package_number",1]]},"key3":{},"key5":{},"key6":{"26220AAT":"","":""},"key8":"[]"},"key7":"[]","keys":[["d_supply_code",""],["d_turnover_month",202603],["d_turnover_day",17]]}
```

### 输出（RMI 调用前参数）

```
remoteClass: Sms01288_s
method:      save
signature:   Sms01288_s.save(Hashtable)
params (1): prm [Hashtable]

  corporate = "X"                                    [String]
  key4      = "26220AAT"                             [String]
  type      = "order"                                [String]
  lang      = "ja"                                   [String]
  key2      = ""                                     [String]
  keys      = Vector<String[]>                       [Vector]
    [0] = ["d_supply_code", ""]                      [String[]]
    [1] = ["d_turnover_month", "202603"]             [String[]]
    [2] = ["d_turnover_day", "17"]                   [String[]]
  key7      = Vector (empty)                         [Vector]
  key       = Hashtable                              [Hashtable]
    key1 = Hashtable<String, String[][]>            [Hashtable]
      "26220AAU" = [["d_product_code","70RSCCD020"], ["d_product_name","SC5000-CD-0002"], ...] [String[][]]
    key2 = Hashtable<String, String[][]>            [Hashtable]
      "26220AAT" = [["d_comment_note","__SMS_TABLE_NULL__"], ["d_package_number","1"]] [String[][]]
    key3 = {}                                        [Hashtable]
    key5 = {}                                        [Hashtable]
    key6 = Hashtable<String, String>                 [Hashtable]
      "26220AAT" = ""                                [String]
    key8 = "[]" 或 String[] 或 Vector               [取决于 JSON 格式]
```

---

## 十一、参考文档

- `SmsWeb/docs/save-api-analysis.md`：客户传参与 Java 期望对比
- `SmsWeb/docs/save-verification-report.md`：新旧实现一致性验证
