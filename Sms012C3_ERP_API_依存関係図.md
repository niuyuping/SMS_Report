# Sms012C3 と ERP 連携インターフェース依赖関係図

## 一、全体アーキテクチャ：画面 → API → サーバ → Pipe → ERP

```mermaid
flowchart TB
    subgraph 画面层["画面 (sms012C3.java)"]
        A1[getPrice]
        A2[getUnitPrice]
        A3[isUnitPriceMaster]
        A4[getConsigns]
        A5[getChargeCustomers]
        A6[getCustomers]
        A7[getProducts]
        A8[getCustomerProducts]
        A9[getCustomerProduct]
        A10[getUserList]
        A11[setFactoryResults]
        A12[moveR3]
        A13[judgmentERP]
    end

    subgraph 服务器层["sms012C3_s / SMSServer2_s"]
        B1[getMaster<br/>FLAG=TYPE=ERP]
        B2[setFactoryResults<br/>直接呼出]
        B3[moveOrderToR3]
        B4[judgmentERP]
    end

    subgraph R3Server["R3Server (sms_common)"]
        C1[getERPMasterPrice<br/>Pipe 71]
        C2[getERPMasterCustomers<br/>Pipe 72]
        C3[getERPMasterProducts<br/>Pipe 74]
        C4[getERPMasterCustomerProduct<br/>Pipe 87]
        C5[insert/update<br/>Pipe 62/64]
    end

    subgraph Pipe层["Pipe (Socket)"]
        D1["コマンド 71<br/>販価"]
        D2["コマンド 72<br/>得意先"]
        D3["コマンド 74<br/>機種"]
        D4["コマンド 87<br/>得意先品目"]
        D5["コマンド 63<br/>試作出荷実績"]
        D6["コマンド 62/64<br/>R3 注文移行"]
    end

    subgraph ERP["ERP システム"]
        E[(SAP/R3)]
    end

    A1 --> B1
    A2 --> B1
    A3 --> B1
    A4 --> B1
    A5 --> B1
    A6 --> B1
    A7 --> B1
    A8 --> B1
    A9 --> B1
    A10 --> B1
    A11 --> B2
    A12 --> B3
    A13 --> B4

    B1 --> C1
    B1 --> C2
    B1 --> C3
    B1 --> C4
    B2 --> D5
    B3 --> C5
    B4 --> C3

    C1 --> D1
    C2 --> D2
    C3 --> D3
    C4 --> D4
    C5 --> D6

    D1 --> E
    D2 --> E
    D3 --> E
    D4 --> E
    D5 --> E
    D6 --> E
```

## 二、Pipe コマンド / データタイプ別 API グループ依赖

```mermaid
flowchart LR
    subgraph Pipe71["Pipe 71 - 販価"]
        P71[getERPMasterPrice<br/>getERPMasterPrices]
        API71a[getPrice]
        API71b[getUnitPrice]
        API71c[isUnitPriceMaster]
        API71d[getUserList 一部]
    end

    subgraph Pipe72["Pipe 72 - 得意先"]
        P72[getERPMasterCustomers]
        API72a[getConsigns]
        API72b[getChargeCustomers]
        API72c[getCustomers]
        API72d[getUserList<br/>getEndUser]
    end

    subgraph Pipe74["Pipe 74 - 機種"]
        P74[getERPMasterProducts]
        API74a[getProducts]
        API74b[judgmentERP 間接]
    end

    subgraph Pipe87["Pipe 87 - 得意先品目"]
        P87[getERPMasterCustomerProduct]
        API87a[getCustomerProducts]
        API87b[getCustomerProduct]
    end

    subgraph Pipe63["Pipe 63 - 試作出荷"]
        P63[setFactoryResults]
    end

    subgraph Pipe62_64["Pipe 62/64 - R3 移行"]
        P62[moveOrderToR3]
        API62[moveR3]
    end

    API71a --> P71
    API71b --> P71
    API71c --> P71
    API71d --> P71
    API72a --> P72
    API72b --> P72
    API72c --> P72
    API72d --> P72
    API74a --> P74
    API74b -.->|製品マスタ| P74
    API87a --> P87
    API87b --> P87
    P63 --> P63
    API62 --> P62
```

## 三、業務呼び出し順序依赖（前後関係）

```mermaid
flowchart TB
    subgraph 主数据检索["マスタ検索（ERP から取得）"]
        direction TB
        M1[getCustomers / getChargeCustomers<br/>getConsigns]
        M2[getProducts]
        M3[getPrice / getUnitPrice<br/>isUnitPriceMaster]
        M4[getCustomerProducts<br/>getCustomerProduct]
        M5[getUserList]
    end

    subgraph 判定["判定系"]
        J1[judgmentERP]
    end

    subgraph 订单持久化["注文永続化（SMS 側）"]
        O1[orderInsert / orderUpdate]
    end

    subgraph 向ERP送数["ERP への送信"]
        S1[moveR3]
        S2[setFactoryResults]
    end

    M1 --> M2
    M2 --> M3
    M2 --> J1
    M4 --> M3
    M1 --> M5

    O1 --> S1
    J1 -.->|移行可否に影響| S1
    S2 -.->|独立：試作出荷実績| S2
```

## 四、ERP と連携する 13 API 一覧と依赖概要

| API | 方向 | Pipe/タイプ | 依赖関係の概要 |
|-----|------|-------------|----------------|
| getPrice | 取得 | 71 販価 | getMaster(ERP PRICE) に依存。他 API と独立 |
| getUnitPrice | 取得 | 71 販価 | 同上 |
| isUnitPriceMaster | 取得 | 71 販価 | 同上 |
| getConsigns | 取得 | 72 得意先 | getMaster(CUSTOMER_LIST_ERP) に依存。独立 |
| getChargeCustomers | 取得 | 72 得意先 | 同上 |
| getCustomers | 取得 | 72 得意先 | 同上 |
| getProducts | 取得 | 74 機種 | getMaster(PRODUCT_LIST_ERP/ERP PRODUCT) に依存。独立 |
| getCustomerProducts | 取得 | 87 得意先品目 | getMaster(CUSTOMER_PRODUCT_ERP2) に依存。独立 |
| getCustomerProduct | 取得 | 87 得意先品目 | getMaster(CUSTOMER_PRODUCT_ERP) に依存。独立 |
| getUserList | 取得 | 71 + 72 | PRICE_LIST_ERP + getEndUser(CUSTOMER_LIST_ERP) |
| setFactoryResults | 送信 | 63 | 直接 Pipe。他 ERP API に非依存 |
| moveR3 | 送信 | 62/64 | **orderInsert/orderUpdate の先行実行に依存**。業務上は注文保存後に呼び出し |
| judgmentERP | 判定 | 間接 74 | 製品マスタ（getProducts/PRICE 由来の可能性）に依存。moveR3 実行可否に影響 |

説明：
- **取得系** 10 個：相互に呼び出し順序の制約はなく、いずれも「必要に応じて ERP からマスタを取得」。
- **送信系** 2 個：**moveR3** は注文の保存済み（orderInsert/orderUpdate）に依存；**setFactoryResults** は独立。
- **判定系** 1 個：**judgmentERP** は製品マスタ（74 機種データ関連）に依存し、後続の moveR3 実行可否に影響する。

---

## 五、ERP に依存するインターフェースがすべて ERP と連携できない場合に、Sms012C3 で完了できない業務

上記 13 個の ERP 連携インターフェースが**すべて ERP システムと正常に連携できない**場合、以下の業務に影響するか、完了できない。

### 1. 完全に完了できない業務（送信系）

| 業務 | 依存する ERP インターフェース | 完了できない意味 |
|------|-------------------------------|------------------|
| **R/3 注文移行** | moveR3（Pipe 62/64） | 注文を SMS から ERP（SAP/R3）に同期できない。保存後の注文は R3 側で生成/更新されず、生産・出荷・在庫等の ERP 側プロセスは当該注文に基づいて進行できない。 |
| **無償試作出荷実績登録** | setFactoryResults（Pipe 63） | 試作出荷実績を ERP に書き戻せない。試作出荷業務は ERP 側で正しい実績データを得られない。 |

### 2. マスタ・検索系：「ERP 由来の最新データ」を取得できない

| 業務 | 依存する ERP インターフェース | 完了できない意味 |
|------|-------------------------------|------------------|
| **得意先・担当・配送先の ERP 元データ** | getCustomers, getChargeCustomers, getConsigns（Pipe 72） | ERP から最新の得意先/担当/配送先マスタを取得できない。画面のドロップダウンが空になるか、ローカル/キャッシュのみとなり、ERP と一致した選択・チェックが保証できない。 |
| **機種一覧の ERP 元データ** | getProducts（Pipe 74） | ERP から最新の機種マスタを取得できない。機種選択/検索が空またはローカルデータのみとなり、新規・変更時の機種選択に影響する。 |
| **販価・標準販価の ERP 元データ** | getPrice, getUnitPrice, isUnitPriceMaster（Pipe 71） | ERP から販価を取得できず、「機種・得意先別」販価存在チェックができない。価格の自動带出、標準販価制限等のロジックが ERP に沿って正しく実行できない。 |
| **得意先品目・NSCM 品目** | getCustomerProducts, getCustomerProduct（Pipe 87） | ERP から得意先品目/NSCM 品目を取得できない。機種・得意先ダイアログ検索、NSCM 品目带出等を ERP データに基づいて完了できない。 |
| **NSCM 最終ユーザ一覧** | getUserList（Pipe 71 + 72） | ERP から最終ユーザ一覧を取得できず、NSCM 画面上の「最終ユーザ」選択を正しく完了できない。 |

### 3. 判定・後続フローへの影響（判定系）

| 業務 | 依存する ERP インターフェース | 完了できない意味 |
|------|-------------------------------|------------------|
| **ERP 連携機種判定** | judgmentERP（74 等に間接依存） | 「当該機種が ERP と連携するか」を正しく判定できない。erpProduct_/erpFlag_ 等のフラグ異常により、R3 に移行すべき注文が移行されない、または移行すべきでないものが移行対象と誤判定され、R3 移行の前提条件が誤る。 |

### 4. 引き続き完了できる業務（上記 ERP インターフェースに非依存）

「ERP との連携がすべて無効」という前提では、以下は完了可能（SMS 自身の DB と主要 API に依存）：

- **注文の新規・更新・削除・枝番追加**（getEntryNo, getOrderSub, getOrder, orderInsert, orderUpdate, orderDelete）  
  → 注文は **SMS 側**で正常に保存・保守できる。
- **画面初期化とラベル表示**（getLabelString）。
- **ERP マスタに依存しないドロップダウン/検索**（SMS またはローカルマスタのみ使用する場合）。

つまり：**SMS 内の「受注入力・保守」は完了できるが、「ERP と一致したマスタ取得」「R3 移行」「試作出荷実績の書き戻し」は完了できない。**

### 5. 影響度別まとめ表

| 影響度 | 業務 | 説明 |
|--------|------|------|
| **完全に不可** | R/3 注文移行 | 注文データを ERP に書き込めない。 |
| **完全に不可** | 無償試作出荷実績登録 | 試作実績を ERP に書き戻せない。 |
| **著しく制限** | 得意先・担当・配送先・機種・販価・得意先品目・NSCM ユーザ等のマスタ | ERP 由来の最新データを取得できず、入力・チェックが ERP と不一致。 |
| **ロジック異常** | ERP 連携機種判定 → 移行判断 | R3 移行の可否の前提条件が誤る。 |
| **完了可能** | SMS 内注文の増削改査、画面操作 | SMS 側データに限り、ERP と未同期。 |
