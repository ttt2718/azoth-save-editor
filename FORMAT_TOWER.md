# 对战塔版（BAG16/PC18）存档格式与开发参考

本文件补充《宝可梦水银 对战塔特别版》（验证 V3.0）的格式结论。对战塔编辑器已并入 index.html（上传存档自动识别版本），其格式逻辑在 `tw-tower-core` 脚本块中。容器/校验思路与 FC 相同，但签名、分区、个体格式均不同。**两版性格顺序相同（FC 自定义顺序），闪光阈值不同（FC=16，对战塔=8）。**

## 1. 容器

- 扇区签名 `0x41313601`（FC 为 0x08012025）。双槽 0–13 / 14–27，计数器大者为当前。
- section 尺寸由 `meta.layout` 推导：sb2=3884、sb1=14560、pc=33540；id1–3 各 3968，id4=sb1-3×3968，id5–12 各 3968，id13=pc-8×3968。
- 校验：与 FC 相同算法；`compat.readSlot` 还验证 PC 头（magic `0x38314350` 等）与 AZTH 扩展 CRC。
- 直接复用 `docs/home/decomp-compat.js` 的 `parseSave` / `writeSave` / `readSlot` / `checksum` / `readRecord` / `writeRecord`，不要重写。

## 2. 箱子宝可梦（61 字节位打包）

- 18 箱 ×30，PC 区偏移 `17 + (box*30+slot)*61`，空槽全 0。
- 字段位宽见 `decomp-compat.js` 的 `fields` 表（personality/otId 32、nick 12×8、ot 7×8、species 11、teraType 5、heldItem 10、exp 21、ball 6、move 4×11、iv 6×5、natureModifier 5、shinyModifier 1、abilityNum 2 等）。
- **性格 = (personality%25) ^ natureModifier**，索引用 FC 自定义性格表；改性格只写 natureModifier，不动 PID。
- **闪光 = (shinyValue<8) ^ shinyModifier**；改闪光只翻 shinyModifier。
- **特性 = abilityNum（0/1/2）**，合法性由 `meta.species[id].abilities=[特性1,特性2,隐藏]` 检查。
- 等级不存，由 `meta.experienceTables[growthRate]`（8 组）从 exp 推算；exp 上限 21 位。
- 图标：`meta.forward.species` 的反向映射把对战塔 ID 转回 FC ID，再用 FC 的 icons。

## 3. 队伍宝可梦（100 字节，标准加密，位打包字段）

- SB1 `0x234`=数量，`0x238` 起 6 只。加密 key=PID^OTID，48 字节数据区，子结构按标准 PID%24 排列，+28 u16 校验（所有子结构 u16 求和）。
- 子结构内容（与原版不同，位打包）：
  - G0：species 11 bit + teraType 5 bit + heldItem 10 bit + ball 6 bit（正好 32 位）。
  - G1：低 21 bit = exp；高位其他数据保留。
  - A：4 招式各 11 bit 连排，随后 PP 等保留。
  - E：前 6 字节为 6 项 EV，其余华丽大赛数据保留。
  - M：M0 病毒/捕获地/相遇信息；M1 = IV 字（6×5bit + 高位标志）；M2 保留。
- **性格顺序与 FC 相同**（6 只成员能力值反推全部吻合，无薄荷干扰）；party 未见 natureModifier/shinyModifier，性格/闪光通过换 PID 实现（保留低字节与闪光状态，阈值 8）。
- 能力值重算：FC 种族值（reverseSpecies 映射）+ 与 FC 相同公式；改后当前 HP 回满。6 只原始成员**原值重写幂等（字节不变）6/6 通过**，可视为格式已完全掌握。
- 队伍特性不提供修改（party 特性位含义未验证）；队伍不可新增/删除。

## 4. BP 对战点数

- **BP 在 `SaveBlock2 + 0xEB8`（u16）**。注意：最初选用的 SB1+0xEE0 实机无效，已排除；SB2+0xEB8 与前者原值均为 645，为当前候选字段，写回时更新两个槽的 section 0 并重建校验和。若仍无效，需让用户改变 BP 后导出做差分（方法同 FC BracerPoints）。

## 5. 其他说明

- 背包/金钱不对战塔开放（用户需求）；队伍能力值依赖 FC 种族值映射，若某物种两版种族值不同需另行校正。
- 回归方法同 FC：Node 提取 script 块 eval（先 decomp-data/decomp-compat/icons，再 tower-core），箱子编辑/新增/清空、队伍幂等与编辑、BP 写回、双槽校验、UI `new Function` 语法检查。
