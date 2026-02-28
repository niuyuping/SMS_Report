# Sms01288 save - WebApi RMI 呼び出し直前の入出力サンプル

## 概要

WebApi（SmsWebApi、IntraMart 環境）が `/sms01288/save` リクエストを処理する際、IntraMart/Web API Maker 経由では `keys`、`key7`、`key8` が JSON 文字列として渡される場合がある。SmsWebApi の `SmsWebApiUtil` は `isJsonArrayLike` によりそれらを Vector にパースする。

---

## 修正前後の比較

### 1. コード修正の比較

| 項目 | 修正前 | 修正後 |
|------|--------|--------|
| **Sms01288Controller targetKeys** | `keys`, `key7`, `key8` | `keys`, `key7`, `key8`, **`key1`, `key2`, `key3`** |
| **RmiEndpointRegistry SMS01288_SAVE_KEYS** | `"keys", "key7", "key8"` | `"keys", "key7", "key8", "key1", "key2", "key3"` |

### 2. パラメータ型の比較（RMI 呼び出し前）

| パラメータパス | 修正前 | 修正後 | RMI 期待型 |
|----------------|--------|--------|------------|
| keys | Vector | Vector | Vector ✓ |
| key7 | Vector | Vector | Vector ✓ |
| key.key8 | Vector | Vector | Vector ✓ |
| **key.key1.26220AAU** | **String[][]** ✗ | **Vector** ✓ | Vector |
| **key.key2.26220AAT** | **String[][]** ✗ | **Vector** ✓ | Vector |
| **key.key3 の各子要素** | **String[][]** ✗ | **Vector** ✓ | Vector |
| key.key5 | Hashtable (空) | Hashtable (空) | Hashtable ✓ |
| key.key6.26220AAT | String | String | String ✓ |

### 3. 結果の比較

| 項目 | 修正前 | 修正後 |
|------|--------|--------|
| **RMI 呼び出し** | `ClassCastException`（key2/key3 が非空のとき） | 正常終了 ✓ |
| **例外発生箇所** | `Sms01288_s.update()` 1035 行目 | — |
| **例外メッセージ** | `String[][] cannot be cast to Vector` | — |

---

## 入力 (Request)

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

### IntraMart 環境で到達しうる形式（配列の文字列化）

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

## 修正前の出力 (RMI 呼び出し直前のパラメータ)

`SmsWebApiUtil.convertJsonToHashtable(input, vectorKeys)` 適用後。  
`vectorKeys = {keys, key7, key8}`（key1/key2/key3 を含まない）

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

### 型の説明（修正前）

| パラメータパス | 実際の型 | RMI 期待型 | 備考 |
|----------------|----------|------------|------|
| keys | Vector | Vector | ✓ 正常 |
| key7 | Vector | Vector | ✓ 正常 |
| key.key8 | Vector | Vector | ✓ 正常 |
| **key.key1.26220AAU** | **String[][]** | **Vector** | ✗ 型不一致、RMI で ClassCastException |
| **key.key2.26220AAT** | **String[][]** | **Vector** | ✗ 型不一致、RMI で ClassCastException |
| key.key3 | Hashtable | Hashtable | ✓ 空、子なし |
| key.key5 | Hashtable | Hashtable | ✓ 空 |
| key.key6.26220AAT | String | String | ✓ 正常 |

### 例外発生箇所

`updateHash`（key2）または `deleteHash`（key3）が非空のとき、`Sms01288_s.update()` 1035 行目で以下を実行：

```java
Vector vector = (Vector)tbl.get(key);
```

このとき `tbl.get("26220AAT")` の戻り値は `String[][]` のため、Vector へのキャスト時に以下が発生：

```
java.lang.ClassCastException: class [[Ljava.lang.String; cannot be cast to class java.util.Vector
```

### WebApi_Spring との違い

| 項目 | WebApi_Spring (X-Simulate-Intramart: true) | WebApi (SmsWebApi) 修正前 |
|------|--------------------------------------------|---------------------------|
| keys | [String] 文字列 | [Vector] |
| key7 | [String] 文字列 | [Vector] |
| key.key8 | [String] 文字列 "[]" | [Vector] |
| key.key2.26220AAT | [String[][]] | [String[][]]（ともに不正、RMI は Vector を期待） |
| key.key1.26220AAU | [String[][]] | [String[][]]（ともに不正、RMI は Vector を期待） |

WebApi は `isJsonArrayLike` により keys/key7/key8 を正しく Vector に変換するが、key1/key2/key3 は vectorKeys に含まれておらず、ネストした配列が String[][] となり、RMI 呼び出し時に異常となる。

---

## 修正後の出力（差分のみ）

修正後は、`key.key1`、`key.key2`、`key.key3` 配下のネスト配列が Vector に変換され、該当行は次のように変わる：

```
║       key1 = {...} [Hashtable]:
║         26220AAU = [[d_product_code, 70RSCCD020], ...] [Vector]     ← 修正前は [String[][]]
║       key2 = {...} [Hashtable]:
║         26220AAT = [[d_comment_note, __SMS_TABLE_NULL__], [d_package_number, 1]] [Vector]     ← 修正前は [String[][]]
```

---

## 修正箇所

| プロジェクト | 修正ファイル | 修正内容 |
|--------------|--------------|----------|
| **WebApi**（SmsWebApi / IntraMart） | `SmsWebApi/.../Sms01288Controller.java` | save メソッドの targetKeys に KEY1, KEY2, KEY3 を追加 |
| **WebApi_Spring** | `WebApi_Spring/.../RmiEndpointRegistry.java` | SMS01288_SAVE_KEYS に "key1", "key2", "key3" を追加 |
