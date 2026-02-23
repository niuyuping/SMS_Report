# WebSms01206 与 Sms01206 的 1:1 画面与接口对应说明

## 一、画面一览（4 个画面）

| 画面       | URL | 说明 |
|------------|-----|------|
| 一覧       | `GET /websms01206/maintenance/list` | オーダーエントリー（エントリーNo.別）一覧。检索条件：エントリー番号 / 得意先 / 機種。分页、新规登録・详细・编辑入口。 |
| 新规表单   | `GET /websms01206/maintenance/create?form` | 新规录入。採番・检索・担当/得意先/機種/明細・配送先/ランク/出荷依頼等。 |
| 编辑表单   | `GET /websms01206/maintenance/{entryNo}/update?form` | 与「新规表单」同一 JSP（createForm.jsp），通过 entryNo 加载既有数据。 |
| 详细       | `GET /websms01206/maintenance/{entryNo}` | 只读显示单件オーダー（表头 + 明細）。服务端通过 getOrder 注入 modelJson。 |
| 登録确认   | `GET /websms01206/maintenance/createConfirm` | 登録前确认。显示与 createForm 同结构；「登録実行」POST 到 create；「修正」回 create?form；「戻る」回 list。 |
| 登録実行   | `POST /websms01206/maintenance/create` | 确认画面「登録実行」提交后处理（当前为 redirect:list）。 |

## 二、本次实现内容

### 1. Controller 新增/调整

- **GET `list`**  
  - 返回一覧 JSP：`spring/websms01206/maintenance/list.jsp`。

- **POST `api/getOrderSub`**  
  - 一覧数据来源。请求体为检索条件（如 `entryNo`, `customer`, `product`），透传到 `WebSms01206Service.getOrderSub`（Imart WebApi）。

- **GET `create?form`**  
  - 返回新规表单 JSP：`createForm.jsp`（原 createOrderForm.jsp 已改为 createForm.jsp 与文件名一致）。

- **GET `{entryNo}/update?form`**  
  - 编辑表单。在 Model 中设置 `entryNo`，返回同一 `createForm.jsp`；页面 mounted 时根据 `data-entry-no` 调用 getOrder 加载数据。

- **GET `{entryNo}`**  
  - 详细画面。调用 `getOrder(entryNo, "0")`，将整份响应 JSON 写入 Model 的 `modelJson`，并设置 `entryNo`，返回 `detail.jsp`。  
  - detail.jsp 内通过 `<script id="page-data">${modelJson}</script>` 读取，解析为 `modelRaw.ans` 作为表头与明細数据源。

- **GET `createConfirm`**  
  - 返回登録确认 JSP：`createConfirm.jsp`。

- **POST `create`**  
  - 接收确认画面 form POST（`entryNo` 等）。当前实现为 `redirect:list`，与旧系统「登録実行」后回到一覧一致。

### 2. 前端 API 基路径统一

- **createForm.jsp**  
  - 原：`contextPath + '/websms01206/maintenance/websms01206'`  
  - 现：`contextPath + '/websms01206/maintenance/api'`  
  - 所有接口调用（getCharges, getOrder, orderInsert, getConsigns, getRankDialogMaster 等）均使用 `api + '/method'`，与 Controller 的 `api/*` 一致。

- **createConfirm.jsp**  
  - getOrder 的 URL 从 `.../websms01206/getOrder` 改为 `.../api/getOrder`。

- **list.jsp**  
  - 一覧数据从 `GET .../list?get` 改为 **POST** `.../api/getOrderSub`，请求体为 `searchParams`（entryNo / customer / product）。  
  - 响应支持 `res.data.data` 或 `res.data.ans` 作为列表数组，`res.data.total` 为总件数（缺省时用列表 length）。

### 3. 命名与视图

- 新规/编辑表单视图名统一为 **createForm.jsp**（与磁盘文件名一致），不再使用 createOrderForm.jsp。

## 三、接口调用关系（与旧系统 1:1）

- **一覧**  
  - 画面：GET list → list.jsp。  
  - 数据：POST api/getOrderSub（searchParams）。

- **新规/编辑表单**  
  - 画面：GET create?form 或 GET {entryNo}/update?form → createForm.jsp。  
  - 数据与操作：全部通过 POST api/{method}，例如：  
    getCharges, getCustomers, getProducts, getVari, getUses, getPriceCodeMaster, getMakerMaster, getPrice, getEntryNo, getOrder, orderInsert, orderUpdate, orderDelete, getCustomerProducts, getConsigns, getSupplys, getDefaultConsigns, judgementCustomerConfidence, judgmentStopProduct, getPSSPeriodProduct, judgmentERP, isUnitPriceMaster, getProductPlant, isDuplicationOrder, isDuplicationOrder2, getRankDialogMaster, getForwadingListHtml, getExcelData, setFactoryResults, getCustomerForwardReadTime, getMaxSalesUpdate, getActiveDate, getOutInData, moveR3 等（与 Controller 已暴露的 api/* 一致）。

- **详细**  
  - 画面：GET {entryNo} → Controller 调用 getOrder → 注入 modelJson → detail.jsp 只读展示，无额外前端 API。

- **登録确认**  
  - 画面：GET createConfirm → createConfirm.jsp。  
  - 数据：sessionStorage 或 GET 参数 entryNo + POST api/getOrder。  
  - 提交：form POST create（entryNo）→ redirect:list。

## 四、组件与交互逻辑（与 Sms01206 一致）

- **一覧**  
  - 检索・重置・分页・新规登録・详细・编辑链接；表格列：エントリー番号 / 得意先コード・名 / 機種コード・名 / 注文No / 納期 / 単価 / 操作。

- **createForm**  
  - 採番 / 检索 / 枝番 0–9 切换；追加・更新・削除・クリア・出荷计划・终了；担当・得意先・機種・納品書・納品・カートン・ユーザー・配送先・納入先・装置メーカー・case mark1–7・機展書No・ランク・販価・顧客機種・用途・扱い・販売ルート等；明細行（注文数/出荷済/出荷残/納期/回答区分等）；配送先弹窗（ConsignDialog）、ランク弹窗（RankDialog）、出荷依頼书弹窗（ForwardRequestDialog）；校验：entryNo/担当/得意先/機種必填、明細注文数非负；与信表示、廃止機種/生販確定期間警告。

- **detail**  
  - 表头 H26/H0/H1/H27/B3/B33/B2/H7/H8… 及明細 D01–Dxx；编辑・一覧に戻る。

- **createConfirm**  
  - 与 createForm 同结构的只读确认；登録実行（POST create）・修正（create?form）・戻る（list）。

## 五、技术栈（与既有 WebSms 一致）

- 后端：Spring MVC，`WebSms01206Controller`，`WebSms01206Service` / `WebSms01206ServiceImpl`，`ImartApiClient`（透传至 sms01206 imart WebApi）。  
- 前端：JSP + Vue 3（CDN）+ axios；createForm 使用 Ant Design Vue（a-select 等）；样式：sms01206.css。  
- 接口：全部 POST、JSON 请求体，路径为 `/websms01206/maintenance/api/{method}`。

用户从一覧进入新规/编辑/详细，再经确认画面登録后回到一覧的流程与旧系统 Sms01206 保持一致，实现 1:1 对应。
