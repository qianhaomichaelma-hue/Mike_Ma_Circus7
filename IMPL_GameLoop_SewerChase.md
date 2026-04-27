# Living Circus — Game Loop 完整实现方案（详细版）

> 生成日期：2026/04/27
> 涵盖范围：GameInstance 扩展 / 公寓门锁系统 / 第二关连接 / 第三关下水道新建 / 结算结束

---

## 一、总体流程图

```
[公寓卧室 spawn]
       │
  (柜门交互 BP_Closet_Trigger)  ← 已有
       ↓
[Lvl_02_FashionShow — 红绿灯]  ← 已有
       │ OnLevelComplete
       │   → MarkLevelComplete("Lvl_02_FashionShow")
       │   → SpawnAreaTag = "Bedroom_Exit"
       │   → Open Level → Lvl_01_Apartment
       ↓
[公寓 — 卧室门(BP_DoorLock)解锁 → 客厅]
       │
  (客厅大门 BP_LevelPortal)
       ↓
[Lvl_Test_Mechanism — 蘑菇地狱]
       │ OnLevelComplete
       │   → MarkLevelComplete("Lvl_Test_Mechanism")
       │   → SpawnAreaTag = "LivingRoom_Exit"
       │   → Open Level → Lvl_01_Apartment
       ↓
[公寓 — 厕所门(BP_DoorLock)解锁]
       │
  (厕所镜子相机交互 → BP_LevelPortal 激活)
       ↓
[Lvl_04_SewerChase — 下水道追逐]  ← 新建
       │ 消灭所有 BP_SewerEnemy
       │   → GM.SpawnPortal() → BP_LevelPortal 出现在玩家旁
       │   → 玩家踩传送门 → OnLevelComplete
       │   → MarkLevelComplete("Lvl_04_SewerChase")
       │   → 显示 WBP_Results
       ↓
[结算画面 / 游戏结束]
```

---

## 步骤 1 — BP_GameInstance01 新增 SpawnAreaTag 变量

**目标**：在公寓关卡 BeginPlay 时知道把玩家传送到哪个出生点。

### 操作步骤

1. 双击打开 `Content/_Circus07/Blueprints/GameInstance/BP_GameInstance01`
2. 在左侧 **Variables（变量）** 面板点击 **+** 新增变量：

| 字段 | 值 |
|---|---|
| Variable Name | `SpawnAreaTag` |
| Variable Type | String |
| Instance Editable | 否 |
| Default Value | `Bedroom` |

3. 点击工具栏 **Compile → Save**

> 说明：`"Bedroom"` 是默认值，代表玩家第一次进公寓时出生在卧室。  
> 每次从子关卡返回前，子关卡的 GM 会把这个值改成对应的 Tag，然后才 Open Level 回公寓。

---

## 步骤 2 — 新建 BP_DoorLock 蓝图

**目标**：一个可复用的带锁门，BeginPlay 检查 GameInstance 判断是否解锁，玩家走近可以尝试开门。

### 2.1 新建蓝图

1. 在 `Content/_Circus07/Blueprints/` 右键 → **Blueprint Class**
2. 父类选 **Actor** → 命名 `BP_DoorLock`
3. 双击打开

### 2.2 添加组件

在 **Components** 面板依次添加：

| 组件 | 类型 | 说明 |
|---|---|---|
| `DoorRoot` | Scene Component | 根节点，控制门轴位置 |
| `DoorMesh` | Static Mesh Component | 挂在 DoorRoot 下，门的外形 |
| `InteractBox` | Box Collision | 挂在根节点下（不跟门一起转），玩家靠近检测范围 |

> **注意**：`DoorMesh` 挂在 `DoorRoot` 下，开门动画旋转 `DoorRoot` 即可，`DoorMesh` 跟着转。  
> `InteractBox` 挂在默认 Root（不是 DoorRoot）下，这样它不会随门旋转，始终在门口位置不动。

组件层级示意：
```
DefaultSceneRoot
  ├── DoorRoot (Scene)
  │     └── DoorMesh (Static Mesh)
  └── InteractBox (Box Collision)
```

### 2.3 添加变量

| Variable Name | Type | Instance Editable | Default | 说明 |
|---|---|---|---|---|
| `RequiredLevelName` | String | ✅ | `""` | 需要通关的关卡名，在关卡里对每个实例单独填 |
| `bUnlocked` | Boolean | ❌ | false | 运行时状态，不需要在编辑器设置 |
| `ClosedRotation` | Rotator | ✅ | (0, 0, 0) | 门关闭时的 Rotation，在关卡里对齐好后填 |
| `OpenRotation` | Rotator | ✅ | (0, 90, 0) | 门打开后的 Rotation（Y轴即Yaw旋转90°） |
| `OpenSpeed` | Float | ✅ | 2.0 | 门打开动画速度 |
| `DoorTimeline` | Timeline | ❌ | — | 控制开门动画（下面创建） |

### 2.4 创建 Timeline 动画

1. 在 Event Graph 空白处右键 → 搜索 **Add Timeline** → 命名 `DoorTimeline`
2. 双击 `DoorTimeline` 节点进入 Timeline 编辑器：
   - 点击左上角 **+Float Track** → 命名 `Alpha`
   - 在轨道上 **右键 → Add key**：
     - Key 1：Time=0，Value=0
     - Key 2：Time=1，Value=1
   - 选中两个 Key，右键 → **Auto**（平滑插值）
   - **Length** 改为 `1.0`（1秒开门）
3. 关闭 Timeline 编辑器

### 2.5 BeginPlay 逻辑

在 Event Graph 连接以下节点：

```
[Event BeginPlay]
       │
       ▼
[Get Game Instance]
       │ Return Value
       ▼
[Cast To BP_GameInstance01]
       │ As BP_GameInstance01
       ▼
[Is Level Complete]  ← 输入：RequiredLevelName (Get变量)
       │
   ┌───┴───┐
  True   False
   │       │
   ▼       ▼
[Set bUnlocked = true]   [Set bUnlocked = false]
```

> Cast 失败分支（Cast Failed pin）可以忽略或接一个 Print String 调试用。

### 2.6 InteractBox Overlap 逻辑

选中 `InteractBox` 组件 → 在 Details 面板底部 Events 区域点击 **OnComponentBeginOverlap** 旁的 **+** 按钮，自动在 Event Graph 生成节点。

连接逻辑：

```
[OnComponentBeginOverlap (InteractBox)]
       │ OtherActor
       ▼
[Cast To BP_FirstPersonCharacter]
       │ Cast成功
       ▼
[Branch] ← Condition: Get bUnlocked
   │
   ├── True ──▶ [Is Valid? Get bDoorAlreadyOpen（可选防重复）]
   │                 │ False
   │                 ▼
   │            [Set bDoorAlreadyOpen = true]
   │                 │
   │                 ▼
   │            [Play (DoorTimeline)]  ← 从头播放开门
   │
   └── False ─▶ [Play Sound At Location]  ← 锁门音效（可选）
                      │
                      ▼
                [Print String "此门未开锁"]  ← 调试用，后期换UI提示
```

### 2.7 Timeline Update 驱动门旋转

`DoorTimeline` 节点有三个输出引脚：`Update`、`Finished`、`Alpha`（Float Track）

```
[DoorTimeline] Update 引脚
       │
       ▼
[Lerp (Rotator)]
  ├── A: Get ClosedRotation
  ├── B: Get OpenRotation
  └── Alpha: DoorTimeline.Alpha
       │ Return Value
       ▼
[Set World Rotation]  ← 目标选择 DoorRoot 组件（不是整个 Actor）
```

> `Set World Rotation` 节点：在节点左上角的目标引脚里，把 `DoorRoot` 组件拖进去，或者先 `Get DoorRoot` 再接过来。

### 2.8 Compile & Save，然后关闭

---

## 步骤 3 — 公寓关卡改造：多出生点 + Level Blueprint

### 3.1 放置 PlayerStart

在 `Lvl_01_Apartment` 关卡中：

1. 在 **Place Actors** 面板搜索 `Player Start`，拖入关卡放置 **3 个**
2. 分别在合适位置调整位置和朝向（Rotation Z 控制朝向）：

| 编号 | 放置位置描述 | 面朝方向 |
|---|---|---|
| PS_Bedroom | 卧室中央，游戏开始站的地方 | 朝向衣柜方向 |
| PS_BedroomExit | 卧室门口内侧（门开了往里走一步） | 朝向客厅方向 |
| PS_LivingRoomExit | 客厅靠厕所一侧 | 朝向厕所方向 |

3. 给每个 PlayerStart 加 Tag：
   - 选中 PlayerStart → Details → 搜索 **Tags**
   - 点击 **+** 添加一个元素
   - 分别填入：`Bedroom` / `Bedroom_Exit` / `LivingRoom_Exit`

### 3.2 修改 Level Blueprint

打开关卡蓝图：菜单栏 **Blueprints → Open Level Blueprint**

在 Event Graph 找到或创建 **Event BeginPlay**，连接以下逻辑：

```
[Event BeginPlay]
       │
       ▼
[Delay] ← Duration: 0.1
（等 World 完全加载再移动玩家，否则有时会失效）
       │
       ▼
[Get Game Instance]
       │
       ▼
[Cast To BP_GameInstance01]
       │ As BP_GameInstance01 (保存为局部变量 GI)
       │
       ▼
[Get SpawnAreaTag]  ← 从 GI 读取
（保存到局部 String 变量 "TagToFind"）
       │
       ▼
[Get All Actors Of Class]  ← Actor Class: Player Start
       │ Out Actors (Array)
       ▼
[For Each Loop]
       │ Array Element (Object Reference)
       ▼
[Get Actor Tags]  ← 输入：Array Element
       │ Return Value (Array of Name)
       ▼
[Contains]  ← 输入：TagToFind (String转Name：加一个 "Make Literal Name" 或直接 Contains接受Name)
       │
   ┌───┴───┐
  True   False
   │       │
   ▼       └── (Continue 循环)
[Get Actor Location]  ← 输入：Array Element（这个 PlayerStart 的位置）
       │
       ▼
[Get Player Character]
       │ Return Value
       ▼
[Set Actor Location And Rotation]
  ├── New Location: PlayerStart的Location
  ├── New Rotation: GetActorRotation(Array Element)  ← 朝向也一起同步
  └── Sweep: false, Teleport: true
       │
       ▼
[Break (For Each Loop)]  ← 找到就停，不继续遍历

─── 循环结束后 ───

[For Each Loop] 的 Completed 引脚
       │
       ▼
[Set SpawnAreaTag]  ← GI变量，设为 "Bedroom"
（重置，防止下次进公寓时还跳到错误出生点）
```

> **Contains 节点注意**：Actor Tags 数组是 `Array of Name`，而 SpawnAreaTag 是 `String`。  
> 解决方法：`Contains` 节点选择 **Array of Name**，然后把 `SpawnAreaTag`（String）用 `Name to String` 的反向——即 `String to Name`（Make Literal Name 或 Convert String to Name）转一下再输入 Contains。  
> 或者更简单：直接搜索 `Get Actor Tags` → `For Each`（第二层） → `Name to String` → `Equal (String)` 与 TagToFind 比较。

### 3.3 简化写法（推荐）

如果上面的 Name/String 转换觉得麻烦，可以改用 **Custom PlayerStart 子类**：

1. 新建 Blueprint Class，父类 `Player Start`，命名 `BP_PlayerStart_Tagged`
2. 加一个 `String` 变量 `SpawnTag`，Instance Editable
3. Level Blueprint 里 `Get All Actors Of Class (BP_PlayerStart_Tagged)`，然后直接比较 `SpawnTag == TagToFind`

---

## 步骤 4 — 公寓放置 BP_DoorLock 实例

### 4.1 卧室门

1. 把 `BP_DoorLock` 从 Content Browser 拖入公寓关卡，放在卧室门口
2. 精确调整位置让 `DoorMesh` 和实际门洞对齐
3. 在 Details 面板设置：
   - `RequiredLevelName` = `Lvl_02_FashionShow`
   - `ClosedRotation` = 当前门关闭时的 Rotation（点 Actor → 看 Transform Rotation，填入）
   - `OpenRotation` = 门打开后的角度（通常 Yaw ±90°，取决于门轴方向）
4. 如果场景里原来有一个静态门模型，选中它 → 在 Details 里把 Mesh 复制下来 → 粘贴到 `BP_DoorLock` 的 `DoorMesh`，然后删掉原来的静态模型

### 4.2 厕所门

重复上面流程，在厕所门口放第二个 `BP_DoorLock` 实例：
- `RequiredLevelName` = `Lvl_Test_Mechanism`
- `ClosedRotation` / `OpenRotation` 同上

---

## 步骤 5 — 公寓厕所：相机交互激活隐藏传送门

### 5.1 放置隐藏传送门

1. 把 `BP_LevelPortal` 拖入厕所区域（镜子后面或墙内，玩家看不到的位置）
2. 在 Details 设置：
   - `TargetLevelName` = `Lvl_04_SewerChase`
   - `bIsActive` = **false**（默认关闭，玩家进去不触发）
3. 可以把传送门缩小或移到墙内先藏起来，激活后再用动画/SetActorLocation 移出来

### 5.2 厕所镜子相机交互触发器

如果厕所没有相机交互物，新建一个触发器 Actor（参考衣柜的 `BP_Closet_Trigger`）：

1. 新建 Blueprint Class，父类 Actor，命名 `BP_Mirror_Trigger`
2. 加组件：
   - `MirrorMesh`（Static Mesh）：镜子外形
   - `CameraDetectBox`（Box Collision）：玩家对准时的检测区域
3. 在 Event Graph，实现相机对准检测：

```
[Event Tick]
       │
       ▼
[Get Player Camera Manager]
       │
       ▼
[Get Camera Location]  (保存 CamLoc)
[Get Camera Rotation] → [Get Forward Vector]  (保存 CamForward)
       │
       ▼
[Get Actor Location]  ← 镜子位置 (保存 MirrorLoc)
       │
       ▼
[Subtract] MirrorLoc - CamLoc → [Normalize] = ToMirror
       │
       ▼
[Dot Product] (CamForward, ToMirror)  → DotVal
       │
       ▼
[Branch]  Condition: DotVal > 0.85  (约30°以内视角对准)
   │
  True
   │
   ▼
[Get Distance To]  ← 玩家到镜子距离，限制只在近处才能触发
  [Branch] Distance < 300.0
     │
    True
     │
     ▼
[Branch] bPortalActivated = false?  ← 防重复激活
     │
    True
     │
     ▼
[Set bPortalActivated = true]
     │
     ▼
[Get All Actors Of Class (BP_LevelPortal)]
     │ Out Actors
     ▼
[For Each Loop]
     │ Array Element
     ▼
[Cast To BP_LevelPortal]
     │ As BP_LevelPortal
     ▼
[Get TargetLevelName] == "Lvl_04_SewerChase"?
     │ True
     ▼
[Set bIsActive = true]  ← 在 BP_LevelPortal 上设置
     │
     ▼
[Set Actor Hidden In Game = false]  ← 让传送门显示出来
     │
     ▼
[Play Sound At Location]  ← 镜子裂开/传送门开启音效
     │
     ▼
[Break]  ← 找到就停
```

> 也可以直接用 Level Blueprint 引用场景里的传送门实例，不用 Get All Actors，更简单直接：  
> Level BP → 在场景里右键传送门实例 → **Create a Reference to...** → 直接操作这个引用。

---

## 步骤 6 — Lvl_Test_Mechanism：第二关通关接入

### 6.1 确认或新建第二关的 GameMode

检查 `Lvl_Test_Mechanism` 的 **World Settings → GameMode Override** 当前是什么：
- 如果已经有 GM（比如继承自 `BP_FirstPersonGameMode`）→ 直接打开它修改
- 如果没有 → 新建 Blueprint Class，父类 `BP_FirstPersonGameMode`，命名 `BP_MushroomGM`，然后在 World Settings 里指定

### 6.2 在 GameMode 里实现 OnLevelComplete 事件

打开第二关的 GM，在 Event Graph：

1. 右键 → 搜索 **Add Custom Event** → 命名 `OnLevelComplete`
2. 连接逻辑：

```
[Custom Event: OnLevelComplete]
       │
       ▼
[Get Game Instance]
       │
       ▼
[Cast To BP_GameInstance01]
       │ As BP_GameInstance01
       │
       ├──▶ [Mark Level Complete]
       │         Input: Level Name = "Lvl_Test_Mechanism"
       │
       └──▶ [Set SpawnAreaTag]
                 Input: "LivingRoom_Exit"
（这两个节点从同一个 Cast 出来的引脚并行连，不分先后）
       │
       ▼（接在两个节点都执行完后，用 Sequence 节点连）
[Open Level (by Name)]
       Input: Level Name = "Lvl_01_Apartment"
```

> 用 **Sequence** 节点（右键搜索）把 `MarkLevelComplete`、`Set SpawnAreaTag`、`Open Level` 串起来，确保顺序执行。

### 6.3 关卡出口传送门

在 `Lvl_Test_Mechanism` 关卡里放一个 `BP_LevelPortal`：
- `TargetLevelName` = `Lvl_01_Apartment`
- `bIsActive` = true（默认激活）

**确认 BP_LevelPortal 的 Overlap 逻辑会调用 GameMode.OnLevelComplete**：

打开 `BP_LevelPortal` 的 Event Graph，找 Overlap 触发链，确认有类似这段：

```
[OnComponentBeginOverlap]
       │ 玩家角色 Cast 成功
       ▼
[bIsActive = true?]
       │ True
       ▼
[Get Game Mode] → [尝试 Cast 到父类 BP_FirstPersonGameMode]
       │
       ▼
[Call OnLevelComplete]  ← 如果父类有这个函数/事件
       │
       ▼
[Open Level (by Name)] ← Target Level Name (变量)
```

如果 `BP_LevelPortal` 里没有调用 `OnLevelComplete`，两种修法：
- 方法A：在 `BP_LevelPortal` 的 Overlap 里加 `Get Game Mode` → `Cast To BP_MushroomGM` → `OnLevelComplete`（每个关卡的 Portal 分别处理）
- 方法B（推荐）：在 `BP_LevelPortal` 的 Overlap 里 Cast 到最基础的父类，父类定义 `OnLevelComplete` 为可覆盖的 **BlueprintImplementableEvent**，子类 GM 各自实现

---

## 步骤 7 — 新建 BP_SewerEnemy

### 7.1 创建蓝图

1. `Content/_Circus07/Blueprints/` 右键 → Blueprint Class
2. 父类选 **Character**（不是 Actor，Character 自带 CharacterMovement）
3. 命名 `BP_SewerEnemy`，双击打开

### 7.2 配置组件

- **Mesh（继承的 Skeletal Mesh）**：在 Details 里选一个临时 Mesh（可用 Mannequin 占位，后期换）
- **Capsule Component（继承）**：Default 即可，记住它会用于 Overlap 检测

### 7.3 添加变量

| Variable Name | Type | Instance Editable | Default Value | Tooltip |
|---|---|---|---|---|
| `MoveSpeed` | Float | ✅ | 300.0 | 追逐速度 cm/s |
| `FacingDotThreshold` | Float | ✅ | 0.7 | 玩家面朝判定阈值（0.7≈45°） |
| `StillSpeedThreshold` | Float | ✅ | 15.0 | 玩家视为静止的速度上限 cm/s |
| `KillTimeRequired` | Float | ✅ | 1.5 | 持续对视+静止多久消灭敌人（秒） |
| `FacingTimer` | Float | ❌ | 0.0 | 内部累计计时 |
| `bAlive` | Boolean | ❌ | true | 防止重复触发死亡逻辑 |
| `CachedPlayerRef` | BP_FirstPersonCharacter Object Reference | ❌ | — | BeginPlay 缓存玩家引用，避免每帧 GetPlayerCharacter |

### 7.4 Event BeginPlay

```
[Event BeginPlay]
       │
       ▼
[Get Character Movement]
       │
       ▼
[Set Max Walk Speed]  ← Value: Get MoveSpeed
       │
       ▼
[Get Player Character]
       │ Return Value
       ▼
[Cast To BP_FirstPersonCharacter]
       │ As BP_FirstPersonCharacter
       ▼
[Set CachedPlayerRef]  ← 存起来

─── 同时 ───

[Get Capsule Component]
       │
       ▼
[Set Generate Overlap Events = true]
```

> 为什么缓存 PlayerRef：`Get Player Character` 每帧调用有轻微开销，Enemy Tick 频繁调用时缓存更好。

### 7.5 Event Tick（拆为两块）

**块 A — 直线追逐**

```
[Event Tick]
       │ Delta Seconds (保存为局部 DeltaT)
       ▼
[Branch]  Condition: Get bAlive
       │ True
       ▼
─── 块 A 开始 ───────────────────────────────────────
[Is Valid (CachedPlayerRef)]  ← 防止玩家未加载时崩溃
       │ Is Valid
       ▼
[Get Actor Location (CachedPlayerRef)]  = PlayerLoc
[Get Actor Location (Self)]             = MyLoc
       │
       ▼
[Vector - Vector]  PlayerLoc - MyLoc  = DirRaw
       │
       ▼
[Normalize (Vector)]  DirRaw → Direction
       │
       ▼
[Add Movement Input]
  ├── World Direction: Direction
  └── Scale Value: 1.0
─── 块 A 结束 ───────────────────────────────────────
```

**块 B — 消灭条件检测（接在块 A 之后，同一 Tick 里）**

```
─── 块 B 开始 ───────────────────────────────────────
[Get Velocity (CachedPlayerRef)]
       │
       ▼
[Vector Length]  = PlayerSpeed

[Get Actor Forward Vector (CachedPlayerRef)]  = PlayerForward

[Get Actor Location (Self)]     = MyLoc
[Get Actor Location (CachedPlayerRef)] = PlayerLoc
[Vector - Vector]  MyLoc - PlayerLoc → [Normalize] = ToEnemy
（注意方向：从玩家指向敌人）

[Dot Product]  (PlayerForward · ToEnemy)  = DotVal
─────────────────────────────────────────────────────

[AND Boolean]
  ├── A: [Less]  PlayerSpeed < Get StillSpeedThreshold
  └── B: [Greater]  DotVal > Get FacingDotThreshold
       │ 结果 = bConditionMet
       ▼
[Branch]  Condition: bConditionMet
   │
   ├── True ──▶
   │       [Float + Float]  Get FacingTimer + DeltaT  → [Set FacingTimer]
   │              │
   │              ▼
   │       [Branch]  FacingTimer >= Get KillTimeRequired
   │              │ True
   │              ▼
   │       [Call Custom Event: OnEnemyKilled]
   │
   └── False ─▶ [Set FacingTimer = 0.0]
─── 块 B 结束 ───────────────────────────────────────
```

### 7.6 Custom Event — OnEnemyKilled

```
[Custom Event: OnEnemyKilled]
       │
       ▼
[Set bAlive = false]  ← 第一件事，防止 Tick 继续触发
       │
       ▼
[Sequence]
  ├── Then 0: [Spawn Emitter At Location]
  │               ← 选一个粒子特效（可暂时用 P_Explosion 占位）
  │               Location: Get Actor Location (Self)
  │
  ├── Then 1: [Play Sound At Location]  ← 消灭音效（可选）
  │
  ├── Then 2:
  │       [Get Game Mode]
  │              │
  │              ▼
  │       [Cast To BP_SewerChaseGM]
  │              │ As BP_SewerChaseGM
  │              ▼
  │       [Call OnEnemyKilled]
  │              Input: Enemy Ref = Self
  │
  └── Then 3: [Destroy Actor]  ← 最后销毁自己
```

> Then 2 的 Cast 失败不会崩，只是 GM 不知道敌人死了。确保关卡 World Settings 已指向 BP_SewerChaseGM。

### 7.7 Capsule Overlap — 触碰玩家即死

选中 **Capsule Component** → Details → Events → **OnComponentBeginOverlap** 点 **+**

```
[OnComponentBeginOverlap (CapsuleComponent)]
       │ Other Actor
       ▼
[Cast To BP_FirstPersonCharacter]
       │ Cast成功
       ▼
[Branch]  Condition: Get bAlive  ← 只有敌人活着才能伤害玩家
       │ True
       ▼
[Get Game Instance]
       │
       ▼
[Cast To BP_GameInstance01]
       │ As BP_GameInstance01
       ▼
[Modify HP]  Input: Amount = -100
（填-100确保直接扣死，ModifyHP内部有Clamp到0）
       │ bIsDead output (Boolean)
       ▼
[Branch]  bIsDead
       │ True
       ▼
[Open Level (by Name)]  ← "Lvl_04_SewerChase"（重启关卡）
或者:
[Get Player Controller] → [Set Input Mode Game Only]
[Create Widget (WBP_DeathScreen)] → [Add to Viewport]（死亡界面）
```

> 简单处理就直接 `Open Level` 重载当前关卡，重开即可。死亡界面是可选的后续优化。

### 7.8 Compile & Save

---

## 步骤 8 — 新建 BP_SewerChaseGM

### 8.1 创建蓝图

1. `Content/_Circus07/Blueprints/GameMode/` 右键 → Blueprint Class
2. 父类选 **BP_FirstPersonGameMode**（继承现有基类）
3. 命名 `BP_SewerChaseGM`，双击打开

### 8.2 添加变量

| Variable Name | Type | Instance Editable | Default | 说明 |
|---|---|---|---|---|
| `TotalEnemies` | Integer | ❌ | 0 | BeginPlay 时自动计数 |
| `KilledEnemies` | Integer | ❌ | 0 | 每死一个敌人+1 |
| `bPortalSpawned` | Boolean | ❌ | false | 防止重复生成传送门 |
| `PortalClass` | Class Reference → BP_LevelPortal | ✅ | BP_LevelPortal | 在编辑器里下拉选择 BP_LevelPortal |

### 8.3 BeginPlay

```
[Event BeginPlay]
       │
       ▼
[Parent: BeginPlay]  ← 右键搜索 "Add call to parent function"，必须调用
       │
       ▼
[Get All Actors Of Class]  ← Actor Class: BP_SewerEnemy
       │ Out Actors (Array of BP_SewerEnemy)
       ▼
[Array Length]  → [Set TotalEnemies]
```

### 8.4 Custom Event — OnEnemyKilled（由 BP_SewerEnemy 调用）

右键 → Add Custom Event → 命名 `OnEnemyKilled`  
添加输入引脚：`EnemyRef`（BP_SewerEnemy Object Reference）

```
[Custom Event: OnEnemyKilled (EnemyRef)]
       │
       ▼
[Integer + Integer]  Get KilledEnemies + 1 → [Set KilledEnemies]
       │
       ▼
[Branch]  KilledEnemies >= TotalEnemies
       │ True
       ▼
[Branch]  NOT bPortalSpawned
       │ True（还没生成过传送门）
       ▼
[Set bPortalSpawned = true]
       │
       ▼
[Call SpawnPortal]  ← 调用下面的函数
```

### 8.5 Function — SpawnPortal

右键 → Add Function → 命名 `SpawnPortal`（函数，不是事件）

```
[SpawnPortal 入口]
       │
       ▼
[Get Player Character]
       │ Return Value
       ▼
[Get Actor Location]  = PlayerLoc
       │
       ▼
[Vector + Vector]
  PlayerLoc + (200, 0, 0)  ← 在玩家前方200cm生成
  （也可以直接用 PlayerLoc，传送门生成在玩家脚下）
  = SpawnLocation
       │
       ▼
[Spawn Actor From Class]
  ├── Class: Get PortalClass
  ├── Spawn Transform:
  │     Location = SpawnLocation
  │     Rotation = (0, 0, 0)
  │     Scale = (1, 1, 1)
  └── Collision Handling: Always Spawn, Ignore Collisions
       │ Return Value (BP_LevelPortal reference)
       ▼
[Cast To BP_LevelPortal]
       │ As BP_LevelPortal
       │
       ├──▶ [Set Target Level Name] = "Lvl_End_Results"
       │        （或 "Lvl_01_Apartment" 如果走结算Widget方案）
       │
       └──▶ [Set bIsActive = true]
       │
       ▼
[Delay] ← Duration: 0.5（可选，给传送门出现动画留点时间）
       │
       ▼
[Print String "传送门已开启！走进去离开关卡"]  ← 调试用，后期删
```

> `Spawn Actor From Class` 的 `Spawn Transform` 需要一个完整的 Transform 结构体，用 **Make Transform** 节点组装 Location、Rotation、Scale。

### 8.6 Custom Event — OnLevelComplete

```
[Custom Event: OnLevelComplete]
       │
       ▼
[Sequence]
  ├── Then 0:
  │     [Get Game Instance] → [Cast To BP_GameInstance01]
  │       ├──▶ [Mark Level Complete]  Input: "Lvl_04_SewerChase"
  │       └──▶ [Add Score]  Input: 1000（通关奖励分，按需）
  │
  └── Then 1:
        （根据结算方案二选一）
        方案A: [Create Widget (WBP_Results)] → [Add to Viewport]
               [Get Player Controller] → [Set Input Mode UI Only]
               [Set Show Mouse Cursor = true]
        方案B: [Open Level (by Name)] → "Lvl_End_Results"
```

### 8.7 Compile & Save

---

## 步骤 9 — 新建 Lvl_04_SewerChase 关卡

### 9.1 创建关卡

1. Content Browser → `Content/_Circus07/Maps/`
2. 右键 → **New Level** → 选 **Empty Level**（空关卡，自己搭建）或 **Basic**（有基础光照）
3. 命名 `Lvl_04_SewerChase`，双击打开

### 9.2 配置 World Settings

菜单 **Window → World Settings**：
- `GameMode Override` → 选择 `BP_SewerChaseGM`

### 9.3 配置 DefaultEngine.ini（确认关卡可以被 Open Level 找到）

已有配置不需要改，`Open Level by Name` 只需要填 `.umap` 的文件名（不含路径和后缀），即 `Lvl_04_SewerChase`。

### 9.4 搭建基础环境

- 放 **Player Start** 一个（玩家进入关卡的出生点）
- 用 **BSP Brush** 或导入 Mesh 搭建下水道走廊（长廊最适合直线追逐玩法）
- 摆 **3~5 个 BP_SewerEnemy** 实例（数量随难度调整）
- 加基础 **光照**（Point Light 或 Spot Light 打出阴暗下水道氛围）
- 加 **BP_LevelPortal** 不放（由 GM 动态生成）或放一个隐藏的（bIsActive=false）

### 9.5 摆放 BP_SewerEnemy 注意事项

- 每个 Enemy 在 Details 里可以调 `MoveSpeed`（初始敌人可以慢一点，300 cm/s 左右）
- 不同 Enemy 的 `FacingDotThreshold` 可以不同，让有的敌人更难消灭（需要更正对才行）
- `KillTimeRequired` 越大越难（需要对视更久）

---

## 步骤 10 — WBP_Results 结算界面

### 10.1 新建 Widget Blueprint

1. `Content/_Circus07/Blueprints/WBP/` 右键 → **User Interface → Widget Blueprint**
2. 命名 `WBP_Results`，双击打开

### 10.2 Designer 面板布局

在 Designer 面板搭建简单 UI：

```
[Canvas Panel]
  ├── [Image]  ── 全屏半透明黑色背景
  │                Color: (0, 0, 0, 0.8)  Fill Screen
  │
  ├── [Vertical Box]  ── 居中对齐，Anchors: 中央
  │     ├── [Text Block]  "The End"
  │     │       Font Size: 72,  Color: White
  │     │
  │     ├── [Text Block] "Score:"
  │     │       Font Size: 36
  │     │
  │     ├── [Text Block]  Name: "ScoreText"
  │     │       Font Size: 48,  Color: Yellow
  │     │       (在 Graph 里绑定变量)
  │     │
  │     ├── [Spacer]  Size: (0, 40)
  │     │
  │     ├── [Button]  Name: "RestartBtn"
  │     │       [Text] "重新开始"
  │     │
  │     └── [Button]  Name: "MainMenuBtn"
  │               [Text] "返回主菜单"
```

### 10.3 Graph 面板逻辑

**绑定 ScoreText 的 Text：**

选中 ScoreText → Details → Content → Text 旁边点 **Bind → Create Binding**

```
[Get Game Instance] → [Cast To BP_GameInstance01]
       │
       ▼
[Get Total Score]
       │ Integer
       ▼
[To Text (Int)]  = Return Value
```

**RestartBtn OnClicked：**

选中按钮 → Details → Events → **OnClicked** 点 **+**

```
[OnClicked (RestartBtn)]
       │
       ▼
[Get Game Instance] → [Cast To BP_GameInstance01]
       │ As BP_GameInstance01 (保存为 GI)
       │
       ▼
[Sequence]
  ├── Then 0: [Clear (CompletedLevels Map)]     ← GI.CompletedLevels
  ├── Then 1: [Set TotalScore = 0]              ← GI.TotalScore
  ├── Then 2: [Set PlayerCurrentHP = 3]         ← GI.PlayerCurrentHP
  ├── Then 3: [Set SpawnAreaTag = "Bedroom"]    ← GI.SpawnAreaTag
  └── Then 4: [Clear (CollectedItems Array)]    ← GI.CollectedItems
       │
       ▼
[Open Level (by Name)] → "Lvl_01_Apartment"
```

> 注意：`Clear` 节点用于 Map 和 Array 类型，在 GI 的对应变量上右键 → Get → 再连 `Clear` 节点。

**MainMenuBtn OnClicked：**

```
[OnClicked (MainMenuBtn)]
       │
       ▼
[Open Level (by Name)] → "Lvl_MainMenu"
```

### 10.4 在 BP_SewerChaseGM 里调用

回到 `BP_SewerChaseGM` 的 `OnLevelComplete`，Then 1 里：

```
[Create Widget]
  ├── Class: WBP_Results
  └── Owning Player: Get Player Controller
       │ Return Value (WBP_Results 引用)
       ▼
[Add to Viewport]
       │
       ▼
[Get Player Controller]
       │
       ▼
[Set Input Mode UI Only]
  └── In Widget To Focus: WBP_Results 引用
       │
       ▼
[Set Show Mouse Cursor = true]
```

---

## 步骤 11 — 确认 FashionShow 关卡通关后设置 SpawnAreaTag

打开 `BP_FashionShowGM`，找 `OnLevelComplete` 事件，确认在 Open Level 之前加上：

```
[Get Game Instance] → [Cast To BP_GameInstance01]
       ├──▶ [Mark Level Complete]  "Lvl_02_FashionShow"
       └──▶ [Set SpawnAreaTag = "Bedroom_Exit"]

（然后）
[Open Level (by Name)] → "Lvl_01_Apartment"
```

如果 `BP_FashionShowGM` 里已有 `OnLevelComplete` 逻辑，只需加 `Set SpawnAreaTag` 那一行。

---

## 步骤 12 — 端到端测试清单

按顺序逐项确认：

- [ ] 进入公寓，玩家出现在卧室（Tag=Bedroom 的 PlayerStart）
- [ ] 卧室门上锁，靠近有音效/提示
- [ ] 柜门交互进入 FashionShow
- [ ] 完成 FashionShow（到达终点/Portal）→ 回到公寓
- [ ] 回到公寓后玩家在 Bedroom_Exit 位置
- [ ] 卧室门变为可以打开（推门动画）
- [ ] 走向客厅，客厅大门 Portal → 进入 Lvl_Test_Mechanism
- [ ] 完成 Lvl_Test_Mechanism → 回到公寓
- [ ] 回到公寓后玩家在 LivingRoom_Exit 位置（靠近厕所）
- [ ] 厕所门变为可以打开
- [ ] 进入厕所，对准镜子 → 传送门出现
- [ ] 走进传送门 → 进入 Lvl_04_SewerChase
- [ ] 敌人追逐玩家（直线移动）
- [ ] 触碰敌人后玩家死亡（关卡重启）
- [ ] 对视+静止 1.5 秒 → 敌人消灭
- [ ] 所有敌人消灭 → 传送门在玩家旁生成
- [ ] 走进传送门 → WBP_Results 显示
- [ ] 分数正确显示
- [ ] [重新开始] → 回卧室，GameInstance 数据已清零
- [ ] [返回主菜单] → 主菜单

---

## 附录 — 常见问题

### Q: 公寓 BeginPlay SetActorLocation 没效果？
A: 加 `Delay(0.1)` 再移动，或者用 `Set Actor Location And Rotation (Teleport=true)`

### Q: GetActorTags Contains 类型不匹配？
A: Tags 是 `Array of Name`，SpawnAreaTag 是 `String`。用 `Name to String` 转换后比较，或自定义 BP_PlayerStart_Tagged 蓝图改用 String 变量。

### Q: BP_SewerEnemy 追着追着穿墙了？
A: 确认父类是 **Character** 不是 Actor，Character 的 CharacterMovement 自带碰撞。检查 Capsule Component 的 Collision Preset 是否为 `Pawn`。

### Q: 敌人 DotProduct 判断不稳定，有时对着看也不消灭？
A: 把 `FacingDotThreshold` 降低到 `0.5`（约60°），`StillSpeedThreshold` 提高到 `30.0`，让判定更宽松。

### Q: 所有敌人死了但传送门没出现？
A: 在 `OnEnemyKilled` 里加 `Print String` 打印 `KilledEnemies / TotalEnemies` 的值，确认计数是否正确。常见问题：TotalEnemies 在 BeginPlay 时为0（敌人还没加载完），同样加 `Delay(0.2)` 再 GetAllActors。

### Q: GameInstance SpawnAreaTag 重置时机不对？
A: 一定要在公寓 Level Blueprint BeginPlay 的**最后**重置，不要在 BeginPlay 的最开始重置（那样移动玩家前就清掉了还没读到）。
