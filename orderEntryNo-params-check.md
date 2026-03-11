# WebSms01206 orderEntryNo 参数处理检查报告

以主版本 `OrderEntryNo.java` 为基准，对 `corporate_code`（SMS法人コード）与 `table_type`（order=アクチャル / plan=基礎計画）在前端的处理逻辑进行对照与补全。

---

## 一、主版本 OrderEntryNo.java 中的约定

### 1. table_type（type_）

- **取值**: `order`（アクチャル・実订单）、`plan`（基礎計画）。
- **使用方式**:
  - 所有 RMI 调用（getOrder / getOrderSub / orderUpdate / orderDelete / insert / getPrice 等）均将 `type_` 作为 `OrderEntryNo_i.TYPE` 传入。
  - **与信管理**: 仅在 `"order".equals(type_)` 时执行（画面表示・得意先変更時の判定・更新後の判定）。  
    - 例: `if ("order".equals(type_)) { ... judgeCustomerConfidence ... }`、  
      `if (!"order".equals(type_)) return ccJudge;`  
  - 即：**plan 时不进行与信管理**（不调 judgementCustomerConfidence、不显示与信限度額、不参与 detailLocked）。

### 2. corporate_code（corporate_）

- **取值**: 主版本中出现的有 X, J, 4, H, N, U, L, M, R, S, V, Q, 3 等（SMS法人コード）。
- **使用方式**:
  - 所有 RMI 调用均将 `corporate_` 作为 `OrderEntryNo_i.CORPORATE` 传入。
  - **与信表示対象**: `ccCorporates_` に含まれる法人のみ与信限度額等を表示（主版本では X, H, O, T, N, K 等）。
  - **NNSN 系**: U, R, L, M, S, V, Q, 3 等で出荷リードタイム取得（getCustomerForwardReadTime）等の分岐あり。
  - **V**: 画面ラベル変更（case mark、機種/工程図番、倉庫/出荷場所等）。
  - **N, U**: 日付・取消関連の特別処理。
  - **isJapan（収益認識等）**: X, N, U, J, 4 を対象とする記述あり。

---

## 二、Web 前端原有实现与缺口

### 1. table_type

- **已有**:  
  - Controller 接收 `table_type`，放入 Model / `WEBSMS01206_CONFIG.tableType`。  
  - 画面初期の `getOrder` のみ `type: (self.tableType || 'order')` を使用。
- **缺口**:  
  - その他 API（getOrder 枝番切替・getOrderSub・orderUpdate・orderDelete・insert・getPrice）はすべて `type: 'order'` 固定。  
  - `table_type=plan` でも与信表示・`judgementCustomerConfidence` が実行されていた（主版本では plan 時は行わない）。  
  - 出荷依頼 Excel 用 `getExcelData` の payload が `type: 'order'` 固定。

### 2. corporate_code

- **已有**:  
  - Controller 接收 `corporate_code`，传入 `corporate`。  
  - `ccCorporates: ['X','H','O','T','N','K']`、`isCcCorporate()`・`showCreditLimit()` で与信表示を制御。  
  - `isJapan` は `['X','N','U','J','4']` で主版本と一致。  
  - `judgementNNSN`・`getTransportList` 等で `corporate` を渡している。
- **缺口**:  
  - 主版本の「V 法人時は case mark / 機展書No. 等ラベル変更」は、Web 版は静的な JSP ラベルのため未対応（**主版本に合わせるなら動的であるべき**。下記「case mark / 機展書No. 動的ラベル」参照）。

---

## 三、本次补全内容

### 1. table_type=plan 時の与信なし

- **showCreditLimit**:  
  `(this.tableType || 'order') !== 'order'` のときは与信限度額を表示しない（主版本の「order のときだけ与信」に合わせた）。
- **judgementCustomerConfidence**:  
  得意先変更時の処理で、`(this.tableType || 'order') === 'order'` のときだけ  
  `reloadCustomerConfidence` / `judgementCustomerConfidence` を呼び、  
  plan のときは `customerConfidenceLimit` と `ccJudge` のみクリア（与信は行わない）。

### 2. 全 API で type に tableType を使用

- 以下をすべて `type: (this.tableType || 'order')` または `type: (self.tableType || 'order')` に変更。  
  - 初期・枝番切替の `getOrder`  
  - `getOrderSub`（2 箇所）  
  - `orderUpdate`  
  - `orderDelete`  
  - insert 用 body の `type`  
  - `getPrice`（4 箇所）
- 出荷依頼 Excel 用 `getExcelData`（dialog-forward.js）の payload を  
  `type: (vm.tableType && vm.tableType.trim()) ? vm.tableType.trim() : 'order'` に変更。

### 3. corporate_code

- 既存の `corporate` 受け渡し・`ccCorporates`・`isCcCorporate`・`isJapan` は主版本と整合しており、今回の変更対象外。  
- **対応済み**: 法人別・NNSN 系・kanagata に応じた動的ラベルを Vue の computed（labelNpmEntryNo, labelProduct, labelCaseMark1～7, consignButtonLabel）で実装し、JSP でバインド済み。

---

## 四、case mark / 機展書No. 動的ラベル（主版本分析）

主版本 OrderEntryNo.java のラベル設定は **type_（table_type）を参照していない**。  
したがって **table_type は case mark・機展書No. の表示ラベルと無関係**。

### 主版本の条件分岐（要約）

| 条件 | 参照しているパラメータ | ラベル変更内容（例） |
|------|------------------------|----------------------|
| デフォルト | （labelHash のみ） | ケースマーク1～7（7487/7488…）、機展書No.（5249） |
| NNSN 系 | **corporate_code** ∈ {U,R,L,M,S,**V**,Q,3} または nnsnFlg_ | 33→前注文番号、**24→在庫引当数**、28～32→顧客ライン等 |
| さらに **V** | **corporate_code == "V"** | **26→倉庫/出荷場所、27→"case mark"、33→機種/工程図番**（24 は「在庫引当数」のまま） |
| DMM金型 | **kidou_parameter の kanagata** | 27→拠点、**24→機展書No.**、26→販売窓口 等（金型用一式） |

- **case mark（Label27）**: デフォルト＝7488。**V 時**＝"case mark"。**kanagata 時**＝"拠点"。→ **corporate_code（と kanagata）に依存、table_type は不使用**。
- **機展書No.（Label24）**: デフォルト＝5249。**NNSN 系**＝在庫引当数（V でも 24 は変更しない）。**kanagata 時**＝機展書No.。→ **corporate_code（と kanagata）に依存、table_type は不使用**。

### 結論

- **動的であるべき**: 主版本は corporate_code と kanagata に応じてラベルを切り替えているため、Web 版も静的なままでよい理由はない。**corporate_code および kanagata に応じた動的ラベルが望ましい**。
- **corporate_code に関係する**: 上記のとおり、V および NNSN 系法人でラベルが変わる。
- **table_type には関係しない**: ラベル設定コードに type_ の参照は存在しない。

---

## 五、パラメータ取り扱い一覧（補全後）

| 参数 | 取值 | 前端处理 |
|------|------|----------|
| **table_type** | order | 全 API に `type: 'order'` を渡す。与信表示・judgementCustomerConfidence を実行。 |
| **table_type** | plan | 全 API に `type: 'plan'` を渡す。与信表示なし・judgementCustomerConfidence 不调用。 |
| **corporate_code** | X, H, N, … | 全 API に `corporate` を渡す。ccCorporates により与信表示の有無を制御。 |
| **corporate_code** | V 等 | 同上。ラベル変更は現状未実装（静的 JSP のため）。 |

以上により、主版本と同様に **corporate_code** と **table_type** の各取值に対して、API の type/corporate の受け渡しおよび「order 時のみ与信・plan 時は与信なし」の扱いがそろった。  
case mark / 機展書No. 等のラベルは、主版本に合わせるなら **corporate_code と kanagata に応じた動的表示**を追加するのが望ましく、**table_type はラベル表示と無関係**である。

---

## 六、Web 与 RMI 主版本对照确认（corporate_code / table_type）

**说明**: 本仓库仅包含 SmsWeb，不包含 RMI_SERVER。以下以文档中记载的主版本 OrderEntryNo 约定为基准，对 Web 端实现做逐项核对。

### 6.1 参数接收与传递

| 项目 | 主版本约定 | Web 实现 | 状态 |
|------|------------|----------|------|
| corporate_code | 接收 SMS法人コード，默认 X；所有 RMI 调用传 CORPORATE | Controller 接收 `corporate_code`，默认 "X"，放入 Model `corporate`；前端所有 getOrder/getOrderSub/orderUpdate/orderDelete/insert/getPrice 及与信・NNSN・运输等 API 均传 `corporate: (this.corporate \|\| 'X')` | ✅ 一致 |
| table_type | order=アクチャル、plan=基礎計画；所有 RMI 传 TYPE；**仅 order 时执行与信** | Controller 接收 `table_type`，默认 "order"，放入 Model `tableType`；全 API 使用 `type: (this.tableType \|\| 'order')`；出荷依頼 Excel 的 getExcelData 使用 `vm.tableType` | ✅ 一致 |

### 6.2 与信逻辑（table_type=plan 时不执行）

| 场景 | 主版本 | Web 实现 | 状态 |
|------|--------|----------|------|
| 与信限度額表示 | order 且 ccCorporates 含当前法人时显示 | `showCreditLimit()` 中 `(this.tableType \|\| 'order') !== 'order'` 则 return false | ✅ 一致 |
| 得意先変更時の与信判定 | 仅 type_=order 时调用 judgementCustomerConfidence / reloadCustomerConfidence | orderEntryNo.js 中 `(this.tableType \|\| 'order') === 'order'` 时才调用上述 API，否则清空 customerConfidenceLimit / ccJudge | ✅ 一致 |
| 订单数据加载时（getOrder 返回后）与信 | 仅 order 时调用 judgementCustomerConfidence | sms01206-order-data.js 的 loadOrderData 中 `(vm.tableType \|\| 'order') === 'order'` 时才调用 judgementCustomerConfidence，否则清空 ccJudge / customerConfidenceLimit | ✅ 已补全 |

### 6.3 corporate_code 相关业务

| 项目 | 主版本 | Web 实现 | 状态 |
|------|--------|----------|------|
| 与信対象法人 | ccCorporates（X,H,O,T,N,K 等） | ccCorporates: ['X','H','O','T','N','K']，isCcCorporate / showCreditLimit 控制显示 | ✅ 一致 |
| isJapan（収益認識等） | X, N, U, J, 4 | 同左 | ✅ 一致 |
| NNSN 系（U,R,L,M,S,V,Q,3） | 出荷リードタイム等分岐、ラベル変更 | judgementNNSN、getTransportList 等传 corporate；isNnsnCorporate、labelNpmEntryNo 等动态标签 | ✅ 一致 |
| V 法人 | case mark / 機種・工程図番 / 倉庫・出荷場所 等ラベル変更 | labelCaseMark1～7、labelProduct、consignButtonLabel 等 computed 按 V 切换 | ✅ 一致 |
| kanagata（金型） | 拠点・機展書No.・販売窓口 等ラベル | isKanagata、对应 label 与 consignButtonLabel | ✅ 一致 |

### 6.4 结论

- **corporate_code（SMS法人コード）**: Web 已正确接收、默认值 X，并贯穿所有需法人区分的 API 与画面逻辑（与信、NNSN、V/金型动态标签）。
- **table_type（order/plan）**: Web 已正确接收、默认 order；所有订单相关 API 与 getExcelData 均传 type；**仅在 table_type=order 时执行与信**（画面表示・得意先変更・订单加载三处均已约束），与主版本一致。
- **业务与交互**: 与信表示/非表示、动态标签（機展書No./機種/ケースマーク等）、出荷依頼 Excel 的 type 传递均已与主版本对齐。若主版本在 RMI_SERVER 中有额外分支（如未在文档中体现的 type/corporate 分支），需在 RMI 侧再对照一次源码确认。
