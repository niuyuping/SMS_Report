# sms012C8 getCustomers 调用链分析报告

**文档版本**：1.0  
**分析对象**：通过 SmsWebApi 调用 sms012C8 的 getCustomers 接口  
**结论摘要**：调用链经 SmsWebApi → RMI sms012C8_s → SMSServer2 → SMSMasterServer2 → Customer2 → R3Server，**需要 R3Server 通过 Pipe 向 ERP 系统查询得意先数据**。

---

## 1. 目的与范围

- **目的**：厘清 SmsWebApi 调用 sms012C8 中 getCustomers 的完整调用链，并确认是否经由 R3Server、通过 Pipe 向 ERP 查询数据。
- **范围**：从 SmsWebApi 入口到 RMI 服务端、SMSServer、R3Server 及 Pipe 的调用路径与数据来源。

---

## 2. 调用链总览

| 序号 | 层级 | 组件 | 说明 |
|-----|------|------|------|
| ① | 入口 | SmsWebApi | HTTP POST `/sms012C8/getCustomers` |
| ② | 控制器 | Sms012c8Controller | JSON → Hashtable，RMI 调用 sms012C8_s |
| ③ | RMI 业务 | sms012C8_s | getCustomers → getMaster(CUSTOMER_LIST_ERP) |
| ④ | 通用服务 | SMSServer2 | getMaster → master_.get(prm) |
| ⑤ | 主数据 | SMSMasterServer2 | CUSTOMER_LIST_ERP → getCustomerListERP |
| ⑥ | 主数据实现 | Customer2 | getCustomerListERP → getERPMasters |
| ⑦ | R3 网关 | SMSServer2_s | getERPMasters → R3Server.getERPMasters |
| ⑧ | R3/ERP 通信 | R3Server | getERPMasterCustomers → **Pipe 向 ERP** |

---

## 3. 各层调用说明

### 3.1 SmsWebApi 入口（① → ②）

- **路径**：`POST /sms012C8/getCustomers`（或框架映射的等价路径）。
- **实现**：`SmsWebApi/src/main/java/jp/co/nidec/smswebapi/rmi/controller/Sms012c8Controller.java`
  - `@Path(PATH_GET_CUSTOMERS)`、`@POST`，接收 `@Body String input`。
  - 使用 `SmsWebApiUtil.convertJsonToHashtable(input)` 将 JSON 转为 Hashtable。
  - 通过 `RmiConnectionManager.getRemoteObject("sms012C8_s")` 取得 RMI 远程对象。
  - 调用 `remoteObject_.getCustomers(prm)`，返回值转为 Map 后返回。

### 3.2 RMI 服务端 sms012C8_s（③）

- **实现**：`RMI_SERVER/business_service/sms012C8/sms012C8_s.java`（约 266–314 行）
- **逻辑**：
  - 从 `prm` 取得 `KEY`（customer）、`LANG`、`CORPORATE`。
  - 组装 `keys = [corporate, customer, "LIKE", "Z001"]`。
  - 构造 `hash`：`LANG`、`TYPE = SMSMasterServer.CUSTOMER_LIST_ERP`、`KEY = keys`。
  - 调用 `remoteObject_.getMaster(hash)`。  
    `remoteObject_` 为 SMSServer 接口，在构造函数中通过 `MSWRemote(path, SMSServerIfc.NAME + "2")` 取得，即 **SMSServer2**。
- **要点**：getCustomers 的数据来源被固定为 **CUSTOMER_LIST_ERP**（来自 ERP 的得意先列表），不是本地 DB 或其它 master 类型。

### 3.3 SMSServer2（④）

- **实现**：`RMI_SERVER/common_service/sms_common/server/SMSServer2_s.java`（约 296–299 行）
- **逻辑**：`getMaster(prm)` 直接委托给 `master_.get(prm)`，`master_` 为 **SMSMasterServer2**。

### 3.4 SMSMasterServer2（⑤）

- **实现**：`RMI_SERVER/common_service/sms_common/server/SMSMasterServer2.java`（约 545–551 行）
- **逻辑**：当 `type == CUSTOMER_LIST_ERP` 时，调用  
  `customer_.getCustomerListERP(lang, table)`，  
  其中 `customer_` 为 **Customer2**。

### 3.5 Customer2（⑥）

- **实现**：`RMI_SERVER/common_service/sms_common/master/Customer2.java`（约 376–406 行）
- **逻辑**：
  - 从 `table`（即传入的 hash）中取 `SMSServerIfc.KEY` 得到 `key[]`。
  - 调用 `server_.getERPMasters(SMSMasterServer.CUSTOMER_LIST_ERP, key[0], key)`。  
    `server_` 在 SMSServer2 体系下为 **SMSServer2_s**，其 `getERPMasters` 会转给 **R3Server**。
  - 将 ERP 返回的得意先数据填入 Hashtable，键为得意先コード等，并包含 `"__ERP__"` 前缀的原始 ERP 行（sms012C8_s 中会过滤掉 `__ERP__` 键，仅用 SMS 侧整理后的数据）。

### 3.6 SMSServer2_s.getERPMasters（⑦）

- **实现**：`RMI_SERVER/common_service/sms_common/server/SMSServer2_s.java`（约 4182–4190 行）
- **逻辑**：
  - 按法人代码 `corp` 从 `R3Servers_` 中取 `R3Server r3`。
  - 调用 `r3.getERPMasters(type, key)`，对 getCustomers 而言 `type == CUSTOMER_LIST_ERP`。

### 3.7 R3Server 与 Pipe（⑧）

- **实现**：`RMI_SERVER/common_service/sms_common/server/R3Server.java`
  - `getERPMasters`（约 669–689 行）：当 `type == SMSMasterServer.CUSTOMER_LIST_ERP` 时，调用 `getERPMasterCustomers(key)`。
  - `getERPMasterCustomers`（约 931–958 行）：
    - 使用 `new Pipe(host_, port_).open("72 \"" + key[0] + "\" \"" + key[1] + "\" " + opt + " " + flg + " X ja")` 向 ERP 端进程发起 Pipe 通信。
    - 按行读取返回内容，解析为 `String[][]`（得意先列表），返回给上层。
- **结论**：**R3Server 通过 Pipe 向 ERP 系统查询得意先数据，是 getCustomers 的必经路径。**

---

## 4. 是否需 R3Server 通过 Pipe 向 ERP 查询

| 项目 | 结论 |
|------|------|
| 是否经 R3Server | **是**。getCustomers 使用 CUSTOMER_LIST_ERP，最终由 SMSServer2 的 getERPMasters 转交 R3Server。 |
| 是否通过 Pipe | **是**。R3Server.getERPMasterCustomers 使用 `Pipe(host_, port_).open(...)` 与 ERP 端通信。 |
| 数据来源 | 得意先列表来自 **ERP 系统**，非仅 RDB 或其它本地 master。 |

因此：**通过 SmsWebApi 调用 sms012C8 的 getCustomers 时，需要 R3Server 通过 Pipe 向 ERP 系统查询数据。** 若 R3/ERP 未配置或 Pipe 不通，该接口可能返回空或异常。

---

## 5. 关键代码索引

| 环节 | 文件 | 行号/位置 |
|------|------|-----------|
| API 入口 | SmsWebApi/.../Sms012c8Controller.java | PATH_GET_CUSTOMERS，getCustomers(@Body)，remoteObject_.getCustomers(prm) |
| 业务实现 | RMI_SERVER/.../sms012C8_s.java | 266–314：getCustomers，CUSTOMER_LIST_ERP，getMaster(hash) |
| 类型分发 | RMI_SERVER/.../SMSMasterServer2.java | 545–551：CUSTOMER_LIST_ERP → getCustomerListERP |
| ERP 调用 | RMI_SERVER/.../Customer2.java | 376–406：getCustomerListERP，getERPMasters(CUSTOMER_LIST_ERP, ...) |
| R3 与 Pipe | RMI_SERVER/.../R3Server.java | 669–689：getERPMasters；931–958：getERPMasterCustomers，Pipe.open("72 ...") |

---

## 6. 小结

- **调用链**：SmsWebApi → Sms012c8Controller → RMI sms012C8_s.getCustomers → SMSServer2.getMaster → SMSMasterServer2（CUSTOMER_LIST_ERP）→ Customer2.getCustomerListERP → SMSServer2.getERPMasters → R3Server.getERPMasterCustomers → **Pipe 向 ERP**。
- **R3Server 与 Pipe**：getCustomers 使用的 CUSTOMER_LIST_ERP 必须经 R3Server 的 Pipe 向 ERP 查询，无“仅查本地 DB、不连 ERP”的分支。
- **依赖**：该接口正常返回依赖 R3Server 配置及 ERP 端 Pipe 服务可用。

---

*报告结束*
