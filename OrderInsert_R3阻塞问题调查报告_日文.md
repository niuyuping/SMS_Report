# OrderInsert R3 ブロック問題調査報告書

**報告日**：2025-02-26  
**問題現象**：InsertOrder が R3 サービス未起動時に約 2.5 分間ブロックした後、結果を返却できる  
**対象モジュール**：SMS オーダー登録、R3/ERP ブリッジ、Pipe 通信

---

## 一、問題の背景

### 1.1 現象の概要

- WebSms が `getTurnoverDate` インターフェースを呼び出した際、リクエストが長時間応答しない
- R3 サービスが未起動の状態でも、`insertOrder` 呼び出しが約 2.5 分間ブロックした後、成功レスポンスを返す
- R3 サービスが未起動であることを確認済み

### 1.2 関連システム構成

```
WebSms → RMI(SMSServer2) → insertOrder / getTurnoverDate
                              ↓
                    getInspectDate / getCustomerReadTime
                              ↓
                    getMaster(CUSTOMER_LIST_ERP)
                              ↓
                    Customer2.getCustomerListERP
                              ↓
                    getERPMasters → R3Server.getERPMasterCustomers
                              ↓
                    Pipe(host, port) → R3 ブリッジプロセス
```

---

## 二、調査プロセス

### 2.1 ブロック発生箇所の特定

ログおよびコード解析により、以下を特定：

- **OrderEntryNo_L.err**：`getTurnoverDate [X][17504][20260303]`
- **SMSServer2.err**：`getTurnoverDate-1 call` → `getCustomerReadTime` → `Customer2.getCustomerListERP` → `SMSServer2.getERPMasters` → `R3Server.getERPMasters [customer_list_erp]`
- **R3Server**：`Pipe(host_, port_).open()` により R3 ブリッジプロセスへ接続し、`reader.readLine()` で応答待機

### 2.2 重要コードの分析

#### 2.2.1 Pipe コンストラクタ（ブロックの主因）

**ファイル**：`sms_common/util/Pipe.java`

```java
public static int RETRY_CNT = 300;

public Pipe(String host, int port) {
    for(int i = 0; i < RETRY_CNT; i++){
        try {
            Socket socket = new Socket(host, port);
            // ... recv_, send_ の初期化
            break;
        } catch (Exception exception) {
            if(i == RETRY_CNT-1) exception.printStackTrace();
        }
        try{
            System.err.println("Pipe retry..."+i);
            Thread.currentThread().sleep(500);  // 失敗のたびに 500ms 待機
        } catch (Exception exception2){
            exception2.printStackTrace();
        }
    }
}
```

- R3 未起動時、`new Socket(host, port)` は即時に失敗（Connection refused）
- 失敗のたびに `sleep(500)` ms 実行
- 最大 300 回リトライ  
**合計ブロック時間：300 × 500ms ≈ 150 秒（約 2.5 分）**

#### 2.2.2 300 回失敗後の挙動

300 回すべて失敗した場合、`recv_` と `send_` は初期化されず `null` のままとなる。

続けて `Pipe.open(cmd)` が呼ばれると：

```java
public BufferedReader open(String command) {
    try {
        send_.write(command);  // send_ が null → NullPointerException
        send_.write(EOD);
        send_.flush();
    } catch (IOException exception) {
        exception.printStackTrace();
    }
    return recv_;
}
```

`send_` が null のため `NullPointerException` が発生し、上位の catch で捕捉されて「null または空テーブルの返却」に変換される。

#### 2.2.3 例外ハンドリングの流れ

| 階層 | クラス/メソッド | 例外処理 | 戻り値 |
|------|-----------------|----------|--------|
| 1 | R3Server.getERPMasterCustomers | catch Exception | (String[][])null を返す |
| 2 | SMSServer2_s.getERPMasters | catch Exception | null を返す |
| 3 | Customer2.getCustomerListERP | erp が null による NPE を catch | 空 Hashtable を返す |
| 4 | getCustomerReadTime | catch Exception | readTime=0 で aHash を返す |
| 5 | getTurnoverDate | readTime=0 等のデフォルト値を使用 | 日付計算を継続 |
| 6 | getInspectDate | catch Exception | return "" |
| 7 | insertOrder | catch Exception | 後続処理を継続し中断しない |

---

## 三、insertOrder 呼び出しチェーンの整理

### 3.1 R3 関連の実行パス

**発火条件**：`isJapan(ccode)` かつ `handle_code=="1"` かつ `d_on_board_date` または `d_inspect_date` が空

```
insertOrder
  └── for each line
        └── try {
              getInspectDate(line, corp, customerCode)
                └── getTurnoverDate(wprm, corp, delivery)
                      └── getCustomerReadTime(wk)
                            └── getMaster(CUSTOMER_LIST_ERP)
                                  └── Customer2.getCustomerListERP
                                        └── getERPMasters(type, corp, key)
                                              └── R3Server.getERPMasterCustomers
                                                    └── new Pipe(host_, port_).open(cmd)
                                                          ├── コンストラクタ：300 回リトライ × 500ms ≈ 2.5 分
                                                          └── open()：NPE → catch される
            } catch(Exception) {
              exception.printStackTrace();  // 例外が握りつぶされる
            }
```

### 3.2 insertOrder 全体フロー

```
insertOrder
├── 1. 収益認識対応（売上日・検収日自動設定）  ← R3 ブロックが発生する箇所
├── 2. 標準販価再取得（getMasterUnitPrice）
├── 3. OUT-IN 編集（editOutInData）
├── 4. insertDeployEntryData
├── 5. getPlantHandle / getCustomerDeployConsign
├── 6. orderTable.insert（DB 挿入）
├── 7. updatePlanUnitPrice / updateForcast
├── 8. insertOutInData
└── 9. OutInCustomer_.insertOutInCustomer
```

R3 呼び出し失敗後、上位は readTime=0 や空日付などのデフォルト値を使用し、DB 挿入などの後続処理はそのまま実行される。

---

## 四、根因の結論

### 4.1 ブロック時間が長い理由

| 要因 | 説明 |
|------|------|
| Pipe のリトライ機構 | RETRY_CNT=300、失敗ごとに sleep 500ms |
| 計算式 | 300 × 500ms = 150 秒 ≈ 2.5 分 |
| 該当ケース | R3 ブリッジプロセス未起動またはポート未監視 |

### 4.2 結果が返る理由

| 要因 | 説明 |
|------|------|
| 複数階層の try-catch | R3Server から insertOrder まで一貫して例外キャッチ |
| デフォルト値によるフォールバック | getCustomerReadTime が空テーブル時に readTime=0 を返す |
| 処理の継続 | insertOrder の catch ではログ出力のみで後続処理を継続 |
| 業務許容 | 売上日/検収日はデフォルト値を使用してもオーダーは DB に登録される |

---

## 五、タイムライン（R3 未起動時の代表的なケース）

| 時刻 | イベント |
|------|----------|
| T=0s | insertOrder が getInspectDate を呼び出す |
| T=0s | 呼び出しチェーンが R3Server.getERPMasterCustomers に到達 |
| T=0s | `new Pipe(host, port)` 開始 |
| T=0~150s | Pipe コンストラクタ：300 回リトライ、各回 sleep 500ms、"Pipe retry...N" 出力 |
| T≈150s | 300 回目で失敗、Pipe.open() で NPE 発生 |
| T≈150s | 例外が上位へ伝播し、各層で catch して null/空テーブル/デフォルト値を返す |
| T≈150s | getCustomerReadTime が readTime=0 で処理継続 |
| T≈150s | getTurnoverDate / getInspectDate がデフォルト値で返却 |
| T≈150s | insertOrder が処理を続行し、DB 挿入完了後に返却 |

---

## 六、改善提案

### 6.1 短期対応（ブロック時間の短縮）

1. **リトライ回数の削減**：`Pipe.RETRY_CNT` を 300 から 3～5 に変更  
2. **待機時間の短縮**：sleep を 500ms から 100ms に変更  
3. **想定効果**：ブロック時間を約 2.5 分から 0.5～3 秒程度まで短縮

### 6.2 長期対応（堅牢性向上）

1. **接続タイムアウト**：`Socket.connect(addr, connectTimeout)` により接続待ち時間を制限  
2. **読み取りタイムアウト**：`socket.setSoTimeout(readTimeout)` により readLine の無限待ちを防止  
3. **R3 稼働監視**：起動時や定期実行で R3 ブリッジプロセスをチェックし、早期にアラート  
4. **フォールバック方針**：R3 利用不可時はローカル／デフォルトデータを明示的に使用し、長時間ブロックを避ける  

---

## 七、付録

### 7.1 関連ファイル一覧

| ファイルパス | 説明 |
|--------------|------|
| sms_common/util/Pipe.java | Pipe 接続およびリトライロジック |
| sms_common/server/R3Server.java | R3 インターフェース、getERPMasterCustomers 等 |
| sms_common/server/SMSServer2_s.java | insertOrder、getInspectDate、getTurnoverDate |
| sms_common/master/Customer2.java | getCustomerListERP |

### 7.2 関連定数

- `Pipe.RETRY_CNT`：300  
- Pipe コンストラクタ内の `Thread.sleep`：500ms
