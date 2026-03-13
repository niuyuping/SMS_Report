# scp05101 と R3/ERP 接続ハング問題 — 現状と対応方針報告書

**文書版**：1.0  
**日付**：2025-03-12  
**対象**：05101 サービス、SMSServer2、R3Server、Pipe 通信および ERP 未接続時のブロック問題

---

## 一、現状概要

### 1.1 事象

- ERP システムが未接続のとき、scp05101 の **insert（追加）**・**update（出荷指示／更新）** インタフェースを呼ぶと**長時間ハング**し、正常に戻らない。
- **delete（削除）** インタフェースはハングしない。

### 1.2 アーキテクチャ（呼び関係）

```
[ 業務サービスプロセス（複数） ]
   scp05101_s, Sms01203, Sms01236, ...  各々独立 bat・独立プロセス
            │
            │  RMI 呼び出し（同一登録名）
            ▼
[ SMSServer2 — 1 プロセスのみ ]
   test_SMSServer2.bat 起動、RMI 名: SMSServerIfc2
   内部保持: R3Servers_（法人コード別 複数 host:port）
            │
            │  TCP Pipe（読取タイムアウトなし）
            ▼
[ ERP 側プロセス / Mock ]
   localhost:64942 等のポート
```

- **SMSServer2**：システム全体で**1 プロセスのみ**。`test_SMSServer2.bat` で起動。全業務サービス（05101 含む）が**共有**。
- **R3Server**：SMSServer2 プロセス内のみに存在。法人コードごとに **host:port** を複数設定。**全サービスが同一の R3 接続設定を共有**。
- **05101**：R3 に直接は接続せず、RMI で SMSServer2 を呼び出し、SMSServer2 内で R3Server 経由で ERP に接続する。

---

## 二、原因分析

### 2.1 R3 を呼ぶインタフェース（ハングするもの）

| インタフェース | R3Server/ERP を呼ぶか | 説明 |
|----------------|------------------------|------|
| **insertFactoryIssuePlan** | はい（条件成立時） | ランク="2" または "1" かつ 回答区分="1" のとき、`updateSmsOrderTable` → `insertSCPOrder` → `createOrder` → `moveOrderToR3` を経て R3Server.insert() を使用 |
| **updateFactoryIssuePlan** | はい（条件成立時） | 出荷指示分岐は上記と同様。cancel 分岐では `cancelSmsOrderTable` → `cancelSCPOrder` を経て R3 へ update |
| **deleteFactoryIssuePlan** | いいえ | 自 DB の論理削除のみ（factory_issue_plan_table の z_del_time 更新）。SMS/R3 は呼ばない |

よって、**insert/update の「出荷指示」または「cancel」のケースでのみ R3 に接続し、ERP 未接続時にハングする。**

### 2.2 ハングの直接原因：Pipe にタイムアウトなし

- R3Server と ERP の通信は **sms_common.util.Pipe** で行う：`new Pipe(host, port)` で TCP 接続、`open(command)` でコマンド送信後、**`reader.readLine()`** で応答待ち。
- **現状の実装**：
  - コンストラクタで **`socket.setSoTimeout(0)`** を指定しており、**読取タイムアウトは無制限**。
  - 接続は **`new Socket(host, port)`** で、**接続タイムアウトは未設定**。失敗時は 300 回リトライ、間隔 500ms。
- **結果**：ERP が無いと、接続で OS の TCP タイムアウトまで待つか、接続だけ成立した場合に **readLine() がブロックし続け**、リクエストが「ハング」したように見える。

### 2.3 Pipe を外部から閉じてブロック解除できるか

- **現状**：Pipe は Socket を公開しておらず、`close()` もない。R3Server 側の `reader` はローカル変数のみで他スレッドに渡していないため、**外部から閉じることはできない**。
- **コード変更する場合**：Pipe に Socket を保持し **`close()`** を追加するか、`open()` の戻り値の reader を「タイムアウト／キャンセル」用スレッドに渡し、別スレッドから **`close()`** を呼んで `readLine()` のブロックを解除する方法がある。

---

## 三、R3 接続設定（SMSServer2 第 6 引数）

R3 の host/port は **SMSServer2 起動コマンドの第 6 引数**で指定。形式は  
`法人コード:host;port[,法人コード:host;port...]`。

現環境では **localhost + 複数ポート**。一覧は以下。

### 3.1 法人コード別 — host / port 一覧

| 番号 | 法人コード | host | port |
|------|------------|------|------|
| 1 | H | localhost | 64972 |
| 2 | P | localhost | 64967 |
| 3 | X | localhost | 64942 |
| 4 | O | localhost | 64938 |
| 5 | F | localhost | 64936 |
| 6 | W | localhost | 64930 |
| 7 | Y | localhost | 64926 |
| 8 | C | localhost | 64924 |
| 9 | D | localhost | 64920 |
| 10 | A | localhost | 64918 |
| 11 | I | localhost | 64916 |
| 12 | R | localhost | 64914 |
| 13 | L | localhost | 64912 |
| 14 | K | localhost | 64932 |
| 15 | T | localhost | 64910 |
| 16 | N | localhost | 64934 |
| 17 | E | localhost | 64945 |
| 18 | U | localhost | 64940 |
| 19 | J | localhost | 64908 |
| 20 | G | localhost | 64906 |
| 21 | M | localhost | 64904 |
| 22 | S | localhost | 64959 |
| 23 | V | localhost | 64951 |
| 24 | Q | localhost | 64964 |
| 25 | B | localhost | 64976 |
| 26 | 3 | localhost | 64971 |
| 27 | 4 | localhost | 64969 |
| 28 | 5 | localhost | 64899 |
| 29 | X1 | localhost | 64942 |
| 30 | X2 | localhost | 64942 |
| 31 | X3 | localhost | 64942 |
| 32 | X4 | localhost | 64942 |
| 33 | X5 | localhost | 64942 |
| 34 | X6 | localhost | 64942 |
| 35 | X7 | localhost | 64942 |
| 36 | X1H | localhost | 64942 |
| 37 | X4H | localhost | 64942 |
| 38 | X5H | localhost | 64942 |
| 39 | X6H | localhost | 64942 |
| 40 | X7H | localhost | 64942 |
| 41 | X2B | localhost | 64942 |
| 42 | X4B | localhost | 64942 |
| 43 | X5B | localhost | 64942 |
| 44 | X6B | localhost | 64942 |
| 45 | X7B | localhost | 64942 |
| 46 | HT | localhost | 64920 |

### 3.2 ポート別まとめ（Mock リスニング用）

**22 種類のポート**がある。Mock を行う場合は、以下の各ポートで listen し、accept 後にすぐ close すればよい。

| port | そのポートを使う法人コード |
|------|----------------------------|
| 64899 | 5 |
| 64904 | M |
| 64906 | G |
| 64908 | J |
| 64910 | T |
| 64912 | L |
| 64914 | R |
| 64916 | I |
| 64918 | A |
| 64920 | D, HT |
| 64924 | C |
| 64926 | Y |
| 64930 | W |
| 64932 | K |
| 64934 | N |
| 64936 | F |
| 64938 | O |
| 64940 | U |
| 64942 | X, X1, X2, X3, X4, X5, X6, X7, X1H, X4H, X5H, X6H, X7H, X2B, X4B, X5B, X6B, X7B |
| 64945 | E |
| 64951 | V |
| 64959 | S |
| 64964 | Q |
| 64967 | P |
| 64969 | 4 |
| 64971 | 3 |
| 64972 | H |
| 64976 | B |

---

## 四、対応方針

### 4.1 方針 A：Mock ERP（暫定・ERP 未接続時の推奨）

**考え方**：R3 設定の host:port（現状は localhost + 上表のポート）で、**接続を受け付けたらすぐ close するだけ**の Mock を立て、R3 側の `readLine()` が接続クローズで即戻る（または例外）ようにし、リクエストが無限に待たないようにする。

**ポイント**：
- リスニングするアドレス/ポートは SMSServer2 第 6 引数と一致させる。05101 でよく使う法人（例：X）だけ検証するなら、まず **localhost:64942** だけ listen してもよい。
- 動作：`accept()` のあと、必要なら少し読んでから、**すぐその接続を close()** する。
- 正常な業務データは返さないため R3 呼び出しはエラー終了するが、**速く戻る**ためシステム全体のハングは防げる。

**メリット**：既存 Java コードを触らず、導入が簡単。過渡期向き。  
**デメリット**：R3 連携はすべて失敗。ERP 接続後に Mock を外す必要あり。

### 4.2 方針 B：Pipe に読取タイムアウトを追加（中期）

**考え方**：**sms_common.util.Pipe** を修正し、Socket を保持したうえで、構築時または `open()` 後に **`setSoTimeout(ミリ秒)`** を設定。`readLine()` がタイムアウトで `SocketTimeoutException` を投げるようにし、上位で捕捉してエラー返却する。

**メリット**：外部 Mock に依存せず、タイムアウト値も設定可能。  
**デメリット**：コード変更・ビルド・リグレッションが必要。

### 4.3 方針 C：Pipe の外部 close 対応（タイムアウト／Mock と併用）

**考え方**：Pipe に Socket を保持し、**`close()`** を公開するか、`open()` の戻り値の reader を「タイムアウト／キャンセル」スレッドに渡す。タイムアウトやユーザーキャンセル時に別スレッドから **`close()`** を呼び、ブロック中の `readLine()` を解除する。

**メリット**：長時間待ちを能動的に打ち切れる。  
**デメリット**：Pipe とその呼び出し側の変更、並行処理・例外の扱いが必要。

### 4.4 方針 D：R3 を呼ばない設定スイッチ（長期オプション）

**考え方**：SMSServer2 または SCPServer に「ERP 無効」などの設定を追加し、ON のときは **moveOrderToR3 を呼ばない**。SMS 側のみ処理するか、エラー返却とする。

**メリット**：業務ロジック上 ERP に繋がない選択ができる。  
**デメリット**：R3 を呼ぶ経路を洗い出し、設定とテストが必要。

---

## 五、実施順序の提案

| フェーズ | 推奨 | 説明 |
|----------|------|------|
| **短期** | **方針 A（Mock ERP）** を採用 | ERP 未接続時、必要なポート（少なくとも localhost:64942 等の常用法人）で Mock を起動し、accept 後すぐ close してリクエストのハングを防ぐ。 |
| **中期** | **方針 B（Pipe 読取タイムアウト）** を実施 | `Pipe` に `setSoTimeout(例: 30 秒)` を設定し、Mock への依存を減らしつつ長時間ブロックを防ぐ。 |
| **長期** | 必要に応じて **方針 D（R3 スイッチ）** を検討 | 「ERP 停止時は R3 を一切呼ばない」要件があれば、設定と分岐を追加する。 |

---

## 六、関連コード位置（改修時の参照用）

| 内容 | パス／場所 |
|------|------------|
| Pipe の読取タイムアウト設定 | `service/common/sms_common/util/Pipe.java` コンストラクタ内 `socket.setSoTimeout(0)` |
| R3Server での Pipe 利用 | `service/common/sms_common/server/R3Server.java` 各所の `new Pipe(host_, port_).open(...)` 直後の `reader.readLine()` |
| scp05101 の SMS 呼び出し | `service/businese/scp05101/scp05101_s/src/scp05101/scp05101_s.java` の `remoteObject_.insertSCPOrder` / `cancelSCPOrder` |
| SMSServer2 の R3 パラメータ解析 | `service/common/sms_common/server/SMSServer_s.java` の `setR3Parameters` 付近（1456–1480 行程度） |
| SCPServer の R3 呼び出し | `service/common/sms_common/server/SCPServer.java` の `createOrder` 内 `server_.moveOrderToR3(r3Hash)` |

---

**以上。** 特定方針の詳細設計やスクリプト例が必要な場合は、本報告書をベースに追記する。
