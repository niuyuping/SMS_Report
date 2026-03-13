# scp05101 与 R3/ERP 连接卡住问题 — 现状与解决方案报告

**文档版本**：1.0  
**日期**：2025-03-12  
**范围**：05101 服务、SMSServer2、R3Server、Pipe 通信与 ERP 未连通时的阻塞问题

---

## 一、现状概述

### 1.1 现象

- ERP 系统未接通时，调用 scp05101 的 **insert（追加）**、**update（出荷指示/更新）** 接口会**长时间卡住**，无法正常返回。
- **delete（削除）** 接口不会卡住。

### 1.2 架构关系（谁调谁）

```
[ 业务服务进程（多个） ]
   scp05101_s, Sms01203, Sms01236, ...  每个有独立 bat，独立进程
            │
            │  RMI 调用（同一注册名）
            ▼
[ SMSServer2 — 仅 1 个进程 ]
   test_SMSServer2.bat 启动，RMI 名: SMSServerIfc2
   内部持有: R3Servers_（按法人代码多组 host:port）
            │
            │  TCP Pipe（无读超时）
            ▼
[ ERP 侧进程 / Mock ]
   localhost:64942 等端口
```

- **SMSServer2**：全系统**只有 1 个**，由 `test_SMSServer2.bat` 启动；所有业务服务（含 05101）**共用**该实例。
- **R3Server**：仅在 SMSServer2 进程内存在，按法人代码配置多组 **host:port**；**所有服务共用同一套 R3 连接配置**。
- **05101**：不直接连 R3，通过 RMI 调用 SMSServer2，由 SMSServer2 内部再调 R3Server 连 ERP。

---

## 二、原因分析

### 2.1 哪些接口会走到 R3（会卡住）

| 接口 | 是否调用 R3Server/ERP | 说明 |
|------|------------------------|------|
| **insertFactoryIssuePlan** | 是（条件满足时） | 当 ランク="2" 或 "1" 且 回答区分="1" 时，会调用 `updateSmsOrderTable` → `insertSCPOrder` → `createOrder` → `moveOrderToR3`，进而使用 R3Server.insert() |
| **updateFactoryIssuePlan** | 是（条件满足时） | 出荷指示分支同上；cancel 分支会调用 `cancelSmsOrderTable` → `cancelSCPOrder` → 内部对 R3 做 update |
| **deleteFactoryIssuePlan** | 否 | 仅做本地 DB 逻辑删除（update factory_issue_plan_table set z_del_time），不调用 SMS/R3 |

因此：**只有 insert/update 在“出荷指示”或“cancel”场景下会连 R3，会因 ERP 未连通而卡住。**

### 2.2 卡住的直接原因：Pipe 无超时

- R3Server 与 ERP 的通信通过 **sms_common.util.Pipe** 完成：`new Pipe(host, port)` 建立 TCP 连接，`open(command)` 发送命令，然后 **`reader.readLine()`** 等待响应。
- **当前实现**：
  - 构造里对 Socket 调用了 **`socket.setSoTimeout(0)`**，即**读超时 = 无限**。
  - 连接阶段使用 **`new Socket(host, port)`**，**未设置连接超时**；失败后重试 300 次、每次间隔 500ms。
- **结果**：ERP 未接通时，要么连接阶段长时间等待系统 TCP 超时，要么连接成功（或由其他进程占坑）后 **readLine() 一直阻塞**，请求表现为“卡住”。

### 2.3 能否从外部关闭 Pipe 解除阻塞

- **当前**：Pipe 未暴露 Socket，也没有 `close()`；R3Server 中 `reader` 为局部变量，未交给其他线程，**无法从“外部”关闭**。
- **若改代码**：可在 Pipe 中保存 Socket 并增加 `close()`，或把 `open()` 返回的 reader 交给“超时/取消”线程，由另一线程调用 `close()` 打断阻塞的 `readLine()`。

---

## 三、R3 连接配置（SMSServer2 第 6 参数）

R3 的 host/port 来自 **SMSServer2 启动命令的第 6 个参数**，格式为：  
`法人代码:host;port[,法人代码:host;port...]`。

当前环境为 **localhost + 多端口**，详细值如下。

### 3.1 按法人代码 — host / port 一览

| 序号 | 法人代码 | host | port |
|------|----------|------|------|
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

### 3.2 按端口汇总（便于 Mock 监听）

共 **22 个不同端口**，若做 Mock 需在以下端口监听（accept 后立即关闭即可）：

| port | 使用该端口的法人代码 |
|------|------------------------|
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

## 四、解决方案

### 4.1 方案 A：Mock ERP（临时，推荐用于 ERP 未接通期间）

**思路**：在 R3 配置的 host:port（当前为 localhost + 上表端口）上启动**仅接受连接并立即关闭**的 Mock 服务，使 R3 侧 `readLine()` 因连接关闭而快速返回（或抛异常），请求不再无限卡住。

**要点**：
- 监听地址/端口与 SMSServer2 第 6 参数一致；若只验证 05101 常用法人（如 X），可先只监听 **localhost:64942**。
- 行为：`accept()` 后可选读少量数据（或直接关闭），然后 **立即 close()** 该连接。
- 不发送合法业务数据，R3 调用会以错误结束，但**能快速返回**，避免整系统卡死。

**优点**：不改现有 Java 代码，部署简单，适合过渡。  
**缺点**：R3 相关功能全部失败，需在 ERP 接通后撤掉 Mock。

### 4.2 方案 B：Pipe 增加读超时（中期）

**思路**：修改 **sms_common.util.Pipe**，保存 Socket 引用，在构造或 `open()` 后对 Socket 调用 **`setSoTimeout(毫秒)`**，使 `readLine()` 在超时后抛出 `SocketTimeoutException`，上层捕获后返回错误。

**优点**：不依赖外部 Mock，可配置超时时间。  
**缺点**：需改代码、编译、回归测试。

### 4.3 方案 C：Pipe 支持外部关闭（配合超时或 Mock）

**思路**：在 Pipe 中保存 Socket 并对外提供 **`close()`**，或把 `open()` 返回的 reader 交给“超时/取消”线程；在超时或用户取消时由另一线程调用 **`close()`**，打断阻塞的 `readLine()`。

**优点**：可主动打断长时间等待。  
**缺点**：需改 Pipe 与调用方式，并处理并发与异常。

### 4.4 方案 D：配置开关跳过 R3（长期可选）

**思路**：在 SMSServer2 或 SCPServer 中增加配置项（如“ERP 未启用”），当该开关打开时**不调用 moveOrderToR3**，仅做 SMS 侧处理或直接返回错误。

**优点**：从业务逻辑上避免连 ERP。  
**缺点**：需梳理所有调用 R3 的路径并做配置与测试。

---

## 五、建议实施顺序

| 阶段 | 建议 | 说明 |
|------|------|------|
| **短期** | 采用 **方案 A（Mock ERP）** | ERP 未接通时在需要的端口（至少 localhost:64942 等常用法人端口）起 Mock，accept 后立即关闭，避免请求卡死。 |
| **中期** | 实施 **方案 B（Pipe 读超时）** | 在 `Pipe` 中设置 `setSoTimeout(例如 30 秒)`，减少对 Mock 的依赖，避免长时间阻塞。 |
| **长期** | 视需求考虑 **方案 D（R3 开关）** | 若需在“ERP 停用”时完全不走 R3，再增加配置与分支逻辑。 |

---

## 六、相关代码位置（便于后续修改）

| 内容 | 路径/位置 |
|------|------------|
| Pipe 读超时设置 | `service/common/sms_common/util/Pipe.java`，构造中 `socket.setSoTimeout(0)` |
| R3Server 使用 Pipe | `service/common/sms_common/server/R3Server.java`，多处 `new Pipe(host_, port_).open(...)` 后 `reader.readLine()` |
| scp05101 调 SMS | `service/businese/scp05101/scp05101_s/src/scp05101/scp05101_s.java`，`remoteObject_.insertSCPOrder` / `cancelSCPOrder` |
| SMSServer2 R3 参数解析 | `service/common/sms_common/server/SMSServer_s.java`，`setR3Parameters`，约 1456–1480 行 |
| SCPServer 调 R3 | `service/common/sms_common/server/SCPServer.java`，`createOrder` 中 `server_.moveOrderToR3(r3Hash)` |

---

**报告结束。** 若需对某一方案做详细设计或脚本示例，可在此基础上再补充。
