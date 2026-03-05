# SmsWeb 项目中添加 012C3 功能 — 指导手册

本文档说明在 SmsWeb 项目中新增 **012C3** 功能时所需的基础配置、文件清单、代码模式及注意事项，便于按既有模块（01206、01288）一致的方式完成开发与集成。

---

## 一、项目与依赖前提

### 1.1 项目结构概览

| 路径 | 用途 |
|------|------|
| `src/main/java/spring/` | Java 源码，包根为 `spring` |
| `src/main/resources/META-INF/spring/` | Spring 配置（当前仅 `applicationContext-main.xml`） |
| `src/main/conf/message/smsweb/` | MessageManager 用 properties（路由、API 名、View Bean 名等） |
| `src/main/webapp/WEB-INF/views/spring/` | JSP 视图，按模块子目录（如 `websms01206/`、`websms01288/`） |
| `src/main/webapp/static/` | 静态资源，按模块与 `common/` 分子目录 |

### 1.2 模块依赖（module.xml）

- SmsWeb 依赖 **mypackage.SmsWebApi**（verified-version 1.0.0）。
- 012C3 的后端 API 需在 **SmsWebApi** 侧实现对应路径（如 `sms012c3`），本手册只涉及 SmsWeb 侧配置与代码。

### 1.3 共通配置（无需为 012C3 单独改）

- **Spring**：`applicationContext-main.xml` 已对 `spring.app`、`spring.domain.service`、`spring.common` 做 component-scan，新 Controller/Service 只要放在对应包下即可被扫描。
- **请求映射**：无项目内 `web.xml`；URL 由 Spring MVC 的 `@RequestMapping` 提供，无需新增 Servlet 映射。

---

## 二、添加 012C3 必须完成的配置与文件

### 2.1 属性配置（必做）

**文件**：`src/main/conf/message/smsweb/smsweb.properties`

在文件末尾追加 012C3 用键值（与 01288 风格一致时示例）：

```properties
# ========== websms012c3 ==========
sms.module012c3.name=sms012c3
sms.module012c3.route.index=spring/websms012c3/maintenance/index.jsp
sms.module012c3.redirect.index=redirect:/websms012c3/maintenance/index
```

说明：

- **sms.module012c3.name**：与 SmsWebApi 侧 API 路径一致（如 `sms012c3`），Service 中通过 MessageManager 读取，用于 `SmsApiClient` 的 `apiPath`。
- **sms.module012c3.route.***：各画面对应的 JSP 路径（相对 `WEB-INF/views/`），与下方「JSP 文件」一致。
- **sms.module012c3.redirect.***：需要重定向时使用（如根路径打到 index）。

若 012C3 有多画面或 Excel 等 Bean 名 View，可继续追加，例如：

```properties
sms.module012c3.route.list=spring/websms012c3/maintenance/list.jsp
sms.module012c3.bean.someDownload=someDownloadView
```

### 2.2 新建 Java 类与包（清单）

| 类型 | 建议路径/包名 | 说明 |
|------|----------------|------|
| Controller | `spring.app.websms012c3.WebSms012C3Controller` | `@Controller` + `@RequestMapping("/websms012c3/maintenance")`（建议与 01206、01288 统一使用前导 `/`） |
| Service 接口 | `spring.domain.service.websms012c3.WebSms012C3Service` | 业务方法定义 |
| Service 实现 | `spring.domain.service.websms012c3.WebSms012C3ServiceImpl` | `@Service`，内部用 `SmsApiClient` 调 SmsWebApi 的 `sms012c3`（apiPath 从 `sms.module012c3.name` 读取） |
| DTO | `spring.domain.service.websms012c3.dto.*` | 每个 API 一对 Request/Response DTO（推荐 01288 风格）；若采用 01206 风格则可主要用 `Map` + 少量 Form |
| Form（可选） | `spring.app.websms012c3.WebSms012C3_Dto_XXXForm` | 有表单画面时使用 |

若 012C3 需要自定义 Excel 等 View：在 `spring.domain.service.websms012c3.util.view` 下增加 View 类，并在同包或共通 `ViewConfig` 中 `@Bean` 注册，并在 properties 中增加 `sms.module012c3.bean.xxx`。

### 2.3 Spring 配置（通常无需改）

- **applicationContext-main.xml**：已 component-scan 上述包，**无需修改**。
- 若 012C3 有**独有 View Bean**：在 012c3 的 `ViewConfig`（或现有 01206 的 ViewConfig）中增加 `@Bean`，并保证该配置类所在包在 `spring.domain.service` 下以便被扫描。

### 2.4 JSP / 视图文件（必做）

- **目录**：`src/main/webapp/WEB-INF/views/spring/websms012c3/maintenance/`
- **最少**：与 properties 中 `sms.module012c3.route.*` 对应的 JSP（如 `index.jsp`）。
- 若有对话框：可建子目录 `dialogs/`，与 01206/01288 类似。
- 若使用 Bean 名 View（如 Excel 下载）：在 ViewConfig 中注册 Bean，Controller 返回该 Bean 名，并在 properties 中配置 `sms.module012c3.bean.xxx`。

### 2.5 URL 与菜单

- **URL**：由 Controller 的 `@RequestMapping` 决定。例如基路径 `websms012c3/maintenance` 时，首页可为 `…/websms012c3/maintenance/index`，根路径可重定向到 index。
- **菜单**：在 **Intramart 平台** 的菜单/权限中配置指向上述 URL 的菜单项；SmsWeb 项目内无菜单定义文件。

---

## 三、代码模式与实现要点

### 3.1 Controller 层（以 01288 风格为例）

- **类注解**：`@Controller`，无共同基类。
- **类级映射**：`@RequestMapping("/websms012c3/maintenance")`（建议与 01206、01288 统一使用前导 `/`）。
- **视图/重定向**：在 `@PostConstruct` 中通过 `MessageManager.getInstance().getMessage("sms.module012c3.route.xxx")` 等取字符串，赋给成员变量，方法内 `return pathXxx` 或 `return redirectXxx`。
- **MessageManager 封装**：建议与 01288 相同，使用静态方法封装 `getMessage(key)`，异常时返回空字符串，避免 NPE。

示例骨架：

```java
@Controller
@RequestMapping("/websms012c3/maintenance")
public class WebSms012C3Controller {

    @Autowired
    private WebSms012C3Service webSms012C3Service;

    private String pathIndex;
    private String redirectIndex;

    @PostConstruct
    private void initViewPaths() {
        pathIndex = getMessage("sms.module012c3.route.index");
        redirectIndex = getMessage("sms.module012c3.redirect.index");
    }

    private static String getMessage(String key) {
        try {
            String v = MessageManager.getInstance().getMessage(key);
            return (v != null && !v.trim().isEmpty()) ? v.trim() : "";
        } catch (Exception e) {
            return "";
        }
    }

    @RequestMapping(value = "index", method = RequestMethod.GET)
    public String index(Model model, HttpServletRequest request) {
        return pathIndex;
    }

    @RequestMapping(method = RequestMethod.GET)
    public String root() {
        return redirectIndex;
    }

    // 各 API 用 @RequestBody XxxRequestDto + @ResponseBody XxxResponseDto，调用 webSms012C3Service
}
```

### 3.2 Service 层（01288 风格）

- **接口**：`WebSms012C3Service`，方法签名为强类型 Request/Response DTO。
- **实现类**：`WebSms012C3ServiceImpl`，`@Service`，注入 `SmsApiClient`。
- **API 路径**：通过 `MessageManager.getInstance().getMessage("sms.module012c3.name")` 读取，未配置时使用默认值（如 `sms012c3`），与 SmsWebApi 侧路径一致。
- **调用方式**：`smsApiClient.post(getApiPath(), path, request, XxxResponseDto.class)`；若无请求体可用 `postNoBody`，再手动封装 Response DTO。

示例片段：

```java
@Service
public class WebSms012C3ServiceImpl implements WebSms012C3Service {

    private static final String MSG_KEY_MODULE_NAME = "sms.module012c3.name";
    private static final String DEFAULT_API_PATH = "sms012c3";
    private static final String PATH_GET_XXX = "getXxx";

    @Autowired
    private SmsApiClient smsApiClient;

    private String getApiPath() {
        try {
            String name = MessageManager.getInstance().getMessage(MSG_KEY_MODULE_NAME);
            if (name != null && !name.trim().isEmpty()) {
                return name.trim();
            }
        } catch (Exception ignored) { }
        return DEFAULT_API_PATH;
    }

    @Override
    public XxxResponseDto getXxx(XxxRequestDto request) {
        return smsApiClient.post(getApiPath(), PATH_GET_XXX, request, XxxResponseDto.class);
    }
}
```

### 3.3 DTO 约定（与 SmsWebApi 契约一致）

- **Request**：POJO，字段与 SmsWebApi 请求 JSON 一致（如 `lang`、`corporate`、`key1` 等），Getter/Setter。
- **Response**：建议包含 `error`（boolean）与 `data`（内部 DTO）；若 API 在根级别返回 `ans` 等字段，可用 `@JsonProperty("ans")` 的 setter 转存到 `data` 中，与 01288 的 GetChargeResponseDto 做法一致。

### 3.4 API 调用方式（SmsApiClient）

- **基地址**：从 MessageManager 的 `sms.webapi.url`（smsweb.properties）读取，SmsApiClient 内部使用，无需在 012C3 中再配。
- **两种用法**：
  - **强类型（推荐 012C3）**：`smsApiClient.post(apiPath, path, requestDto, responseDtoClass)`。
  - **透传（01206 风格）**：`smsApiClient.postPassthrough(apiPath, methodName, map)`，入出参均为 `Map`。
- **apiPath**：必须与 SmsWebApi 侧模块路径一致；012C3 使用 `sms.module012c3.name` 配置即可。

### 3.5 若采用 01206 风格（多画面 + Map 透传）

- Controller：类级 `@RequestMapping("/websms012c3/maintenance")`（可带前导 `/`），多个 `route.*`（list、createform、detail 等），`@PostConstruct` 中初始化 path/redirect/bean 名。
- Service：方法为 `Map<String,Object> methodName(Map<String,Object> input)`，实现内 `smsApiClient.postPassthrough(getApiPath(), "methodName", input)`。
- 表单：使用 `@ModelAttribute` 提供 Form 对象，GET 显示、POST 可只做 redirect 或再调 API。

### 3.6 自定义 View（如 Excel 下载）

- 在 `spring.domain.service.websms012c3.util.view` 下实现 View 类，并在同包或共用 ViewConfig 中 `@Bean(name = "xxxView")` 注册。
- properties 中：`sms.module012c3.bean.xxx=xxxView`。
- Controller 方法：`return viewBeanName`，Model 中放入 View 所需属性（如文件名、内容等）。
- 需确保 `BeanNameViewResolver` 的 order 较高（如 0），以便优先于 JSP 解析（可参考 01206 的 ViewConfig）。

---

## 四、前端项目文件结构规则与共通组件指导

新增 012C3 时，前端资源需遵循既有目录与命名规则，并优先复用共通组件与脚本，保证风格一致、维护简单。

### 4.1 前端根路径与引用方式

- **根路径**：所有静态资源位于 `src/main/webapp/` 下。
- **JSP 中引用**：必须带应用上下文，统一使用 `<%=request.getContextPath()%>`，例如：
  ```jsp
  <script src="<%=request.getContextPath()%>/static/websms012c3/js/xxx.js"></script>
  <link rel="stylesheet" href="<%=request.getContextPath()%>/static/websms012c3/css/websms012c3.css">
  ```
- **不要**在 JSP 中写死 `/static/` 或绝对路径，否则部署到子路径时资源会 404。

### 4.2 静态资源目录结构规则

| 路径 | 用途 | 012C3 示例 |
|------|------|-------------|
| **static/common/** | 全模块共用：共通组件、工具脚本、第三方库 | 不新建，直接引用 |
| **static/common/js/** | 共通 JS：common.js、axios.min.js、vue.global.js、dayjs 等 | — |
| **static/common/js/components/** | 共通 Web Components（applet-*） | — |
| **static/common/css/components/** | 共通组件样式（与 js/components 对应） | — |
| **static/websms012c3/** | 012C3 专用静态资源 | 新建 |
| **static/websms012c3/js/** | 012C3 业务脚本、页面入口、对话框逻辑 | 如 `websms012c3.js`、`sms012c3-api-util.js` |
| **static/websms012c3/js/components/** | 012C3 独有组件（若需要） | 如 Jbk* 风格组件 |
| **static/websms012c3/js/dialogs/** | 012C3 弹窗相关脚本 | 按需 |
| **static/websms012c3/css/** | 012C3 画面样式 | 如 `websms012c3.css`、`sms012c3-common.css` |
| **static/websms012c3/css/components/** | 012C3 独有组件样式 | 按需 |
| **static/websms012c3/css/dialogs/** | 012C3 弹窗样式 | 按需 |

**规则要点**：

- **模块隔离**：每个功能模块（websms01206、websms01288、websms012c3）拥有独立目录 `static/websmsXXXXX/`，模块内再按 `js/`、`css/` 及子目录划分。
- **命名**：模块级样式可命名为 `websms012c3.css` 或与业务相关的 `sms012c3-common.css`、`sms012c3-createForm.css` 等，与现有 01206/01288 保持一致风格。
- **共通优先**：能放 `static/common/` 的脚本与样式（工具函数、通用 UI）不放模块目录；模块目录只放该模块特有逻辑与样式。

### 4.3 共通组件（static/common）使用指导

以下组件为项目已有，012C3 可直接在 JSP 中按需引用，无需重复实现。

| 组件 | 路径 | 标签名 | 说明 |
|------|------|--------|------|
| 容器 | `common/js/components/AppletContainer.js` | `<applet-container>` | 画面外枠、布局容器 |
| 标题 | `common/js/components/HeadLabel.js` → 见 AppletContainer | `<applet-head-label>` | 画面标题 |
| 状态栏 | `common/js/components/StatusBar.js` | `<applet-status-bar>` | 底部状态栏 |
| 表单字段 | `common/js/components/FormField.js` | `<applet-form-field>` | 标签 + 输入框 |
| 按钮 | `common/js/components/Button.js` | `<applet-button>` | 按钮（支持 disabled、尺寸、位置等） |
| 下拉 | `common/js/components/Select.js` | `<applet-select>` | 原生 select 风格 |
| 组合框 | `common/js/components/ComboBox.js` | `<applet-combobox>` | 输入 + datalist 选择 |
| 复选框 | `common/js/components/Checkbox.js` | `<applet-checkbox>` | 复选框 |
| 单选 | `common/js/components/Radio.js` | `<applet-radio>` | 单选 |
| 对话框 | `common/js/components/Dialog.js` | `<applet-dialog>` | 弹窗外枠（ overlay + 标题 + 内容 slot） |

**引用顺序建议**（在 JSP `<head>` 内）：

1. Vue（若用）：模块自己的 `vue.global.js` 或 `static/common/js/vue.global.js`。
2. 模块专用 CSS：如 `static/websms012c3/css/websms012c3.css`。
3. 共通组件（按依赖顺序）：AppletContainer → StatusBar → FormField → Button → Select → ComboBox → Checkbox → Radio → Dialog。
4. 模块业务脚本：最后加载 012C3 的 js。

**共通组件特性**：

- 多为 **Web Components**（Custom Elements），使用 Shadow DOM，样式通过组件自带 CSS（如 `Button.css`）或属性（如 `width`、`height`、`position`、`left`、`top`）控制。
- 组件脚本会通过 `document.currentScript.src` 解析同名的 CSS 路径（如 `Button.js` → `Button.css`），因此共通 CSS 需放在 `static/common/css/components/` 下与组件同名的文件中。
- 使用示例（与 01288 index.jsp 一致）：
  ```html
  <applet-container>
    <applet-head-label text="画面标题"></applet-head-label>
    <applet-form-field id="entryNoField" label="エントリーNo" label-width="88px" input-width="112px" ...></applet-form-field>
    <applet-button id="btnRead" onclick="doRead()">検索</applet-button>
    <applet-dialog id="dlg">...</applet-dialog>
  </applet-container>
  ```

### 4.4 模块专用组件（01206 风格，可选）

01206 在 **模块内** 使用了 Jbk 系组件（JbkTableView、JbkComboBox、JbkButton、JbkLabel、JbkEdit、JbkCell 等），位于各模块的 `static/websms01206/js/components/` 与 `css/components/`。若 012C3 采用类似表格/表单风格，可：

- **方案 A**：在 `static/websms012c3/js/components/` 下复制或精简 Jbk 系组件，仅给 012C3 使用。
- **方案 B**：将 Jbk 系抽到 `static/common/`，供多模块共用（需评估 01206 是否一起改为引用共通）。

新建模块专用组件时，建议与共通组件一致：**JS 与 CSS 同名**，CSS 放在 `static/websms012c3/css/components/`，便于维护。

### 4.5 共通脚本（common.js）与第三方库

- **common.js**（`static/common/js/common.js`）：提供 `SmsUtils` 等工具，如 `showLoading(show)`、`showMessage(message, type, duration)`、`formatDate(date, format)` 等。012C3 若需统一 loading、消息提示、日期格式，可先引用该脚本再调用。
- **第三方库**：如 axios（`common/js/axios.min.js`）、Vue（`common/js/vue.global.js`）、dayjs 及其插件（`common/js/dayjs.min.js`、advancedFormat、customParseFormat 等）。优先使用 `static/common/` 下已有版本；若模块当前像 01206 一样自备一份（如 `static/websms01206/js/vue.global.js`），可逐步统一到 common，避免重复。

### 4.6 012C3 前端最小可做清单

1. **目录**：新建 `static/websms012c3/`，其下 `js/`、`css/`（及按需的 `js/dialogs/`、`css/dialogs/`、`js/components/`、`css/components/`）。
2. **样式**：至少 `static/websms012c3/css/websms012c3.css`（可先空或从 01288 复制再改）。
3. **脚本**：至少一个入口脚本（如 `websms012c3.js` 或 `index.js`），负责页面初始化、事件绑定、调用后端 API（与 Controller 的 `@RequestMapping("/websms012c3/maintenance")` 对应）。
4. **JSP**：在 `index.jsp`（或对应画面 JSP）的 `<head>` 中按 4.3 顺序引用共通组件 + 本模块 CSS/JS，路径使用 `<%=request.getContextPath()%>/static/...`。
5. **API 调用**：前端请求的 URL 需与 Controller 一致（如 `contextPath + '/websms012c3/maintenance/xxx'`），若使用共通 `common.js` 或 axios，注意 baseURL 或 contextPath 不要写死。

按上述规则，012C3 的前端即可与现有 01206/01288 对齐，并最大化复用共通组件与脚本。

---

## 五、实施步骤小结（012C3 最小可运行）

1. **配置**  
   - 在 `smsweb.properties` 中增加 `sms.module012c3.name`、`sms.module012c3.route.index`、`sms.module012c3.redirect.index`。

2. **Java**  
   - 新建 `WebSms012C3Controller`（`spring.app.websms012c3`），实现 index 与 root 重定向，必要时加 1～2 个 API 方法。  
   - 新建 `WebSms012C3Service` 与 `WebSms012C3ServiceImpl`（`spring.domain.service.websms012c3`），实现 `getApiPath()` 与 1～2 个 API 调用。  
   - 按需新建 `dto/XxxRequestDto`、`dto/XxxResponseDto`。

3. **视图与前端**  
   - 新建 `WEB-INF/views/spring/websms012c3/maintenance/index.jsp`（内容可与 01288 的 index 类似或占位）。  
   - 前端静态资源按 **第四章** 规则在 `static/websms012c3/` 下建立目录与入口 CSS/JS，并按需引用共通组件。

4. **后端 API**  
   - 在 **SmsWebApi** 侧实现 `sms012c3` 对应接口，保证 URL、JSON 契约与 SmsWeb 的 apiPath、DTO 一致。

5. **菜单与权限**  
   - 在 Intramart 平台中配置菜单项指向 `…/websms012c3/maintenance/index`（或 list 等）。

6. **验证**  
   - 访问配置的 URL，确认页面打开、API 调用与 `sms.webapi.url` 连通性。

---

## 六、注意事项与常见坑

- **包名与扫描**：Controller 必须在 `spring.app` 下，Service 实现必须在 `spring.domain.service` 下，否则不会被 applicationContext-main.xml 扫描到。
- **MessageManager 键**：键名必须与 properties 中完全一致（如 `sms.module012c3.name`），否则取不到值会回退到默认或空。
- **API 路径一致**：`sms.module012c3.name` 必须与 SmsWebApi 提供的路径一致（如 `sms012c3`），否则 404 或连错模块。
- **RequestMapping 风格**：01206、01288、012C3 均建议使用前导 `/`（如 `/websms01206/maintenance`、`/websms012c3/maintenance`），避免混用。
- **View 与 JSP**：返回 Bean 名时需注册 `BeanNameViewResolver` 且 order 优先；返回的字符串为 JSP 路径时，无需加前缀，与 properties 中 `route.*` 一致即可。
- **依赖**：确保 SmsWebApi 已部署且版本符合 module.xml 的 `verified-version`，且 `sms.webapi.url` 指向正确环境。

---

## 七、参考现有模块

| 项目 | 路径/文件 |
|------|-----------|
| Controller（多画面 + 透传 API） | `spring.app.websms01206.WebSms01206Controller` |
| Controller（单画面 + 强类型 API） | `spring.app.websms01288.WebSms01288Controller` |
| Service（透传） | `spring.domain.service.websms01206.WebSms01206ServiceImpl` |
| Service（强类型） | `spring.domain.service.websms01288.WebSms01288ServiceImpl` |
| 属性配置 | `src/main/conf/message/smsweb/smsweb.properties` |
| View Bean 配置 | `spring.domain.service.websms01206.util.view.ViewConfig` |
| 共通 HTTP 客户端 | `spring.common.client.SmsApiClient` |

按上述清单与模式，即可在 SmsWeb 中完成 012C3 的基础配置与最小可运行功能；后续再按业务增加画面、API 与 DTO 即可。
