# Intramart 环境下 `@Value("${sms.webapi.url}")` 无法解析的原因分析

## 现象

- 使用 `@Value("${sms.webapi.url}")`（无默认值）时，Intramart Server 启动报错：  
  `Could not resolve placeholder 'sms.webapi.url' in value "${sms.webapi.url}"`
- 使用 `@Value("${sms.webapi.url:http://10.0.2.9:80/imart}")`（带默认值）时，Server 可正常启动。

说明在 Intramart 运行时，**占位符 `sms.webapi.url` 在解析时没有被注册进当前使用的 Spring 环境**。

---

## 原因分析（结合 Intramart 环境）

### 1. Intramart 的 Spring 根上下文由谁、怎样加载

- 启动日志中有：  
  `WebApp[production/webapp/default/imart,STARTING] Initializing Spring root WebApplicationContext`  
  即 **根 WebApplicationContext** 是在 **imart** 这个 Web 应用下初始化的，不是仅在 SmsWeb 模块内。
- 根上下文的配置来源由 **Intramart 平台** 决定（平台自己的 `web.xml` 或扩展的 `SpringServletContextListener` 等），一般会设置 `contextConfigLocation`，指向**平台侧**的配置文件（例如 `WEB-INF/applicationContext.xml` 或平台约定的若干 XML），而**不一定**包含各业务模块的 `META-INF/spring/applicationContext-main.xml`。

因此可以归纳为：

- **根 WebApplicationContext** 的 `contextConfigLocation` 由 **imart 平台** 决定。
- 平台通常只加载**平台自己的** Spring 配置，**不会**自动把 SmsWeb 的 `META-INF/spring/applicationContext-main.xml` 加入根 context 的配置列表。

### 2. Bean 定义来自哪里（为什么我们的 Bean 会在根 context 里）

- SmsWeb 的 Bean（如 `SmsApiClient`、`WebSms01288Controller`）在包 `spring.app`、`spring.common`、`spring.domain.service` 下，并使用了 `@Component` / `@Service` / `@Controller`。
- 根 context 的配置里，很可能有一个**大范围的 component-scan**（例如扫描多个根包或整个 classpath），这样会把**已部署到 imart 的 SmsWeb 类**扫描进**根 context**。
- 也就是说：**Bean 定义是在根 context 里注册并实例化的**，和“是否加载了 SmsWeb 的 applicationContext-main.xml”无关。

### 3. PropertySourcesPlaceholderConfigurer 在哪里

- `PropertySourcesPlaceholderConfigurer` 和占位符默认值（`<props>`）只定义在 **SmsWeb** 的 `applicationContext-main.xml` 里。
- 若该文件**没有被**纳入根 WebApplicationContext 的 `contextConfigLocation`，则：
  - 根 context 在初始化时**不会**加载这份 XML；
  - 根 context 里就**没有**我们配置的 `PropertySourcesPlaceholderConfigurer`；
  - 根 context 的 `Environment` / 占位符解析器中也**不会**出现 `sms.webapi.url` 等属性。

### 4. 报错发生的时机

- 根 context 刷新时，会实例化所有单例 Bean（包括上述带 `@Value("${sms.webapi.url}")` 的类）。
- 解析 `@Value("${sms.webapi.url}")` 时，Spring 会在**当前 context** 的占位符解析器中查找 `sms.webapi.url`。
- 由于根 context 里没有我们的 `PropertySourcesPlaceholderConfigurer`，也没有从 `application.properties` 或 XML 默认属性中注入该 key，所以**占位符无法解析**，抛出 `Could not resolve placeholder 'sms.webapi.url'`。

### 5. 小结：为什么“加默认值就能启动”

- **不加默认值**：`@Value("${sms.webapi.url}")` 必须解析出值，而根 context 里没有该属性 → 报错。
- **加默认值**：`@Value("${sms.webapi.url:http://10.0.2.9:80/imart}")` 在占位符解析不到时，会使用 `:` 后面的默认值，不依赖根 context 是否加载了我们的 XML 或 properties → 能正常启动。

本质是：**提供占位符的配置（applicationContext-main.xml）和实际创建 Bean 的 context（根 WebApplicationContext）不是同一个上下文**，所以在 Intramart 环境下，不能依赖“根 context 一定会加载 applicationContext-main.xml”。

---

## 结论与建议

| 项目 | 说明 |
|------|------|
| **根本原因** | 根 WebApplicationContext 由 Intramart 平台配置加载，通常不包含 SmsWeb 的 `META-INF/spring/applicationContext-main.xml`，因此根 context 中没有本模块的 `PropertySourcesPlaceholderConfigurer`，无法解析 `sms.webapi.url`。 |
| **Bean 所在 context** | SmsWeb 的 Bean 被平台的 component-scan 扫进**根 context**，在根 context 中完成 `@Value` 解析和注入。 |
| **推荐做法** | 所有使用 `sms.webapi.url`（以及可能仅在本模块 XML 中定义的属性）的 `@Value`，**一律在注解中写默认值**，例如：<br>`@Value("${sms.webapi.url:http://localhost:27879/imart}")`。 |
| **application.properties** | 若希望用文件覆盖默认值，需要保证该文件在**根 context 能访问到的 classpath** 上，并且根 context 的配置中有能加载它的 `PropertySourcesPlaceholderConfigurer` 或等价机制；在 Intramart 默认部署下往往不满足，因此默认值仍是保障启动的关键。 |
| **applicationContext-main.xml** | 其中的 `PropertySourcesPlaceholderConfigurer` 在本模块被单独加载时（例如单独测试或子 context 使用该 XML 时）仍会生效；在 Intramart 根 context 中则可能不生效，故不在根 context 上依赖它解析 `sms.webapi.url`。 |

遵循以上做法后，在 Intramart 环境下即可稳定启动，同时保留在能加载到 `application.properties` 或 XML 默认属性的环境中被覆盖的能力。
