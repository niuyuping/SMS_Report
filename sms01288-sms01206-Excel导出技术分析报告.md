# sms01288 与 sms01206 Excel 导出技术分析报告

**文档版本**：1.0  
**创建日期**：2025-03-05  
**适用系统**：SMS 同梱品・オーダーエントリー連携
验证用数据24122ABM
---

## 1. 概述

### 1.1 背景

本报告针对 **sms01288（同梱品設定・修正画面）** 与 **sms01206（Order Entry 画面）** 之间的联动关系，以及如何导出与同梱品相关 Excel 数据的操作逻辑进行技术分析。

### 1.2 分析结论摘要

| 项目 | 结论 |
|------|------|
| **sms01288** | 无 Excel 导出功能，需通过联动打开 sms01206 实现 |
| **sms01206** | 具备 Excel 导出功能（出荷依頼票出力 ダイアログ） |
| **关联方式** | sms01288 点击粉色エントリーNo → 新窗口打开 sms01206 并传入 entry_no |
| **导出流程** | sms01206 中点击注文数列 → 出荷依頼票ダイアログ → 「to excel」按钮 |

---

## 2. 系统架构与关系

### 2.1 画面职责

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│  sms01288                       │     │  sms01206                       │
│  同梱品設定・修正画面              │ ──► │  Order Entry 画面               │
│                                 │     │                                 │
│  • 同梱品（包裹）数据管理          │     │  • 订单数据管理                   │
│  • 无 Excel 导出功能              │     │  • 出荷依頼票出力                 │
│  • 可联动打开 sms01206            │     │  • Excel 导出功能                 │
└─────────────────────────────────┘     └─────────────────────────────────┘
```

### 2.2 联动触发条件

**sms01288 中点击表格行时，满足以下条件可打开 sms01206：**

| 条件项 | 说明 |
|--------|------|
| 点击列 | エントリーNo 列（G_ENTRY） |
| 单元格样式 | 粉色背景（`java.awt.Color.pink`） |
| 权限 | `sdHoujinFlg_ == true` |
| 数据内容 | エントリーNo 非空 |

**粉色单元格的设定场景：**

- 出荷済数不为 0 的数据行（`setlockLines` 中设定）
- 新追加的数据行（`applyDetailData` 中设定）

---

## 3. 技术实现详细分析

### 3.1 sms01288 → sms01206 联动实现

**调用链：**

```
表格点击 → kidouEntryApplet() → kidouOrderEntryNo()
```

**关键代码：**

- 文件：`java applet/sms01288/Sms01288.java`
- 方法：`kidouEntryApplet`（行 2453）、`kidouOrderEntryNo`（行 2469）

**URL 生成逻辑：**

```
http:{host}:8080/stemapki.jsp?program_nm=sms01206
  &userid={userid}
  &passwd={passwd}
  &Language={lang}
  &kidou_parameter={kidou},entry_no={entry},entry_sub_no=0
  &ScreenWidth={width}
  &ScreenHeight={height}
  &time={timestamp}
```

**参数说明：**

| 参数 | 用途 |
|------|------|
| `entry_no` | エントリー番号，sms01206 加载时使用 |
| `entry_sub_no` | 枝番，默认 "0" |
| `kidou_parameter` | 起動参数，用于后续联动 |

### 3.2 sms01206 初始化与数据加载

**sms01206 启动时：**

1. 从 URL 解析 `entry_no`、`entry_sub_no`，存入 `firstEntry_`、`firstEntrySub_`
2. 调用 `entryNo_.setText(firstEntry_)` 设置エントリーNo
3. 调用 `searchEntryNo()` 执行 RMI 查询
4. 查询成功后设置 `updateSw_ = UPDATE`，进入更新模式
5. 若 `firstSw_ == "on"`，自动执行 `setSearchData(firstEntrySub_, "sub")` 显示明细

**因此，从 sms01288 打开后，sms01206 自动处于可编辑状态，无需额外操作。**

### 3.3 Excel 导出入口与触发条件

**ForwardRequestDialog（出荷依頼票出力 ダイアログ）触发条件：**

| 条件 | 说明 |
|------|------|
| 点击列 | 表格「注文数」列（NO_DETAIL，第一列） |
| 画面模式 | `updateSw_ == UPDATE` |
| 法人 | 属于 iraihyouHoujin_（X, H, O, W, N, P） |
| 出荷模式 | 非試作出荷（`!isFactoryNumberUpdate()`） |

**调用链：**

```
表格 NO_DETAIL 列点击 → displayForwardDialog() → ForwardRequestDialog.show()
```

**Excel 导出：**

- 用户点击「to excel」按钮
- 调用 `toExcel(key, fdata, riekiCenterItem, sRiyuu)`
- 同梱品时通过 `editDoukonExcel()` 获取父子エントリー数据一并导出

### 3.4 同梱品 Excel 导出逻辑

**同梱品判定：**

- `d_package_number` 非空，或 `d_package_entry_no` 非空

**处理流程：**

1. `editDoukonExcel(exdata)` 根据父エントリー的 `d_entry_no`、`d_entry_sub_no`、`d_detail_no` 作为条件
2. 调用 `remoteObject_.getKoOrder(prm)` 取得子エントリー
3. 将父・子数据合并后生成 Excel 用 Tab 分隔文本
4. 复制到剪贴板，打开 Excel 模板 `Sms01206_01.xls`

**数据来源：**

- sms01206 的 RMI（`OrderEntryNo_s`）及 `SMS01206.select1`
- 不涉及 sms01288 的接口调用

#### 3.4.1 同梱品判定逻辑详解

**涉及数据库字段：**

| 字段 | 含义 |
|------|------|
| `d_package_number` | 同梱品数（親：有值时表示该订单为同梱品的親） |
| `d_package_entry_no` | 同梱品エントリーNo（子：指向親的 entry_no） |
| `d_package_entry_sub_no` | 同梱品エントリー枝番 |
| `d_package_detail_no` | 同梱品明细番号 |

- **親**：`d_package_number` 非空
- **子**：`d_package_number` 为空，但 `d_package_entry_no` 非空（指向親的エントリー）

**sms01288 的判定**（`Sms01288.java` 约 2388-2401 行）

```java
public boolean isPackageEntry(String[] data){
    if (!isEmpty(data[PACKAGE_NUMBER])){    // data[41] 同梱品数
        return true;   // 親
    }
    if (!isEmpty(data[PACKAGE_ENTRY_NO])){   // data[42] 同梱品エントリーNo
        return true;   // 子
    }
    return false;
}
```

**sms01206 的判定**（`OrderEntryNo.java`）

- **画面用** `isPackageData()`（约 15146-15156 行）：`return (oyaPackege_) || (koPackege_);`
- **oyaPackege_ / koPackege_ 的设定** 在 `setMidashiLabel(data)`（约 12175-12201 行）：
  - `d_package_number` 非空 → `oyaPackege_ = true`（親）
  - `d_package_entry_no` 非空 → `koPackege_ = true`（子）
- **Excel 导出时**（约 12446-12450 行）：仅检查 `d_package_number` 非空

**总结：** 親看 `d_package_number`，子看 `d_package_entry_no`，任一满足即判定为同梱品。

#### 3.4.2 d_package_number 的获取来源与取得时机

`d_package_number` 对应数据库表 `order_tbl` 的列 `package_number`，`d_` 为显示层命名前缀。

**1. 存储位置**

- 表：`order_tbl`
- 列：`package_number`（`OrderLine01` 中对应 cols_[77]）

**2. 对应接口**

| 场景 | RMI 接口 | 方法 | 实现类 |
|------|----------|------|--------|
| 画面表示 | `OrderEntryNo_i` | `getOrder(Hashtable prm)` | `OrderEntryNo_s` |
| Excel 导出 | `OrderEntryNo_i` | `getExcelData(Hashtable prm)` | `OrderEntryNo_s` |
| 同梱品子数据 | `OrderEntryNo_i` | `getKoOrder(Hashtable prm)` | `OrderEntryNo_s` |

- 接口：`java applet/Sms01206/Sms01206_s/src/SMS01206/OrderEntryNo_i.java`
- 实现：`java applet/Sms01206/Sms01206_s/src/SMS01206/OrderEntryNo_s.java`
- RMI 名：`OrderEntryNo_s`（`OrderEntryNo_L.java` 中 bind）

**getOrder 返回结构中 d_package_number 的位置：**

- 返回值：`Hashtable`，key `ANS` 对应 `subHash`
- `subHash` 结构：`"D1"`, `"D2"`, … 表示各明细行，每行对应一个 `detailHash`
- `detailHash` 结构：key 为明细列索引字符串（`"0"`, `"1"`, …, `"20"`, …）
- **d_package_number 的 key**：`"20"`
- 取值：`detailHash.get("20")` 即 d_package_number 的值
- `createOneData(detailHash, MAX_DETAIL)` 将其转为 `String[]`，`data[20]` 对应 d_package_number

**代码依据**（`OrderEntryNo_s.java`）：

1. **detailItems_ 定义**（约 99-122 行）：`detailItems_[20] = "d_package_number"`
2. **常量定义**（约 209 行）：`PACKAGE_NUMBER_DETAIL = 20`
3. **getOrder 写入**（约 866-870 行）：
   ```java
   for (int iDetail = 0; iDetail < detailItems_.length; iDetail++){
       if (!"".equals(detailItems_[iDetail])){
           item = toValue(lines[i].get(detailItems_[iDetail]));  // iDetail=20 时取 "d_package_number"
           String iStr = new Integer(iDetail).toString();       // iStr = "20"
           detailHash.put(iStr, item);                           // key="20"
       }
   }
   ```
4. **实际使用**（约 1733 行）：`packageNumber = (String)subDetail.get(new Integer(PACKAGE_NUMBER_DETAIL).toString());` → `subDetail.get("20")`

**3. 取得时机（什么时候拿到）**

| 场景 | 取得时机 | 调用链 |
|------|----------|--------|
| **sms01206 画面表示** | 选择枝番并调用 `setSearchData` 时 | `setSearchData` → `remoteObject_.getOrder(prmHash)`（约 4990 行）→ `OrderEntryNo_s.getOrder` → `selectOrder`（SMS01206.select）→ 查询 order_tbl → 返回的 `lines[i]` 中 `lines[i].get("d_package_number")` 取得 → 填入 `detailHash` → `createOneData` 生成 `data1[i]` → `setMidashiLabel(data1[i])` 时使用 `data1[i][PACKAGE_NUMBER_DETAIL]` |
| **sms01206 Excel 导出** | 用户点击「to excel」后执行 `toExcel` 时 | `toExcel` → `getExcelData(key)` → `OrderEntryNo_s.getExcelData` → `selectOrder`（SMS012016.select42）→ 查询 order_tbl → `lines[i].get("d_package_number")` 取得 → 填入返回的 `exVector` → `exdata[getExcelItemIndex("d_package_number")]`（约 12447 行）|

**4. 读取路径（sms01206 Excel 导出）**

```
toExcel → getExcelData(key) → OrderEntryNo_s.getExcelData
  → remoteObject_.selectOrder(hash)
  → SQL: SMS012016.select42
  → WHERE: d_entry_no = ? AND d_entry_sub_no = ?
  → 从 order_tbl 查询，返回行包含 package_number
  → 映射为 exdata[getExcelItemIndex("d_package_number")]
```

**5. 写入路径**

| 场景 | 文件 | 说明 |
|------|------|------|
| sms01288 同梱品新增 | `Sms01288_s.java` | `insertCopyData`（约 1407 行）：親（i==0）时 `line[0].set("d_package_number", pNumber)`，`pNumber` 为同梱品数 |
| sms01288 同梱品更新 | `Sms01288_s.java` | `updateCopyData`（约 1459 行）：親 时 `line[0].set("d_package_number", pNumber)`，子 时设为 NULL |
| 同梱品数更新 | `PackageOrderUpdater.java` | `updateParent`（约 181 行）：`package_number = 'count_'` 更新親记录的件数 |
| 同梱品作成 | `PackageOrderClient.java` | `head.set("package_number", "" + bodys_.size())`（约 754 行）：親头行写入同梱件数 |

**6. 数据流概要**

```
order_tbl.package_number（DB 列）
    ↑ 写入                    ↓ 读取
sms01288 / PackageOrder*    selectOrder (SMS01206.select / SMS012016.select42)
    → INSERT/UPDATE            → getOrder / getExcelData 返回时即已取得
                                   → 画面：setSearchData → setMidashiLabel
                                   → Excel：toExcel → exdata 中取得
```

#### 3.4.3 getKoOrder 接口入参说明

**用途**：根据親エントリー的 entry_no / entry_sub_no / detail_no，查询同梱品子エントリー列表。

**入参**（Hashtable prm）：

| Key（OrderEntryNo_i 常量） | 类型 | 必填 | 说明 |
|----------------------------|------|------|------|
| `KEY`（"key"） | `String[]` | ✓ | 親エントリー：[0]=entry_no, [1]=entry_sub_no, [2]=detail_no；子记录以 d_package_entry_no / d_package_entry_sub_no / d_package_detail_no 匹配 |
| `KEY1`（"key1"） | `String[]` | ✓ | 返回子数据的列名数组，如 `koExcelItems_` 或 `koItems_` |
| `CORPORATE`（"corporate"） | `String` | ✓ | 法人コード |
| `TYPE`（"type"） | `String` | ✓ | テーブル種別 |
| `LANG`（"lang"） | `String` | ✓ | 言語（ja/en 等） |

**KEY 构造示例**（editDoukonExcel，约 13204-13210 行）：

```java
String[] oyaEntry = new String[3];
oyaEntry[0] = oyaData[getExcelItemIndex("d_entry_no")];
oyaEntry[0] = getCutEntryNo(oyaEntry[0]);  // entry_no 需经 getCutEntryNo 处理
oyaEntry[1] = oyaData[getExcelItemIndex("d_entry_sub_no")];               // 親 entry_sub_no
oyaEntry[2] = oyaData[getExcelItemIndex("d_detail_no")];                  // 親 detail_no
prm.put(OrderEntryNo_i.KEY, oyaEntry);
prm.put(OrderEntryNo_i.KEY1, koExcelItems_);  // Excel 用 14 列
```

**KEY1 可选值**：

- **koExcelItems_**（Excel 出力用 14 列）：d_entry_no, d_entry_sub_no, d_order_no, d_product_code, d_product_name, d_forward_date, d_reply_date, d_reply_code, d_on_board_date, d_inspect_date, d_order_number, d_unit_price, d_currency_name, d_currency_code
- **koItems_**（画面用 17 列）：同上 + d_detail_no, d_delivery_date, d_forward2_date, d_status 等

**内部查询条件**（OrderEntryNo_s.java 约 2129-2131 行）：

```sql
d_package_entry_no = oya[0] 
AND d_package_entry_sub_no = oya[1] 
AND d_package_detail_no = oya[2]
```

**返回值**：`Hashtable`，key `ANS` 对应 `Vector`，元素为 `String[]`，顺序与 KEY1 一致。

### 3.5 点击 to excel 后的详细逻辑解读

#### 阶段 1：按钮点击与对话框关闭

**ForwardRequestDialog.java**（约 623-638 行）

```java
public void excelButton___action_actionPerformed(java.awt.event.ActionEvent e) {
    // 試作修正时需先选择修正理由，否则禁用
    if (isShisakuSyuusei()){
        Vector vector = getSyuuseiRiyuu();
        if (0 == vector.size()){ excelButton_.setEnabled(false); }
    }
    if (excelButton_.isEnabled()) {
        excel_ = true;      // 标记为 Excel 出力
        confirm_ = false;  // 非确认（出荷依頼表印刷）
        dispose();         // 关闭对话框
    }
}
```

- 设置 `excel_ = true`，关闭对话框，返回到 `displayForwardDialog` 的后续逻辑。

#### 阶段 2：分支判断与参数收集

**OrderEntryNo.java**（约 4157-4220 行）

`displayForwardDialog` 中对话框关闭后的处理：

```java
if (forwardDialog_.isConfirm()) {
    // 用户点击的是「確認」→ 出荷依頼表 HTML 印刷
    // ...
} else {
    if (forwardDialog_.isExcel()) {  // 用户点击的是「to excel」
        String[] key = new String[2];
        key[0] = label;   // entry_no（当前选中枝番的エントリーNo）
        key[1] = sub;     // entry_sub_no
        Hashtable ansHash = forwardDialog_.getValue1();  // 从对话框获取用户输入
        String[] fdata = (String[])ansHash.get("ans");    // 印刷种类、备注等
        String riekiCenterItem = toValue((String)ansHash.get("ans1"));  // 利益センター
        Vector sRiyuu = (Vector)ansHash.get("ans2");     // 試作修正时的修正理由
        toExcel(key, fdata, riekiCenterItem, sRiyuu);
    }
}
```

**getValue1()**（ForwardRequestDialog.java 约 491-528 行）从对话框取得：

- `ans`：`printSw1_`（新规/修正/削除）+ `note_` 中的 5 个 token（依頼内容）
- `ans1`：利益センター选中的代码（list1_ 选中行第 2 列）
- `ans2`：試作修正时的修正理由向量（販価/数量/販売モデル/その他）

#### 阶段 3：toExcel 主流程

**OrderEntryNo.java**（约 12423-12493 行）

| 步骤 | 处理 | 代码位置 / 说明 |
|------|------|-----------------|
| 3.1 | 光标改为 WAIT | `setCursor(Cursor.WAIT_CURSOR)` |
| 3.2 | RMI 取得订单明细 | `exVector = getExcelData(key)` → `OrderEntryNo_s.getExcelData`，SQL `SMS012016.select42`，按 entry_no/sub_no 查 order_tbl |
| 3.3 | 机种/試作/同梱判定 | `d_product_code` 头 3 位 `NP-` → 機展；`judgeShisakuProduct` → 試作；`d_package_number` 非空 → 同梱品 |
| 3.4 | 拼装 Excel 文本 | 见下方「printData 拼装」 |
| 3.5 | 复制到剪贴板 | `Clipboard.setContents(StringSelection(printData), null)` |
| 3.6 | 打开 Excel 模板 | `explorer.exe "http://{server}/excel/Sms01206/Sms01206_01.html"`，HTML 重定向到 `Sms01206_01.xls` |

#### 阶段 4：printData 拼装结构

`printData` 为 Tab 分隔、换行分隔的 TSV 文本，由以下方法顺序拼成：

| 序号 | 方法 | 内容 |
|------|------|------|
| 1 | `editExcel1` → `editExcelOutData1` | 表头：出荷依頼票标题、日期、依頼No、同梱品/会社間试作标记、担当/得意先/機種等、列名行 |
| 2 | `editExcel2` → `editExcelOutData2/3` | 明细表：每行 `d_delivery_date`～`d_comment_note`，最多 10 行，不足补空行 |
| 3 | `editExcel3` 或 `editExcel4` | 一般机种：連絡事項、配送先、ケースマーク等；試作机种：機展書 No、Rev、製品分野等 |
| 4 | `editExcel5` | 出荷日回答、利益センター、商品企画流程等 |
| 5 | `editDoukonExcel`（同梱品时） | 同梱品リスト：親データ（`editDoukonExcel1`）+ 子データ（`editDoukonExcel2`，由 `getKoOrder` 取得） |

#### 阶段 5：同梱品时的子数据取得

**editDoukonExcel**（OrderEntryNo.java 约 13198-13233 行）

1. 以父的 `d_entry_no`、`d_entry_sub_no`、`d_detail_no` 为条件
2. 调用 RMI `remoteObject_.getKoOrder(prm)`
3. **OrderEntryNo_s.getKoOrder**（约 2108 行）：按 `d_package_entry_no`、`d_package_entry_sub_no`、`d_package_detail_no` 查子エントリー
4. 用 `editDoukonExcel1` 输出親データ，`editDoukonExcel2` 输出子エントリー列表

#### 阶段 6：输出与打开

- **剪贴板**：TSV 文本已放入系统剪贴板
- **模板**：通过 `Sms01206_01.html` 重定向到 `Sms01206_01.xls`，用 `explorer.exe` 打开
- **使用方式**：在打开的 Excel 中粘贴（Ctrl+V）即可将剪贴板内容填入模板；或依赖模板内既有结构配合粘贴

#### 调用链总览

```
excelButton 点击
  → excel_=true, dispose()
  → displayForwardDialog 中 isExcel() 为 true
  → getValue1() 取得 fdata, riekiCenterItem, sRiyuu
  → toExcel(key, fdata, riekiCenterItem, sRiyuu)
       → getExcelData(key)        [RMI: getExcelData → SMS012016.select42]
       → editExcel1/2/3|4/5        [拼装 TSV]
       → editDoukonExcel(exdata)  [同梱品时, RMI: getKoOrder]
       → 剪贴板.setContents(printData)
       → explorer.exe → Sms01206_01.html → Sms01206_01.xls
```

### 3.6 整条逻辑链贯通解读

本节从用户操作出发，按时间顺序串联 sms01288 → sms01206 → Excel 导出的完整数据流与决策分支。

---

#### 阶段 0：同梱品数据在 DB 中的形态

**数据模型**（`order_tbl`）：

| 记录类型 | package_number | d_package_entry_no | d_package_entry_sub_no | d_package_detail_no |
|----------|----------------|--------------------|------------------------|---------------------|
| **親**   | 非空（同梱件数） | 空                 | 空                     | 空                  |
| **子**   | 空              | 親的 entry_no       | 親的 entry_sub_no      | 親的 detail_no      |

- 親记录：`d_package_number` 表示该订单下同梱品数量。
- 子记录：`d_package_entry_no` / `d_package_entry_sub_no` / `d_package_detail_no` 指向親，建立父子关系。
- 写入来源：sms01288、PackageOrderClient 等；读取来源：sms01206 的 selectOrder（SMS01206.select / SMS012016.select42）。

---

#### 阶段 1：sms01288 中用户点击粉色エントリーNo

**触发条件**（Sms01288.java `kidouEntryApplet` 约 2453-2466 行）：

1. 点击列为 `G_ENTRY`（エントリーNo）
2. 单元格边框颜色为 `Color.pink`
3. `sdHoujinFlg_ == true`（法人权限）
4. エントリーNo 非空

**处理流程**：

```
kidouEntryApplet(entry, row)
  → kidouOrderEntryNo(entry, row)
  → 拼接 URL：stemapki.jsp?program_nm=sms01206&entry_no={entry}&entry_sub_no=0&...
  → getAppletContext().showDocument(url, "_blank")
```

**结果**：新窗口打开 sms01206，URL 中携带 `entry_no`、`entry_sub_no=0`。

---

#### 阶段 2：sms01206 启动与数据加载

**解析与初始化**：

1. 从 URL 解析 `entry_no`、`entry_sub_no` → 存入 `firstEntry_`、`firstEntrySub_`
2. `entryNo_.setText(firstEntry_)` 设置エントリーNo 输入框
3. 调用 `searchEntryNo()` 执行 RMI 查询
4. 成功后 `updateSw_ = UPDATE`，进入更新模式
5. 若 `firstSw_ == "on"`，自动调用 `setSearchData(firstEntrySub_, "sub")` 显示明细

**数据取得**（`setSearchData` → 约 4990 行）：

```
remoteObject_.getOrder(prmHash)
  → OrderEntryNo_s.getOrder
  → remoteObject_.selectOrder(hash)  // SMS01206.select
  → SQL 查询 order_tbl，返回 OrderLine01[] lines
  → 各明细行 lines[i].get("d_package_number") 填入 detailHash.put("20", ...)
  → createOneData 生成 data1[i]，data1[i][20] 即为 d_package_number
  → setMidashiLabel(data1[i]) 设定 oyaPackege_ / koPackege_
```

**此时**：画面已拿到 `d_package_number`、`d_package_entry_no` 等，用于同梱品标记显示。

---

#### 阶段 3：用户点击注文数列（NO_DETAIL）

**触发条件**（OrderEntryNo.java 约 9006-9024 行）：

1. `column == NO_DETAIL`（注文数，第一列）
2. `updateSw_.equals(UPDATE)`
3. `isItemAri(iraihyouHoujin_, corporate_)`（法人 X/H/O/W/N/P 等）
4. `!isFactoryNumberUpdate()`（非試作出荷）
5. 枝番在 1～10 范围内（出荷依頼表仅支持枝番 10 以内）

**处理**：

```
displayForwardDialog()
  → forwardDialog_.setValue(...)
  → forwardDialog_.show()
```

**结果**：弹出「出荷依頼票出力 ダイアログ」，用户可选择「確認」（HTML 印刷）或「to excel」。

---

#### 阶段 4：用户点击「to excel」

**ForwardRequestDialog**（约 623-638 行）：

```java
excel_ = true;
confirm_ = false;
dispose();
```

**displayForwardDialog 后续**（约 4211-4219 行）：

```java
if (forwardDialog_.isExcel()) {
    key[0] = label;   // 当前枝番对应 entry_no
    key[1] = sub;     // entry_sub_no
    toExcel(key, fdata, riekiCenterItem, sRiyuu);
}
```

**此时**：`key` 为当前选中明细的 `[entry_no, entry_sub_no]`，即 Excel 导出的查询键。

---

#### 阶段 5：toExcel 主流程

**5.1 取得 Excel 用明细**

```
getExcelData(key)
  → remoteObject_.getExcelData(prm)
  → OrderEntryNo_s.getExcelData
  → remoteObject_.selectOrder(hash)  // SMS012016.select42
  → WHERE d_entry_no = key[0] AND d_entry_sub_no = key[1]
  → 返回 exVector，exdata = exVector.elementAt(0)
```

**5.2 同梱品判定**（约 12446-12449 行）：

```java
String packageNo = exdata[getExcelItemIndex("d_package_number")];
if (!isEmpty(packageNo)) { doukon = true; }
```

- `exdata` 来自 `getExcelData`，列顺序由 `excelItems_` 定义，`d_package_number` 对应 DB 的 `package_number`。
- `doukon = true` 表示当前行为同梱品親，需要追加同梱品列表。

**5.3 拼装 printData**

| 步骤 | 方法 | 内容 |
|------|------|------|
| 1 | editExcel1 | 表头、同梱品标记、列名等 |
| 2 | editExcel2 | 明细表（最多 10 行） |
| 3 | editExcel3 或 editExcel4 | 一般/試作机种差异区 |
| 4 | editExcel5 | 出荷日回答、利益センター等 |
| 5 | **editDoukonExcel**（同梱品时） | 親データ + 子データ列表 |

**5.4 同梱品子数据取得**（editDoukonExcel 约 13198-13228 行）

```
exdata（親）→ 取出 d_entry_no, d_entry_sub_no, d_detail_no
  → oyaEntry[0] = getCutEntryNo(exdata["d_entry_no"])
  → oyaEntry[1] = exdata["d_entry_sub_no"]
  → oyaEntry[2] = exdata["d_detail_no"]
  → prm.put(KEY, oyaEntry)
  → remoteObject_.getKoOrder(prm)
```

**OrderEntryNo_s.getKoOrder**（约 2108-2152 行）：

```
prm.get(KEY) → oya[]
  → WHERE d_package_entry_no = oya[0] 
        AND d_package_entry_sub_no = oya[1] 
        AND d_package_detail_no = oya[2]
  → remoteObject_.selectOrder(hash)  // SMS01206.select1
  → 返回子エントリー的 OrderLine01[] lines
  → 按 koExcelItems_ 列名取出，组装 Vector<String[]>
  → ansHash.put(ANS, vector)
```

**逻辑含义**：以親的 `entry_no / entry_sub_no / detail_no` 为条件，在 order_tbl 中查找所有 `d_package_*` 指向该親的子记录。

**5.5 输出与打开**

```
printData += editDoukonExcel1(oyaData, midashi1)   // 親 1 行
printData += editDoukonExcel2(koVector, midashi1)  // 子 N 行
  → Clipboard.setContents(printData)
  → explorer.exe "http://{server}/excel/Sms01206/Sms01206_01.html"
  → HTML 重定向到 Sms01206_01.xls
```

用户在 Excel 中粘贴（Ctrl+V）即可将数据填入模板。

---

#### 阶段 6：整条链路中的数据流小结

```
order_tbl.package_number / d_package_*
    │
    ├─ 画面表示：getOrder → detailHash["20"] → setMidashiLabel（oyaPackege_/koPackege_）
    │
    └─ Excel 导出：
         ├─ getExcelData → exdata["d_package_number"] → doukon 判定
         └─ doukon 时：editDoukonExcel
              ├─ 親：exdata（来自 getExcelData 的当前行）
              └─ 子：getKoOrder(親的 entry/sub/detail) → koVector
                   → WHERE d_package_entry_no/sub_no/detail_no = 親
```

---

#### 关键决策点汇总

| 位置 | 条件 | 分支 |
|------|------|------|
| sms01288 表格点击 | 粉色 + sdHoujinFlg_ | 打开 sms01206 |
| sms01206 注文数列点击 | UPDATE + iraihyouHoujin_ + 非試作出荷 | 弹出出荷依頼票ダイアログ |
| ダイアログ关闭 | isExcel() | 进入 toExcel |
| toExcel 内 | d_package_number 非空 | 调用 editDoukonExcel |
| editDoukonExcel | - | getKoOrder 查子エントリー，拼入 printData |

---

#### 代码引用速查

| 环节 | 文件 | 方法/行 |
|------|------|---------|
| sms01288 点击判定 | Sms01288.java | kidouEntryApplet 2453, kidouOrderEntryNo 2469 |
| sms01206 参数接收 | OrderEntryNo.java | firstEntry_, searchEntryNo |
| 注文数列点击 | OrderEntryNo.java | 9006-9019, displayForwardDialog 4125 |
| to excel 按钮 | ForwardRequestDialog.java | excelButton 623-638 |
| toExcel 主流程 | OrderEntryNo.java | toExcel 12423-12493 |
| 同梱判定 | OrderEntryNo.java | 12446-12449 |
| editDoukonExcel | OrderEntryNo.java | editDoukonExcel 13198-13233 |
| getKoOrder | OrderEntryNo_s.java | getKoOrder 2108-2152 |
| getOrder 中 d_package_number | OrderEntryNo_s.java | detailItems_[20], 866-870 |
| getExcelData | OrderEntryNo_s.java | getExcelData → SMS012016.select42 |

---

## 4. 完整操作流程（用户视角）

### 4.1 导出与 sms01288 相关 Excel 的操作步骤

| 步骤 | 画面 | 操作内容 |
|:----:|------|----------|
| 1 | sms01288 | 打开同梱品設定・修正画面，定位到需要导出的同梱品行 |
| 2 | sms01288 | 点击该行的**粉色**エントリーNo 单元格 |
| 3 | 新窗口 | 自动打开 sms01206，并带入该 entry_no，自动加载订单数据 |
| 4 | sms01206 | 在表格中点击对应行的**注文数**列（第一列）单元格 |
| 5 | 出荷依頼票出力 ダイアログ | 弹出对话框，选择「to excel」按钮 |
| 6 | - | 生成 Excel 文件（同梱品时自动包含父子エントリー）<sup>1</sup> |

<sup>1</sup> **代码依据**：

- **同梱品判定与调用**（`OrderEntryNo.java` 约 12446-12463 行）：
  ```java
  String packageNo = exdata[getExcelItemIndex("d_package_number")]; 
  if (!isEmpty(packageNo)){ doukon = true; }
  // ...
  if (doukon){ printData = printData + editDoukonExcel(exdata); }  //040116
  ```
- **父子数据取得与拼接**（`OrderEntryNo.java` 约 13198-13228 行）：
  ```java
  public String editDoukonExcel(String[] oyaData){
      // 以父 d_entry_no/d_entry_sub_no/d_detail_no 为条件
      prm.put(OrderEntryNo_i.KEY, oyaEntry);
      Hashtable ansHash = remoteObject_.getKoOrder(prm);  // RMI 取得子エントリー
      koVector = (Vector)ansHash.get(OrderEntryNo_i.ANS);
      printData = editDoukonExcel1(oyaData, midashi1);   // 親データ部
      printData = printData + editDoukonExcel2(koVector, midashi1);  // 子データ部
  }
  ```
- **子エントリー取得实现**：`OrderEntryNo_s.java` 的 `getKoOrder(Hashtable prm)`（约 2108 行）

### 4.2 流程示意图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  sms01288    │     │  sms01206    │     │  Excel       │
│  同梱品画面   │     │  Order Entry │     │  出力        │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │ ① 点击粉色          │                    │
       │    エントリーNo      │                    │
       │ ② 新窗口打开         │                    │
       │    ────────────────►│                    │
       │                    │                    │
       │                    │ ③ 点击注文数列      │
       │                    │ ④ 弹出出荷依頼票     │
       │                    │    ダイアログ        │
       │                    │ ⑤ 点击「to excel」   │
       │                    │ ───────────────────►│
       │                    │                    │
       │                    │                    │ ⑥ Excel 生成
       │                    │                    │    （同梱品含父子）
       │                    │                    │
```

---

## 5. 相关代码清单

| 功能 | 文件路径 | 方法/类 |
|------|----------|---------|
| sms01288 打开 sms01206 | `java applet/sms01288/Sms01288.java` | `kidouEntryApplet`, `kidouOrderEntryNo` |
| sms01206 参数接收与初始化 | `java applet/Sms01206/Sms01206/src/SMS01206/OrderEntryNo.java` | `firstEntry_`, `searchEntryNo` |
| 出荷依頼票ダイアログ入口 | `OrderEntryNo.java` | `displayForwardDialog`（表格 NO_DETAIL 列点击） |
| 出荷依頼票ダイアログ | `java applet/Sms01206/Sms01206/src/SMS01206/ForwardRequestDialog.java` | `excelButton_`（"to excel"） |
| Excel 导出主逻辑 | `OrderEntryNo.java` | `toExcel`, `editDoukonExcel` |
| 同梱品子数据取得 | `java applet/Sms01206/Sms01206_s/src/SMS01206/OrderEntryNo_s.java` | `getKoOrder` |

---

## 6. 注意事项与限制

### 6.1 功能限制

- sms01288 本身**不具备** Excel 导出功能
- Excel 导出必须经由 sms01206 的出荷依頼票出力 ダイアログ完成
- 同梱品导出时，父子数据由 sms01206 的 RMI 取得，不依赖 sms01288

### 6.2 权限与条件

- 法人需在 `iraihyouHoujin_` 范围内（X, H, O, W, N, P）
- 需满足 `sdHoujinFlg_` 权限（sms01288 侧）
- 試作出荷画面不可使用出荷依頼票 Excel 导出

### 6.3 出荷依頼票行数限制

- 法人 X 时，出荷依頼票为 10 行显示限制（注释：060823）

---

## 7. 附录

### 7.1 术语对照

| 日文 | 中文 |
|------|------|
| 同梱品 | 同捆品/包裹 |
| エントリーNo | エントリー号 |
| 出荷依頼票 | 出荷依頼票 |
| 枝番 | 枝番 |
| 注文数 | 注文数 |
| 採番 | 採番 |
| 試作出荷 | 試作出荷 |



*文档结束*
