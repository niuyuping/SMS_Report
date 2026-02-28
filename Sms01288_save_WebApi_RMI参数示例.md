# Sms01288 save - WebApi RMI 调用前 输入输出示例

## 场景说明

WebApi（SmsWebApi，IntraMart 环境）处理 `/sms01288/save` 请求时，经 IntraMart/Web API Maker 后，`keys`、`key7`、`key8` 可能以 JSON 字符串形式到达。SmsWebApi 的 `SmsWebApiUtil` 具备 `isJsonArrayLike` 处理，会将它们解析为 Vector。

---

## 修复前后对比

### 1. 代码修改对比

| 项目 | 修复前 | 修复后 |
|------|--------|--------|
| **Sms01288Controller targetKeys** | `keys`, `key7`, `key8` | `keys`, `key7`, `key8`, **`key1`, `key2`, `key3`** |
| **RmiEndpointRegistry SMS01288_SAVE_KEYS** | `"keys", "key7", "key8"` | `"keys", "key7", "key8", "key1", "key2", "key3"` |

### 2. 参数类型对比（RMI 调用前）

| 参数路径 | 修复前 | 修复后 | RMI 期望 |
|----------|--------|--------|----------|
| keys | Vector | Vector | Vector ✓ |
| key7 | Vector | Vector | Vector ✓ |
| key.key8 | Vector | Vector | Vector ✓ |
| **key.key1.26220AAU** | **String[][]** ✗ | **Vector** ✓ | Vector |
| **key.key2.26220AAT** | **String[][]** ✗ | **Vector** ✓ | Vector |
| **key.key3 各子值** | **String[][]** ✗ | **Vector** ✓ | Vector |
| key.key5 | Hashtable (空) | Hashtable (空) | Hashtable ✓ |
| key.key6.26220AAT | String | String | String ✓ |

### 3. 结果对比

| 项目 | 修复前 | 修复后 |
|------|--------|--------|
| **RMI 调用** | `ClassCastException`（key2/key3 非空时） | 正常执行 ✓ |
| **异常位置** | `Sms01288_s.update()` 第 1035 行 | — |
| **异常信息** | `String[][] cannot be cast to Vector` | — |

---

## 输入 (Request)

```
POST /sms01288/save
Content-Type: application/json

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

### IntraMart 环境下可能到达的形式（数组被字符串化）

```json
{
  "corporate": "X",
  "key4": "26220AAT",
  "type": "order",
  "lang": "ja",
  "key2": null,
  "key": {
    "key1": {"26220AAU": [["d_product_code","70RSCCD020"], ...]},
    "key2": {"26220AAT": [["d_comment_note","__SMS_TABLE_NULL__"], ["d_package_number",1]]},
    "key3": {},
    "key5": {},
    "key6": {"26220AAT": "", "": ""},
    "key8": "[]"
  },
  "key7": "[]",
  "keys": "[[\"d_supply_code\",\"\"],[\"d_turnover_month\",202603],[\"d_turnover_day\",17]]"
}
```

---

## 修复前输出 (RMI 呼び出し直前のパラメータ)

`SmsWebApiUtil.convertJsonToHashtable(input, vectorKeys)` 適用後。  
`vectorKeys = {keys, key7, key8}`（未包含 key1/key2/key3）

```
╔══════════════════════════════════════════════════════════════════
║ API 呼び出し [/sms01288/save]
╠══════════════════════════════════════════════════════════════════
║ 【Request】
║   method:      POST
║   uri:         /sms01288/save
║   contentType: application/json
║   body.raw:    {"corporate":"X","key4":"26220AAT","type":"order","lang":"ja","key2":null,"key":{"key1":{"26220AAU":[["d_product_code","70RSCCD020"],["d_product_name","SC5000-CD-0002"],["d_price_code",8],["d_unit_price",0.0],["d_currency_code","Y"],["d_order_number",1],["d_factory_number",0],["d_remain_number",1],["d_comment_note","__SMS_TABLE_NULL__"],["d_package_entry_no","26220AAT"],["d_package_entry_sub_no",0],["d_package_detail_no",1]]},"key2":{"26220AAT":[["d_comment_note","__SMS_TABLE_NULL__"],["d_package_number",1]]},"key3":{},"key5":{},"key6":{"26220AAT":"","":""},"key8":[]},"key7":[],"keys":[["d_supply_code",""],["d_turnover_month",202603],["d_turnover_day",17]]}
╠══════════════════════════════════════════════════════════════════
║ 【RMI 呼び出し直前のパラメータ】SmsWebApiUtil.convertJsonToHashtable 適用後
║   remoteClass: Sms01288_s
║   method:      save
║   signature:   Sms01288_s.save(Hashtable)
║   params (8):
║     corporate = X [String]
║     key2 = "" [String]
║     key4 = 26220AAT [String]
║     key = {key1={26220AAU=[[d_product_code, 70RSCCD020], [d_product_name, SC5000-CD-0002], [d_price_code, 8], [d_unit_price, 0.0], [d_currency_code, Y], [d_order_number, 1], [d_factory_number, 0], [d_remain_number, 1], [d_comment_note, __SMS_TABLE_NULL__], [d_package_entry_no, 26220AAT], [d_package_entry_sub_no, 0], [d_package_detail_no, 1]]}, key2={26220AAT=[[d_comment_note, __SMS_TABLE_NULL__], [d_package_number, 1]]}, key3={}, key5={}, key6={26220AAT="", =""}, key8=[]} [Hashtable]:
║       key1 = {26220AAU=[[d_product_code, 70RSCCD020], [d_product_name, SC5000-CD-0002], [d_price_code, 8], [d_unit_price, 0.0], [d_currency_code, Y], [d_order_number, 1], [d_factory_number, 0], [d_remain_number, 1], [d_comment_note, __SMS_TABLE_NULL__], [d_package_entry_no, 26220AAT], [d_package_entry_sub_no, 0], [d_package_detail_no, 1]]} [Hashtable]:
║         26220AAU = [[d_product_code, 70RSCCD020], [d_product_name, SC5000-CD-0002], [d_price_code, 8], [d_unit_price, 0.0], [d_currency_code, Y], [d_order_number, 1], [d_factory_number, 0], [d_remain_number, 1], [d_comment_note, __SMS_TABLE_NULL__], [d_package_entry_no, 26220AAT], [d_package_entry_sub_no, 0], [d_package_detail_no, 1]] [String[][]]
║       key2 = {26220AAT=[[d_comment_note, __SMS_TABLE_NULL__], [d_package_number, 1]]} [Hashtable]:
║         26220AAT = [[d_comment_note, __SMS_TABLE_NULL__], [d_package_number, 1]] [String[][]]
║       key3 = {} [Hashtable]:
║       key5 = {} [Hashtable]:
║       key6 = {26220AAT="", =""} [Hashtable]:
║         26220AAT = "" [String]
║       key8 = [] [Vector]
║     key7 = [] [Vector]
║     keys = [[d_supply_code, ], [d_turnover_month, 202603], [d_turnover_day, 17]] [Vector]
║     lang = ja [String]
║     type = order [String]
╚══════════════════════════════════════════════════════════════════
```

### 类型说明（修复前）

| 参数路径 | 实际类型 | RMI 期望类型 | 说明 |
|----------|----------|--------------|------|
| keys | Vector | Vector | ✓ 正常 |
| key7 | Vector | Vector | ✓ 正常 |
| key.key8 | Vector | Vector | ✓ 正常 |
| **key.key1.26220AAU** | **String[][]** | **Vector** | ✗ 类型不符，RMI 端会 ClassCastException |
| **key.key2.26220AAT** | **String[][]** | **Vector** | ✗ 类型不符，RMI 端会 ClassCastException |
| key.key3 | Hashtable | Hashtable | ✓ 空，无子值 |
| key.key5 | Hashtable | Hashtable | ✓ 空 |
| key.key6.26220AAT | String | String | ✓ 正常 |

### 异常发生位置

当 `updateHash`（key2）或 `deleteHash`（key3）非空时，`Sms01288_s.update()` 第 1035 行执行：

```java
Vector vector = (Vector)tbl.get(key);
```

此时 `tbl.get("26220AAT")` 返回的是 `String[][]`，强转为 Vector 时抛出：

```
java.lang.ClassCastException: class [[Ljava.lang.String; cannot be cast to class java.util.Vector
```

### 与 WebApi_Spring 的差异

| 项目 | WebApi_Spring (X-Simulate-Intramart: true) | WebApi (SmsWebApi) 修复前 |
|------|--------------------------------------------|---------------------------|
| keys | [String] 字符串 | [Vector] |
| key7 | [String] 字符串 | [Vector] |
| key.key8 | [String] 字符串 "[]" | [Vector] |
| key.key2.26220AAT | [String[][]] | [String[][]]（均错误，RMI 期望 Vector） |
| key.key1.26220AAU | [String[][]] | [String[][]]（均错误，RMI 期望 Vector） |

WebApi 具备 `isJsonArrayLike`，keys/key7/key8 会正确转为 Vector；但 key1/key2/key3 未在 vectorKeys 中，其嵌套数组会变成 String[][]，导致 RMI 调用异常。

---

## 修复后输出（仅差异部分）

修复后，`key.key1`、`key.key2`、`key.key3` 下的嵌套数组会转为 Vector，输出中对应行变化为：

```
║       key1 = {...} [Hashtable]:
║         26220AAU = [[d_product_code, 70RSCCD020], ...] [Vector]     ← 修复前为 [String[][]]
║       key2 = {...} [Hashtable]:
║         26220AAT = [[d_comment_note, __SMS_TABLE_NULL__], [d_package_number, 1]] [Vector]     ← 修复前为 [String[][]]
```
