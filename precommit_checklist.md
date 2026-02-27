# Pre-commit Checklist（提交前检查清单）

本文档适用于 **SmsWeb** 模块的提交前自检。请逐项确认后再执行 commit。

---

## 提交前自检清单（勾选）

- [ ] 仅提交了 SmsWeb 相关变更
- [ ] `sms.webapi.url` 不包含模块端点路径
- [ ] `static` 已同步到 `public`
- [ ] `SmsWeb/module.xml` 中 `id` 为 `SmsWeb`
- [ ] 新增 Controller 放在 `spring/app/{模块名称}/`
- [ ] 新增 Service 放在 `spring/domain/service/{模块名称}/`
- [ ] 新增前端静态资源放在 `webapp/static/{模块名称}/`
- [ ] 新增 JSP 模板放在 `webapp/WEB-INF/views/spring/{模块名称}/maintenance/`

---

## 1. 提交范围

- [ ] 仅提交了 SmsWeb 相关变更

**说明**：提交时只包含 `SmsWeb/` 目录下的变更。

| 示例类型 | 以 Sms01206（websms01206 模块）为例 |
|----------|------------------------------------|
| ✅ 正确 | `SmsWeb/` 下变更：如 `SmsWeb/src/main/java/spring/app/websms01206/`、`SmsWeb/src/main/webapp/static/websms01206/` |
| ❌ 错误 | 提交了 `SmsWebApi/`、`RMI_SERVER/`、`Sms01206/`（旧版 Java）、`SmsWebApi/src/...` 等无关项目 |

---

## 2. sms.webapi.url

- [ ] `sms.webapi.url` 不包含模块端点路径

**说明**：配置文件 `SmsWeb/src/main/conf/message/mymaster/mymaster-message.properties`，配置项 `sms.webapi.url`，值中**不得包含**模块端点路由路径。

| 示例类型 | 以 Sms01206 为例 |
|----------|-----------------|
| ✅ 正确 | `sms.webapi.url=http://10.0.2.9:80/imart` |
| ❌ 错误 | `sms.webapi.url=http://10.0.2.9:80/imart/sms01206` |

模块端点路径 `/sms01206` 在 `ImartApiClient.java` 内拼接，完整 URL = `sms.webapi.url` + `/sms01206`。

---

## 3. static 与 public 同步

- [ ] `static` 已同步到 `public`

**说明**：每次提交前，将 `webapp/static` 的内容复制到 `public/static`。

| 示例类型 | 以 Sms01206 为例 |
|----------|-----------------|
| ✅ 正确 | `src/main/public/static/websms01206/` 与 `src/main/webapp/static/websms01206/` 内容一致，包含同版本 js/css 等 |
| ❌ 错误 | 只改了 `webapp/static/websms01206/` 未同步到 `public/static/websms01206/`，或两处文件不一致 |

```bash
# 从 SmsWeb 项目根目录执行
mkdir -p src/main/public/static
cp -r src/main/webapp/static/* src/main/public/static/
```

---

## 4. module.xml 检查

- [ ] `SmsWeb/module.xml` 中 `id` 为 `SmsWeb`

**说明**：每次提交前确认 `SmsWeb/module.xml` 中的 **id** 为 `SmsWeb`。

| 示例类型 | 以 Sms01206 模块所在项目为例 |
|----------|-----------------------------|
| ✅ 正确 | `<id>SmsWeb</id>` |
| ❌ 错误 | `<id>mypackage.SmsWeb</id>`、`<id>SmsWebApi</id>`、`<id>Sms01206</id>` |

---

## 5. 代码结构约定

- [ ] 新增 Controller 放在 `spring/app/{模块名称}/`
- [ ] 新增 Service 放在 `spring/domain/service/{模块名称}/`
- [ ] 新增前端静态资源放在 `webapp/static/{模块名称}/`
- [ ] 新增 JSP 模板放在 `webapp/WEB-INF/views/spring/{模块名称}/maintenance/`

**说明**：以下以 Sms01206（模块名 websms01206）为例：

| 类型 | ✅ 正确示例 | ❌ 错误示例 |
|------|------------|------------|
| 后端 Controller | `SmsWeb/src/main/java/spring/app/websms01206/WebSms01206Controller.java` | `spring/app/websms01206/` 放在其他包下，或写成 `spring/app/sms01206/` |
| 后端 Service | `SmsWeb/src/main/java/spring/domain/service/websms01206/WebSms01206ServiceImpl.java` | `spring/domain/service/sms01206/` 或 `spring/domain/websms01206/` |
| 前端静态资源 | `SmsWeb/src/main/webapp/static/websms01206/js/websms01206-createForm.js` | `webapp/static/sms01206/` 或 `webapp/static/websms01206-createForm.js`（未按模块分子目录） |
| 前端 JSP 模板 | `SmsWeb/src/main/webapp/WEB-INF/views/spring/websms01206/maintenance/createForm.jsp` | `views/websms01206/createForm.jsp`（缺少 maintenance）、或 `views/sms01206/` |

**路由约定**：根路由 `/{模块名称}/maintenance`，API 在 `api` 下。以 websms01206 为例：`/websms01206/maintenance`、`/websms01206/maintenance/api/getOrder`。
