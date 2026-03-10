# 前端 MOCK 方案在各模块中的难度和风险对比

## 要点

- **服务与 ERP 交互的规模对比**  
  - 用表格对比 012C3 / 01206 / 01288 的接口数、取数来源、Pipe 类型、主数据退路等。
- **为何 Sms01206 / Sms01288 更容易通过 MOCK 与参数规避**  
  - **01288**：仅 judgmentERP + save 内 R3 移行依赖 ERP，主数据未设 FLAG "ERP"，无 setFactoryResults / getUserList，规避点少。  
  - **01206**：通常模式下 5 个取数先走 SMS，不传 KEY1/KEY2="ERP" 时不连 ERP，仅 setFactoryResults、moveR3、judgmentERP 需规避或 MOCK。
- **为何 Sms012C3 更难执行且风险更高**  
  - 13 个接口均需做 MOCK，且强耦合，调用链路多，很难设计出覆盖全部业务逻辑的MOCK数据，实施面大、难度高。
- **前端 vs 服务侧 Mock 数据量估算（Sms012C3，覆盖全部场景）**  
  - **前端 API 侧**：最小可跑通一条链约 **10～12 组**；中等规模（如 5 得意先×10 机种）覆盖全部分支约 **400～500 组**，规模增大可达 **1500 组以上**（仅 10 个取数接口）。  
  - **服务侧 Mock ERP（Pipe 另一端）**：最小约 **10 组内**（7 类命令各 1 组）；同等中等规模覆盖全部场景约 **350～700 组**，一份 71/72 数据可被多个接口共用，维护点更集中。

---

## 一、三个服务与 ERP 交互的规模对比


| 维度                                | Sms012C3                       | Sms01206                                      | Sms01288                             |
| --------------------------------- | ------------------------------ | --------------------------------------------- | ------------------------------------ |
| **明确依赖 ERP 的接口数量**                | **13 个**                       | 通常模式仅 **3 类**（送数2+判定1）；5 取数为「先 SMS、条件满足才 ERP」 | **2 类**（judgmentERP + save 内 R3 移行）  |
| **取数类（主数据）**                      | 10 个，均走 ERP                    | 5 个，**通常模式仅用 SMS**（画面不传 KEY1/KEY2="ERP"）      | 0 个明确（主数据用 SMS 类型）                   |
| **送数类**                           | moveR3 + **setFactoryResults** | moveR3 + setFactoryResults                    | 仅 save 内 moveR3（无 setFactoryResults） |
| **判定类**                           | judgmentERP                    | judgmentERP                                   | judgmentERP                          |
| **使用的 Pipe 类型**                   | 71 / 72 / 74 / 87 / 63 / 62·64 | 71 / 74 / 87 / 63 / 62·64（取数仅在試作出荷モード时可能走）    | 仅 62·64（显式）                          |
| **得意先·配送先·担当来源**                  | **全部 ERP（72）**                 | SMS（CUSTOMER_LIST_1/CONSIGN）                  | SMS（CUSTOMER_LIST_1/CONSIGN）         |
| **是否具备 getUserList（最终用户）**        | **有（71+72）**                   | 无                                             | 无                                    |
| **是否具备 getCustomerProduct（单品目）**  | **有（87）**                      | 无（内部有，非公开 5 取数）                               | 有但类型为 CUSTOMER_PRODUCT（非 ERP）        |
| **主数据 getMaster 是否显式 FLAG "ERP"** | **是，多处**                       | **仅当调用方传 KEY1/KEY2/KEY3="ERP" 时**（通常画面不传）     | **否**（PRICE/PRODUCT_LIST_1 等）        |


Sms012C3 的 ERP 接触面远大于另外两个：**接口多、Pipe 种类全、且关键主数据（客户/配送/担当）没有“用 SMS 主数据代替”的退路**，因此仅靠“少数几个参数 + 局部 MOCK”很难系统性地避开 ERP。**Sms01206** 在通常受注模式下 5 个取数不连 ERP（源码结论见《Sms01206_ERP_API_依赖关系图》第四节），故与 01288 一样易于在无 ERP 时维持业务。

---

## 二、为何 Sms01206 / Sms01288 更容易通过 MOCK 与参数规避

### 2.1 Sms01288：规避点少、主数据可不走 ERP

- **明确必须与 ERP 交互的只有 2 类**：  
  - **judgmentERP**（是否 ERP 连携机种）；  
  - **save / saveNewPackage 内部触发的 moveR3 → moveOrderToR3**（R3 移行）。
- **主数据**（getUnitPrice、getProducts、getCustomers、getConsigns 等）在实现中使用的是 **PRICE、PRODUCT_LIST_1、CUSTOMER_LIST_1、CUSTOMER_CONSIGN** 等，**未设置 FLAG "ERP"**，因此有机会从 **SMS 主数据或缓存** 取数，不一定连 ERP。
- **规避策略**简单：  
  - 用 MOCK 让 **judgmentERP** 返回“不连携”或固定值，使画面不进入“必须移行 R3”的分支；  
  - 或在不执行“移行 R3”的保存路径下操作（若业务允许），则 **不会触发 moveOrderToR3**；  
  - 主数据若共通层未走 ERP，则 **无需对主数据做 MOCK**，只需保证与 SMS 库一致即可。
- **无 setFactoryResults**：不存在“试作出荷实绩登録”按钮/流程，少一个必须 MOCK 或禁用的送数点。
- **无 getUserList / getCustomerProduct（ERP 版）**：没有依赖 Pipe 71+72 的最终用户列表、也没有显式 ERP 单品目查询，**需要保持一致的 MOCK 维度少**。

因此：**通过“MOCK judgmentERP + 控制是否执行 R3 移行”和“主数据可能天然不走 ERP”**，Sms01288 较容易在 ERP 全挂时仍让业务大体执行，**难度低、风险相对可控**。

### 2.2 Sms01206：通常模式下 5 个取数不连 ERP，仅送数·判定需规避

- **源码结论**（详见《Sms01206_ERP_API_依赖关系图》第四节）：  
  - **getPrice / getUnitPrice / isUnitPriceMaster / getProducts** 均为 **先 getMaster(PRICE 或 PRODUCT) 不设 FLAG**，仅当调用方传入 **KEY2="ERP" 或 KEY1="ERP"** 且 SMS 无数据时才再带 FLAG "ERP" 访问 ERP。  
  - 画面 **OrderEntryNo出荷依頼表改造.java** 仅在 **試作出荷モード**（`isFactoryNumberUpdate()`）下对上述接口传 KEY1/KEY2="ERP"。**通常受注画面不传**，故这 4 个取数在无 ERP 时 **仅用 SMS，可用**。  
  - **getCustomerProducts** 仅使用 **SMSMasterServer.CUSTOMER_PRODUCT**，无任何 FLAG "ERP" 或 ERP 分支，**始终只用 SMS**。  
  - **getConsigns / getCustomers** 使用 **CUSTOMER_CONSIGN、CUSTOMER_LIST_1**，来自 **SMS 主数据**。
- **因此通常模式下**，真正会连 ERP、必须规避或 MOCK 的只有 **3 类**：**setFactoryResults**、**moveR3**、**judgmentERP**。
- **规避策略**：  
  - **主数据**：通常模式下 **无需对 5 个取数做 MOCK**，它们已走 SMS；仅当在 **試作出荷モード** 下若 SMS 无数据会走 ERP，此时无 ERP 则无法补查，可考虑不进入该模式或 MOCK。  
  - **判定**：MOCK **judgmentERP** 或接受“无法判定是否 ERP 连携机种”。  
  - **送数**：通过 **参数或画面约束**（如不点“试作出荷实绩”“R3 移行”），或对 **setFactoryResults / moveR3** 做 MOCK“成功”并明确标识“未真正送 ERP”，即可让流程在 SMS 内跑通。
- **无 getUserList、无 getCustomerProduct（公开单品 ERP）**：比 012C3 少两个 ERP 强依赖，MOCK 维度少。

因此：**在通常受注模式下，无需对 5 个取数做 MOCK**，仅需对 **judgmentERP + 2 个送数** 做规避或 MOCK，Sms01206 即可在 ERP 不可用时维持业务大体执行，**难度低、风险低**（与 01288 接近）。

---

## 三、为何 Sms012C3 更难执行且风险更高

- **13 个接口、10 个取数均走 ERP**，客户/配送/担当无 SMS 退路；独有 **getUserList**（最终用户）、**getCustomerProduct**（单品目）、**setFactoryResults**（试作出荷实绩），规避点分散且维度多。
- **API 调用链耦合度高**，链路数量随业务环节呈指数级增加，要为每一条链路都设计一组严格约束的 MOCK 数据，否则任意一环不一致都会导致保存失败、移行失败或后续逻辑错误，**实施面大、难度高、风险高**。

---

## 四、从前端 API 调用侧 Mock 时的难点（以 Sms012C3 为例）

以下从**前端/API 调用侧**对 Sms012C3 的接口做 Mock（即对 getCustomers、getConsigns、getUserList、getPrice、moveR3 等 13 个 RMI 接口在网关或前端层拦截并返回手写数据）时的典型难点。

**场景一：受注录入到保存/R3 移行——多接口串联，一环不一致就断链**

- 用户操作顺序通常是：选担当 → **getCustomers** → 选得意先 → **getConsigns** → 选机种 → **getProducts** → 带出贩价 **getPrice/getUnitPrice** → **judgmentERP** → 保存 → **moveR3**。前端会**依次**调用上述多个 API。
- 若在**前端 API 调用侧**对每个接口分别 Mock：  
  - **getCustomers** 返回的得意先码，必须在 **getConsigns** 的 Mock 里作为入参能查到对应配送先；  
  - **getConsigns** 返回的配送先、**getProducts** 返回的机种，必须与 **getPrice/getUnitPrice** 的 Mock 中 (得意先, 机种) 维度一致，否则贩价带不出或报错；  
  - **moveR3** 的 Mock 要接受前面所有步骤拼出来的订单结构（法人/得意先/机种/贩价等），若与 getCustomers、getProducts、getPrice 等返回的码或格式不一致，保存/移行会报错或逻辑异常。
- **难点**：前端侧要维护的是**整条调用链**上多份 Mock 数据的一致性；改一个接口的返回值（例如多一个得意先），往往要同步改 getConsigns、getPrice、moveR3 等多处 Mock，否则任意一环不一致都会导致画面报错或保存失败，是典型的**多接口强耦合、前端侧 Mock 成本高**的场景。

**场景二：同一接口需多组 Mock 数据才能覆盖不同交互逻辑**

- 同一支接口在不同入参或不同业务分支下会走不同画面逻辑，若只做「单点」Mock（例如固定返回一组数据），只能覆盖一种路径，其它路径会报错或表现异常，因此**同一接口往往需要多组 Mock 数据**，按入参或场景切换返回。
- **例一：getPrice / getUnitPrice**  
  入参为 (法人, 得意先, 机种)（或 机种/得意先 顺序视实现而定）。画面会根据「是否有贩价」「贩价是否为零」走不同分支：有有效贩价时带出价格、贩价为零时可能提示「无偿出荷以外贩价 ZERO」、无贩价时提示「贩价マスター未登録」。要覆盖这三种交互，就需要至少三组 Mock：  
  - 对 (法人A, 得意先X, 机种P) 返回有效贩价；  
  - 对 (法人A, 得意先Y, 机种Q) 返回零或贩价区分表示无偿；  
  - 对 (法人A, 得意先Z, 机种R) 返回空/无数据。  
  若还要覆盖「不同担当下不同得意先列表」导致的多种 (得意先, 机种) 组合，Mock 组数会进一步增加。
- **例二：judgmentERP**  
  入参为 机种コード；返回是否 ERP 连携机种。为 true 时后续会走 R3 移行（moveR3），为 false 时不移行。要同时覆盖「走移行」与「不移行」两条交互路径（例如验证移行按钮是否可用、保存后是否调 moveR3），就需要至少两组 Mock：对机种 M1 返回 true、对机种 M2 返回 false，并在前面 getProducts/getPrice 等 Mock 里也提供 M1/M2，整条链才都能跑通。
- **难点**：同一接口的多组 Mock 不仅要满足「入参 → 返回值」一一对应，还要与其它接口的 Mock 在数据维度上一致（例如 judgmentERP 用到的 M1/M2 必须在 getProducts 的 Mock 里存在），否则仍会断链或走不到目标分支。

---

## 五、10 个取数接口 Mock 组数估算（覆盖全部场景）

以下基于 Sms012C3 的**业务与交互逻辑**（代码与依赖关系图），对 10 个取数接口在**前端 API 侧 Mock** 时所需「Mock 组」数量做估算。**1 组**指：在某一确定的入参（或入参组合）下，为覆盖一种业务分支或一种选择路径而准备的一类返回值。

### 5.1 各接口的入参维度与分支

| 接口 | 主要入参维度 | 返回值形态 | 为覆盖「全部场景」需区分的 Mock 组 |
|------|----------------------------|------------|----------------------------------------------|
| **getChargeCustomers** | 法人、担当コード | 列表（得意先・名称・担当） | 按**担当**区分：每 1 个担当 1 组列表。设担当数 = C_charge。 |
| **getCustomers** | 法人、检索键(key)、KEY1(机种) | 列表（得意先・名称・担当） | 两条路径：① 按担当/检索键 LIKE（Z001）→ 每 1 种 key 1 组；② 按机种 getCustomers2（CUSTOMER_PRODUCT_ERP2）→ 每 1 机种 1 组。设 key 形态数 = K_cust，机种数 = P。 |
| **getConsigns** | 法人、得意先 | 列表（配送先等） | 按**得意先**区分：每 1 个得意先 1 组配送先列表。设得意先数 = N_c。 |
| **getProducts** | 法人、机种检索键、KEY1(得意先[]) | 列表（机种・名称・品种等） | 两条路径：① 按检索键（%/前缀/完全一致）→ 每 1 种检索键 1 组；② 代理店用按得意先 getProductsDairiten → 每 1 得意先 1 组。设检索键形态数 = K_prod，参与代理店检索的得意先数 = N_d。 |
| **getCustomerProducts** | 法人、KEY1(得意先[]) | 列表（机种・名称） | 内部走 getProductsDairiten，按**得意先**区分：每 1 得意先 1 组。同上 N_d。 |
| **getPrice** | 法人、机种、得意先 | 单条（14+1 列：贩价・通貨・相手機種名等） | 按 **(法人, 得意先, 机种)** 区分；且需覆盖**有数据 / 无数据**等分支。设有效 (得意先, 机种) 组合数 = N_c × P，再乘以分支数 B_price（如 有/无 ≈ 2）。 |
| **getUnitPrice** | 法人、机种、得意先、有効日 | 单条（贩价・通貨 2 列） | 同上 key；另需覆盖**新贩价/现贩价**（由 s[8] 与 yyyymmdd 比较）、**无数据**。分支数 B_unit ≈ 3。 |
| **isUnitPriceMaster** | 法人、机种、得意先 | 布尔 | 同上 key；**true / false** 两种分支。2 组 per key，key 数 = N_c × P。 |
| **getCustomerProduct** | 法人、得意先、机种（prm 中的 KEY） | 单条（品目明细） | 按 **(法人, 得意先, 机种)** 区分；有/无数据。约 N_c × P × 2。 |
| **getUserList** | 法人(key[0])、当前最终用户(KEY1) | 列表 + 当前行（最终用户列表） | 按**法人**区分即可（内部 71+72 拼出列表）。设法人数 = Corp。 |

说明：上述「得意先数 N_c」「机种数 P」等是指**在 Mock 中需要区分的、能参与链路的**主数据条数，不是真实系统总量；且各接口之间必须**共用同一套** 法人/担当/得意先/机种 编码，否则链会断。

### 5.2 最小可跑通一条链（单路径）

- 1 法人、1 担当、1 得意先、1 配送先、1 机种、1 组贩价（有数据）、getUserList 1 组、getCustomerProduct 可选 1 组。  
- **约：** getChargeCustomers 1 + getCustomers 1～2 + getConsigns 1 + getProducts 1～2 + getCustomerProducts 1 + getPrice 1 + getUnitPrice 1 + isUnitPriceMaster 1 + getCustomerProduct 0～1 + getUserList 1  
- **合计约 10～12 组**（仅保证一条「选担当 → 选得意先 → 选配送先 → 选机种 → 带出贩价 → 保存」路径可跑通）。

### 5.3 覆盖「分支」与「多选择」后的估算式

在保证**链上数据一致**的前提下（同一批 担当/得意先/机种 在各接口中复用），需要：

- **getChargeCustomers**：C_charge 组（每担当一组列表）。
- **getCustomers**：K_cust 组（按担当/检索键）+ P 组（按机种 getCustomers2），共 **K_cust + P** 组。
- **getConsigns**：**N_c** 组（每得意先一组配送先）。
- **getProducts**：K_prod 组（按检索键）+ N_d 组（代理店按得意先），共 **K_prod + N_d** 组。
- **getCustomerProducts**：**N_d** 组（每得意先一组品目列表）。
- **getPrice**：**(N_c × P) × B_price** 组（B_price 建议 ≥ 2：有/无）。
- **getUnitPrice**：**(N_c × P) × B_unit** 组（B_unit 建议 ≥ 3：新贩价/现贩价/无）。
- **isUnitPriceMaster**：**(N_c × P) × 2** 组（true/false）。
- **getCustomerProduct**：**(N_c × P) × 2** 组（有/无，NSCM 品目）。
- **getUserList**：**Corp** 组（每法人一组最终用户列表）。

**合计（数量级）：**  
2·(N_c × P) + (N_c × P)·(B_price + B_unit) + (N_c × P)·2 + (N_c × P)·2 + (K_cust + P) + (K_prod + N_d) + N_c + N_d + C_charge + Corp  
≈ **(N_c × P)·(4 + B_price + B_unit) + K_cust + K_prod + P + 2·N_d + N_c + C_charge + Corp**。

代入**中等规模**示例：N_c = 5，P = 10，C_charge = 3，Corp = 2，K_cust = 2，K_prod = 3，N_d = 3，B_price = 2，B_unit = 3：

- (N_c×P)·(4+2+3) = 50×9 = **450**
- K_cust + K_prod + P + 2·N_d + N_c + C_charge + Corp = 2+3+10+6+5+3+2 = **31**
- **合计约 480 组**（仅 10 个取数接口，且未含 judgmentERP / moveR3 / setFactoryResults 的 Mock）。

### 5.4 服务侧 Mock ERP（Pipe 另一端）需多少组数据才能覆盖全部场景

在**服务侧**做 Mock 时，前端/ R3Server 不会直接调用「getPrice / getCustomers」等 13 个接口，而是由 **R3Server / sms012C3_s** 通过 **Pipe 命令**（71 / 72 / 74 / 87 / 62 / 64 / 63）与 Mock 服务通信。因此，Mock 侧只需按**命令号 + 请求参数**返回约定格式的报文，**一组数据**对应：某一条命令在某一类参数下的一个返回值（单行或多行）。

- **71（贩价）**：请求为 (corp, customer, product, date 等)。getPrice / getUnitPrice / isUnitPriceMaster 都走 71，**共用同一套 71 数据**，不必按「3 个接口」各做一份。覆盖全部场景需区分的组 = **N_corp × N_c × P** × 响应类型数（有/无/新贩价·现贩价 等，约 2～3）。  
  示例：2×5×10×2 = **200 组**。
- **72（得意先）**：请求为 (corp, customer, opt, flg)，flg 有 Z001（配送先等）、Z002、Z007（最终用户）。getConsigns / getCustomers / getEndUser 都走 72，**共用同一套 72 数据**，按 (corp, customer, opt, flg) 区分即可。组数 ≈ **N_corp × N_c × (opt 数 × flg 数)**，opt 常用 EQ/LIKE，flg 常用 Z001/Z002/Z007，约 2×3 = 6。  
  示例：2×5×6 = **60 组**。
- **74（机种）**：请求为 (corp, plant, product, opt)。getProducts / judgmentERP 间接用 74，**一套 74 数据**即可。组数 ≈ **N_corp × N_plant × P × 2**（EQ/LIKE），N_plant 常为 1。  
  示例：2×1×10×2 = **40 组**。
- **87（得意先品目）**：请求为 (corp, customer, cust_opt, product, prod_opt)。getCustomerProducts / getCustomerProduct 都走 87，**共用 87 数据**。组数 ≈ **N_corp × N_c × P × (cust_opt × prod_opt)**，约 2×2 = 4。  
  示例：2×5×10×4 = **400 组**（若不做 opt 区分可压到 100 组）。
- **62 / 64 / 63**：送数类，按「成功 / 失败」等分支即可。各 2 组左右，共约 **6 组**。

**合计（与 5.3 同等规模 N_corp=2, N_c=5, P=10, N_plant=1）：**  
71 约 200 + 72 约 60 + 74 约 40 + 87 约 100～400 + 62/64/63 约 6 → **约 406～706 组**。若 87 不做 opt 细分、71 只做有/无两种响应，可落在 **约 350～450 组**。

与**前端 API 侧**对比：同一业务规模下，前端 10 个取数接口约 **480 组**（见 5.3），服务侧约 **350～700 组**。服务侧**不会**因为 getPrice / getUnitPrice / isUnitPriceMaster 三个接口而把 71 数据复制三份，也不会因为 getConsigns / getCustomers / getEndUser 而把 72 复制多份，所以**组数略少或相当**；但服务侧要保证 71/72/74/87 的**报文格式**（列数、分隔符、顺序）与 R3Server 解析一致，否则仍会 NPE 或错位。

### 5.5 结论

- **最小可跑通一条链**：前端 10 个取数接口约 **10～12 组**；服务侧 Pipe 约 **7 类命令 × 各 1 组 ≈ 10 组内**（单法人/单得意先/单机种/单路径）。
- **覆盖全部场景（中等规模）**：  
  - **前端 API 侧**：10 个取数接口约 **400～500 组**，N_c×P 增大后可超 1500 组。  
  - **服务侧 Mock ERP**：71/72/74/87/62/64/63 合计约 **350～700 组**（与 N_c×P 相关，且 87 是否按 opt 细分会影响上限）。  
- 服务侧 Mock 的组数**与前端同量级**，但**一份 71 数据可同时满足 getPrice / getUnitPrice / isUnitPriceMaster**，**一份 72 可同时满足 getConsigns / getCustomers / getUserList 内的 getEndUser**，维护点更集中；若要「覆盖全部场景」，两者都需**数百组**数据，且必须与 judgmentERP、moveR3 等逻辑在数据维度上一致。
