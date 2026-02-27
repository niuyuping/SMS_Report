# SmsWebApiUtil における Vector 対象 key の JSON 配列文字列対応

## 概要

IntraMart 環境で Sms01288 の Save インターフェースを呼び出す際、`key8` パラメータの変換時に「String から Vector への変換異常」が発生する問題の原因と解決策を説明する。

## 発生現象

- **対象 API**: `/sms01288/save`
- **発生環境**: IntraMart 上で稼働する SmsWebApi
- **エラー内容**: `key8` パラメータの変換時に `ClassCastException` が発生
  - `java.lang.String cannot be cast to java.util.Vector`

## 原因分析

### 1. 想定されるリクエスト形式

正常時、クライアントから送信される JSON は以下の形式である：

```json
{
  "key": {
    "key1": {...},
    "key2": {...},
    "key8": []
  },
  "key7": [],
  "keys": [["d_supply_code", ""], ...]
}
```

この場合、Jackson により `key8` は `List`（空の `ArrayList`）としてパースされ、`convertListToVector` で `Vector` に変換される。

### 2. IntraMart/Bloom Maker 環境における差異

 IntraMart の Web API Maker や Bloom Maker 経由では、リクエストの受け渡し方法が異なる場合がある：

- 複雑なオブジェクトを `JSON.stringify` により文字列化して送信
- フォームパラメータ等を介した渡し方で、配列値が文字列 `"[]"` として二重エンコードされる
- プロキシやゲートウェイによる再エンコード

その結果、`key8` の値が本来の配列 `[]` ではなく、**文字列 `"[]"`** として届くことがある。

### 3. 既存コードの挙動

`deserializeAnalyzeJson` では、JSON オブジェクト形式の文字列 `{...}` のみを再パース対象としている：

```java
// isJsonLike: 先頭が '{' かつ 末尾が '}' であれば解析対象とみなす
return trimmed.startsWith("{") && trimmed.endsWith("}");
```

JSON 配列形式 `"[]"` は `[` と `]` で囲まれるため、`isJsonLike` の条件に合致せず、**再パースされずに String のまま残る**。

さらに `convertMapToHashtable` では、`value instanceof String` の場合はそのまま Hashtable に格納する：

```java
} else if (value instanceof String || value instanceof Boolean || value instanceof Number) {
    prm.put(key, value);
}
```

その結果、RMI サーバ側（`Sms01288_s.save`）で以下の処理を行う際に `ClassCastException` が発生する：

```java
Vector changeVector = (Vector) prm2.get(KEY8);  // String "[]" を Vector にキャストできない
```

### 4. なぜ Spring Boot（WebApi_Spring）では発生しないか

WebApi_Spring では、標準の `@RequestBody` により生の HTTP ボディがそのまま渡される。  
`Content-Type: application/json` で正しい JSON が送信される限り、Jackson により `key8` は `List` としてパースされるため、本問題は発生しない。

一方、IntraMart 環境ではリクエストの経路や形式が異なり、上記のように `key8` が文字列 `"[]"` として届く可能性がある。

## 解決策

### 実装内容

`convertMapToHashtable` および `convertMapToHashtableIncludeKey` において、以下の条件を満たす場合に JSON 配列文字列をパースして Vector に変換する処理を追加した：

- 対象 key が `vectorKeys` に含まれる、または `isVector` が true
- 値が `String` 型
- 文字列が JSON 配列形式（`[` で始まり `]` で終わる）

### 追加したメソッド

```java
/**
 * 文字列が解析対象（JSON配列）に近い形式か判定します。
 * IntraMart/Bloom Maker 等で Vector 対象 key が JSON 配列文字列として届く場合の解析用。
 */
private static boolean isJsonArrayLike(String str)
```

### 処理フロー

1. `isJsonArrayLike` で文字列が `[` で始まり `]` で終わるか判定
2. 条件を満たす場合、`ObjectMapper.readValue(str, List.class)` で List にパース
3. 既存の `convertListToVector` により Vector に変換
4. パースに失敗した場合は元の String をそのまま格納し、ワーニングをログ出力

### 修正対象ファイル

- `SmsWebApi/src/main/java/jp/co/nidec/smswebapi/rmi/util/SmsWebApiUtil.java`
  - `convertMapToHashtable` に JSON 配列文字列の解析処理を追加
  - `convertMapToHashtableIncludeKey` に同様の処理を追加
  - `isJsonArrayLike` メソッドを新規追加

## 影響範囲

- **対象 key**: `keys`, `key7`, `key8` 等、`vectorKeys` に登録されている key
- **影響 API**: Sms01288 save をはじめ、`vectorKeys` を指定して `convertJsonToHashtable` を呼び出すすべての API
- **後方互換性**: 通常の配列形式 `[]` で届く場合は従来どおり List として処理されるため、既存の動作には影響しない

## 検証方法

WebApi_Spring の IntraMart シミュレーション機能（ヘッダ `X-Simulate-Intramart: true`）を使用すると、JSON 配列が文字列化されたリクエストを再現できる。  
修正後は、シミュレーション適用時でも `key8` が Vector として正しく変換されることを確認すること。
