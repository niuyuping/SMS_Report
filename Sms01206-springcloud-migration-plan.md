# Sms01206 → Spring Cloud 微服务改造完整方案

> 文档版本：v1.0  
> 编写日期：2026-03-12  
> 前置文档：[Sms01206-containerization-plan.md](./Sms01206-containerization-plan.md)

---

## 目录

1. [改造背景与目标](#1-改造背景与目标)
2. [现状技术债分析](#2-现状技术债分析)
3. [目标架构设计](#3-目标架构设计)
4. [服务拆分方案](#4-服务拆分方案)
5. [项目结构设计](#5-项目结构设计)
6. [核心模块改造详解](#6-核心模块改造详解)
   - 6.1 数据库访问层改造（JDBC-ODBC → MyBatis）
   - 6.2 RMI接口 → REST API 映射
   - 6.3 分布式EntryNo采番设计
   - 6.4 外部系统集成改造
   - 6.5 配置管理改造
   - 6.6 服务发现与注册
   - 6.7 熔断与限流
   - 6.8 API网关
7. [依赖技术栈选型](#7-依赖技术栈选型)
8. [数据模型设计](#8-数据模型设计)
9. [完整API接口设计](#9-完整api接口设计)
10. [工程脚手架代码](#10-工程脚手架代码)
11. [迁移策略（渐进式）](#11-迁移策略渐进式)
12. [非功能性需求设计](#12-非功能性需求设计)
13. [改造工作量估算](#13-改造工作量估算)

---

## 1. 改造背景与目标

### 1.1 驱动力

| 问题 | 现状痛点 | Spring Cloud解决方案 |
|------|---------|---------------------|
| 通信协议过时 | Java RMI，依赖 RMI Registry，无法水平扩容 | REST/HTTP，无状态，天然支持 K8s 水平扩缩容 |
| 数据库驱动废弃 | `sun.jdbc.odbc.JdbcOdbcDriver`（JDK 8+ 已移除） | Microsoft JDBC Driver + HikariCP 连接池 |
| 客户端技术废弃 | Java Applet（浏览器已全面禁用） | REST API，前端可接入 React/Vue 或新版 Swing 应用 |
| 无健康检查 | 无任何可观测性手段 | Spring Actuator + Prometheus + Grafana |
| 配置硬编码 | Windows 绝对路径写在 `.prm` 文件中 | Spring Cloud Config + K8s ConfigMap/Secret |
| 单点故障 | 全部服务单进程，一个崩溃影响全局 | 独立部署，Resilience4j 熔断隔离 |
| 无版本管理 | 文件名加日期的原始方式 | Git + 语义化版本 + CI/CD |

### 1.2 改造目标

```
改造前（当前）                        改造后（目标）
─────────────────────                ──────────────────────────────────────
Java Applet（浏览器）                 REST Client（新客户端/前端）
    │ RMI                                │ HTTP/JSON
    ▼                                    ▼
OrderEntryNo_s（单进程）          Spring Cloud Gateway
    │ RMI ──→ SMSSQLServerIfc            │
    │ RMI ──→ SMSServerIfc2         sms01206-service（Spring Boot）
    │ RMI ──→ scp_common_i               │ Feign ──→ sms-master-service
Windows Server裸机                       │ Feign ──→ scp-service
                                         │ MyBatis ──→ SQL Server
                                    K8s Deployment（可水平扩容）
```

---

## 2. 现状技术债分析

### 2.1 接口规模统计

| 接口文件 | 方法数 | 说明 |
|---------|-------|------|
| `OrderEntryNo_i.java` | **65个** | Sms01206 自身对外暴露的业务方法 |
| `SMSServerIfc.java` | **30+个** | 依赖的主业务服务（SMSServer2）接口 |
| `SMSSQLServerIfc.java` | **4个** | SQL代理接口（SELECT/UPDATE/getSQL/getInfo） |
| `scp_common_i.java` | 若干 | SCP联动接口 |
| `RemoteIfc.java` | 4个 | 基础管理接口（clearRegister/exit/status/init） |

### 2.2 数据传输模式

所有接口的入参和返回值均为 `java.util.Hashtable`，以字符串键值对传递：

```java
// 典型调用模式（现状）
Hashtable prm = new Hashtable();
prm.put("corporate", "15185");
prm.put("entry_no", "2024001");
Hashtable result = remoteObject_.referOrder(prm);
String orderDate = (String) result.get("order_date");
```

**改造方向**：为每个业务方法定义强类型 DTO，消除 `Hashtable`，引入 Jackson JSON 序列化。

### 2.3 SQL代理模式分析

`OrderEntryNo_s` 不直接持有数据库连接，所有SQL通过 `SMSSQLServerIfc` 代理执行：

```java
// 现状：SQL以字符串传入，通过RMI发送给SQL代理执行
Hashtable sqlPrm = new Hashtable();
sqlPrm.put("sql", "SELECT entry_no, ... FROM order_table WHERE ...");
Hashtable result = mswc_.executeQuery(sqlPrm);
```

**改造方向**：直接在 `sms01206-service` 内部通过 MyBatis Mapper 执行 SQL，消除 SQL 代理这一额外网络跳转。

### 2.4 EntryNo 采番关键风险

`getEntryNo()` 无参数方法，内部通过 `SMSServerIfc.getNewEntryNo()` 实现，后者在 `SMSServerIfc2` 单进程内通过数据库序列/自增保证唯一性。

**改造风险**：Spring Cloud 环境多实例并发下，必须保证 EntryNo 全局唯一、有序。方案见 [6.3节](#63-分布式entryno采番设计)。

---

## 3. 目标架构设计

### 3.1 整体微服务架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           K8s 集群 (sms namespace)                       │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                    Spring Cloud Gateway                           │    │
│  │   路由 / 认证 / 限流 / 链路追踪注入                               │    │
│  └──────────────────────┬───────────────────────────────────────────┘    │
│                          │ HTTP/JSON                                      │
│        ┌─────────────────┼──────────────────────┐                        │
│        │                 │                       │                        │
│   ┌────▼─────┐    ┌──────▼──────┐    ┌──────────▼───────┐               │
│   │sms01206  │    │sms-master   │    │scp-service       │               │
│   │-service  │    │-service     │    │（SCP联动）        │               │
│   │          │    │（主数据缓存）│    │                  │               │
│   │Spring    │    │Spring Boot  │    │Spring Boot       │               │
│   │Boot      │    │Caffeine     │    │                  │               │
│   └────┬─────┘    └──────┬──────┘    └──────────────────┘               │
│        │ MyBatis          │ MyBatis                                       │
│        │                 │                                               │
│   ┌────▼─────────────────▼──────┐                                        │
│   │      SQL Server             │  ← 外部（企业内网，不容器化）           │
│   └─────────────────────────────┘                                        │
│                                                                           │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────────────────┐  │
│  │  Eureka Server │  │ Config Server  │  │ Zipkin / Prometheus        │  │
│  │  （服务注册）  │  │  （配置中心）  │  │ （链路追踪 / 指标监控）   │  │
│  └────────────────┘  └────────────────┘  └───────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘

外部系统（不改造，仅适配接口）：
  SAP R/3 ←→ R3AdapterService（HTTP调用）
  NPM      ←→ NpmAdapterService（TCP Socket封装为HTTP）
  IMS      ←→ ImsAdapterService（TCP + HTTP封装）
```

### 3.2 Spring Cloud 技术选型总览

| 功能 | 选型 | 版本 |
|------|------|------|
| 核心框架 | Spring Boot | 3.3.x |
| 微服务框架 | Spring Cloud | 2023.0.x |
| 服务注册/发现 | Spring Cloud Eureka（或 Nacos） | 4.1.x |
| 配置中心 | Spring Cloud Config Server | 4.1.x |
| 服务间调用 | Spring Cloud OpenFeign | 4.1.x |
| API网关 | Spring Cloud Gateway | 4.1.x |
| 熔断/限流 | Resilience4j | 2.2.x |
| 数据库访问 | MyBatis-Spring-Boot | 3.0.x |
| JDBC驱动 | Microsoft JDBC Driver for SQL Server | 12.6.x |
| 连接池 | HikariCP（Spring Boot 内置） | — |
| 本地缓存 | Caffeine（主数据缓存） | 3.x |
| 分布式缓存 | Redis（EntryNo采番/Session） | 7.x |
| 链路追踪 | Micrometer Tracing + Zipkin | — |
| 指标监控 | Micrometer + Prometheus | — |
| API文档 | SpringDoc OpenAPI 3.0 | 2.5.x |
| 日志 | Logback + ELK Stack | — |
| 容器化 | Docker + K8s | — |
| Java版本 | JDK 21（LTS） | — |

---

## 4. 服务拆分方案

### 4.1 拆分原则

1. **按原RMI服务边界拆分**：原先每个 RMI 服务实例（OrderEntryNo_s、SMSServer2、SMSSQLServer、scp_common_s）对应一个 Spring Boot 服务
2. **消除 SQL 代理层**：`SMSSQLServerIfc` 仅是 JDBC 的网络包装，拆分后各服务直连数据库，消除这层
3. **共通模块下沉为 Starter**：`it_common`、`sms_common` 中的工具类改造为 Spring Boot AutoConfiguration Starter
4. **外部系统封装为 Adapter**：SAP R/3、NPM、IMS 封装为独立的 Adapter 服务

### 4.2 服务清单

| 新服务名 | 对应原组件 | 职责 | 优先级 |
|---------|-----------|------|--------|
| `sms01206-service` | `OrderEntryNo_s` | 订单登录号业务核心 | **P0（本期）** |
| `sms-master-service` | `SMSMasterServer` / `SMSMasterServer2` | 主数据缓存（客户、机种、价格等） | P1 |
| `scp-service` | `scp_common_s` / `SCPServer` | SCP生产销售调整系统联动 | P1 |
| `sms-gateway` | — | API网关（路由/认证/限流） | P1 |
| `sms-config-server` | — | Spring Cloud Config服务 | P0（支撑） |
| `sms-eureka-server` | — | 服务注册中心 | P0（支撑） |
| `r3-adapter-service` | `R3Server` | SAP R/3 RFC 调用封装 | P2 |
| `npm-adapter-service` | `NPMServer` | 机展书 TCP Socket 封装 | P2 |
| `ims-adapter-service` | `IMSServer` | IMS库存系统联动封装 | P2 |
| `sms-common-starter` | `it_common` + `sms_common` 工具部分 | 公共工具库（AutoConfiguration） | P0（支撑） |

### 4.3 本期（P0）范围

本文档详细设计 **`sms01206-service`** 的 Spring Cloud 改造，其他服务的改造方案参考本文档模式进行。

---

## 5. 项目结构设计

### 5.1 Maven 多模块项目结构

```
sms-platform/                              ← 根 POM（多模块父项目）
├── pom.xml                                ← 父POM，管理所有依赖版本（BOM）
│
├── sms-common-starter/                    ← 公共工具 AutoConfiguration Starter
│   ├── pom.xml
│   └── src/main/java/jp/co/sankyoseiki/sms/common/
│       ├── config/SmsAutoConfiguration.java
│       ├── parameter/ParameterProperties.java
│       ├── util/FiscalYear.java
│       ├── util/Katakana.java
│       └── ...（从 it_common/util 迁移）
│
├── sms-eureka-server/                     ← Eureka 注册中心
│   ├── pom.xml
│   └── src/main/java/
│
├── sms-config-server/                     ← Spring Cloud Config 服务
│   ├── pom.xml
│   └── src/main/java/
│
├── sms-gateway/                           ← API 网关
│   ├── pom.xml
│   └── src/main/java/
│
├── sms01206-service/                      ← 【核心】OrderEntryNo 业务服务
│   ├── pom.xml
│   ├── Dockerfile
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/jp/co/sankyoseiki/sms/sms01206/
│   │   │   │   ├── Sms01206Application.java           ← 主启动类
│   │   │   │   ├── controller/
│   │   │   │   │   ├── OrderEntryController.java       ← 订单 CRUD
│   │   │   │   │   ├── MasterController.java           ← 主数据查询
│   │   │   │   │   └── EntryNoController.java          ← 采番
│   │   │   │   ├── service/
│   │   │   │   │   ├── OrderEntryService.java          ← 业务接口
│   │   │   │   │   ├── impl/OrderEntryServiceImpl.java ← 从 OrderEntryNo_s 迁移
│   │   │   │   │   ├── EntryNoService.java
│   │   │   │   │   └── impl/EntryNoServiceImpl.java
│   │   │   │   ├── mapper/
│   │   │   │   │   ├── OrderMapper.java                ← 从 executeQuery SQL 迁移
│   │   │   │   │   ├── PriceMapper.java
│   │   │   │   │   ├── CustomerMapper.java
│   │   │   │   │   └── ...（按业务域分组）
│   │   │   │   ├── domain/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   ├── OrderEntity.java            ← 对应 order_table
│   │   │   │   │   │   ├── EntryNoEntity.java          ← 对应 entry_no_table
│   │   │   │   │   │   └── ...
│   │   │   │   │   ├── dto/request/
│   │   │   │   │   │   ├── OrderInsertRequest.java
│   │   │   │   │   │   ├── OrderUpdateRequest.java
│   │   │   │   │   │   └── ...
│   │   │   │   │   └── dto/response/
│   │   │   │   │       ├── OrderResponse.java
│   │   │   │   │       ├── EntryNoResponse.java
│   │   │   │   │       └── ...
│   │   │   │   ├── client/                             ← Feign 客户端
│   │   │   │   │   ├── SmsMasterClient.java            ← 调用 sms-master-service
│   │   │   │   │   ├── ScpClient.java                  ← 调用 scp-service
│   │   │   │   │   ├── R3AdapterClient.java            ← 调用 r3-adapter-service
│   │   │   │   │   └── NpmAdapterClient.java           ← 调用 npm-adapter-service
│   │   │   │   ├── config/
│   │   │   │   │   ├── DataSourceConfig.java
│   │   │   │   │   ├── RedisConfig.java
│   │   │   │   │   ├── ResilienceConfig.java
│   │   │   │   │   └── SecurityConfig.java
│   │   │   │   └── exception/
│   │   │   │       ├── SmsBusinessException.java
│   │   │   │       └── GlobalExceptionHandler.java
│   │   │   └── resources/
│   │   │       ├── application.yml
│   │   │       ├── application-dev.yml
│   │   │       ├── application-prod.yml
│   │   │       ├── logback-spring.xml
│   │   │       └── mapper/                            ← MyBatis XML
│   │   │           ├── OrderMapper.xml
│   │   │           ├── PriceMapper.xml
│   │   │           └── ...
│   │   └── test/
│   │       └── java/jp/co/sankyoseiki/sms/sms01206/
│   │           ├── controller/OrderEntryControllerTest.java
│   │           └── service/OrderEntryServiceTest.java
│   └── k8s/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml
│
├── sms-master-service/                    ← 主数据服务（P1）
├── scp-service/                           ← SCP服务（P1）
├── r3-adapter-service/                    ← SAP R/3 适配器（P2）
├── npm-adapter-service/                   ← NPM 适配器（P2）
└── ims-adapter-service/                   ← IMS 适配器（P2）
```

---

## 6. 核心模块改造详解

### 6.1 数据库访问层改造（JDBC-ODBC → MyBatis）

#### 现状问题

```java
// 现状：SQL 作为字符串通过 RMI 传给代理服务执行
Hashtable sqlPrm = new Hashtable();
sqlPrm.put("sql", "SELECT O.ENTRY_NO, O.ORDER_DATE, O.CUSTOMER_CODE, " +
    "O.PRODUCT_CODE, O.QUANTITY, O.UNIT_PRICE " +
    "FROM ORDER_TABLE O WHERE O.ENTRY_NO = '" + entryNo + "'");
Hashtable result = mswc_.executeQuery(sqlPrm);  // RMI调用
```

**问题**：
1. SQL 拼接存在 SQL 注入风险
2. 通过 RMI 网络传输 SQL 字符串，多一跳延迟
3. 无类型安全，结果集也是 `Hashtable`
4. JDBC-ODBC Bridge 在 JDK 8+ 不存在

#### 目标改造

**`application.yml` 数据源配置**：

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:sqlserver://ms1152.cmp.sankyoseiki.co.jp:1433;databaseName=SMSDB;encrypt=true;trustServerCertificate=true}
    username: ${DB_USER}
    password: ${DB_PASSWD}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    hikari:
      pool-name: Sms01206HikariPool
      maximum-pool-size: 10
      minimum-idle: 3
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-test-query: SELECT 1

mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: jp.co.sankyoseiki.sms.sms01206.domain.entity
  configuration:
    map-underscore-to-camel-case: true
    default-fetch-size: 100
    default-statement-timeout: 30
```

**Entity 设计示例（OrderEntity）**：

```java
// 从 sms_common/table/OrderTable.java 和 EntryNoTable.java 迁移字段定义
@Data
@TableName("ORDER_TABLE")
public class OrderEntity {
    /** エントリーNo（订单号，主键） */
    private String entryNo;
    /** サブNo */
    private String subNo;
    /** 受注日 */
    private LocalDate orderDate;
    /** 法人コード */
    private String corporateCode;
    /** 得意先コード */
    private String customerCode;
    /** 機種コード */
    private String productCode;
    /** 数量 */
    private BigDecimal quantity;
    /** 単価 */
    private BigDecimal unitPrice;
    /** ステータス */
    private String status;
    /** 最終更新日時 */
    private LocalDateTime lastUpdateTime;
    // ... 其他字段从 OrderTable.java 中迁移
}
```

**Mapper 接口设计（类型安全）**：

```java
@Mapper
public interface OrderMapper {

    // 对应原 OrderEntryNo_s.getOrder(Hashtable)
    @Select("SELECT * FROM ORDER_TABLE WHERE ENTRY_NO = #{entryNo} AND SUB_NO = #{subNo}")
    Optional<OrderEntity> findByEntryNoAndSubNo(
        @Param("entryNo") String entryNo,
        @Param("subNo") String subNo);

    // 对应原 OrderEntryNo_s.getOrderSub(Hashtable)（按EntryNo查所有SubNo）
    List<OrderEntity> findAllByEntryNo(@Param("entryNo") String entryNo);

    // 对应原 orderInsert(Hashtable)
    int insert(OrderEntity order);

    // 对应原 orderUpdate(Hashtable)
    int update(OrderEntity order);

    // 对应原 orderDelete(Hashtable)
    int deleteByEntryNoAndSubNo(
        @Param("entryNo") String entryNo,
        @Param("subNo") String subNo);

    // 复杂动态查询在 XML 中定义
    List<OrderEntity> searchOrders(OrderSearchCriteria criteria);
}
```

**MyBatis XML（动态SQL示例，对应原来的字符串拼接）**：

```xml
<!-- src/main/resources/mapper/OrderMapper.xml -->
<mapper namespace="jp.co.sankyoseiki.sms.sms01206.mapper.OrderMapper">

    <resultMap id="OrderResultMap" type="OrderEntity">
        <id property="entryNo" column="ENTRY_NO"/>
        <result property="subNo" column="SUB_NO"/>
        <result property="orderDate" column="ORDER_DATE"/>
        <result property="corporateCode" column="CORPORATE_CODE"/>
        <result property="customerCode" column="CUSTOMER_CODE"/>
        <result property="productCode" column="PRODUCT_CODE"/>
        <result property="quantity" column="QUANTITY"/>
        <result property="unitPrice" column="UNIT_PRICE"/>
        <result property="status" column="STATUS"/>
        <result property="lastUpdateTime" column="LAST_UPDATE_TIME"/>
    </resultMap>

    <!-- 动态条件查询（替代原拼接SQL） -->
    <select id="searchOrders" resultMap="OrderResultMap">
        SELECT * FROM ORDER_TABLE
        <where>
            <if test="entryNo != null">AND ENTRY_NO = #{entryNo}</if>
            <if test="corporateCode != null">AND CORPORATE_CODE = #{corporateCode}</if>
            <if test="customerCode != null">AND CUSTOMER_CODE = #{customerCode}</if>
            <if test="productCode != null">AND PRODUCT_CODE = #{productCode}</if>
            <if test="orderDateFrom != null">AND ORDER_DATE >= #{orderDateFrom}</if>
            <if test="orderDateTo != null">AND ORDER_DATE &lt;= #{orderDateTo}</if>
            <if test="status != null">AND STATUS = #{status}</if>
        </where>
        ORDER BY ENTRY_NO DESC, SUB_NO ASC
    </select>

    <!-- 批量插入 -->
    <insert id="batchInsert" parameterType="list">
        INSERT INTO ORDER_TABLE (ENTRY_NO, SUB_NO, ORDER_DATE, CORPORATE_CODE,
            CUSTOMER_CODE, PRODUCT_CODE, QUANTITY, UNIT_PRICE, STATUS, LAST_UPDATE_TIME)
        VALUES
        <foreach collection="list" item="item" separator=",">
            (#{item.entryNo}, #{item.subNo}, #{item.orderDate}, #{item.corporateCode},
             #{item.customerCode}, #{item.productCode}, #{item.quantity}, #{item.unitPrice},
             #{item.status}, GETDATE())
        </foreach>
    </insert>

</mapper>
```

---

### 6.2 RMI接口 → REST API 映射

**设计原则**：
- 原 `Hashtable prm` 入参 → 强类型 Request DTO（`@RequestBody`）或路径参数
- 原 `Hashtable` 返回值 → 强类型 Response DTO，统一包装为 `ApiResponse<T>`
- 无参数方法（`getEntryNo()`、`getQSeihinHash()`）→ `GET` 请求
- 查询类方法（`getSales`、`getOrder`等）→ `GET` 请求，参数通过 `@RequestParam` 或路径变量传递
- 写操作（`orderInsert`、`orderUpdate`、`orderDelete`）→ `POST`/`PUT`/`DELETE`
- 状态变更（`moveR3`、`updateSmsStatus`等）→ `POST`（动作语义）

#### 统一响应体

```java
@Data
@Builder
public class ApiResponse<T> {
    private boolean success;
    private T data;
    private String errorCode;
    private String errorMessage;
    private long timestamp;

    public static <T> ApiResponse<T> ok(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .timestamp(System.currentTimeMillis())
            .build();
    }

    public static <T> ApiResponse<T> error(String code, String message) {
        return ApiResponse.<T>builder()
            .success(false)
            .errorCode(code)
            .errorMessage(message)
            .timestamp(System.currentTimeMillis())
            .build();
    }
}
```

#### 完整的 RMI → REST 方法映射表

| 原RMI方法 | HTTP方法 | REST路径 | 说明 |
|-----------|---------|---------|------|
| `getEntryNo()` | GET | `/api/v1/entry-no/generate` | 采番（无参数） |
| `getQSeihinHash()` | GET | `/api/v1/masters/product-categories` | 製品区分取得 |
| `getLabelString(prm)` | GET | `/api/v1/labels?lang={lang}&screen={screen}` | 画面标签 |
| `getOrder(prm)` | GET | `/api/v1/orders/{entryNo}/{subNo}` | 订单详情 |
| `getOrderSub(prm)` | GET | `/api/v1/orders/{entryNo}` | 按EntryNo查所有SubNo |
| `orderInsert(prm)` | POST | `/api/v1/orders` | 订单登录 |
| `orderUpdate(prm)` | PUT | `/api/v1/orders/{entryNo}/{subNo}` | 订单更新 |
| `orderDelete(prm)` | DELETE | `/api/v1/orders/{entryNo}/{subNo}` | 订单删除 |
| `insert(prm)` | POST | `/api/v1/prices` | 販価マスター登录 |
| `update(prm)` | PUT | `/api/v1/prices/{priceId}` | 販価マスター更新 |
| `getPrice(prm)` | GET | `/api/v1/prices?corporate={code}&product={code}&customer={code}` | 価格検索 |
| `getUnitPrice(prm)` | GET | `/api/v1/prices/unit?...` | 販価マスター検索（追加版） |
| `getSales(prm)` | GET | `/api/v1/sales?corporate={code}` | 他社向販売検索 |
| `getPattern(prm)` | GET | `/api/v1/production-patterns?...` | 生産パターン |
| `getCustomers(prm)` | GET | `/api/v1/masters/customers?section={code}` | 得意先検索 |
| `getCharges(prm)` | GET | `/api/v1/masters/charges?section={code}` | 担当者検索 |
| `getChargeCustomers(prm)` | GET | `/api/v1/masters/charges/{chargeCode}/customers` | 担当者別得意先 |
| `getConsigns(prm)` | GET | `/api/v1/masters/consigns?customer={code}` | 配送先検索 |
| `getConsign(prm)` | GET | `/api/v1/masters/consigns/{consignCode}` | 配送先詳細 |
| `getProducts(prm)` | GET | `/api/v1/masters/products?...` | 機種検索 |
| `getUses(prm)` | GET | `/api/v1/masters/uses` | 用途マスター |
| `getVari(prm)` | GET | `/api/v1/masters/vari?...` | 扱い区分検索 |
| `getSupplys(prm)` | GET | `/api/v1/masters/supplys?customer={code}` | 納入先検索 |
| `getMakerMaster(prm)` | GET | `/api/v1/masters/makers?...` | メーカーマスター |
| `getPriceCodeMaster(prm)` | GET | `/api/v1/masters/price-codes?...` | 販価决定区分 |
| `updatePriceCode(prm)` | PUT | `/api/v1/masters/price-codes/{id}` | 販価决定区分更新 |
| `getProductDialogMaster(prm)` | GET | `/api/v1/dialogs/product?...` | 機種選択Dialog |
| `getForwardDialogMaster(prm)` | GET | `/api/v1/dialogs/forward?...` | 出荷依赖票Dialog |
| `getRankDialogMaster(prm)` | GET | `/api/v1/dialogs/rank?...` | ランクDialog |
| `getDefaultConsigns(prm)` | GET | `/api/v1/orders/defaults/consigns?customer={code}` | 初期値取得 |
| `moveR3(prm)` | POST | `/api/v1/orders/{entryNo}/move-to-r3` | R/3移行 |
| `updateSmsStatus(prm)` | POST | `/api/v1/orders/{entryNo}/status/q` | ステータスST_Q設置 |
| `updateSmsStatusClear(prm)` | POST | `/api/v1/orders/{entryNo}/status/clear` | ステータスクリア |
| `distributeDeploy(prm)` | POST | `/api/v1/orders/{entryNo}/distribute-deploy` | 会社間派生処理 |
| `getDistributeDeploy(prm)` | GET | `/api/v1/orders/{entryNo}/distribute-deploy` | 会社間派生データ取得 |
| `getDeployEntry(prm)` | GET | `/api/v1/orders/{entryNo}/deploy-entry` | 継続プラントエントリー |
| `isDeployPriceMaster(prm)` | GET | `/api/v1/checks/deploy-price?...` | 継続プラント価格マスターチェック |
| `judgmentERP(prm)` | GET | `/api/v1/checks/erp-product?product={code}` | ERP連携判定 |
| `isUnitPriceMaster(prm)` | GET | `/api/v1/checks/unit-price?...` | 販価マスター存在チェック |
| `isHoliday(prm)` | GET | `/api/v1/calendar/is-holiday?date={date}` | 休日チェック |
| `getActiveDate(prm)` | GET | `/api/v1/calendar/active-date?date={date}&days={n}` | 稼動日計算 |
| `judgmentStopProduct(prm)` | GET | `/api/v1/checks/stop-product?product={code}` | 廃機種判定 |
| `judgementCustomerConfidence(prm)` | GET | `/api/v1/checks/customer-confidence?customer={code}` | 与信管理判定 |
| `reloadCustomerConfidence(prm)` | POST | `/api/v1/cache/customer-confidence/reload` | 与信キャッシュ再取得 |
| `getNpmEntry(prm)` | GET | `/api/v1/npm/entries?...` | NPM機展書検索 |
| `getNpmLine(prm)` | GET | `/api/v1/npm/lines?...` | NPMコメント情報 |
| `getNpmProductDivision(prm)` | GET | `/api/v1/npm/product-division?...` | NPM製品区分 |
| `getOutInData(prm)` | GET | `/api/v1/out-in/data?...` | OUT-IN連携データ取得 |
| `updateOutinData(prm)` | PUT | `/api/v1/out-in/data/{entryNo}` | OUT-IN連携データ更新 |
| `setFactoryResults(prm)` | POST | `/api/v1/factory-results` | 無償試作出荷実績登録 |
| `getMaxSalesUpdate(prm)` | GET | `/api/v1/orders/{entryNo}/max-update-date` | 最終更新日取得 |
| `judgementNNSN(prm)` | GET | `/api/v1/checks/nnsn?...` | NNSN判定 |
| `isDuplicationOrder(prm)` | GET | `/api/v1/checks/duplicate-order?orderNo={no}` | OrderNo重複チェック |
| `isDuplicationOrder2(prm)` | GET | `/api/v1/checks/duplicate-mold?...` | 金型品目重複チェック |
| `getCustomerForwardReadTime(prm)` | GET | `/api/v1/masters/customer-lead-time?customer={code}` | リードタイム取得 |
| `getProductFromCustomerProduct(prm)` | GET | `/api/v1/masters/products/by-customer-product?...` | 機種コード取得 |
| `getInvoiceNoCount(prm)` | GET | `/api/v1/orders/invoice-count?invoiceNo={no}` | インボイスNo件数 |
| `getTurnoverDate(prm)` | GET | `/api/v1/calendar/turnover-date?...` | 売上日取得 |
| `getExcelData(prm)` | GET | `/api/v1/orders/{entryNo}/excel-data` | Excel出力データ |
| `getForwadingListHtml(prm)` | GET | `/api/v1/orders/{entryNo}/forwarding-list` | 出荷依赖表作成 |
| `getKoOrder(prm)` | GET | `/api/v1/orders/{entryNo}/bundle-children` | 同梱品子エントリー |
| `getOyaOrder(prm)` | GET | `/api/v1/orders/{entryNo}/bundle-parent` | 同梱品親エントリー |
| `updateKoOrder(prm)` | PUT | `/api/v1/orders/{entryNo}/bundle-children` | 同梱品子エントリー更新 |
| `getMultiRate(prm)` | GET | `/api/v1/masters/exchange-rates?...` | 換算レート取得 |
| `getTransportList(prm)` | GET | `/api/v1/masters/transport?corporate={code}` | 輸送手段マスター |
| `getPSSPeriodProduct(prm)` | GET | `/api/v1/pss-period/products?...` | 生販調整確定期間 |
| `getProductPlant(prm)` | GET | `/api/v1/masters/product-plant?product={code}` | 自動設定プラント |
| `getConporison(prm)` | GET | `/api/v1/masters/product-classification?product={code}` | 製品区分 |
| `getCustomerProducts(prm)` | GET | `/api/v1/masters/customer-products?customer={code}` | 得意先별販価マスター |
| `clearRegister()` | POST | `/api/v1/admin/cache/clear` | キャッシュクリア |
| `status()` | GET | `/actuator/health` | Spring Actuator代替 |

---

### 6.3 分布式EntryNo采番设计

**原实现**：`SMSServerIfc.getNewEntryNo()` 在单进程内通过数据库序列保证唯一性。

**问题**：多实例并发下，简单的 `MAX(ENTRY_NO)+1` 会产生重复号。

**改造方案：数据库段式采番（推荐，与现有SQL Server兼容）**

```sql
-- 在 SQL Server 中创建采番管理表
CREATE TABLE ENTRY_NO_SEQUENCE (
    SEQUENCE_NAME  NVARCHAR(50) PRIMARY KEY,    -- 序列名（按法人/年度分）
    CURRENT_VALUE  BIGINT       NOT NULL,        -- 当前最大值
    STEP           INT          NOT NULL DEFAULT 100,  -- 每次申请的步长
    PREFIX         NVARCHAR(20) NOT NULL,        -- 前缀（法人代码+年度）
    LAST_UPDATED   DATETIME2   NOT NULL DEFAULT GETDATE()
);

-- 初始化
INSERT INTO ENTRY_NO_SEQUENCE VALUES ('15185_2026', 2026000000, 100, '2026', GETDATE());
```

```java
// EntryNoService：段式采番，每次从DB申请一段号码
@Service
@Slf4j
public class EntryNoServiceImpl implements EntryNoService {

    private final EntryNoMapper entryNoMapper;
    private final RedisTemplate<String, String> redisTemplate;

    // 本地号段缓存（线程安全）
    private final ConcurrentHashMap<String, AtomicLong> localCounter = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Long> segmentMax = new ConcurrentHashMap<>();
    private static final int SEGMENT_SIZE = 100;  // 每次从DB申请100个号

    @Override
    @Transactional
    public String generateEntryNo(String corporateCode) {
        String sequenceKey = buildSequenceKey(corporateCode);

        // 用 Redis 分布式锁防止并发申请号段
        String lockKey = "entry_no_lock:" + sequenceKey;
        String lockValue = UUID.randomUUID().toString();

        try {
            // 申请本地号段（先检查本地是否还有可用号）
            long nextNo = getNextLocalNo(sequenceKey);
            if (nextNo == -1) {
                // 本地号段用完，从 DB 申请新号段（带分布式锁）
                nextNo = applyNewSegment(sequenceKey, lockKey, lockValue);
            }
            return formatEntryNo(corporateCode, nextNo);
        } finally {
            releaseLock(lockKey, lockValue);
        }
    }

    private long applyNewSegment(String sequenceKey, String lockKey, String lockValue) {
        // 使用 Redis SET NX 分布式锁
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, 5, TimeUnit.SECONDS);

        if (!Boolean.TRUE.equals(locked)) {
            // 等待并重试
            try { Thread.sleep(50); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return getNextLocalNo(sequenceKey); // 可能其他实例已申请完成
        }

        // 使用悲观锁从 DB 原子性地分配号段
        EntryNoSegment segment = entryNoMapper.applySegmentWithLock(sequenceKey, SEGMENT_SIZE);
        localCounter.put(sequenceKey, new AtomicLong(segment.getStartValue()));
        segmentMax.put(sequenceKey, segment.getEndValue());
        return localCounter.get(sequenceKey).getAndIncrement();
    }

    private long getNextLocalNo(String sequenceKey) {
        AtomicLong counter = localCounter.get(sequenceKey);
        Long maxVal = segmentMax.get(sequenceKey);
        if (counter == null || maxVal == null) return -1;
        long next = counter.getAndIncrement();
        return next <= maxVal ? next : -1;
    }

    private String formatEntryNo(String corporateCode, long no) {
        // 格式：法人代码(5位) + 年(4位) + 流水号(6位) = 15位
        int year = LocalDate.now().getYear();
        return String.format("%s%04d%06d", corporateCode, year, no % 1000000L);
    }
}
```

```xml
<!-- EntryNoMapper.xml -->
<update id="applySegmentWithLock" statementType="CALLABLE">
    -- SQL Server 事务内原子更新
    UPDATE ENTRY_NO_SEQUENCE WITH (UPDLOCK, ROWLOCK)
    SET CURRENT_VALUE = CURRENT_VALUE + #{step},
        LAST_UPDATED = GETDATE()
    WHERE SEQUENCE_NAME = #{sequenceName};

    SELECT CURRENT_VALUE - #{step} + 1 AS startValue,
           CURRENT_VALUE AS endValue
    FROM ENTRY_NO_SEQUENCE
    WHERE SEQUENCE_NAME = #{sequenceName};
</update>
```

---

### 6.4 外部系统集成改造

#### SAP R/3 集成（原 `R3Server.java`）

**现状**：通过 TCP Socket 直连 R/3 系统，参数从 `.prm` 文件读取 `法人代码:host;port` 格式。

**改造方案**：封装为独立的 `r3-adapter-service`，Sms01206 通过 Feign 调用：

```java
// r3-adapter-service 中的 Controller
@RestController
@RequestMapping("/api/v1/r3")
public class R3AdapterController {

    @PostMapping("/orders/{entryNo}/transfer")
    public ApiResponse<R3TransferResult> transferOrder(
        @PathVariable String entryNo,
        @RequestBody R3TransferRequest request) {
        // 封装原 R3Server 中的 TCP 通信逻辑
    }
}

// sms01206-service 中的 Feign 客户端
@FeignClient(
    name = "r3-adapter-service",
    fallbackFactory = R3AdapterClientFallbackFactory.class
)
public interface R3AdapterClient {

    @PostMapping("/api/v1/r3/orders/{entryNo}/transfer")
    ApiResponse<R3TransferResult> transferOrder(
        @PathVariable("entryNo") String entryNo,
        @RequestBody R3TransferRequest request);
}
```

#### NPM 机展书集成（原 `NPMServer.java`）

**现状**：通过 TCP Socket + `Pipe` 类与机展书数据库通信。

**改造方案**：`npm-adapter-service` 内部封装 Socket 通信，对外暴露 REST：

```java
@FeignClient(name = "npm-adapter-service")
public interface NpmAdapterClient {

    // 对应原 getNpmEntry(Hashtable)
    @GetMapping("/api/v1/npm/entries")
    ApiResponse<List<NpmEntry>> getEntries(@RequestParam Map<String, String> params);

    // 对应原 getNpmLine(Hashtable)
    @GetMapping("/api/v1/npm/lines")
    ApiResponse<NpmLineInfo> getLine(@RequestParam Map<String, String> params);
}
```

#### SCP 服务集成（原 `scp_common_s`）

```java
@FeignClient(
    name = "scp-service",
    fallbackFactory = ScpClientFallbackFactory.class
)
public interface ScpClient {

    @PostMapping("/api/v1/scp/orders/sync")
    ApiResponse<Void> syncOrder(@RequestBody ScpSyncRequest request);
}

// 熔断降级：SCP 系统不影响主业务流程
@Component
public class ScpClientFallbackFactory implements FallbackFactory<ScpClient> {
    @Override
    public ScpClient create(Throwable cause) {
        return request -> {
            log.warn("SCP service unavailable, skipping sync: {}", cause.getMessage());
            return ApiResponse.ok(null);  // SCP同步失败不阻断主流程
        };
    }
}
```

---

### 6.5 配置管理改造

#### `application.yml` 完整设计

```yaml
# sms01206-service/src/main/resources/application.yml
spring:
  application:
    name: sms01206-service
  config:
    import: "optional:configserver:${CONFIG_SERVER_URL:http://sms-config-server:8888}"
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

  # 数据源
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWD}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    hikari:
      pool-name: Sms01206Pool
      maximum-pool-size: ${DB_POOL_MAX:10}
      minimum-idle: ${DB_POOL_MIN:3}

  # Redis（EntryNo采番 + 缓存）
  data:
    redis:
      host: ${REDIS_HOST:redis}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      database: 0
      lettuce:
        pool:
          max-active: 8
          max-idle: 4

  # 字符编码
  mandatory-file-encoding: UTF-8

server:
  port: ${SERVER_PORT:8206}
  tomcat:
    threads:
      max: 200
      min-spare: 20
  compression:
    enabled: true

# Eureka 服务注册
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://sms-eureka-server:8761/eureka/}
    registry-fetch-interval-seconds: 30
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30

# MyBatis
mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: jp.co.sankyoseiki.sms.sms01206.domain.entity
  configuration:
    map-underscore-to-camel-case: true
    default-fetch-size: 100
    default-statement-timeout: 30
    cache-enabled: false  # 由 Caffeine 统一管理缓存

# Feign 客户端
feign:
  client:
    config:
      default:
        connect-timeout: 5000
        read-timeout: 10000
        logger-level: BASIC
      r3-adapter-service:
        read-timeout: 30000  # R/3 移行较慢
  circuitbreaker:
    enabled: true

# Resilience4j 熔断配置
resilience4j:
  circuitbreaker:
    instances:
      sms-master-service:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
      scp-service:
        sliding-window-size: 5
        failure-rate-threshold: 60
        wait-duration-in-open-state: 60s
      r3-adapter-service:
        sliding-window-size: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 120s

# Actuator（健康检查/指标）
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,loggers
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}

# 日志
logging:
  level:
    jp.co.sankyoseiki.sms: INFO
    org.springframework.cloud.openfeign: DEBUG
    com.zaxxer.hikari: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{traceId}] %-5level %logger{36} - %msg%n"

# 业务配置
sms01206:
  entry-no:
    segment-size: ${ENTRY_NO_SEGMENT_SIZE:100}  # 号段大小
    sequence-prefix: ${ENTRY_NO_PREFIX:}         # 前缀
  corporate:
    default-code: ${DEFAULT_CORPORATE_CODE:15185}
  npm:
    host: ${NPM_HOST}
    port: ${NPM_PORT:8080}
  r3:
    hosts: ${R3_HOSTS}  # 格式: 法人代码:host;port
  ims:
    host: ${IMS_HOST}
    port: ${IMS_PORT}
```

---

### 6.6 服务发现与注册

```java
// sms-eureka-server/src/main/java/
@SpringBootApplication
@EnableEurekaServer
public class SmsEurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SmsEurekaServerApplication.class, args);
    }
}

// sms01206-service/src/main/java/
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@MapperScan("jp.co.sankyoseiki.sms.sms01206.mapper")
public class Sms01206Application {
    public static void main(String[] args) {
        SpringApplication.run(Sms01206Application.class, args);
    }
}
```

---

### 6.7 熔断与限流

```java
// 服务调用示例（含熔断）
@Service
@Slf4j
public class OrderEntryServiceImpl implements OrderEntryService {

    private final SmsMasterClient smsMasterClient;
    private final ScpClient scpClient;
    private final R3AdapterClient r3AdapterClient;

    // R/3 移行（带熔断）
    @Override
    @CircuitBreaker(name = "r3-adapter-service", fallbackMethod = "moveR3Fallback")
    @Retry(name = "r3-adapter-service", fallbackMethod = "moveR3Fallback")
    @TimeLimiter(name = "r3-adapter-service")
    public CompletableFuture<R3TransferResult> moveR3(String entryNo, R3TransferRequest request) {
        return CompletableFuture.supplyAsync(() ->
            r3AdapterClient.transferOrder(entryNo, request).getData()
        );
    }

    // R/3 移行熔断降级
    public CompletableFuture<R3TransferResult> moveR3Fallback(
            String entryNo, R3TransferRequest request, Exception ex) {
        log.error("R3 transfer failed for entryNo={}, reason={}", entryNo, ex.getMessage());
        // R/3 移行失败记录到本地队列，后续重试
        r3RetryQueue.add(new R3RetryTask(entryNo, request));
        return CompletableFuture.completedFuture(
            R3TransferResult.pending(entryNo, "R3 service unavailable, queued for retry")
        );
    }
}
```

---

### 6.8 API网关

```yaml
# sms-gateway/src/main/resources/application.yml
spring:
  application:
    name: sms-gateway
  cloud:
    gateway:
      routes:
        - id: sms01206-service
          uri: lb://sms01206-service
          predicates:
            - Path=/sms01206/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            - name: CircuitBreaker
              args:
                name: sms01206-cb
                fallbackUri: forward:/fallback/sms01206

        - id: sms-master-service
          uri: lb://sms-master-service
          predicates:
            - Path=/sms-master/**
          filters:
            - StripPrefix=1

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
```

---

## 7. 依赖技术栈选型

### 7.1 父 POM（BOM 版本锁定）

```xml
<!-- sms-platform/pom.xml -->
<project>
    <groupId>jp.co.sankyoseiki.sms</groupId>
    <artifactId>sms-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
    </parent>

    <modules>
        <module>sms-common-starter</module>
        <module>sms-eureka-server</module>
        <module>sms-config-server</module>
        <module>sms-gateway</module>
        <module>sms01206-service</module>
        <module>sms-master-service</module>
        <module>scp-service</module>
        <module>r3-adapter-service</module>
        <module>npm-adapter-service</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.3</spring-cloud.version>
        <mybatis-spring-boot.version>3.0.3</mybatis-spring-boot.version>
        <mssql-jdbc.version>12.6.1.jre11</mssql-jdbc.version>
        <resilience4j.version>2.2.0</resilience4j.version>
        <springdoc.version>2.5.0</springdoc.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok.version>1.18.32</lombok.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- Spring Cloud BOM -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!-- SQL Server JDBC -->
            <dependency>
                <groupId>com.microsoft.sqlserver</groupId>
                <artifactId>mssql-jdbc</artifactId>
                <version>${mssql-jdbc.version}</version>
            </dependency>
            <!-- MyBatis -->
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>${mybatis-spring-boot.version}</version>
            </dependency>
            <!-- Resilience4j -->
            <dependency>
                <groupId>io.github.resilience4j</groupId>
                <artifactId>resilience4j-spring-boot3</artifactId>
                <version>${resilience4j.version}</version>
            </dependency>
            <!-- OpenAPI -->
            <dependency>
                <groupId>org.springdoc</groupId>
                <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
                <version>${springdoc.version}</version>
            </dependency>
            <!-- MapStruct（DTO转换） -->
            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### 7.2 sms01206-service POM

```xml
<!-- sms01206-service/pom.xml -->
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Spring Cloud Eureka Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <!-- Spring Cloud Config Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <!-- OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <!-- Resilience4j -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>
    <!-- MyBatis -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    <!-- SQL Server JDBC -->
    <dependency>
        <groupId>com.microsoft.sqlserver</groupId>
        <artifactId>mssql-jdbc</artifactId>
    </dependency>
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Micrometer Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <!-- Zipkin 链路追踪 -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
    <!-- OpenAPI 文档 -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    </dependency>
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <!-- MapStruct -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
    </dependency>
    <!-- 公共 Starter -->
    <dependency>
        <groupId>jp.co.sankyoseiki.sms</groupId>
        <artifactId>sms-common-starter</artifactId>
        <version>${project.version}</version>
    </dependency>
    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 8. 数据模型设计

### 8.1 核心表对应关系（现有表 → Entity 类）

| 数据库表（推断名） | 原 Java 类 | 新 Entity 类 |
|-----------------|-----------|-------------|
| ORDER_TABLE | `OrderTable.java`、`OrderTable2.java` | `OrderEntity.java` |
| ENTRY_NO_TABLE | `EntryNoTable.java`、`EntryNoTable2.java` | `EntryNoEntity.java` |
| PRICE_TABLE | `PriceTable.java` | `PriceEntity.java` |
| CUSTOMER_TABLE | `CustomerTable.java` | `CustomerEntity.java` |
| PRODUCT_TABLE | `ProductTable.java` | `ProductEntity.java` |
| CONSIGN_TABLE | `ConsignTable.java` | `ConsignEntity.java` |
| CALENDAR_TABLE | `CalendarTable.java` | `CalendarEntity.java` |
| DEPLOY_TABLE | `DeployTable.java` | `DeployEntity.java` |
| SUPPLY_TABLE | `SupplyTable.java` | `SupplyEntity.java` |
| TRANSPORT_TABLE | `TransportTable.java` | `TransportEntity.java` |
| OUT_IN_TABLE | `OutInTable.java` | `OutInEntity.java` |
| PSS_PERIOD_TABLE | `PSSPeriodTable.java` | `PssPeriodEntity.java` |
| STOP_PRODUCT_TABLE | `StopProductTable.java` | `StopProductEntity.java` |
| SUMMARY_TABLE | `SummaryTable.java` | `SummaryEntity.java` |
| ENTRY_NO_SEQUENCE | （新增） | `EntryNoSequenceEntity.java` |

### 8.2 DTO 设计示例

```java
// 订单插入请求 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderInsertRequest {

    @NotBlank(message = "法人コードは必須です")
    @Size(max = 10)
    private String corporateCode;

    @NotBlank(message = "得意先コードは必須です")
    @Size(max = 20)
    private String customerCode;

    @NotBlank(message = "機種コードは必須です")
    @Size(max = 30)
    private String productCode;

    @NotNull(message = "受注日は必須です")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate orderDate;

    @NotNull
    @Positive
    private BigDecimal quantity;

    @NotNull
    @PositiveOrZero
    private BigDecimal unitPrice;

    private String consignCode;
    private String supplyCode;
    private String useCode;
    private String variCode;
    private String chargeCode;
    private String memo;
    // ... 其他字段
}

// 订单响应 DTO
@Data
@Builder
public class OrderResponse {
    private String entryNo;
    private String subNo;
    private String corporateCode;
    private String customerCode;
    private String customerName;   // JOIN 结果
    private String productCode;
    private String productName;    // JOIN 结果
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate orderDate;
    private BigDecimal quantity;
    private BigDecimal unitPrice;
    private BigDecimal totalAmount;
    private String status;
    private String statusName;     // 状态中文名
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime lastUpdateTime;
}
```

---

## 9. 完整API接口设计

### 9.1 Controller 实现示例

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Slf4j
@Tag(name = "OrderEntry", description = "訂単登録号管理 API")
public class OrderEntryController {

    private final OrderEntryService orderEntryService;
    private final EntryNoService entryNoService;

    @Operation(summary = "订单详情查询", description = "对应原 getOrder(Hashtable)")
    @GetMapping("/{entryNo}/{subNo}")
    public ApiResponse<OrderResponse> getOrder(
            @PathVariable @NotBlank String entryNo,
            @PathVariable @NotBlank String subNo) {
        return ApiResponse.ok(orderEntryService.getOrder(entryNo, subNo));
    }

    @Operation(summary = "按EntryNo查所有SubNo", description = "对应原 getOrderSub(Hashtable)")
    @GetMapping("/{entryNo}")
    public ApiResponse<List<OrderResponse>> getOrderSub(@PathVariable String entryNo) {
        return ApiResponse.ok(orderEntryService.getOrderSub(entryNo));
    }

    @Operation(summary = "订单登录", description = "对应原 orderInsert(Hashtable)")
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<OrderResponse> insertOrder(
            @Valid @RequestBody OrderInsertRequest request) {
        return ApiResponse.ok(orderEntryService.insertOrder(request));
    }

    @Operation(summary = "订单更新", description = "对应原 orderUpdate(Hashtable)")
    @PutMapping("/{entryNo}/{subNo}")
    public ApiResponse<OrderResponse> updateOrder(
            @PathVariable String entryNo,
            @PathVariable String subNo,
            @Valid @RequestBody OrderUpdateRequest request) {
        return ApiResponse.ok(orderEntryService.updateOrder(entryNo, subNo, request));
    }

    @Operation(summary = "订单删除", description = "对应原 orderDelete(Hashtable)")
    @DeleteMapping("/{entryNo}/{subNo}")
    public ApiResponse<Void> deleteOrder(
            @PathVariable String entryNo,
            @PathVariable String subNo) {
        orderEntryService.deleteOrder(entryNo, subNo);
        return ApiResponse.ok(null);
    }

    @Operation(summary = "R/3移行", description = "对应原 moveR3(Hashtable)")
    @PostMapping("/{entryNo}/move-to-r3")
    public ApiResponse<R3TransferResult> moveR3(
            @PathVariable String entryNo,
            @RequestBody R3TransferRequest request) {
        return ApiResponse.ok(orderEntryService.moveR3(entryNo, request));
    }
}

@RestController
@RequestMapping("/api/v1/entry-no")
@RequiredArgsConstructor
@Tag(name = "EntryNo", description = "エントリーNo採番 API")
public class EntryNoController {

    private final EntryNoService entryNoService;

    @Operation(summary = "EntryNo採番", description = "対応原 getEntryNo()")
    @GetMapping("/generate")
    public ApiResponse<String> generateEntryNo(
            @RequestParam(defaultValue = "15185") String corporateCode) {
        return ApiResponse.ok(entryNoService.generateEntryNo(corporateCode));
    }
}
```

### 9.2 全局异常处理

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(SmsBusinessException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ApiResponse<Void> handleBusinessException(SmsBusinessException ex) {
        log.warn("Business exception: code={}, message={}", ex.getCode(), ex.getMessage());
        return ApiResponse.error(ex.getCode(), ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Void> handleValidationException(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ApiResponse.error("VALIDATION_ERROR", message);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<Void> handleUnexpectedException(Exception ex) {
        log.error("Unexpected error", ex);
        return ApiResponse.error("INTERNAL_ERROR", "予期しないエラーが発生しました");
    }
}
```

---

## 10. 工程脚手架代码

### 10.1 主启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "jp.co.sankyoseiki.sms.sms01206.client")
@MapperScan("jp.co.sankyoseiki.sms.sms01206.mapper")
@EnableCaching
public class Sms01206Application {

    public static void main(String[] args) {
        // 日文字符编码（原系统使用 Shift-JIS，迁移后统一 UTF-8）
        System.setProperty("file.encoding", "UTF-8");
        SpringApplication.run(Sms01206Application.class, args);
    }

    @Bean
    public CacheManager cacheManager() {
        // 本地缓存（主数据，TTL 5分钟）
        CaffeineCacheManager cacheManager = new CaffeineCacheManager(
            "customers", "products", "charges", "consigns",
            "uses", "vari", "makers", "transportList"
        );
        cacheManager.setCaffeine(
            Caffeine.newBuilder()
                .expireAfterWrite(5, TimeUnit.MINUTES)
                .maximumSize(1000)
                .recordStats()
        );
        return cacheManager;
    }
}
```

### 10.2 Service 层骨架

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class OrderEntryServiceImpl implements OrderEntryService {

    private final OrderMapper orderMapper;
    private final PriceMapper priceMapper;
    private final CustomerMapper customerMapper;
    private final EntryNoService entryNoService;
    private final SmsMasterClient smsMasterClient;
    private final ScpClient scpClient;
    private final R3AdapterClient r3AdapterClient;
    private final OrderMapper2DTO orderMapper2DTO;  // MapStruct

    @Override
    public OrderResponse getOrder(String entryNo, String subNo) {
        return orderMapper.findByEntryNoAndSubNo(entryNo, subNo)
            .map(orderMapper2DTO::toResponse)
            .orElseThrow(() -> new SmsBusinessException(
                "ORDER_NOT_FOUND",
                String.format("注文が見つかりません: entryNo=%s, subNo=%s", entryNo, subNo)
            ));
    }

    @Override
    @Transactional
    public OrderResponse insertOrder(OrderInsertRequest request) {
        // 1. 採番
        String entryNo = entryNoService.generateEntryNo(request.getCorporateCode());

        // 2. 価格チェック・取得
        PriceEntity price = priceMapper.findPrice(
            request.getCorporateCode(),
            request.getCustomerCode(),
            request.getProductCode()
        ).orElseThrow(() -> new SmsBusinessException("PRICE_NOT_FOUND", "販価マスターが存在しません"));

        // 3. 与信チェック（キャッシュ済み）
        CustomerConfidenceResult confidence = checkCustomerConfidence(request.getCustomerCode());
        if (!confidence.isApproved()) {
            throw new SmsBusinessException("CREDIT_DENIED", "与信管理により受注できません");
        }

        // 4. 重複チェック
        if (request.getOrderNo() != null && isDuplicateOrder(request.getOrderNo())) {
            throw new SmsBusinessException("DUPLICATE_ORDER", "OrderNoが重複しています");
        }

        // 5. 注文登録
        OrderEntity order = orderMapper2DTO.toEntity(request);
        order.setEntryNo(entryNo);
        order.setSubNo("01");
        order.setUnitPrice(price.getUnitPrice());
        order.setStatus("ST_0");
        orderMapper.insert(order);

        // 6. SCP 同期（非同期、熔断保护）
        syncToScpAsync(order);

        log.info("Order inserted: entryNo={}, customer={}, product={}",
            entryNo, request.getCustomerCode(), request.getProductCode());
        return orderMapper2DTO.toResponse(order);
    }

    @Async
    protected void syncToScpAsync(OrderEntity order) {
        try {
            scpClient.syncOrder(ScpSyncRequest.fromOrder(order));
        } catch (Exception e) {
            log.warn("SCP sync failed for entryNo={}: {}", order.getEntryNo(), e.getMessage());
        }
    }
}
```

---

## 11. 迁移策略（渐进式）

### 11.1 绞杀者模式（Strangler Fig Pattern）

不做一次性全量重写，而是通过「绞杀者」模式逐步替换：

```
阶段一（并行运行）：
  旧客户端（Applet）→ RMI → 旧 OrderEntryNo_s（继续运行）
  新客户端（HTTP）  → REST → 新 sms01206-service（新增）
  共享同一个数据库（SQL Server）

阶段二（流量切换）：
  旧客户端（Applet）→ RMI → 旧 OrderEntryNo_s（只读模式）
  新客户端          → REST → 新 sms01206-service（主要流量）
  同一数据库，通过双写保证一致性

阶段三（下线旧系统）：
  新客户端          → REST → 新 sms01206-service（全量）
  旧 OrderEntryNo_s 下线
```

### 11.2 分阶段实施计划

```
Phase 0（基础设施，第1-2周）
├── 搭建 sms-eureka-server
├── 搭建 sms-config-server
├── 搭建 sms-gateway（基本路由）
└── 建立 GitHub 仓库 + CI/CD 基础流水线

Phase 1（数据层，第3-5周）
├── 分析 SQL Server 数据库实际表结构
├── 逆向工程生成 Entity 类（MyBatis Generator）
├── 编写核心 Mapper 并通过单测验证 SQL 正确性
└── 搭建测试数据库环境

Phase 2（核心业务，第6-10周）
├── 实现 EntryNo 采番服务（含分布式锁）
├── 实现订单 CRUD（orderInsert/Update/Delete/getOrder）
├── 实现价格主数据查询（getPrice/getUnitPrice）
├── 实现客户/机种查询接口（getCustomers/getProducts）
└── 单测覆盖率达到 80%+

Phase 3（外部集成，第11-13周）
├── 实现 r3-adapter-service（SAP R/3 TCP 通信封装）
├── 实现 npm-adapter-service（NPM Socket 封装）
├── 实现 moveR3 + 熔断降级
└── 集成测试

Phase 4（次要功能，第14-16周）
├── 实现其余 40+ 个接口
├── OUT-IN 联动接口
├── PSS Period / 与信管理 / 廃機種判定等
└── Excel出力 / HTML出荷依赖表生成

Phase 5（并行验证，第17-20周）
├── 新旧系统并行运行，对比业务数据一致性
├── 性能压测
├── 逐步将真实流量切换到新系统
└── 旧系统下线

Phase 6（后续优化，持续）
├── 改造 sms-master-service（独立主数据缓存服务）
├── 改造 scp-service
└── 前端客户端改造（替代 Java Applet）
```

### 11.3 数据一致性保证

在并行运行阶段，通过**双写 + 最终一致性**保证：

```java
// 旧系统同步适配器（桥接层，部署在旧 Windows 服务器上）
// 监听新系统数据库变更，同步更新旧系统 RMI 状态
@Component
public class LegacyBridgeAdapter {

    @EventListener
    public void onOrderInserted(OrderInsertedEvent event) {
        // 通知旧 SMSMasterServer 刷新缓存
        try {
            legacyRmiClient.clearRegister();
        } catch (Exception e) {
            log.warn("Legacy cache clear failed: {}", e.getMessage());
        }
    }
}
```

---

## 12. 非功能性需求设计

### 12.1 可观测性

```
链路追踪：Micrometer Tracing → Zipkin
  每个 HTTP 请求生成 TraceId，贯穿所有微服务调用

指标监控：Micrometer → Prometheus → Grafana
  关键指标：
  - http_server_requests_seconds（API 响应时间）
  - entry_no_generation_total（EntryNo 采番次数）
  - db_connection_pool_active（数据库连接池状态）
  - resilience4j_circuitbreaker_state（熔断器状态）

日志聚合：Logback → Filebeat → Elasticsearch → Kibana
  日志格式包含：timestamp、traceId、spanId、serviceName、level、message
```

### 12.2 安全性

| 安全层面 | 方案 |
|---------|------|
| API认证 | Spring Security + JWT（统一在Gateway验证） |
| 数据库凭据 | K8s Secret（生产环境用 Vault 管理） |
| 服务间通信 | mTLS（Istio Service Mesh，可选） |
| SQL注入防护 | MyBatis 参数绑定（`#{}` 预编译），禁止 `${}` 字符串拼接 |
| 敏感字段 | 客户价格等敏感数据传输加密（HTTPS） |

### 12.3 性能目标

| 指标 | 目标 | 说明 |
|------|------|------|
| API 响应时间（P99） | < 200ms | 主数据查询（Caffeine缓存） |
| API 响应时间（P99） | < 500ms | 订单CRUD操作 |
| EntryNo 采番延迟 | < 50ms | Redis 号段缓存 |
| 并发请求数 | 100 TPS | 初期目标 |
| 服务可用性 | 99.9%（月停机 < 44分钟） | — |

---

## 13. 改造工作量估算

| 工作项 | 预估工作量（人天） | 说明 |
|--------|-----------------|------|
| 基础设施搭建（Eureka/Config/Gateway） | 5天 | 相对标准化 |
| 数据库逆向 + Entity/Mapper | 10天 | 需分析现有 SQL Server 表结构 |
| EntryNo 采番服务 | 5天 | 含分布式锁设计和测试 |
| 核心订单 CRUD（20个接口） | 15天 | 主要业务逻辑 |
| 主数据查询接口（20个接口） | 10天 | 相对简单的查询 |
| 外部系统适配（R/3/NPM/IMS） | 20天 | TCP 协议封装，需联调 |
| 次要功能接口（25个接口） | 15天 | OUT-IN/SCP等联动 |
| 并行运行验证 | 10天 | 含数据一致性比对 |
| 性能测试与优化 | 5天 | — |
| 文档与知识转移 | 5天 | — |
| **合计** | **100人天** | **约5人×4个月** |

> **注意**：工作量估算不含前端（Applet 替代客户端）的改造，如需同步改造客户端，预计额外增加 60～80 人天。

---

## 附录A：`OrderEntryNo_i.java` 完整接口 → REST 端点对照表

见第 [6.2节](#62-rmi接口--rest-api-映射) 完整映射表（含 65+ 个方法）。

## 附录B：编码规范约定

| 约定项 | 规范 |
|--------|------|
| 字符编码 | 统一 UTF-8（原系统 Shift-JIS 配置文件需转换） |
| 时区 | 所有日期时间使用 `LocalDate`/`LocalDateTime`，JVM 时区设置为 `Asia/Tokyo` |
| 日期格式 | JSON 中统一 `yyyy-MM-dd` / `yyyy-MM-dd HH:mm:ss` |
| 金额类型 | `BigDecimal`（禁止使用 `float`/`double`） |
| ID 命名 | 原 `ENTRY_NO` 等全大写下划线命名，Entity 字段用驼峰（MyBatis 自动映射） |
| 包名 | `jp.co.sankyoseiki.sms.{服务名}.{层名}` |
| 异常码 | 大写下划线，如 `ORDER_NOT_FOUND`、`PRICE_NOT_FOUND` |

## 附录C：原 `.prm` 参数文件 → Spring 配置映射

| 原参数文件 | 原 Key | 新配置方式 |
|-----------|--------|----------|
| `rmi_ms1152.prm` | `RMI=rmi://ms1225:1099/` | `eureka.client.service-url.defaultZone` |
| `rmi_ms1152.prm` | `SQL=ms1152` | `spring.datasource.url` (K8s Secret) |
| `server_program.prm` | 各服务程序路径 | K8s Deployment 中各服务独立部署 |
| `reset.prm` | `X=15185`（法人コード） | `sms01206.corporate.default-code` (ConfigMap) |
| `master.prm` | 各Master服务器IP | `spring.cloud.discovery.client`（Eureka自动发现） |
| `r3division.prm` | R/3 划分代码 | `sms01206.r3.hosts` (ConfigMap) |

---

*本文档基于对 OrgSms 项目代码的深度静态分析编写，涵盖 65+ 个 RMI 接口方法的完整改造映射。  
具体实施时，需在分析真实 SQL Server 数据库表结构（DDL）后，对 Entity 和 Mapper 做相应调整。*
