# SmsWebApiUtil 中 Vector 目标 key 的 JSON 数组字符串对应

## 概要

在 IntraMart 环境下调用 Sms01288 的 Save 接口时，对 `key8` 参数进行转换会出现「String 向 Vector 转换异常」。本文档说明该问题的原因及解决方案。

## 现象

- **对象 API**：`/sms01288/save`
- **发生环境**：运行于 IntraMart 上的 SmsWebApi
- **错误内容**：在 `key8` 参数转换时发生 `ClassCastException`
  - `java.lang.String cannot be cast to java.util.Vector`

## 原因分析

### 1. 期望的请求形式

正常情况下，客户端发送的 JSON 如下：

```json
{
  "key": {
    "key1": {...},
    "key2": {...},
    "key8": []
  },
  "key7": [],
  "keys": [["d_supply_code", ""], ...]
}
```

此时，Jackson 会将 `key8` 解析为 `List`（空的 `ArrayList`），再经 `convertListToVector` 转为 `Vector`。

### 2. IntraMart/Bloom Maker 环境下的差异

经由 IntraMart 的 Web API Maker 或 Bloom Maker 时，请求的传递方式可能有所不同：

- 使用 `JSON.stringify` 将复杂对象序列化为字符串后发送
- 通过表单参数等方式传递时，数组值可能被二次编码为字符串 `"[]"`
- 代理或网关进行再编码

因此，`key8` 有时会以**字符串 `"[]"`** 的形式到达，而不是数组 `[]`。

### 3. 既有代码的行为

`deserializeAnalyzeJson` 仅对 JSON 对象形式的字符串 `{...}` 进行二次解析：

```java
// isJsonLike: 仅当以 '{' 开头且以 '}' 结尾时视为解析对象
return trimmed.startsWith("{") && trimmed.endsWith("}");
```

JSON 数组 `"[]"` 以 `[` 和 `]` 包围，不满足 `isJsonLike` 条件，**不会被再次解析，而保持为 String**。

在 `convertMapToHashtable` 中，当 `value instanceof String` 时，会直接放入 Hashtable：

```java
} else if (value instanceof String || value instanceof Boolean || value instanceof Number) {
    prm.put(key, value);
}
```

因此，RMI 服务端（`Sms01288_s.save`）执行以下代码时会发生 `ClassCastException`：

```java
Vector changeVector = (Vector) prm2.get(KEY8);  // 无法将 String "[]" 强转为 Vector
```

### 4. 为何 WebApi_Spring 中不发生

WebApi_Spring 中，通过标准的 `@RequestBody` 接收原始的 HTTP body。  
在以 `Content-Type: application/json` 发送正确 JSON 的前提下，Jackson 会将 `key8` 解析为 `List`，因此本问题不会发生。

而 IntraMart 环境下的请求路径和格式不同，可能出现上述 `key8` 以字符串 `"[]"` 形式到达的情况。

## 解决方案

### 实现内容

在 `convertMapToHashtable` 和 `convertMapToHashtableIncludeKey` 中，对满足下列条件的值增加将 JSON 数组字符串解析并转换为 Vector 的处理：

- 对应 key 在 `vectorKeys` 中，或 `isVector` 为 true
- 值为 `String` 类型
- 字符串符合 JSON 数组形式（以 `[` 开头、以 `]` 结尾）

### 新增方法

```java
/**
 * 判断字符串是否为需解析的 JSON 数组形式。
 * 用于 IntraMart/Bloom Maker 环境下，Vector 目标 key 以 JSON 数组字符串形式到达时的解析。
 */
private static boolean isJsonArrayLike(String str)
```

### 处理流程

1. 使用 `isJsonArrayLike` 判断字符串是否以 `[` 开头、以 `]` 结尾
2. 满足条件时，用 `ObjectMapper.readValue(str, List.class)` 解析为 List
3. 使用现有的 `convertListToVector` 转换为 Vector
4. 解析失败时，保留原始 String，并输出 warning 日志

### 修改文件

- `SmsWebApi/src/main/java/jp/co/nidec/smswebapi/rmi/util/SmsWebApiUtil.java`
  - 在 `convertMapToHashtable` 中增加 JSON 数组字符串的解析处理
  - 在 `convertMapToHashtableIncludeKey` 中增加同样处理
  - 新增 `isJsonArrayLike` 方法

## 影响范围

- **对象 key**：`keys`、`key7`、`key8` 等，在 `vectorKeys` 中注册的 key
- **对象 API**：Sms01288 save，以及所有通过指定 `vectorKeys` 调用 `convertJsonToHashtable` 的 API
- **向后兼容性**：当以正常数组形式 `[]` 到达时，仍按 List 处理，不影响既有行为

## 验证方法

可使用 WebApi_Spring 的 IntraMart 模拟功能（请求头 `X-Simulate-Intramart: true`）模拟 JSON 数组被字符串化的情况。  
修复后，在模拟场景下也应能确认 `key8` 被正确转换为 Vector。
