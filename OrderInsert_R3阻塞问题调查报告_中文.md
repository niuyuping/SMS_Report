# OrderInsert R3 阻塞问题调查报告

**报告日期**：2025-02-26  
**问题现象**：InsertOrder 在 R3 服务未启动时阻塞约 2.5 分钟后仍能返回结果  
**涉及模块**：SMS 订单插入、R3/ERP 桥接、Pipe 通信

---

## 一、问题背景

### 1.1 现象描述

- WebSms 调用 `getTurnoverDate` 接口时，请求长时间无响应
- `insertOrder` 调用在 R3 服务未启动的情况下，阻塞约 2.5 分钟后仍能返回成功
- 已确认 R3 服务未启动

### 1.2 相关系统架构

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
                    Pipe(host, port) → R3 桥接进程
```

---

## 二、调查过程

### 2.1 阻塞点定位

通过日志与代码分析，定位到：

- **OrderEntryNo_L.err**：`getTurnoverDate [X][17504][20260303]`
- **SMSServer2.err**：`getTurnoverDate-1 call` → `getCustomerReadTime` → `Customer2.getCustomerListERP` → `SMSServer2.getERPMasters` → `R3Server.getERPMasters [customer_list_erp]`
- **R3Server**：通过 `Pipe(host_, port_).open()` 连接 R3 桥接进程，`reader.readLine()` 等待返回

### 2.2 关键代码分析

#### 2.2.1 Pipe 构造函数（核心阻塞点）

**文件**：`sms_common/util/Pipe.java`

```java
public static int RETRY_CNT = 300;

public Pipe(String host, int port) {
    for(int i = 0; i < RETRY_CNT; i++){
        try {
            Socket socket = new Socket(host, port);
            // ... 初始化 recv_, send_
            break;
        } catch (Exception exception) {
            if(i == RETRY_CNT-1) exception.printStackTrace();
        }
        try{
            System.err.println("Pipe retry..."+i);
            Thread.currentThread().sleep(500);  // 每次失败休眠 500ms
        } catch (Exception exception2){
            exception2.printStackTrace();
        }
    }
}
```

- R3 未启动时，`new Socket(host, port)` 快速失败（Connection refused）
- 每次失败后 `sleep(500)` ms
- 最多重试 300 次  
**总阻塞时间：300 × 500ms ≈ 150 秒（约 2.5 分钟）**

#### 2.2.2 300 次失败后的处理

300 次均失败时，`recv_` 与 `send_` 未被初始化，保持为 `null`。

随后调用 `Pipe.open(cmd)`：

```java
public BufferedReader open(String command) {
    try {
        send_.write(command);  // send_ 为 null → NullPointerException
        send_.write(EOD);
        send_.flush();
    } catch (IOException exception) {
        exception.printStackTrace();
    }
    return recv_;
}
```

`send_` 为 null 会触发 `NullPointerException`，该异常被上层捕获并转化为“返回 null / 空表”。

#### 2.2.3 异常处理链路

| 层级 | 类/方法 | 异常处理 | 返回行为 |
|------|---------|----------|----------|
| 1 | R3Server.getERPMasterCustomers | catch Exception | 返回 (String[][])null |
| 2 | SMSServer2_s.getERPMasters | catch Exception | 返回 null |
| 3 | Customer2.getCustomerListERP | erp 为 null 导致 NPE，catch | 返回空 Hashtable |
| 4 | getCustomerReadTime | catch Exception | readTime=0，返回 aHash |
| 5 | getTurnoverDate | 使用 readTime=0 等默认值 | 继续计算日期 |
| 6 | getInspectDate | catch Exception | return "" |
| 7 | insertOrder | catch Exception | 继续后续逻辑，不中断 |

---

## 三、insertOrder 调用链梳理

### 3.1 与 R3 相关的执行路径

**触发条件**：`isJapan(ccode)` 且 `handle_code=="1"` 且 `d_on_board_date` 或 `d_inspect_date` 为空

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
                                                          ├── 构造函数：300 次重试 × 500ms ≈ 2.5 分钟
                                                          └── open()：NPE → 被 catch
            } catch(Exception) {
              exception.printStackTrace();  // 异常被吞掉
            }
```

### 3.2 insertOrder 整体流程概览

```
insertOrder
├── 1. 収益認識対応（売上日・検収日自動設定）  ← R3 阻塞发生在此
├── 2. 標準販価再取得（getMasterUnitPrice）
├── 3. OUT-IN 编辑（editOutInData）
├── 4. insertDeployEntryData
├── 5. getPlantHandle / getCustomerDeployConsign
├── 6. orderTable.insert（DB 插入）
├── 7. updatePlanUnitPrice / updateForcast
├── 8. insertOutInData
└── 9. OutInCustomer_.insertOutInCustomer
```

R3 调用失败后，上层使用默认值（如 readTime=0、空日期），后续 DB 插入等步骤照常执行。

---

## 四、根因结论

### 4.1 阻塞时间长的原因

| 因素 | 说明 |
|------|------|
| Pipe 重试机制 | RETRY_CNT=300，每次失败后 sleep 500ms |
| 计算公式 | 300 × 500ms = 150 秒 ≈ 2.5 分钟 |
| 适用场景 | R3 桥接进程未启动或端口无监听 |

### 4.2 仍能返回结果的原因

| 因素 | 说明 |
|------|------|
| 多层 try-catch | 从 R3Server 到 insertOrder 均有异常捕获 |
| 默认值兜底 | getCustomerReadTime 在空表时返回 readTime=0 |
| 流程不中断 | insertOrder 在 catch 中仅打印异常，继续执行 |
| 业务容忍 | 売上日/検収日使用默认值，订单仍能插入 DB |

---

## 五、时间线（R3 未启动时典型情况）

| 时刻 | 事件 |
|------|------|
| T=0s | insertOrder 调用 getInspectDate |
| T=0s | 调用链到达 R3Server.getERPMasterCustomers |
| T=0s | `new Pipe(host, port)` 开始 |
| T=0~150s | Pipe 构造函数：300 次重试，每次 sleep 500ms，打印 "Pipe retry...N" |
| T≈150s | 第 300 次失败，Pipe.open() 触发 NPE |
| T≈150s | 异常沿调用链向上传播，各层 catch 并返回 null/空表/默认值 |
| T≈150s | getCustomerReadTime 使用 readTime=0 继续 |
| T≈150s | getTurnoverDate / getInspectDate 用默认值返回 |
| T≈150s | insertOrder 继续执行，完成 DB 插入并返回 |

---

## 六、改进建议

### 6.1 短期措施（降低阻塞时间）

1. **减少重试次数**：将 `Pipe.RETRY_CNT` 从 300 调整为 3~5  
2. **缩短休眠时间**：将 sleep 从 500ms 调整为 100ms  
3. **效果预估**：阻塞时间从约 2.5 分钟降至 0.5~3 秒

### 6.2 长期措施（增强健壮性）

1. **连接超时**：使用 `Socket.connect(addr, connectTimeout)` 限制连接等待时间  
2. **读取超时**：对 `socket.setSoTimeout(readTimeout)` 避免 readLine 无限等待  
3. **R3 可用性检测**：启动或定期检查 R3 桥接进程，提前告警  
4. **降级策略**：R3 不可用时，明确使用本地/默认数据并记录日志，避免长时间阻塞  

---

## 七、附录

### 7.1 涉及文件列表

| 文件路径 | 说明 |
|----------|------|
| sms_common/util/Pipe.java | Pipe 连接与重试逻辑 |
| sms_common/server/R3Server.java | R3 接口封装，getERPMasterCustomers 等 |
| sms_common/server/SMSServer2_s.java | insertOrder、getInspectDate、getTurnoverDate |
| sms_common/master/Customer2.java | getCustomerListERP |

### 7.2 相关常量

- `Pipe.RETRY_CNT`：300  
- Pipe 构造函数中 `Thread.sleep`：500ms
