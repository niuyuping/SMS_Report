# Sms012C8 update 接口 NullPointerException 分析文档

## 1. 问题现象

调用 update 时出现如下日志与异常：

```
call update
update sta= [JA24517AHP][1][未]
updateSms sta= [order]
java.lang.NullPointerException: Cannot load from object array because "s" is null
	at sms012C8.sms012C8_s.getStockType(sms012C8_s.java:860)
	at sms012C8.sms012C8_s.moveOrderR3(sms012C8_s.java:583)
	at sms012C8.sms012C8_s.update(sms012C8_s.java:532)
	...
```

---

## 2. 日志含义与输出条件

### 2.1 实际出错的服务

- 堆栈显示为 **`sms012C8_s`**（Sms012C8 服务），不是 Sms012C3。
- 若通过 SmsWebApi 调用的是「Sms012C3 的 update」，则可能是 012C3 转调 012C8，或实际连接的就是 012C8。

### 2.2 各条日志含义

| 日志内容 | 含义 | 代码位置（sms012C8_s.java） |
|----------|------|-----------------------------|
| `call update` | 进入 `update(prm)` 方法 | 约 404 行 |
| `update sta= [JA24517AHP][1][未]` | 正在处理某一行：entry_no、detail_no、印刷フラグ（如「未」） | 约 479 行 |
| `updateSms sta= [order]` | 对该行执行 SMS 更新，type 为 "order" | 约 654 行 |

### 2.3 会输出这段日志并进入 R 付け 的条件

- 调用了 **Sms012C8 的 `update(prm)`**，且 `prm` 中带有 `KEY1`（Vector，每元素为一行 String[]）。
- 对某一行：`data[3]` 为「未」、且 `corporate` 为「J」时，会做印刷/更新逻辑。
- 随后在 **R 付け** 循环中对该行执行 `moveOrderR3()`。
- 在 `moveOrderR3` 内：**受注区分 `data[4]` 不为 `"2"`**（非見込生産），且数量/贩价等检查通过，且该品目不在 `productStockDivision` 缓存中，则会调用 **`getStockType(corporate, edata[0])`**（edata[0] 为品目 code）。
- **NPE 发生点**：`getStockType` 内部通过 `remoteObject_.getMaster(hash)` 拉取 **ERP 机种マスター**（FLAG=ERP, TYPE=PRODUCT），若返回的 `ans` 中对该 corp+product 的 key 没有对应 value（为 null），则 `s = (String[])ans.get(...)` 为 **null**，下一行执行 `item = s[14]` 时触发 **“Cannot load from object array because "s" is null”**。

因此：**只要某次 update 中有一行满足「需要做 R 付け且该品目需要查在库区分」，就会进入 `getStockType`；若此时 ERP 机种数据取不到，就会在此处 NPE。**

---

## 3. 调用链与 ERP/Pipe 依赖

### 3.1 调用链

```
update(prm)
  → 打印 "call update"
  → 循环 vector，打印 "update sta= ..."，执行 updateSms → 打印 "updateSms sta= ..."
  → R 付け 循环：moveOrderR3(corporate, data, productStockDivision, salesStock)
       → 受注区分 data[4] != "2" 且其他检查通过
       → productStockDivision 中无该品目缓存
       → getStockType(corporate, edata[0])
            → remoteObject_.getMaster(hash)  // FLAG=ERP, TYPE=PRODUCT
            → 返回 ans 中无该 key 或 value 为 null
            → s = (String[])ans.get(...) 为 null
            → item = s[14]  → NullPointerException
```

### 3.2 是否从 ERP 通过 Pipe 读取数据

**结论：是。该请求会通过 getMaster(FLAG=ERP) 从 ERP 侧（经 Pipe）读取机种マスター；ERP 未连接或 Pipe 失败时，会导致此处 NPE。**

依据：

- `getStockType()`（约 811–831 行）中：
  - `hash.put(SMSServerIfc.FLAG, "ERP");`
  - `hash.put(SMSServerIfc.TYPE, SMSMasterServer.PRODUCT);`
  - `hash.put(SMSServerIfc.KEY, keys);`  // keys = [corporate, product]
  - `Hashtable ans = remoteObject_.getMaster(hash);`
  - `String[] s = (String[])ans.get(Utility.getKeyString(keys));`
  - `item = s[14];`  // 此处 s 为 null 时 NPE

- 共通层（如 `R3Server`）对 FLAG=ERP、TYPE=PRODUCT 会调用 **`getERPMasterProduct(key)`**，其内部使用：
  - **`new Pipe(host_, port_).open("74 \"corp\" \"plant\" \"product\" EQ ja")`**
  - 通过 Pipe 向 ERP 进程发命令并读取一行返回。

- 若 **ERP 未连接或 Pipe 不通**：Pipe 可能抛异常、超时或返回空，`getERPMasterProduct` 返回 null（或异常被上层捕获后未向 `ans` 放入该 key），导致 `ans.get(Utility.getKeyString(keys))` 为 null，进而 `s` 为 null，访问 `s[14]` 即 NPE。

---

## 4. 不修改代码时的规避方式

仅通过 **调整调用参数**，使服务端 **不进入会调用 `getStockType` 的分支**，即可避免 NPE。

- 在 `moveOrderR3()` 中，第一道判断为（约 535 行）：
  - `if("2".equals(data[4])) return "";`
  - 即：**受注区分 = "2"（見込生産）时，直接返回，不执行 R 付け，也不会调用 `getStockType`。**

- `update()` 的入参中，`KEY1` 为 Vector，每个元素为一行的 `String[] data`，其中：
  - **`data[4]` = 受注区分**（例如 1=受注，2=見込生産 等）。

**建议：**

通过 SmsWebApi 调用 Sms012C8（或经 012C3 转 012C8）的 update 时，在传入的 **KEY1** 中，对 **每一行** 将 **受注区分 `data[4]` 设为 `"2"`（見込生産）**。这样 `moveOrderR3` 会直接 return，不会查在库区分，也就不会调用 `getStockType`，从而避免因 ERP/Pipe 无数据导致的 NPE。

**注意：** 这样做的业务含义是「这批单都按見込生産处理、不执行 R 付け」。若业务上必须执行 R 付け，则无法在不改代码的前提下仅靠参数规避，必须保证 ERP 连接正常且该品目在 ERP 机种マスター中存在，或修改服务端代码在 `ans.get(...)` 为 null 时做防护。

---

## 5. 小结表

| 项目 | 说明 |
|------|------|
| **日志来源** | Sms012C8 的 `update()` → 每行打印 `update sta=` → `updateSms` 打印 `updateSms sta=` → R 付け 循环中 `moveOrderR3()` → `getStockType()` 中 NPE。 |
| **NPE 直接原因** | `getStockType` 通过 getMaster(FLAG=ERP, 机种) 从 ERP（经 Pipe）取品目主数据；ERP 未连接或该品目无数据时，`ans.get(key)` 为 null，未判空即访问 `s[14]` 导致 NPE。 |
| **是否依赖 ERP/Pipe** | **是**；该请求会经 Pipe 向 ERP 取机种マスター，ERP 未连接或不通会导致此类异常。 |
| **不修改代码的规避** | 调用 update 时，在 KEY1 的每一行中将 **受注区分 `data[4]` 设为 `"2"`**，使 R 付け 逻辑不执行，从而不调用 `getStockType`，避免 NPE。 |

---

## 6. 相关代码位置（sms012C8_s.java）

| 功能 | 行号（约） |
|------|------------|
| `update()` 入口与 "call update" | 399–404 |
| 行循环与 "update sta=" | 470–479 |
| R 付け 循环与 moveOrderR3 调用 | 504–522 |
| moveOrderR3、受注区分判断、getStockType 调用 | 532–560 |
| getStockType（getMaster ERP、s[14]） | 811–831 |
| updateSms 与 "updateSms sta=" | 651–654 |

---

*文档基于当前代码库分析，行号可能随版本略有变化。*
