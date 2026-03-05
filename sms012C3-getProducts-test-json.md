# 012C3 getProducts 测试 JSON（SmsWebApi 用）

通过 SmsWebApi 的 **POST /sms012C3/getProducts** 发送时，请求体为 JSON，键名与 RMI 参数对应：`key`、`lang`、`type`、`corporate`、`key1`（可选）。

---

## 说明：012C3 与 R3Server

在现有 RMI 实现（`sms012C3_s.getProducts`）中：

- **不传 key1**：走默认分支，会带 `FLAG "ERP"` 或 `TYPE PRODUCT_LIST_ERP` 调用 `getMaster`，会经 **R3Server/Pipe 查 ERP**。
- **传 key1（代理店用）**：走 `getProductsDairiten`，内部使用 `CUSTOMER_PRODUCT_ERP2`，仍会调用 **R3Server**（`getERPMasters` 等）。

因此**当前 012C3 的 getProducts 没有“完全不走 R3Server”的分支**。若 R3 未配置或未启动，可能返回空或异常。  
若需要**完全不经过 R3** 的机种检索，可改用 **01206 的 getProducts**（先查 RDB，仅在一定条件下再查 ERP）。

下面给出两组**合法、可用的测试 JSON**，用于验证接口与参数形态；实际是否访问 R3 取决于环境配置。

---

## 1. 默认检索（按机种/法人，会走 ERP）

用于「不传 key1」的路径：按法人 + 机种条件查，后端会走 PRODUCT / PRODUCT_LIST_ERP（经 R3）。

```json
{
  "key": "X12345",
  "lang": "ja",
  "type": "order",
  "corporate": "X"
}
```

- **前方一致（模糊）**：key 首字符为 `%` 时走 PRODUCT_LIST_ERP，例如：  
  `"key": "%78T"`

---

## 2. 代理店用检索（传 key1，仍会走 ERP）

传 `key1` 为得意先代码数组时，走代理店分支 `getProductsDairiten`（内部仍用 CUSTOMER_PRODUCT_ERP2，会经 R3）。

```json
{
  "key": "X12345",
  "lang": "ja",
  "type": "order",
  "corporate": "X",
  "key1": ["C001"]
}
```

多个得意先时：

```json
{
  "key": "%78T",
  "lang": "ja",
  "type": "order",
  "corporate": "X",
  "key1": ["C001", "C002"]
}
```

---

## 3. 最小必填（仅必填项）

`key` 至少 2 字符（首字符可为 `%`），否则 `key.substring(1)` 可能异常。

```json
{
  "key": "X1",
  "lang": "ja",
  "corporate": "X"
}
```

---

## 4. 调用示例（curl）

```bash
# 默认检索
curl -X POST "http://<host>:<port>/sms012C3/getProducts" \
  -H "Content-Type: application/json" \
  -d '{"key":"X12345","lang":"ja","type":"order","corporate":"X"}'

# 代理店用
curl -X POST "http://<host>:<port>/sms012C3/getProducts" \
  -H "Content-Type: application/json" \
  -d '{"key":"X12345","lang":"ja","type":"order","corporate":"X","key1":["C001"]}'
```

---

## 参数一览

| 键名       | 类型     | 必填 | 说明 |
|------------|----------|------|------|
| key        | string   | 是   | 检索键：法人 1 字符 + 机种（如 `X12345` 或 `%78T` 表示前方一致） |
| lang       | string   | 是   | 语言，如 `ja` |
| corporate  | string   | 是   | 法人代码，如 `X` |
| type       | string   | 否   | 类型，如 `order` |
| key1       | string[] | 否   | 有值时走代理店分支；数组为得意先代码，如 `["C001"]` |

**注意**：以上 JSON 均可正常进入 012C3 getProducts 逻辑；是否访问 R3Server 由 RMI 端与 R3 配置决定，当前实现下两条分支都会涉及 ERP/R3。
