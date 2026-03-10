# websms01288 同梱品设定・修正画面 处理流程

本文档说明从 `WebSms01288Controller` 到 `index.jsp` 的请求处理与画面初始化流程。

---

## 1. 路由与入口

### 1.1 URL 映射

| URL | 方法 | Controller 方法 | 说明 |
|-----|------|-----------------|------|
| `/websms01288/maintenance` | GET | `root()` | 重定向到 index |
| `/websms01288/maintenance/index` | GET | `index()` | 显示画面 |

### 1.2 配置文件

**smsweb.properties**（`SmsWeb/src/main/conf/message/smsweb/smsweb.properties`）:

```properties
sms.module01288.name=sms01288
sms.module01288.route.index=spring/websms01288/maintenance/index.jsp
sms.module01288.redirect.index=redirect:/websms01288/maintenance/index
```

Controller 的 `@PostConstruct` 通过 `MessageManager` 读取上述配置，将 JSP 路径设置到 `pathIndex`。

---

## 2. Controller 层（WebSms01288Controller.java）

### 2.1 index 方法处理

```java
@RequestMapping(value = "index", method = RequestMethod.GET)
public String index(
        @RequestParam(value = "entryNo", required = false) String entryNo,
        @RequestParam(value = "corporate_code", required = false) String corporateCode,
        @RequestParam(value = "table_type", required = false) String tableType,
        Model model,
        HttpServletRequest request) {
    if (entryNo != null && !entryNo.isEmpty()) {
        model.addAttribute("entryNo", entryNo);
    }
    model.addAttribute("corporate", (corporateCode != null && !corporateCode.isEmpty()) ? corporateCode : "X");
    model.addAttribute("type", (tableType != null && !tableType.isEmpty()) ? tableType : "order");
    return pathIndex;
}
```

### 2.2 启动参数

| 参数 | 必填 | 默认值 | Model 键 | 说明 |
|------|------|--------|----------|------|
| `entryNo` | 否 | - | `entryNo` | 初始显示的エントリーNo |
| `corporate_code` | 否 | `"X"` | `corporate` | 法人代码（传给 API 的 corporate） |
| `table_type` | 否 | `"order"` | `type` | 表类型（传给 API 的 type） |

### 2.3 返回值

返回 `pathIndex`（`spring/websms01288/maintenance/index.jsp`），由 Spring MVC 将该 JSP 作为 View 渲染。

---

## 3. 向 View 层（index.jsp）传递

### 3.1 Model → JSP 的传递

Controller 通过 `model.addAttribute()` 设置的值，在 JSP 中可如下引用：

| Model 键 | JSP 引用 | 用途 |
|----------|----------|------|
| `entryNo` | `${entryNo}` | 启动时自动检索用エントリーNo |
| `corporate` | `${corporate}` | API 调用时的 corporate 参数 |
| `type` | `${type}` | API 调用时的 type 参数 |

### 3.2 JSP 中的接收

**body 的 data 属性**（用于启动参数 entryNo）:

```html
<body data-initial-entry-no="<c:out value='${entryNo}' />">
```

**JavaScript 变量**（用于 API 调用）:

```javascript
var corporate = '<c:out value="${corporate}" default="X"/>';
var tableType = '<c:out value="${type}" default="order"/>';
```

---

## 4. 画面初始化流程（index.jsp）

### 4.1 整体流程

```
页面加载
    │
    ├─ HTML 解析（body data-initial-entry-no、各组件）
    │
    ├─ 脚本开始执行
    │   │
    │   ├─ 设置 contextPath、corporate、tableType
    │   │
    │   └─ Promise.all([ initChargeList(), initCustomerList(), initPCList(), initMakerList(), initVari() ])
    │           │
    │           │  ※ 5 个 API 并行获取基础信息
    │           │     - getCharges（担当）
    │           │     - getCustomers（得意先）
    │           │     - getPriceCodeMaster（販価区分）
    │           │     - getMakerMaster（メーカー）
    │           │     - getVari（回答区分・通貨・扱い区分・ルート区分）
    │           │
    │           └─ .then() → initTable() → initFormEvents()
    │                   │
    │                   ├─ 绑定表单事件
    │                   │
    │                   └─ 若存在 data-initial-entry-no
    │                           │
    │                           ├─ 设置 entryNoField 的值
    │                           └─ setTimeout(0) → btnRead.click()
    │                                   │
    │                                   └─ doRead() → doRefference() → getDetail API
    │                                           │
    │                                           └─ applyDetailData() → 数据渲染到画面・enableForm
    │
    └─ DOMContentLoaded → showBody() → 显示 body.ready
```

### 4.2 基础信息获取（Promise.all）

以下 5 个初始化函数**并行**调用 API，全部完成后执行 `initTable()` 和 `initFormEvents()`。

| 函数 | API | 获取数据 |
|------|-----|----------|
| `initChargeList()` | getCharges | 担当者列表（datalist） |
| `initCustomerList()` | getCustomers | 得意先列表（select） |
| `initPCList()` | getPriceCodeMaster | 販価区分列表 |
| `initMakerList()` | getMakerMaster | 使用ユーザー・装置メーカー |
| `initVari()` | getVari | 回答区分・通貨・扱い区分・ルート区分 |

### 4.3 存在启动参数 entryNo 时的自动检索

1. 在 `initFormEvents()` 中读取 `data-initial-entry-no`
2. 将值设置到 `entryNoField`
3. 通过 `setTimeout(0)` 在下一事件循环执行 `btnRead.click()`
4. 与用户点击「検索」按钮的行为完全一致
5. 经由 `doRead()` → `doRefference()` → `getDetail` API → `applyDetailData()` 完成数据展示和表单启用

---

## 5. API 调用时的参数对应

画面内各 API 调用使用从启动参数得到的 `corporate` 和 `tableType`。

| 启动参数 | 前端变量 | API 参数名 |
|----------|----------|------------|
| corporate_code | `corporate` | `corporate` |
| table_type | `tableType` | `type` |

示例（getDetail）:

```javascript
body: JSON.stringify({
  lang: 'ja',
  type: tableType,
  corporate: corporate,
  key: entry,
  key1: ITEMS_,
  key2: priceOption
})
```

---

## 6. 访问示例

### 6.1 基本访问

```
/websms01288/maintenance/index
```

### 6.2 指定法人・表类型

```
/websms01288/maintenance/index?corporate_code=X&table_type=order
```

### 6.3 指定エントリーNo（自动检索）

```
/websms01288/maintenance/index?corporate_code=X&table_type=order&entryNo=25206ARR
```

### 6.4 完整 URL 示例（context path 为 /SmsWeb 时）

```
http://localhost:8080/SmsWeb/websms01288/maintenance/index?corporate_code=X&table_type=order&entryNo=25206ARR
```

