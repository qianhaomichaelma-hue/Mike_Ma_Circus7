# Living Circus — 项目开发上下文

> **用途**：在新的 AI 对话窗口中，将本文件内容作为 Prompt 发送，以恢复项目上下文继续开发。
> **最后更新**：2026/3/22

---

## 项目概览

- **项目名称**：Living Circus（马戏团）
- **引擎**：Unreal Engine 5.7
- **项目路径**：`f:\Github\Mike_Ma_Circus7`
- **项目文件**：`Ma_Mike_Circus7.uproject`
- **类型**：第一人称叙事恐怖独立游戏
- **核心机制**：玩家在灰色公寓（Hub关卡）中使用相机发现彩色入口，进入三个代表不同心理状态的"内心世界"关卡

---

## 当前作业目标

**Week 08 Assignment: Level Streaming and Game System Organization**

要求：
1. ✅ 创建新关卡并实现关卡间过渡（使用 Open Level，因为项目启用了 World Partition）
2. ✅ 实现工作的关卡切换机制
3. ✅ 将游戏逻辑重构到 GameMode / GameInstance
4. ⬜ 录制 2-4 分钟演示视频

---

## 项目架构

### 关卡结构

| 关卡 | 文件 | GameMode | 状态 |
|---|---|---|---|
| 公寓（Hub） | `Lvl_01_Apartment.umap` | `BP_FirstPersonGameMode` | ✅ 已有 |
| 时装秀（关卡一） | `Lvl_02_FashionShow.umap` | `BP_FashionShowGM` | ✅ 已创建 |
| 城市街道（关卡二） | `Lvl_03_CityStreet.umap` | 待创建 | ⬜ 未开始 |
| 下水道（关卡三） | `Lvl_04_SewerChase.umap` | 待创建 | ⬜ 未开始 |
| 蘑菇地狱 | `Lvl_PCG_ShroomHell.umap` | — | ✅ 已有（PCG） |
| 机制测试 | `Lvl_Test_Mechanism.umap` | — | ✅ 已有 |

### 关卡过渡方式

使用 **Open Level (by Name)** 方案（非 Level Streaming，因为 World Partition 已启用）。
跨关卡数据通过 GameInstance 保存。

### 蓝图继承关系

```
Game Mode Base
    └── BP_FirstPersonGameMode（公寓通用）
            └── BP_FashionShowGM（时装秀专用，子类继承）
                    ├── 继承：收集物/计分逻辑
                    └── 新增：红绿灯规则、移动检测
```

---

## 已完成的蓝图清单

### 1. BP_GameInstance01
**路径**：`Content/_Circus07/Blueprints/GameInstance/`
**已配置**：`DefaultEngine.ini` 中 GameInstanceClass 已指向此蓝图

**变量：**
| 变量名 | 类型 | 公开 | 用途 |
|---|---|---|---|
| `CompletedLevels` | Map (String→Boolean) | ✅ | 记录通关状态 |
| `CollectedItems` | Array of String | ✅ | 收集物品列表 |
| `PlayerCurrentHP` | Integer (默认3) | ✅ | 当前生命值 |
| `PlayerMaxHP` | Integer (默认3) | ✅ | 最大生命值 |
| `TotalScore` | Integer (默认0) | ✅ | 跨关卡总分 |

**函数：**
| 函数名 | 输入 | 输出 | 用途 |
|---|---|---|---|
| `MarkLevelComplete` | LevelName (String) | — | 标记关卡通关 |
| `IsLevelComplete` | LevelName (String) | bIsComplete (Bool) | 查询通关状态 |
| `AddScore` | ScoreAmount (Int) | — | 增加分数 |
| `AddCollectedItem` | ItemName (String) | — | 添加收集物（去重） |
| `ModifyHP` | Amount (Int) | bIsDead (Bool) | 修改生命值（Clamp 0~Max） |

---

### 2. BP_FashionShowGM
**路径**：`Content/_Circus07/Blueprints/GameMode/`
**父类**：`BP_FirstPersonGameMode`（已 Reparent）

**变量：**
| 变量名 | 类型 | 公开 | 默认值 |
|---|---|---|---|
| `bIsRedLight` | Boolean | ✅ | false |
| `bGameStarted` | Boolean | ❌ | false |
| `GreenLightDuration` | Float | ❌ | 3.0 |
| `RedLightDuration` | Float | ❌ | 4.0 |
| `PlayerStartLocation` | Vector | ❌ | (0,0,0) |
| `MovementThreshold` | Float | ❌ | 50.0 |
| `RespawnPoint` | Vector | ❌ | (0,0,0) |
| `bIsWarning` | Boolean | ✅ | false |
| `bBlinkState` | Boolean | ✅ | true |
| `WarningDuration` | Float | ❌ | 1.5 |
| `CurrentBlinkInterval` | Float | ❌ | 0.4 |
| `BlinkTimerHandle` | Timer Handle | ❌ | — |

**自定义事件：**
| 事件名 | 用途 |
|---|---|
| `StartGame` | BeginPlay 2秒后调用，开始游戏 |
| `SwitchToGreen` | 切绿灯，定时到 StartWarning |
| `SwitchToRed` | 切红灯，记录玩家位置，定时到 StartWarning |
| `StartWarning` | 开始闪烁警告，调用 DoBlink |
| `DoBlink` | 递归闪烁（间隔×0.75），结束后切换灯 |
| `OnPlayerCaught` | 红灯移动被抓，传送回起点 |
| `OnLevelComplete` | 通关时调用 GameInstance 记录 |

**Event Tick 逻辑**：
- bGameStarted → bIsRedLight → 检测玩家移动距离 > Threshold → OnPlayerCaught
- ⚠️ 已修复 Bug：红灯闪烁期间也检测移动（移除了 NOT bIsWarning 判断）

**Event BeginPlay**：
- 先调用 `Parent: BeginPlay`（继承父类逻辑）
- 然后 Delay 2秒 → StartGame

---

### 3. BP_TrafficLight
**路径**：`Content/_Circus07/Blueprints/`

**组件**：
- `SignalLight` (Point Light): Intensity 50000, Attenuation 3000
- `LightMesh` (Static Mesh): SM_Cube, Scale (3,3,3)

**Event Tick 逻辑**：
- Get Game Mode → Cast To BP_FashionShowGM
- bIsWarning=true → 根据 bBlinkState 切换亮(50000)/灭(0)，亮时颜色取决于 bIsRedLight
- bIsWarning=false → 正常显示：bIsRedLight=true→红色, false→绿色

---

### 4. BP_LevelPortal（通用传送门）
**路径**：`Content/_Circus07/Blueprints/`

**变量（Instance Editable）**：
- `TargetLevelName` (String): 目标关卡名，每个实例单独配置
- `bIsActive` (Boolean, 默认 true): 是否激活

**逻辑**：
- Overlap → 检查 bIsActive → Cast 玩家 → 验证 TargetLevelName 非空 → 尝试 Cast GameMode 调用 OnLevelComplete → Open Level

---

### 5. BP_Closet_Trigger
**路径**：`Content/_Circus07/Blueprints/`
**用途**：衣柜触发器，Open Level 跳转到 Lvl_02_FashionShow

---

### 6. 其他已有蓝图
- `BP_Light_Switch`: 灯光开关
- `BP_RespawnArea` / `BP_RespawnBox`: 重生区域
- `BP_FirstPersonCharacter`: 第一人称玩家角色
- `BP_FirstPersonCameraManager`: 相机管理器
- `BP_FirstPersonPlayerController`: 玩家控制器
- `BPI_Interactble`: 交互接口
- `WBP_CameraHUD` / `WBP_InteractionTip` / `WBP_PlayerHUD`: UI Widgets
- 收集品系统: `BP_BaseCoin`, `BP_BaseHealth`, `BP_BaseShield`, `BP_Master_Interactable`

---

## 待完成事项

### 🔴 本周必须完成

| # | 任务 | 状态 |
|---|---|---|
| 1 | ~~搭建 Level 架构（Open Level）~~ | ✅ 完成 |
| 2 | ~~时装秀核心玩法（红绿灯）~~ | ✅ 完成 |
| 3 | ~~重构 GameInstance 持久数据~~ | ✅ 完成 |
| 4 | 公寓进度门控（通关解锁门） | ⬜ **下一步** |
| 5 | 录制 2-4 分钟演示视频 | ⬜ 待做 |

### 🟡 第四步详细计划：公寓进度门控

需要实现：
1. 创建 `BP_DoorLock` 蓝图（或修改现有门蓝图）
2. 在 BeginPlay 中读取 GameInstance → `IsLevelComplete("Lvl_02_FashionShow")`
3. 根据结果控制卧室门是否可以打开
4. 添加视觉反馈（锁定/解锁状态的视觉变化）
5. 通关关卡一后解锁卧室门 → 引导玩家去客厅

### 🟢 后续迭代

| 任务 | 说明 |
|---|---|
| 关卡二：城市街道 | 空旷街道，收集发光假人 |
| 关卡三：下水道追逐 | 影子追逐，站定面对消灭 |
| 相机机制完善 | 滤镜、HUD、自动关闭 |
| 公寓环境完善 | 黑白调、粗轮廓线材质 |
| 音频系统 | 音效、音乐节奏与红绿灯联动 |

---

## 关键配置

### DefaultEngine.ini 修改记录
```ini
GameInstanceClass=/Game/_Circus07/Blueprints/GameInstance/BP_GameInstance01.BP_GameInstance01_C
GameDefaultMap=/Game/_Circus07/Maps/Lvl_01_Apartment.Lvl_01_Apartment
GlobalDefaultGameMode=/Game/FirstPerson/Blueprints/BP_FirstPersonGameMode.BP_FirstPersonGameMode_C
```

### 关卡 World Settings
| 关卡 | GameMode Override |
|---|---|
| Lvl_01_Apartment | BP_FirstPersonGameMode（全局默认） |
| Lvl_02_FashionShow | BP_FashionShowGM |

---

## 策划文档核心要点（Living Circus.docx 摘要）

- **公寓**：灰色、黑白单色调、Hub关卡
- **相机**：应用滤镜、看到可互动对象、短时间后自动关闭
- **关卡一（衣柜→时装秀）**：一二三木头人，红绿灯信号
- **关卡二（公寓大门→街道）**：收集发光假人
- **关卡三（浴室镜子→下水道）**：影子追逐，站定面对才能消灭
- **流程引导**：卧室→通关关卡一→解锁卧室门→客厅→关卡二→解锁浴室→关卡三
