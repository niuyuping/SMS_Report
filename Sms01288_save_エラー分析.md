# 1288機能におけるSave API保存処理未反映事象について

現在、当チームにて「1288」機能の開発を進めておりますが、Save APIを呼び出した際に、エラーは返却されないものの、更新・削除・追加処理が反映されない事象が発生しております。レスポンス上は正常終了となっておりますが、データベース上では対象データの変更が確認できない状況です。

本件の確認資料として、AIによるAPIコードのシミュレーション分析結果、および当チームで使用した入力パラメータとその返却値をPDFにまとめて添付しております。

当チーム側の実装またはパラメータ設定に起因する可能性も含め、原因の切り分けを行いたく存じますが、ローカル環境では当該APIの直接的なテストおよびデバッグ実行ができないため、詳細な処理フローの確認が困難な状況です。

つきましては、APIご担当チームにてデバッグモードで上記入力パラメータをご確認いただき、当該パラメータ実行時の処理内容をご教示いただけますと幸いです。

また、可能でございましたら、正常に保存処理が実行される入力パラメータのサンプルをご共有いただけますと幸いです。

お手数をおかけいたしますが、ご確認のほどよろしくお願いいたします。

# テストデータ

### ・入力パラメータ

```
{
  "lang" : "ja",
  "type" : "order",
  "corporate" : "X",
  "keys" : [ [ "d_supply_code", "" ] ],
  "key" : {
    "key1" : {
      "26223ACS" : [ [ "d_product_code", "70RSR102890" ], [ "d_product_name", "SR9930G0008" ], [ "d_price_code", 8 ], [ "d_unit_price", 0 ], [ "d_currency_code", "Y" ], [ "d_order_number", 1 ], [ "d_factory_number", 0 ], [ "d_remain_number", 1 ], [ "d_comment_note", "__SMS_TABLE_NULL__" ], [ "d_package_entry_no", "26223ACN" ], [ "d_package_entry_sub_no", 0 ], [ "d_package_detail_no", 1 ] ]
    },
    "key2" : {
      "26223ACN" : [ [ "d_package_number", 3 ] ]
    },
    "key3" : { },
    "key6" : {
      "26223ACN" : "",
      "26223ACP" : "",
      "26223ACQ" : ""
    },
    "key8" : [ ]
  },
  "key2" : "",
  "key4" : "26223ACN",
  "key5" : { },
  "key7" : [ ]
}
```

### ・出力パラメータ

```
{
    "error": false,
    "data": {
        "ans2": {}
    }
}
```

<div style="page-break-after: always;"></div>

### ・入力パラメータ

```
{
  "keys": [],
  "key": {
    "key1": {},
    "key2": {
      "26223ABU": [
        [
          "d_package_number",
          2
        ]
      ]
    },
    "key3": {
      "26223ABX": [
        [
          "d_package_entry_no",
          ""
        ],
        [
          "d_package_entry_sub_no",
          ""
        ],
        [
          "d_package_detail_no",
          ""
        ],
        [
          "d_order_number",
          0
        ],
        [
          "d_remain_number",
          0
        ]
      ]
    },
    "key6": {
      "26223ABU": "",
      "26223ABV": "",
      "26223ABW": "",
      "26223ABX": ""
    },
    "key8": [],
    "key5": {}
  },
  "key2": null,
  "key4": "26223ABU",
  "key5": {},
  "key7": [],
  "corporate": "X",
  "type": "order",
  "lang": "ja"
}
```

### ・出力パラメータ

```
{
    "error": false,
    "data": {
        "ans2": {}
    }
}
```

<div style="page-break-after: always;"></div>

------



# SMS01288 save API 入参エラー分析

## 1. 概要

save リクエスト（JSON）を従来の `Sms01288_s.save(Hashtable)` に渡す際、**key2（更新行）／key3（削除行）** の value の型が期待と一致せず、`update()` 内で **ClassCastException** が発生する事象について分析した結果をまとめます。

---

## 2. エラー発生箇所

| 項目 | 内容 |
|------|------|
| **対象クラス** | `sms01288api/Sms01288_s.java` |
| **メソッド** | `update(Hashtable prm, Hashtable tbl, Vector p3UpVector, ...)` |
| **行** | `vector` をループし、`String[] data = (String[]) vector.elementAt(i);` でキャストしている箇所（約 1037–1038 行付近） |
| **例外** | `java.lang.ClassCastException`（`Vector` を `String[]` にキャストしようとして失敗） |

該当コードイメージ：

```java
Vector vector = (Vector) tbl.get(key);   // key2 または key3 の 1 エントリ分
for (int i = 0; i < vector.size(); i++) {
    String[] data = (String[]) vector.elementAt(i);  // ← ここで ClassCastException
    line[0].set(data[0], data[1]);
}
```

---

## 3. 期待される型と実際の型

| 項目 | 内容 |
|------|------|
| **期待される型** | `tbl` の各 value は **`Vector<String[]>`**。  
  つまり、1 要素が「列名・値」の 2 要素の **String 配列**（例: `String[]{"d_package_number", "2"}`）。 |
| **実際に渡っていた型** | JSON をそのまま Map/List から Hashtable/Vector に変換した場合、  
  `["d_package_number", 2]` が **Vector**（要素は String と Integer）として変換され、  
  結果として **Vector<Vector>**（または Vector と数値の混在）になっていた。 |
| **結果** | `vector.elementAt(i)` が **String[]** ではなく **Vector** 等を返すため、  
  `(String[]) vector.elementAt(i)` の時点で **ClassCastException** が発生。 |

---

## 4. 根本原因

- **エラーの原因は `Sms01288_s.update()` のロジックではなく、入参の変換処理にある。**
- SmsWeb 側で、フロントから受信した JSON（Map/List）を `Sms01288_s.save(Hashtable)` に渡す前に **Hashtable/Vector に変換**している。
- その変換処理（`Sms01288SaveParamUtil.toHashtable()` の**修正前**）では、  
  **key.key2** および **key.key3** の value について、
  - 期待: 各 entry の value が **Vector<String[]>**（[列名, 値] の配列のベクタ）
  - 実際: 汎用の List→Vector 変換により、**[列名, 値] が Vector として格納**され、  
    結果が **Vector<Vector>** 相当になり、`update()` の前提と不一致。

この型の不一致が、上記 ClassCastException の根本原因です。

<div style="page-break-after: always;"></div>

---

## 5. 対応内容（修正方針と実装）

| 項目 | 内容 |
|------|------|
| **修正対象** | `SmsWeb/.../websms01288/util/Sms01288SaveParamUtil.java` の `toHashtable()` および関連メソッド。 |
| **方針** | **key.key2** と **key.key3** の value に限り、  
  JSON の `[ ["列名", 値], ... ]` を **Vector<String[]>** に変換する専用処理を行う。  
  各 [列名, 値] は **String[] { 列名, String.valueOf(値) }** にし、それを Vector に格納する。 |
| **実装** | ・`toLegacyKeyInner()` を追加し、`key` 内の **key2** / **key3** のみ上記ルールで変換。  
  ・`toVectorOfStringArrays()` を追加し、List の各要素を `String[]` に変換して Vector を組み立てる。  
  ・`toHashtable()` から `key` を変換する際に `toLegacyKeyInner()` を利用するよう変更。 |

この対応により、同じ JSON を送っても `update()` 内で ClassCastException は発生せず、  
その後の deleteOrder や setTotalPrice まで処理が進む想定です。

---

## 6. 参考（問題再現用の入参例）

以下のような save リクエストで、修正前は **手順 3（update(updateHash)）** の実行時に上記 ClassCastException が再現されていました。

- `keys`: []
- `key.key1`: {}
- `key.key2`: `{ "26223ABU": [ ["d_package_number", 2] ] }`
- `key.key3`: `{ "26223ABX": [ ["d_package_entry_no",""], ... ] }`
- `key.key6`: 各 entry のステータス
- `key.key8`: []
- `key4`: `"26223ABU"`
- その他: corporate, type, lang 等

詳細なステップごとの挙動は `Sms01288_save_param_simulation.md` に記載しています。

---

## 7. まとめ

| 項目 | 内容 |
|------|------|
| **どこでエラーか** | `Sms01288_s.update()` 内の `(String[]) vector.elementAt(i)` の行。 |
| **何のエラーか** | ClassCastException（要素が Vector のため String[] にキャストできない）。 |
| **根本原因** | 入参変換で key2/key3 の value が Vector<String[]> ではなく Vector<Vector> 相当になっていたこと。 |
| **対応** | `Sms01288SaveParamUtil` で key2/key3 の value を **Vector<String[]>** に変換するよう修正済み。 |

---

<div style="page-break-after: always;"></div>

## 8. ClassCastException なのに「正常」が返る理由

save() メソッド全体が try-catch で囲まれており、catch で **Exception を握りつぶしている**ためです。

- 発生箇所: `Sms01288_s.save()` の try ブロック内（例: update() 内で ClassCastException）
- catch: `} catch (Exception exception) { exception.printStackTrace(); }` で例外を捕捉し、**再スローしていない**
- その直後に **ans.put(ANS2, kekkaTbl); return ans;** が必ず実行される

このため、例外が発生してもメソッドは必ず Hashtable を返し、呼び出し元（API）は 200 OK と「error: false」「data: { ans2: ... }」のような正常レスポンスを返してしまいます。実際には保存が途中で失敗している可能性があります。修正案としては、少なくとも例外時は ans にエラー情報を入れるか、例外を再スローして呼び出し元で error: true を返すようにする必要があります。

以上がエラー分析の内容となります。ご不明点等がございましたら、ご連絡いただけますと幸いです。
