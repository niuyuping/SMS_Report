# orderEntryNo 传入参数快速验证指南

通过不同 URL 参数访问画面，对照下表确认「控制台输出」与「画面变化」即可快速验证参数是否生效。

---

## 一、控制台确认（必做）

1. 打开画面后按 **F12** 打开开发者工具 → **Console**。
2. 应看到两条日志：
   - `[SMS01206 orderEntryNo] 传入参数:` + 对象（corporate、tableType、entryNo、kidouParameter 等）
   - 若有 kidouParameter：`[SMS01206 orderEntryNo] kidouParameter 拼接结果:` + 字符串

若没有这两条，说明前端未正确拿到配置或脚本未执行到 mounted。

---

## 二、URL 示例（替换为你的主机与 context path）

假设画面地址为：  
`http://localhost:8080/smsweb/websms01206/maintenance/orderEntryNo`

以下均为在该地址后加 `?` 和参数。

---

## 三、按参数验证画面变化

### 1. table_type（最容易看出差异）

| 传参 | URL 示例 | 画面变化 |
|------|----------|----------|
| **order**（默认） | `?corporate_code=X&table_type=order` | 选择得意先后，**会显示**「与信限度額: xxx」；控制台 `tableType: 'order'`。 |
| **plan** | `?corporate_code=X&table_type=plan` | 选择得意先后，**不显示**「与信限度額」；控制台 `tableType: 'plan'`。 |

**快速验证**：同一得意先，分别用 `table_type=order` 和 `table_type=plan` 打开，对比是否有「与信限度額」即可。

---

### 2. corporate_code（看标签文字）

| 传参 | URL 示例 | 画面变化（主标签） |
|------|----------|--------------------|
| **X**（默认） | `?corporate_code=X` | 機展書No.、機種、ケースマーク1～7、按钮「配送先」。 |
| **V** | `?corporate_code=V` | **機種** → 「機種/工程図番」；**第1列** → 「倉庫/出荷場所」；**第2列** → 「case mark」；按钮仍「配送先」。 |
| **U** 或 **R/L/M/S/Q/3**（NNSN 系） | `?corporate_code=U` | **機展書No.** → 「在庫引当数」；**機種** → 「前注文番号」；ケースマーク3～7 变为「顧客ライン」「納入時刻」「登録者」「EDIエラーNo」「企業コード」。 |
| **V + NNSN** | `?corporate_code=V` | V 优先：機種/工程図番、倉庫/出荷場所、case mark 等（同上）。 |

**快速验证**：  
- 用 `corporate_code=V` 打开，看「機種」旁是否变成「機種/工程図番」、第1/2列是否变成「倉庫/出荷場所」「case mark」。  
- 用 `corporate_code=U` 打开，看「機展書No.»是否变成「在庫引当数」、「機種」是否变成「前注文番号」。

---

### 3. kanagata（金型）

| 传参 | URL 示例 | 画面变化 |
|------|----------|----------|
| 无 | 不传 `kanagata` | 上述默认或 V/NNSN 标签。 |
| 有值 | `?corporate_code=X&kanagata=1` | **機種** → 「機種」；**第1列** → 「販売窓口」；**第2列** → 「拠点」；ケースマーク3～7 → 「国内支援」「販売種」「型加工拠点」「型納入拠点」「成形製造拠点」；按钮 → **「得意先詳細」**。 |

**快速验证**：加 `kanagata=1` 打开，看按钮是否从「配送先」变为「得意先詳細」，以及第2列是否变为「拠点」。

---

### 4. 组合示例

- **V 法人 + order**：  
  `?corporate_code=V&table_type=order`  
  → 与信显示 + V 用标签（機種/工程図番、倉庫/出荷場所、case mark）。

- **plan + 不显示与信**：  
  `?corporate_code=X&table_type=plan`  
  → 无「与信限度額」。

- **金型**：  
  `?corporate_code=X&kanagata=1`  
  → 金型用标签 + 按钮「得意先詳細」。

---

## 四、小结

| 想验证的内容 | 建议操作 |
|--------------|----------|
| 参数是否传到前端 | 看控制台是否有 `[SMS01206 orderEntryNo] 传入参数:` 及 `kidouParameter`。 |
| table_type 是否生效 | 对比 `table_type=order` 与 `table_type=plan` 下是否显示「与信限度額」。 |
| corporate_code 是否生效 | 对比 `corporate_code=X` 与 `corporate_code=V`（或 U）的标签文字。 |
| kanagata 是否生效 | 加 `kanagata=1` 看按钮是否变为「得意先詳細」、第2列是否变为「拠点」。 |

按上述步骤即可快速确认传入参数已在画面和控制台生效。
