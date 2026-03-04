# orderInsert 全流程分析与卡点

## 一、整体调用链

```
[1] SmsWeb
    WebSms01206Controller.api/orderInsert
    → WebSms01206ServiceImpl.orderInsert
    → applyOrderInsertDummyAvoidR3(input)   // 预填 7/10/19
    → ImartApiClient.post("orderInsert", input)
         HTTP POST → SmsWebApi

[2] SmsWebApi
    Sms01206Controller.orderInsert
    → SmsWebApiUtil.convertJsonToHashtable(input)
    → remoteObject_.orderInsert(prm)        // RMI 调用

[3] RMI 服务端 - OrderEntryNo_s (你看到日志的进程)
    orderInsert(prm)
    → System.err.println("call orderInsert")
    → System.err.println("insert= [26303ADA][0][1]")
    → getTurnoverMonth(...)                 → System.err.println("getTurnoverMonth= [202603]")
    → isInsertEntry(...)                    → (isInsertEntry= [26303ADA01])
    → getCMIReplyDate(...)                 → System.err.println("CALL getCMIReplyDate [X][X080][20260303][...]")
    → 此处返回 ""，不进入 getLastDeployMaster/getReplyDate
    → remoteObject_.insertOrder(hash)       // ★ 下一步：RMI 调用，日志在此之后消失

[4] 若 insertOrder 被调用（SMSServer2_s）
    insertOrder(prm)
    → System.err.println("call insertOrder2")   // 若看不到，说明卡在 [3]→[4] 的 RMI
    → insertDeployEntryData(type, prm)          // ★ 可能卡点 1
    → orderTable.insert(id, lines)              // ★ 可能卡点 2
    → updatePlanUnitPrice(type, lines)          // ★ 可能卡点 3
    → updateForcast(type, INSERT, lines)        // ★ 可能卡点 4
    → return table
```

你当前日志停在 `CALL getCMIReplyDate` 后，说明**卡在 [3] 里紧接着的 `remoteObject_.insertOrder(hash)` 调用**：要么 RMI 没返回，要么返回前在 [4] 的某一步阻塞。

---

## 二、卡点 0：RMI 调用 insertOrder 本身

- **位置**：`OrderEntryNo_s.java` 约 1611 行  
  `Hashtable ans = remoteObject_.insertOrder(hash);`
- **含义**：当前进程（OrderEntryNo_s）调用了另一对象 `remoteObject_`（通常是 SMSServer2_s）的 `insertOrder`。
- **可能情况**：
  1. **不同 JVM**：RMI 走网络，若对端未起、网络不通或超时，会一直等或超时失败。
  2. **同一 JVM**：若 SMSServer2_s.insertOrder 内部阻塞，这里也会一直拿不到返回值。

**如何确认**：  
看 RMI 服务端（跑 SMSServer2_s 的那台/JVM）是否有 `"call insertOrder2"` 日志。

- 若**从没有** `"call insertOrder2"` → 卡在 **RMI 连接/分发**（对端未收到或未执行到第一行日志）。
- 若有 `"call insertOrder2"` 但后面没有更多日志 → 卡在 **insertOrder 内部**下面某一处。

---

## 三、卡点 1：insertDeployEntryData

- **位置**：`SMSServer2_s.insertOrder` 内  
  `String[][] plant = insertDeployEntryData(type, prm);`
- **流程**：
  - `DeployData.insert(prm)` →
  - 对每条 line：`getDeployMaster(corporate, plant, deployCorp)` →
  - `server_.getMaster(hash)`，type = `PLANT_DEPLOY`（本机 master_.get，可能查库/缓存）
  - 若为「会社間取引」：`insert(master, lines[i])` →
  - 内层再调 **`server_.insertOrder(hash)`**（带 DEPLOY_DATA），即**递归 insertOrder** →
  - 这次会跳过 insertDeployEntryData，直接做 `orderTable.insert` / updatePlanUnitPrice / updateForcast。
- **可能卡住**：
  - `getMaster(PLANT_DEPLOY)` 慢或阻塞（DB/缓存/锁）。
  - 递归的 `insertOrder` 里任意一步慢（尤其是下面的 SQL/updatePlanUnitPrice/updateForcast）。

**日志**：若能看到 `DeployData.getDeployMaster[...]`，说明已进入 insertDeployEntryData，卡在 getMaster 或之后的递归 insertOrder。

---

## 四、卡点 2：orderTable.insert（DB 写入）

- **位置**：`SMSServer2_s.insertOrder` 内  
  `int ans = orderTable.insert(id, lines);`
- **流程**：
  - `OrderTable2.insert(id, lines)` →
  - 每条 line 转成 SQL INSERT，最终 `executeUpdate(id, sql)` →
  - `sql_.executeUpdate(table)`（Table 的 `sql_` 是 SMSSQLServerIfc，多为 **RMI 调 SQL 服务** 或本地 Pipe 到 DB）。
- **可能卡住**：
  - SQL 服务未起、网络不通、超时。
  - 数据库锁、长事务、慢 SQL、死锁。
  - 连接池满，一直等连接。

**日志**：Table2 里有 `System.err.println("Table2 insert call");`，若在 SMSServer2_s 所在进程能看到这条，说明已进入 DB 插入，卡在 SQL 执行或返回。

---

## 五、卡点 3：updatePlanUnitPrice

- **位置**：`SMSServer2_s.insertOrder` 内  
  `updatePlanUnitPrice(type, lines);`
- **流程**：
  - `PlanUnitPrice.update(lines)`，对每条 line：
  - 若缺关键字段：`selectData(...)` → `server_.selectOrder(hash)`（查 order）
  - `selectWakuData(...)`（枠情報）
  - 组 SQL：`server_.executeUpdate(hash)` 更新 plan 相关字段。
- **可能卡住**：
  - `selectOrder` / `executeUpdate` 对应的 SQL 慢、锁表、或对端服务无响应。

---

## 六、卡点 4：updateForcast

- **位置**：`SMSServer2_s.insertOrder` 内  
  `updateForcast(type, OrderTable2.INSERT, lines);`
- **流程**：
  - `ForcastData.update(INSERT, lines)` →
  - 内部 select / insert / update 枠データ，同样通过 Table 的 executeQuery/executeUpdate → `sql_`。
- **可能卡住**：
  - 同上：SQL 执行慢、锁、连接或 RMI 到 SQL 服务阻塞。

---

## 七、结论与建议排查顺序

| 现象 | 最可能卡点 |
|------|------------|
| 只有 OrderEntryNo_s 的日志，完全没有 `"call insertOrder2"` | **卡点 0**：RMI 到 insertOrder 未完成（连接/对端未执行到第一行） |
| 有 `"call insertOrder2"`，没有 `"Table2 insert call"` | **卡点 1**：insertDeployEntryData（getDeployMaster 或递归 insertOrder） |
| 有 `"Table2 insert call"`，之后无新日志 | **卡点 2**：orderTable.insert → sql_.executeUpdate（DB/SQL 服务） |
| 有 Table2 insert 相关日志，但一直不结束 | **卡点 2/3/4**：某一步 SQL 或 RMI 极慢/锁 |

**建议**：

1. **确认日志来源**  
   看 `"call insertOrder2"`、`"Table2 insert call"`、`DeployData.getDeployMaster` 是否和 `"call orderInsert"` 在同一进程/同一日志文件。若 RMI 在另一进程，需要同时看该进程的日志。

2. **在 SMSServer2_s.insertOrder 加简短日志**（若可改 RMI 服务端）  
   在 `insertOrder` 入口、`insertDeployEntryData` 前后、`orderTable.insert` 前后、`updatePlanUnitPrice` 前后、`updateForcast` 前后各打一行日志，确认停在哪一步。

3. **查 SQL / RMI**  
   - 查数据库当前活动会话、长时间运行 SQL、锁等待。  
   - 查 SMSSQLServer（或执行 SQL 的进程）是否存活、是否有对应时间点的请求与错误。

4. **仅改 SmsWeb 时**  
   无法取消 RMI 或服务端内部步骤，只能通过预填（例如已做的 7/10/19）减少后端分支或重算，无法直接避免 insertOrder 内的 DB/R3 等调用；若卡在 RMI 或 SQL，需在 RMI/SQL 侧排查。

---

## 八、流程简图（文字）

```
OrderEntryNo_s.orderInsert
  ├─ getTurnoverMonth        ← 已出现日志
  ├─ isInsertEntry           ← 已出现日志
  ├─ getCMIReplyDate         ← 已出现日志并返回
  └─ remoteObject_.insertOrder(hash)   ← 日志在此之后消失
       │
       └─ SMSServer2_s.insertOrder
            ├─ insertDeployEntryData   ← 可能卡：getMaster / 递归 insertOrder
            ├─ orderTable.insert       ← 可能卡：sql_.executeUpdate
            ├─ updatePlanUnitPrice     ← 可能卡：selectOrder / executeUpdate
            └─ updateForcast           ← 可能卡：select/insert/update 枠
```

上述内容即为「目前代码可能卡在哪里」的完整分析；实际定位时以「是否有 call insertOrder2 / Table2 insert / getDeployMaster」为准判断卡点 0/1/2。

---

## 九、Sms01206 追加按钮 vs WebSms createForm：两个值为何出错

### 9.0 两个值在 Sms01206 中的来源

- **getTransportList 的 key（RMI 端 KEY = String[]）**
  - **来源**：在 `getTransportList2()`（OrderEntryNo出荷依頼表改造.java 约 19584–19592 行）里**就地组装**：
    - `String[] keys = new String[1];`
    - `keys[0] = corporate_;`
  - **corporate_**：画面成员变量，在 **Applet 初始化**时由参数设定（约 3338 行）：
    - `corporate_ = sms_common.AppletParameter.get(this, sms_common.AppletParameter.CORPORATE_CODE);`
  - 即：**法人代码**来自 **Applet 启动参数**（CORPORATE_CODE），原版每次调 getTransportList 时都用这个 corporate_ 组成 `key = [corporate_]`。

- **getConporison 的 key（RMI 端 KEY = String[]）**
  - **来源**：在 `judgementConporison(String product)`（约 13509–13519 行）里**就地组装**：
    - `String[] keys = new String[2];`
    - `keys[0] = corporate_;`  // 同上，Applet 参数
    - `keys[1] = product;`    // 方法入参
  - **product**：**机种代码**，由**调用方**传入。调用场景包括：
    - 检索后设表数据时：`judgementConporison(data[PRODUCT_HEAD])`，product = 表头/检索结果里的机种列（约 4656 行）；
    - 机种变更/设定时：`setErpProduct(product)` 里 `judgementConporison(product)`，product = `getNewProduct()` 等取得的当前机种（约 10588 行）；
    - 其他：表头 key 取得机种列再传 `judgementConporison(product)`（约 18133 行）。
  - 即：**key = [法人代码, 机种代码]**，法人来自 Applet 参数，机种来自当前画面/表数据。

- **lang_**（两个接口都传）：同样在 Applet 初始化时设定（约 3292–3295 行），`lang_ = getParameter(ct_a.P_Language)`，空时默认 `"ja"`。

因此，在 Sms01206 里这两个 key **不是从 orderInsert 或追加按钮来的**，而是：
- getTransportList 的 key：**画面初始化时的法人参数** → `[corporate_]`；
- getConporison 的 key：**同一法人 + 当前操作的机种** → `[corporate_, product]`。

---

### 9.1 原版 Sms01206 点击「追加」后的逻辑

1. **入口**  
   `OrderEntryNo出荷依頼表改造.java`（或 OrderEntryNo_240522 等）：  
   `insertButton___action_actionPerformed` → `insert("insertButton")`。

2. **orderInsert 的入参**（约 7037–7047 行）  
   - `prmHash.put(OrderEntryNo_i.KEY, beforeHash_);`  // KEY = 整表 Hashtable（表头 + 明細）  
   - `prmHash.put(OrderEntryNo_i.KEY6, inComent1);`   // KEY6 = `"insertButton"`  
   - 以及 CORPORATE, TYPE, LANG, OUTIN 等。  
   - 即：**追加时只调 orderInsert，KEY 是 beforeHash_（表单数据），格式正确。**

3. **getTransportList 的调用时机与入参**（非追加时，画面用）  
   - 在 `getTransportList2()`（约 19584–19592 行）中调用。  
   - 入参：`KEY = new String[] { corporate_ }`，`LANG = lang_`。  
   - 即：**原版始终传了 KEY（长度为 1 的 String 数组，元素为法人）。**

4. **getConporison 的调用时机与入参**（非追加时，机种判定用）  
   - 在 `judgementConporison(String product)`（约 13509–13519 行）中调用。  
   - 入参：`KEY = new String[] { corporate_, product }`，`LANG = lang_`。  
   - 即：**原版始终传了 KEY（长度为 2 的 String 数组：法人 + 机种）。**

### 9.2 WebSms createForm 的对应逻辑

1. **追加按钮**  
   - JSP：`@click="saveOrder"` → `websms01206-createForm.js` 的 `saveOrder()` → `validateForm` → `checkOrderNoDuplication` → `doOrderInsert()`。

2. **orderInsert 的入参**（doOrderInsert，约 1123 行）  
   - `body = { key: this.buildKeyForInsert(), outin: {}, key6: 'insertButton', lang: 'ja', corporate: ..., type: 'order' }`  
   - `buildKeyForInsert()` = `buildKey(formData, currentSubNo)`（`sms01206-order-key.js`），得到 H0/B2/…/D01 结构的 **key 对象**，与原版 beforeHash_ 等价。  
   - 即：**追加时 orderInsert 的 key 是正确的，不会导致 RMI 端 NPE。**

3. **getTransportList / getConporison 的调用时机与入参**  
   - 在 **画面打开时** 的 `loadMasterData`（约 480、490 行）中调用：  
     - `getTransportList`: `axios.post(api + '/getTransportList', { lang: 'ja', corporate: self.corporate || 'X' })`  
     - `getConporison`: `axios.post(api + '/getConporison', { lang: 'ja', corporate: (self.corporate || 'X') })`  
   - 两者都**没有传 `key`**，只传了 `lang` 和 `corporate`。

### 9.3 两个值出错的原因（对应日志中的两个 NPE）

| 接口 | 原版 Sms01206 传的 KEY | WebSms createForm 传的 KEY | RMI 端行为 | 结果 |
|------|------------------------|----------------------------|------------|------|
| **getTransportList** | `String[] { corporate_ }` | **未传（缺失）** | `keys = (String[])prm.get(KEY)` → `null`，随后 `tbl.put(KEY, keys)` | **Hashtable.put 不允许 value=null → NPE**（OrderEntryNo_s.java:4113） |
| **getConporison**     | `String[] { corporate_, product }` | **未传（缺失）** | `keys = (String[])prm.get(KEY)` → `null`，随后 `hash.put(KEY, keys)` | **同上 → NPE**（OrderEntryNo_s.java:3617） |

- **出错的两个值**：就是 **getTransportList** 和 **getConporison** 的入参 **`key`**（RMI 端期望的 `KEY` 对应 String[]）。  
- 原版在调用这两个接口时都会组好并传入 KEY；WebSms 在画面加载时调用时没有传 key，导致 RMI 端 `prm.get(KEY)` 为 null，再 `put(KEY, null)` 触发 NPE。  
- 这两个请求发生在 **CreateForm 打开/加载主数据时**（与是否点击追加无直接关系）；若在打开画面后立刻点追加，日志里会同时看到这两个 NPE 和 orderInsert 相关日志（如 isInsertEntry double err）。

### 9.4 小结

- **orderInsert 的 key**：WebSms 用 `buildKeyForInsert()` 组 key，格式正确，不是 NPE 原因。  
- **getTransportList / getConporison 的 key**：WebSms 在 loadMasterData 中未传 key，RMI 端收到 null 后 put 进 Hashtable 导致两个 NPE。  
- **修复**（仅改 WebSms）：在 `WebSms01206ServiceImpl` 的 `getTransportList`、`getConporison` 中，若请求里没有 `key`，则用 `corporate` 补全 `key`（例如 `key = [corporate]`），再透传给 RMI，避免 put(null)。  
- **Exception isInsertEntry double err**：表示订单号重复（业务校验），与上述两个 NPE 无关；可在前端根据 orderInsert 返回做重复提示或防重复提交。
