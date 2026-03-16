本计划是在完整遍历当前 ProjectE 代码实现后的“可执行改写方案”，目标是在尽量复用现有体系（注册、容器、EMC、datagen、渲染）的前提下实现新的等价交换流程。

---

## 一、现状梳理（基于仓库已有实现）

### 1) 现有配方入口

- 所有主配方在 `src/datagen/java/moze_intel/projecte/common/recipe/PERecipeProvider.java`。
- 关键方法：
	- `fuelUpgradeRecipe`：当前炼金煤炭/莫比乌斯/永恒燃料升级依赖贤者之石。
	- `addTransmutationTableRecipes`：当前转化桌、便携式转化桌配方。
	- `addConversionRecipes` + `philoConversionRecipe`：当前“贤者之石+材料”交换（铁->金、金->钻石等）。

### 2) 现有转化桌与便携式转化桌入口

- 方块：`src/main/java/moze_intel/projecte/gameObjs/blocks/TransmutationStone.java`
	- 右键直接打开 `TransmutationContainer`。
- 物品：`src/main/java/moze_intel/projecte/gameObjs/items/TransmutationTablet.java`
	- 右键打开同一个 `TransmutationContainer`，但走手持来源。
- 容器：`src/main/java/moze_intel/projecte/gameObjs/container/TransmutationContainer.java`
- 背后逻辑：`src/main/java/moze_intel/projecte/gameObjs/container/inventory/TransmutationInventory.java`
	- 左侧 8 个输入槽 + 1 锁定槽会读取 EMC（含卡莱恩之星 EMC）。
	- EMC 余额由“玩家知识 EMC + 输入槽内可充能物品 EMC”共同组成。

### 3) 可复用的台座与卡莱恩之星机制

- 暗物质台座方块：`src/main/java/moze_intel/projecte/gameObjs/blocks/Pedestal.java`
- 台座方块实体：`src/main/java/moze_intel/projecte/gameObjs/block_entities/DMPedestalBlockEntity.java`
	- 单槽存储，可稳定读取放置物。
- EMC 工具：`src/main/java/moze_intel/projecte/utils/EMCHelper.java`
	- `getKleinStarMaxEmc`、EMC 持有能力读取逻辑可直接复用。

### 4) 注册与客户端接入点

- 方块注册：`PEBlocks`
- 方块实体注册：`PEBlockEntityTypes`
- 容器注册：`PEContainerTypes`
- 客户端 Screen 注册：`ClientRegistration`
- 方块状态/模型 datagen：`PEBlockStateProvider`、`PEItemModelProvider`
- 掉落表 datagen：`PEBlockLootTable`

### 5) EMC 映射与“禁止 EMC 获取”能力

- 映射管线：`EMCMappingHandler`
- Mapper 扩展点：`src/main/java/moze_intel/projecte/emc/mappers/*`
	- 已有 `OreBlacklistMapper`/`RawOreBlacklistMapper` 使用 `setValueBefore/After(..., 0)` 的做法，适合复用实现“转化桌/便携式转化桌不可通过 EMC 获取”。

---

## 二、目标规则（本分支最终行为）

### 1) 配方层

- 删除原“转化桌”与“便携式转化桌”配方。
- 删除或改写“贤者之石参与”的普通合成/交换配方。
- 炼金煤炭、莫比乌斯、永恒燃料升级不再依赖贤者之石。
- 贤者之石前期定位改为“便携式工作台功能道具”（不再承担燃料升级和交换配方触发）。

### 2) 新核心：多方块结构替代转化桌

- 结构中心为暗物质台座（D），并要求 D 上放置对应等级卡莱恩之星。
- tier 判定：
	- 依据 K 区块“满足条件1的方块种类数”
	- 依据 D 上卡莱恩之星等级
	- 依据 L 区块“暮色森林 boss 奖杯数量”。
- tier 每日转化额度限制：
	- 每个游戏日上限： $100000 * tier^2$ EMC。
- EMC 来源限制：
	- 可调用台座内卡莱恩之星 EMC。
	- 仅允许读取该结构唯一对应 D 的台座，不读取玩家背包左槽物品。

### 3) GUI 行为

- 继续复用转化桌 GUI/容器（减少前端改造成本）。
- 左侧“能量来源区”改为只读显示（不可手动放入/取出）或映射为虚拟槽位，数据来自 D 台座。

### 4) 交换与仪式

- 原贤者之石物品交换（铁<->金、金<->钻石等）迁移到该多方块结构触发。
- tier6 仪式 A：
	- 投入贤者之石 + 结构总 EMC >= 100000000 + 存在牛。
	- 强制雷击（无视避雷针）并击杀牛。
	- 产物：贤者之石 -> 转化桌。
- tier6 仪式 B：
	- 投入转化桌 + 结构总 EMC >= 100000000 + 存在牛。
	- 同样雷击流程。
	- 产物：转化桌 -> 便携式转化桌。
- 转化桌与便携式转化桌不可通过 EMC 学习/产出。

---

## 三、多方块结构定义（保留）

W 是水，K 是任意满足条件1的方块，L 是任意满足条件2的方块，D 是放有卡莱恩能量之星的暗物质台座，A 是空气。

条件1（K）：煤炭块，铁块，金块，钻石块，绿宝石块，下界合金块，暗物质块，红物质块。

条件2（L）：暮色森林任一 boss 奖杯，或空气。

第一层：
```
WWWWW
WWWWW
WWWWW
WWWWW
WWWWW
```

第二层：
```
AAAAA
AKKKA
AKWKA
AKKKA
AAAAA
```

第三层：
```
AAAAA
ALLLA
ALDLA
ALLLA
AAAAA
```

tier：

- tier1：符合结构，D 上至少一级卡莱恩之星。
- tier2：K 至少 2 种，D 上至少二级卡莱恩之星。
- tier3：K 至少 3 种，D 上至少三级卡莱恩之星。
- tier4：K 至少 4 种，D 上至少四级卡莱恩之星。
- tier5：K 至少 5 种，D 上至少五级卡莱恩之星，L 至少 4 个奖杯。
- tier6：K 至少 6 种，D 上至少六级卡莱恩之星，L 全为奖杯。

---

## 四、按文件施工计划（重点：哪些旧文件改、哪些新文件加）

以下分为“必须修改的现有文件”和“建议新增文件”。

### A. 必须修改的现有文件

1. `src/datagen/java/moze_intel/projecte/common/recipe/PERecipeProvider.java`
- 改 `fuelUpgradeRecipe`：去掉 `requires(PEItems.PHILOSOPHERS_STONE)`。
- 改 `addTransmutationTableRecipes`：删除转化桌与便携式转化桌合成。
- 改 `addConversionRecipes`：移除全部 `philoConversionRecipe` 调用或迁移为新结构专属配方体系。
- 保留贤者之石自身合成（是否保留由平衡再定）。

2. `src/main/java/moze_intel/projecte/gameObjs/registries/PEContainerTypes.java`
- 新增结构专用容器类型（建议 `TRANSMUTATION_ALTAR_CONTAINER`），避免直接改坏旧容器语义。

3. `src/main/java/moze_intel/projecte/gameObjs/registries/PEBlockEntityTypes.java`
- 注册新方块实体（结构核心 BE）。

4. `src/main/java/moze_intel/projecte/gameObjs/registries/PEBlocks.java`
- 注册新结构核心方块（如果采用“新核心方块 + 多方块外壳校验”方案）。

5. `src/main/java/moze_intel/projecte/ClientRegistration.java`
- 给新容器挂接 GUI（可复用 `GUITransmutation`，或新建子类）。

6. `src/main/java/moze_intel/projecte/utils/text/PELang.java`
- 增加结构 tier、每日限额、仪式成功/失败、结构缺失、只读槽提示等语言 key。

7. `src/main/resources/assets/projecte/lang/zh_cn.json`
- 补充上述新文案。

8. `src/datagen/java/moze_intel/projecte/client/PEBlockStateProvider.java`
- 新方块 blockstate/model 生成。

9. `src/datagen/java/moze_intel/projecte/client/PEItemModelProvider.java`
- 新方块对应物品模型。

10. `src/datagen/java/moze_intel/projecte/common/loot/PEBlockLootTable.java`
- 新方块掉落逻辑。

11. `src/main/java/moze_intel/projecte/emc/mappers/*`（新增 mapper 后，自动注解加载）
- 用 mapper 把转化桌与便携式转化桌 EMC 置 0（before+after），阻止 EMC 获取链。

### B. 建议新增文件（核心）

1. `src/main/java/moze_intel/projecte/gameObjs/blocks/TransmutationAltarCore.java`
- 新结构核心方块。
- 负责右键打开新容器。
- 不承担复杂逻辑，复杂度下沉到 BE + 结构服务类。

2. `src/main/java/moze_intel/projecte/gameObjs/block_entities/TransmutationAltarBlockEntity.java`
- 核心状态：
	- `currentTier`
	- `dailyEmcUsed`
	- `lastResetDay`
	- `cachedPedestalPos`
	- 仪式冷却/进行中状态
- 核心方法：
	- `scanAndValidateStructure()`
	- `computeTier()`
	- `getAvailableEmcFromPedestalStar()`
	- `canTransmute(emcCost)`（含每日额度校验）
	- `consumeEmc(emcCost)`
	- `tryPerformRitual(ItemStack catalyst)`

3. `src/main/java/moze_intel/projecte/gameObjs/container/TransmutationAltarContainer.java`
- 复用 `TransmutationContainer` 交互协议，但约束左侧槽。
- 关键：
	- 屏蔽玩家向左侧输入槽的写入。
	- 左侧显示映射到 D 台座物品快照。
	- 输出逻辑改为调用结构 BE 的 `consumeEmc`。

4. `src/main/java/moze_intel/projecte/gameObjs/container/inventory/TransmutationAltarInventory.java`
- 继承/改写 `TransmutationInventory`。
- 覆盖 `getAvailableEmc()`：来源改为结构 BE + 台座卡莱恩之星。
- 覆盖添加/扣除 EMC 行为：不再操作玩家个人 EMC。

5. `src/main/java/moze_intel/projecte/gameObjs/container/slots/transmutation/SlotReadOnlyEmcSource.java`
- 专门的只读槽，替代原 `SlotInput` 在新容器的左侧行为。

6. `src/main/java/moze_intel/projecte/gameObjs/logic/transmutation/AltarStructureRules.java`
- 定义 K、L 判定白名单。
- K 区块种类统计、L 奖杯统计。
- 与具体方块实体解耦，方便后续调平衡。

7. `src/main/java/moze_intel/projecte/gameObjs/logic/transmutation/AltarRitualLogic.java`
- 仪式入口与结果计算。
- 检测牛、触发雷击、物品替换、失败回滚。

8. `src/main/java/moze_intel/projecte/gameObjs/logic/transmutation/AltarDailyBudget.java`
- 每日额度计算：`limit = 100000 * tier * tier`。
- 根据 `level.getDayTime()/24000` 做自然日切换。

9. `src/main/java/moze_intel/projecte/emc/mappers/TransmutationProgressionBlacklistMapper.java`
- 把 `projecte:transmutation_table` 与 `projecte:transmutation_tablet` 设置 `setValueBefore=0` + `setValueAfter=0`。

10. `src/main/java/moze_intel/projecte/gameObjs/gui/GUITransmutationAltar.java`（可选）
- 若要明确显示“tier/当日额度/结构状态/仪式提示”，建议新 GUI 类继承 `GUITransmutation`。

### C. 建议新增资源文件

1. `src/main/resources/assets/projecte/textures/block/transmutation_altar_core/*`
2. `src/main/resources/assets/projecte/textures/gui/transmute_altar.png`（若做专属 GUI）
3. `src/main/resources/data/projecte/tags/blocks/transmutation_altar_k_candidates.json`
4. `src/main/resources/data/projecte/tags/items/twilight_trophies.json`

说明：K/L 建议走 tag，而非硬编码；代码读取 tag，配置层可扩展。

---

## 五、关键实现细节（避免返工）

### 1) 结构校验触发时机

- 方块更新时标记“脏”。
- 在核心 BE server tick 中做节流校验（如每 10 tick）。
- 打开容器前强制校验一次，避免显示过期 tier。

### 2) 每日额度重置

- 使用 `currentDay = level.getDayTime() / 24000`。
- `currentDay != lastResetDay` 时：
	- `dailyEmcUsed = 0`
	- `lastResetDay = currentDay`

### 3) EMC 消耗顺序

- 优先从 D 台座卡莱恩之星扣除。
- 不足则从玩家个人 EMC 补差。

### 4) 仪式雷击

- 使用服务端生成雷电实体并设置精确位置。
- 不依赖自然雷暴判定。
- 不受避雷针影响：直接在核心位触发，避免走 vanilla “引雷目标选择”。

### 5) 与旧转化系统兼容

- 保留 `TransmutationContainer` 给旧入口（若还存在）。
- 新结构走独立容器，避免破坏原功能链路。

---

## 六、分阶段里程碑

### 阶段 1：配方与 EMC 封口

- 完成配方改写：去贤者之石依赖，移除桌子/平板配方。
- 完成 mapper：桌子/平板 EMC=0。
- 验收：JEI/合成表中不存在旧配方，EMC 无法产出桌子/平板。

### 阶段 2：结构核心 + tier 判定 + 每日限额

- 核心方块/BE/结构判定服务完成。
- 能读取 D 台座星级并计算 tier。
- 每日额度与 EMC 消耗限制生效。

### 阶段 3：GUI/容器接入

- 新容器与 GUI 接入。
- 左侧只读显示来自 D 台座。
- 可正常进行 EMC 转化，且受 tier 日额度限制。

### 阶段 4：物品交换迁移 + 终极仪式

- 铁金钻等交换迁移到新结构。
- tier6 双仪式完整实现（贤者石->桌子，桌子->平板）。

### 阶段 5：资源与数据生成收尾

- 补模型、纹理、掉落、语言、进度。
- 跑 datagen 并检查生成物。

---

## 七、测试清单（必须过）

- 配方：
	- 转化桌/便携式转化桌无普通合成配方。
	- 燃料升级不再需要贤者之石。
- 结构：
	- 每个 tier 的判定边界正确（K 种类、L 奖杯数、星级）。
- 限额：
	- 不同 tier 的当日上限严格生效。
	- 跨天自动重置。
- EMC 来源：
	- 仅读取 D 台座中的卡莱恩之星。
	- 玩家背包/其他台座不参与。
- 仪式：
	- 条件不足时不触发且无物品损失。
	- 条件满足时雷击、杀牛、产物替换正确。
- EMC 获取封禁：
	- 转化桌与便携式转化桌不可通过 EMC 产出。

---

## 八、已知风险与对策

- 风险：直接魔改 `TransmutationInventory` 可能影响原桌子/平板逻辑。
	- 对策：新建 `TransmutationAltarInventory`，最小侵入。
- 风险：结构扫描频率过高导致服务器负担。
	- 对策：缓存 + 节流 + 邻居更新脏标记。
- 风险：boss 奖杯跨模组命名不稳定。
	- 对策：用 tag 管理奖杯集合，避免硬编码物品 ID。

---

## 九、本计划对原始需求的对应关系

- 去除旧桌子/平板及贤者石配方依赖：已覆盖。
- 燃料升级去贤者石：已覆盖。
- 多方块 tier + 每日限额：已覆盖。
- 只读取 D 上卡莱恩之星 + GUI 左侧只读：已覆盖。
- 原贤者石交换迁移：已覆盖。
- tier6 双仪式与雷击杀牛：已覆盖。
- 桌子/平板不可 EMC 获取：已覆盖（新增 mapper 方案）。
