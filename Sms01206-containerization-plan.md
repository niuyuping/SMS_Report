# Sms01206（OrderEntryNo）容器化方案

> 文档版本：v1.0  
> 编写日期：2026-03-12  
> 适用对象：Sms01206 服务端（`OrderEntryNo_s`）独立容器化部署

---

## 1. 背景与目标

### 1.1 现状概述

Sms01206（订单登录号管理，OrderEntryNo）是 OrgSms 销售管理系统的核心业务模块之一，现状如下：

| 维度 | 现状 |
|------|------|
| 技术栈 | Java（JDK 6/7 时代），Fujitsu APWorks JBK3 框架 |
| 架构模式 | 三层 C/S 架构，客户端为 Java Applet，服务端通过 Java RMI 暴露接口 |
| 数据库访问 | `sun.jdbc.odbc.JdbcOdbcDriver`（JDBC-ODBC Bridge，JDK 8+ 已移除） |
| 构建方式 | 无 Maven/Gradle，手动编译，依赖 Windows `.bat` 脚本启动 |
| 部署方式 | On-Premises，Windows Server 裸机部署 |
| 版本管理 | 无 Git，以文件名附加日期的方式保存历史版本 |

### 1.2 容器化目标

**本期目标（第一步）**：仅对 **Sms01206 服务端**（`OrderEntryNo_s` RMI 服务）独立容器化，打包成可在 Kubernetes 集群中运行的容器镜像，通过 GitHub Actions Workflow 实现自动构建与制品发布。

**本文档范围**：容器化方案设计，不涉及源码修改，不涉及客户端（Applet）的改造。

---

## 2. 技术挑战与应对策略

在开始容器化之前，必须正视以下核心技术障碍：

### 2.1 【高风险】JDBC-ODBC Bridge 已废弃

**问题**：`MSWConnection.java` 中使用 `sun.jdbc.odbc.JdbcOdbcDriver`，该驱动在 JDK 8 中已完全移除。容器环境无法使用 Windows ODBC DSN。

**应对**：在构建阶段，将 `MSWConnection.java` 中的驱动替换为 Microsoft SQL Server 官方 JDBC 驱动（`com.microsoft.sqlserver.jdbc.SQLServerDriver`），并将连接字符串由 ODBC DSN 名称改为标准 JDBC URL 格式。

```
# 原配置（Windows ODBC DSN）
SERVER=jdbc:odbc:ms1152
USER=xxx
PASSWD=xxx

# 改造后（直接 JDBC URL）
SERVER=jdbc:sqlserver://ms1152.cmp.sankyoseiki.co.jp:1433;databaseName=SMSDB
USER=xxx
PASSWD=xxx
```

> **注意**：这是唯一不可避免的源码改动，改动范围极小，仅涉及 `it_common/parameter/MSWConnection.java` 一个文件中的驱动类名。

### 2.2 【高风险】Fujitsu 专有库的获取

**问题**：以下 JAR 包为富士通（Fujitsu）商业专有库，无法从公共仓库下载：

| JAR 文件 | 路径（原 Windows 环境） |
|---------|------------------------|
| `apworks13.jar` | `D:\APW\classes\apworks13.jar` |
| `dbalibjv12.jar` | `C:\...\FJDBALIB\JAVA\dbalibjv12.jar` |
| `trcclient.jar` | `D:\APW\JBK3\TRCClient\lib\trcclient.jar` |
| `isejb.jar` | `D:\APW\JBK3\jdk\jre\lib\isejb.jar` |
| `xmlpro.jar` | `C:\...\FujitsuXML\xmlpro.jar` |
| `xmltrans.jar` | `C:\...\FujitsuXML\xmltrans.jar` |

**应对**：
- 从现有 Windows 服务器的对应路径提取这些 JAR 包
- 将其归档到项目内部的 `lib/fujitsu/` 目录（不提交至 GitHub，需加入 `.gitignore`）
- 在 GitHub Actions 中，通过 **GitHub Secrets + 内部制品仓库**（如 Nexus/JFrog）分发
- 或：使用 GitHub Packages 作为私有 Maven 仓库存放这些 JAR

### 2.3 【中风险】Java RMI 在容器/K8s 中的行为

**问题**：Java RMI 在默认配置下会将本机 hostname 注册到 RMI Registry，容器环境中 hostname 是随机 Pod 名称，客户端无法通过 RMI stub 中记录的地址回连。

**应对**：
- 在 JVM 启动参数中显式设置 RMI 服务地址：
  ```
  -Djava.rmi.server.hostname=<K8s Service的外部IP或域名>
  ```
- 将 RMI Registry（1099端口）和 RMI 服务动态端口都通过 K8s Service 固定暴露

### 2.4 【中风险】JDK 版本兼容性

**问题**：源码使用 JDK 6/7 语法编写（如 `DriverManager.class.forName()`、原始类型 `Hashtable`），且依赖 `RMISecurityManager`（JDK 17+ 已移除）。

**应对**：容器内使用 **JDK 8**（最后一个包含 JDBC-ODBC Bridge 的版本，但需配合 2.1 的替换方案改用 JDK 11），推荐使用 `eclipse-temurin:11-jre-focal` 作为基础镜像，原因如下：
- JDK 11 是当前 LTS 版本中与 Java 6/7 代码兼容性最好的
- `RMISecurityManager` 在 JDK 17 中已废弃并移除，JDK 11 仍可使用
- 仅改动 JDBC 驱动即可让代码正常编译

---

## 3. 架构设计

### 3.1 容器化范围

```
┌─────────────────────────────────────────────────────────────┐
│                        K8s 集群                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              sms01206 Pod（本期容器化范围）            │   │
│  │                                                      │   │
│  │   ┌─────────────────────────────────────────────┐   │   │
│  │   │  OrderEntryNo_s（RMI Server）                │   │   │
│  │   │  ├── OrderEntryNo_i（RMI 接口）              │   │   │
│  │   │  ├── it_common（参数/DB连接工具）             │   │   │
│  │   │  ├── sms_common（SMS业务逻辑）                │   │   │
│  │   │  └── scp_common（SCP联动接口）               │   │   │
│  │   └─────────────────────────────────────────────┘   │   │
│  │   Port: 1099 (RMI Registry) + 动态端口               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────┐   │
│  │ SMSServer   │  │ SMSSQLServer│  │ SMSMasterServer   │   │
│  │（已有/待迁） │  │（已有/待迁）│  │（已有/待迁）      │   │
│  └─────────────┘  └─────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │ JDBC (mssql-jdbc)        │ RMI (1099)
         ▼                          ▼
┌──────────────────┐    ┌──────────────────────┐
│ SQL Server       │    │ 外部系统              │
│ ms1152 (外部)    │    │ SAP R/3 / NPM / IMS  │
└──────────────────┘    └──────────────────────┘
```

### 3.2 依赖关系清单

Sms01206 服务端运行时依赖以下组件：

| 类型 | 组件 | 说明 | 容器内处理方式 |
|------|------|------|----------------|
| 自研公共库 | `it_common` | 参数读取、DB连接、工具类 | 直接打包进 JAR |
| 自研公共库 | `sms_common` | RMI接口、SQL服务代理、业务逻辑 | 直接打包进 JAR |
| 自研公共库 | `scp_common` | SCP联动接口 | 直接打包进 JAR |
| 富士通专有库 | `apworks13.jar` | JBK3 GUI框架（服务端仅用少量工具类） | 放入 `lib/fujitsu/`，通过私有仓库分发 |
| 富士通专有库 | `dbalibjv12.jar` | Fujitsu DB适配层 | 同上 |
| 富士通专有库 | `trcclient.jar` | 事务控制 | 同上 |
| 微软官方库 | `mssql-jdbc-12.x.jar` | SQL Server JDBC 驱动（**替换 JDBC-ODBC Bridge**） | Maven Central 下载 |
| 数据库 | SQL Server (ms1152) | 企业内部数据库，不容器化 | 通过环境变量配置连接串 |
| 外部服务 | SAP R/3 | ERP系统，不容器化 | 通过 ConfigMap/Secret 注入连接参数 |
| 外部服务 | NPM Server | 机展书系统，TCP Socket | 通过 ConfigMap 注入 host:port |
| 外部服务 | IMS Server | 库存管理，TCP + RMI | 通过 ConfigMap 注入 |
| RMI 依赖 | SMSSQLServer | SQL代理服务，RMI 调用 | K8s Service 内部 DNS 解析 |
| RMI 依赖 | SMSServer | 主业务服务，RMI 调用 | K8s Service 内部 DNS 解析 |

---

## 4. 构建系统改造

### 4.1 引入 Gradle 构建

原项目无标准构建系统，需在 `service/businese/Sms01206/` 目录下创建 Gradle 构建文件。

**推荐目录结构（改造后）**：

```
service/
└── businese/
    └── Sms01206/
        ├── build.gradle              ← 新增：Gradle 构建定义
        ├── settings.gradle           ← 新增：项目设置
        ├── gradle/wrapper/           ← 新增：Gradle Wrapper
        ├── Dockerfile                ← 新增：容器镜像构建文件
        ├── src/
        │   └── main/
        │       └── java/             ← 将现有 .java 源文件迁移至此
        │           ├── OrderEntryNo_s.java
        │           ├── OrderEntryNo_i.java
        │           ├── OrderEntryNo_L.java
        │           └── ...（其他对话框类）
        ├── config/                   ← 运行时配置文件（替代 .prm 文件）
        │   ├── rmi.prm
        │   ├── server_program.prm
        │   └── reset.prm
        └── lib/
            └── fujitsu/              ← 富士通专有 JAR（加入 .gitignore）
                ├── apworks13.jar
                ├── dbalibjv12.jar
                └── trcclient.jar
```

**`build.gradle` 核心内容**：

```groovy
plugins {
    id 'java'
    id 'application'
}

group = 'jp.co.sankyoseiki.sms'
version = System.getenv('APP_VERSION') ?: 'dev'
sourceCompatibility = '11'
targetCompatibility = '11'

repositories {
    mavenCentral()
    // 私有仓库（用于富士通 JAR）
    maven {
        url = uri(System.getenv('NEXUS_URL') ?: 'https://nexus.internal/repository/maven-releases/')
        credentials {
            username = System.getenv('NEXUS_USER')
            password = System.getenv('NEXUS_PASS')
        }
    }
}

dependencies {
    // Microsoft SQL Server JDBC 驱动（替代 JDBC-ODBC Bridge）
    implementation 'com.microsoft.sqlserver:mssql-jdbc:12.6.1.jre11'

    // 富士通专有库（从私有仓库或本地 lib 目录引用）
    implementation fileTree(dir: 'lib/fujitsu', include: ['*.jar'])

    // 共通模块（从本地 common 目录引用）
    implementation project(':it_common')
    implementation project(':sms_common')
    implementation project(':scp_common')
}

application {
    mainClass = 'OrderEntryNo_L'
    applicationDefaultJvmArgs = [
        '-Djava.rmi.server.hostname=${RMI_HOSTNAME}',
        '-Djava.security.policy=rmi.policy',
        '-Xmx512m'
    ]
}

jar {
    manifest {
        attributes 'Main-Class': 'OrderEntryNo_L'
    }
    // 打包所有依赖为 fat JAR
    from {
        configurations.runtimeClasspath.collect {
            it.isDirectory() ? it : zipTree(it)
        }
    }
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

### 4.2 多模块项目配置（`settings.gradle`）

```groovy
rootProject.name = 'sms01206'

// 引用 common 模块
includeBuild('../../common/it_common')
includeBuild('../../common/sms_common')
includeBuild('../../common/scp_common')
```

---

## 5. Dockerfile 设计

### 5.1 多阶段构建 Dockerfile

```dockerfile
# ===== Stage 1: Build =====
FROM eclipse-temurin:11-jdk-focal AS builder

WORKDIR /workspace

# 复制 Gradle 构建文件
COPY gradle/wrapper/ gradle/wrapper/
COPY gradlew build.gradle settings.gradle ./

# 预下载依赖（利用 Docker 层缓存）
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# 复制共通模块源码
COPY ../../common/it_common /workspace/../common/it_common
COPY ../../common/sms_common /workspace/../common/sms_common
COPY ../../common/scp_common /workspace/../common/scp_common

# 复制业务模块源码
COPY src/ src/
COPY lib/ lib/

# 构建 Fat JAR
RUN ./gradlew jar --no-daemon


# ===== Stage 2: Runtime =====
FROM eclipse-temurin:11-jre-focal AS runtime

# 安装必要工具（时区同步）
RUN apt-get update && apt-get install -y --no-install-recommends \
    tzdata \
    && rm -rf /var/lib/apt/lists/* \
    && ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

WORKDIR /app

# 从构建阶段复制 JAR
COPY --from=builder /workspace/build/libs/sms01206-*.jar app.jar

# 复制运行时配置文件
COPY config/ config/

# 复制 RMI 安全策略
COPY rmi.policy rmi.policy

# 容器运行用户（非 root）
RUN groupadd -r smsapp && useradd -r -g smsapp smsapp
RUN chown -R smsapp:smsapp /app
USER smsapp

# 暴露 RMI Registry 端口及服务端口
EXPOSE 1099
EXPOSE 1200

# 启动命令（参数通过环境变量注入）
ENTRYPOINT ["java", \
    "-Djava.rmi.server.hostname=${RMI_HOSTNAME}", \
    "-Djava.security.manager", \
    "-Djava.security.policy=/app/rmi.policy", \
    "-Xmx512m", \
    "-jar", "app.jar"]

CMD ["config/rmi.prm", "config/server_program.prm"]
```

### 5.2 RMI 安全策略文件（`rmi.policy`）

```
grant {
    permission java.net.SocketPermission "*:1024-65535", "connect,accept,resolve";
    permission java.net.SocketPermission "*:80,443", "connect";
    permission java.io.FilePermission "/app/config/-", "read";
    permission java.util.PropertyPermission "*", "read";
};
```

---

## 6. 配置文件容器化策略

### 6.1 现有配置文件映射

| 原文件 | 原路径 | 容器化方案 | K8s 资源类型 |
|--------|--------|-----------|-------------|
| `rmi_ms1152.prm` | `sms_common/rmi_ms1152.prm` | 改为 JDBC URL 格式 | K8s Secret |
| `server_program.prm` | `sms_common/server_program.prm` | 移除 Windows 路径，改为服务名 | K8s ConfigMap |
| `reset.prm` | `sms_common/reset.prm` | 法人代码配置 | K8s ConfigMap |
| `master.prm` | `sms_common/master.prm` | 主数据服务器配置 | K8s ConfigMap |
| `r3division.prm` | `sms_common/r3division.prm` | SAP R3 划分代码配置 | K8s ConfigMap |

### 6.2 环境变量替换方案

以下参数通过环境变量覆盖，不再硬编码在 `.prm` 文件中：

| 环境变量 | 用途 | K8s 资源 |
|---------|------|---------|
| `RMI_HOSTNAME` | RMI 服务对外暴露的 IP/域名 | ConfigMap |
| `DB_URL` | SQL Server JDBC URL | Secret |
| `DB_USER` | 数据库用户名 | Secret |
| `DB_PASSWD` | 数据库密码 | Secret |
| `RMI_REGISTRY_HOST` | 注册中心地址（SMSServerManager） | ConfigMap |
| `NPM_HOST` | NPM Server 主机名 | ConfigMap |
| `NPM_PORT` | NPM Server 端口 | ConfigMap |
| `R3_HOSTS` | SAP R/3 host:port 列表（逗号分隔） | ConfigMap |
| `IMS_HOST` | IMS 系统地址 | ConfigMap |

---

## 7. GitHub Actions Workflow 设计

### 7.1 完整 CI/CD 流程

```
GitHub Push (main/release分支)
        │
        ▼
┌──────────────────────────────┐
│  Job 1: build-and-test       │
│  ├── Checkout 代码           │
│  ├── 设置 JDK 11             │
│  ├── 下载富士通私有 JAR       │
│  ├── Gradle 构建             │
│  └── 单元测试（如有）        │
└──────────────────────────────┘
        │ 成功
        ▼
┌──────────────────────────────┐
│  Job 2: build-and-push-image │
│  ├── Docker Build（多阶段）  │
│  ├── 镜像扫描（Trivy）       │
│  └── 推送至 GHCR            │
│      ghcr.io/org/sms01206   │
└──────────────────────────────┘
        │ 成功
        ▼
┌──────────────────────────────┐
│  Job 3: deploy-to-k8s        │
│  ├── 更新 K8s 清单中镜像版本 │
│  ├── kubectl apply           │
│  └── 等待 Rollout 完成       │
└──────────────────────────────┘
```

### 7.2 Workflow 文件（`.github/workflows/sms01206.yml`）

```yaml
name: Sms01206 CI/CD

on:
  push:
    branches: [ main, release/** ]
    paths:
      - 'service/businese/Sms01206/**'
      - 'service/common/it_common/**'
      - 'service/common/sms_common/**'
      - 'service/common/scp_common/**'
      - '.github/workflows/sms01206.yml'
  pull_request:
    branches: [ main ]
    paths:
      - 'service/businese/Sms01206/**'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository_owner }}/sms01206

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: service/businese/Sms01206

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 11
        uses: actions/setup-java@v4
        with:
          java-version: '11'
          distribution: 'temurin'

      - name: Download Fujitsu proprietary JARs from Nexus
        env:
          NEXUS_URL: ${{ secrets.NEXUS_URL }}
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASS: ${{ secrets.NEXUS_PASS }}
        run: |
          mkdir -p lib/fujitsu
          curl -u "${NEXUS_USER}:${NEXUS_PASS}" \
            "${NEXUS_URL}/jp/co/fujitsu/apworks/apworks13/1.0/apworks13-1.0.jar" \
            -o lib/fujitsu/apworks13.jar
          # 同样方式下载其他富士通 JAR...

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew jar --no-daemon

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: sms01206-jar
          path: service/businese/Sms01206/build/libs/*.jar

  build-and-push-image:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Download built JAR
        uses: actions/download-artifact@v4
        with:
          name: sms01206-jar
          path: service/businese/Sms01206/build/libs/

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=git-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: service/businese/Sms01206
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:git-${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  deploy-to-k8s:
    needs: build-and-push-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config

      - name: Update image in K8s manifests
        run: |
          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:git-${{ github.sha }}"
          sed -i "s|image: .*sms01206.*|image: ${IMAGE}|g" \
            k8s/sms01206/deployment.yaml

      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/sms01206/
          kubectl rollout status deployment/sms01206 -n sms --timeout=300s
```

---

## 8. Kubernetes 清单设计

### 8.1 目录结构

```
k8s/
└── sms01206/
    ├── namespace.yaml          # 命名空间
    ├── deployment.yaml         # Deployment
    ├── service.yaml            # Service（RMI 端口）
    ├── configmap.yaml          # 非敏感配置
    ├── secret.yaml             # 数据库凭据（实际用 Sealed Secrets 或 Vault）
    └── hpa.yaml                # 水平自动扩缩容（可选）
```

### 8.2 关键 K8s 清单

**`namespace.yaml`**：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sms
  labels:
    app.kubernetes.io/part-of: orgsms
```

**`configmap.yaml`**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sms01206-config
  namespace: sms
data:
  # RMI 连接配置
  RMI_REGISTRY_HOST: "sms-server.sms.svc.cluster.local"
  RMI_REGISTRY_PORT: "1099"

  # 外部系统连接
  NPM_HOST: "npm-server.internal"
  NPM_PORT: "8080"
  IMS_HOST: "ims-server.internal"

  # SAP R/3（格式：法人代码:host;port）
  R3_HOSTS: "15185:r3.internal;3300"

  # 参数文件挂载路径
  CONFIG_DIR: "/app/config"
```

**`secret.yaml`**（实际环境应使用 Sealed Secrets 或 Vault）：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sms01206-db-secret
  namespace: sms
type: Opaque
stringData:
  DB_URL: "jdbc:sqlserver://ms1152.cmp.sankyoseiki.co.jp:1433;databaseName=SMSDB;encrypt=true;trustServerCertificate=true"
  DB_USER: "sms_user"
  DB_PASSWD: "your-password-here"
```

**`deployment.yaml`**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sms01206
  namespace: sms
  labels:
    app: sms01206
    version: "1.0"
spec:
  replicas: 1  # RMI 服务注意：扩容需配合 RMI Registry 方案
  selector:
    matchLabels:
      app: sms01206
  strategy:
    type: Recreate  # RMI 服务不支持滚动更新（RMI 注册表单例问题）
  template:
    metadata:
      labels:
        app: sms01206
    spec:
      containers:
        - name: sms01206
          image: ghcr.io/your-org/sms01206:latest
          ports:
            - name: rmi-registry
              containerPort: 1099
              protocol: TCP
            - name: rmi-service
              containerPort: 1200
              protocol: TCP
          env:
            - name: RMI_HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP  # 或使用 K8s Service IP
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: sms01206-db-secret
                  key: DB_URL
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: sms01206-db-secret
                  key: DB_USER
            - name: DB_PASSWD
              valueFrom:
                secretKeyRef:
                  name: sms01206-db-secret
                  key: DB_PASSWD
          envFrom:
            - configMapRef:
                name: sms01206-config
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          livenessProbe:
            exec:
              command:
                - java
                - -cp
                - /app/app.jar
                - it_common.parameter.HealthCheck
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            tcpSocket:
              port: 1099
            initialDelaySeconds: 20
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: sms01206-config
      imagePullSecrets:
        - name: ghcr-secret
```

**`service.yaml`**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sms01206
  namespace: sms
spec:
  selector:
    app: sms01206
  type: ClusterIP
  ports:
    - name: rmi-registry
      port: 1099
      targetPort: 1099
    - name: rmi-service
      port: 1200
      targetPort: 1200
---
# 如需从集群外部（旧客户端）访问，额外创建 LoadBalancer Service
apiVersion: v1
kind: Service
metadata:
  name: sms01206-external
  namespace: sms
spec:
  selector:
    app: sms01206
  type: LoadBalancer
  ports:
    - name: rmi-registry
      port: 1099
      targetPort: 1099
    - name: rmi-service
      port: 1200
      targetPort: 1200
```

---

## 9. 需要最低限度修改的源码清单

按照"不改变业务逻辑，仅做基础设施适配"的原则，以下是**不可避免**的源码改动，所有改动均属于基础设施层（非业务代码）：

| 文件 | 修改内容 | 原因 |
|------|---------|------|
| `it_common/parameter/MSWConnection.java` | 驱动类名：`sun.jdbc.odbc.JdbcOdbcDriver` → `com.microsoft.sqlserver.jdbc.SQLServerDriver` | JDBC-ODBC Bridge 在 JDK 8+ 不存在 |
| `it_common/parameter/MSWConnection.java` | 连接方式：从 ODBC DSN 名称改为读取 JDBC URL 环境变量 | 容器内无 ODBC 配置 |
| `OrderEntryNo_L.java` | 将启动参数从硬编码改为从环境变量读取 | 支持容器环境注入配置 |
| `sms_common/server/SMSServer_L.java`（关联依赖） | 同上 | 同上 |

> **注意**：以上改动不涉及任何业务逻辑，仅是将 Windows 平台特有的基础设施代码替换为跨平台标准实现。

---

## 10. 已知风险与遗留问题

| 风险项 | 等级 | 说明 | 缓解措施 |
|--------|------|------|---------|
| Java Applet 客户端不可用 | 严重 | 现代浏览器（Chrome 45+、Firefox 52+）已不支持 Java Applet，本方案只容器化了服务端 | 客户端改造（Swing 应用/Web 化）需另立项目 |
| Fujitsu APWorks JBK3 服务端依赖 | 高 | 若 `apworks13.jar` 中有服务端代码依赖，需要从现有 Windows 环境提取 | 提前在 Windows 机器上分析实际引用的类 |
| RMI 动态端口问题 | 高 | Java RMI 默认使用动态端口传输数据，K8s NetworkPolicy 和防火墙可能拦截 | 使用 `java.rmi.server.port` 固定服务端口（如1200），并在 Service 中明确暴露 |
| RMI 不支持水平扩容 | 中 | 多个 Pod 无法共享同一个 RMI Registry，无法简单水平扩展 | 初期 `replicas: 1`；长期需考虑将接口改造为 gRPC/REST |
| 文件路径硬编码 | 中 | 源码中可能有 Windows 绝对路径（如 `D:\inetpub\...`），在 Linux 容器中失败 | 全面排查 `.prm` 文件引用和文件 IO 操作 |
| 数据库网络连通性 | 中 | SQL Server 在企业内网，K8s Pod 需要能访问 `ms1152.cmp.sankyoseiki.co.jp` | 配置 K8s 网络策略，或使用 VPN/专线打通 |
| 字符编码 | 低 | `.prm` 文件内有日文字符（Shift-JIS编码），Linux 容器默认 UTF-8 | 在 JVM 启动参数中添加 `-Dfile.encoding=UTF-8`，并将配置文件转码为 UTF-8 |
| 时区 | 低 | 服务运行在日本时区（JST, UTC+9） | 容器内设置 `TZ=Asia/Tokyo` |

---

## 11. 实施路线图

### 第一阶段（本期）：容器化准备（预计 2 周）

- [ ] 1.1 从现有 Windows 服务器提取 Fujitsu 专有 JAR 包
- [ ] 1.2 将 JAR 包上传至 Nexus/GitHub Packages 私有仓库
- [ ] 1.3 在 `it_common/MSWConnection.java` 中替换 JDBC 驱动（最小改动）
- [ ] 1.4 创建 Gradle 构建文件，验证本地能完整编译
- [ ] 1.5 本地 `docker build` 验证 Dockerfile 正确性
- [ ] 1.6 本地 `docker run` 验证容器能正常启动和注册 RMI

### 第二阶段：CI/CD 接入（预计 1 周）

- [ ] 2.1 在 GitHub 仓库中配置 Secrets（Nexus 凭据、KUBECONFIG、DB 凭据）
- [ ] 2.2 创建 GitHub Actions Workflow 文件
- [ ] 2.3 验证自动构建和镜像推送
- [ ] 2.4 配置 Trivy 镜像扫描

### 第三阶段：K8s 部署（预计 1 周）

- [ ] 3.1 创建 K8s 命名空间和 RBAC 配置
- [ ] 3.2 创建 ConfigMap 和 Secret
- [ ] 3.3 部署到开发/测试集群，验证功能
- [ ] 3.4 验证与 SQL Server、SAP R/3、NPM 等外部系统的连通性
- [ ] 3.5 部署到生产集群

### 第四阶段（后续）：长期演进建议

- [ ] 4.1 将客户端从 Java Applet 改造为独立 Swing 应用（解决浏览器兼容问题）
- [ ] 4.2 逐步将 RMI 接口迁移为 gRPC（解决 K8s 扩展性问题）
- [ ] 4.3 引入 Spring Boot 包装层，利用其 Actuator 提供健康检查端点
- [ ] 4.4 引入统一配置中心（Spring Cloud Config / etcd）替代 `.prm` 文件

---

## 附录 A：相关文件清单

### 当前源码关键文件
- `service/businese/Sms01206/OrderEntryNo_s.java` — RMI 服务端实现（核心）
- `service/businese/Sms01206/OrderEntryNo_i.java` — RMI 接口定义
- `service/businese/Sms01206/OrderEntryNo_L.java` — 服务器启动器
- `service/common/it_common/parameter/MSWConnection.java` — 数据库连接（需改造）
- `service/common/it_common/parameter/ParameterFile.java` — 参数文件读取
- `service/common/sms_common/RemoteIfc.java` — RMI 基础接口

### 运行时配置文件
- `service/businese/Sms01206/sms_common/rmi_ms1152.prm` — RMI/数据库连接参数
- `service/businese/Sms01206/sms_common/server_program.prm` — 服务程序注册配置
- `service/businese/Sms01206/sms_common/reset.prm` — 法人代码配置
- `service/businese/Sms01206/sms_common/master.prm` — 主数据服务器配置

### 待新增文件（本文档方案输出）
- `service/businese/Sms01206/build.gradle` — Gradle 构建文件
- `service/businese/Sms01206/settings.gradle` — Gradle 项目设置
- `service/businese/Sms01206/Dockerfile` — 容器镜像构建文件
- `service/businese/Sms01206/rmi.policy` — RMI 安全策略
- `.github/workflows/sms01206.yml` — CI/CD Workflow
- `k8s/sms01206/*.yaml` — K8s 清单文件

---

## 附录 B：GitHub Secrets 配置清单

在 GitHub 仓库的 `Settings → Secrets and variables → Actions` 中需配置以下 Secrets：

| Secret 名称 | 说明 |
|-------------|------|
| `NEXUS_URL` | 私有 Nexus 仓库地址 |
| `NEXUS_USER` | Nexus 用户名 |
| `NEXUS_PASS` | Nexus 密码 |
| `KUBECONFIG` | K8s 集群访问凭据（Base64编码） |
| `DB_URL` | SQL Server JDBC URL |
| `DB_USER` | 数据库用户名 |
| `DB_PASSWD` | 数据库密码 |

---

*本文档由 AI 辅助生成，基于对 OrgSms 项目代码的静态分析。具体实施时请以实际运行环境和测试结果为准。*
