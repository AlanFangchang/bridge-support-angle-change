# 实现机制

[English](IMPLEMENTATION.md) · [返回首页](../README.zh-CN.md)

这个 Mod 的核心是改变**放置预览计算出的支撑端点**。游戏原本就能根据端点生成支柱，因此只需要让预览提供不同方向的端点，再把结果交回原版放置流程。

## 1. 从配置进入蓝图

`Content/DataTables/DT_ClassManager/Class_BridgeSupportBootstrap` 是一个 ClassManager 行资产。它使用 `ClassManager.EntityDirector` 标签，`ObjectClass` 指向 `BP_BridgeBootstrap`，由游戏的 ClassManager 将启动组件接入世界。

`BP_BridgeBootstrap` 的 Tick 间隔为一秒，逻辑只在服务器端执行：

1. 获取当前所有玩家控制器。
2. 检查控制器是否已有 `BP_BridgeSession`。
3. 没有则添加，并调用 `SetIsReplicated(true)`。

周期性检查让后来加入的玩家也能获得会话组件。单人游戏同样经过这条路径。

## 2. 每位玩家各自管理放置会话

`BP_BridgeSession` 是挂在玩家控制器上的 ActorComponent。它首先检查 `IsLocalController`，只有拥有该控制器的本地玩家继续处理输入和界面。

会话组件逐帧检查玩家是否处于放置模式，并取得：

```text
玩家角色
  → DeployableOverlapActor（ChildActorComponent）
  → ChildActor（当前放置预览）
```

随后读取预览的 `LinkedItemRowHandle`。目前接管的物品行是木板桥 `bridge_t1` 和碳素桥 `bridge_t2`。

如果预览类精确等于原版 `DeployProxy_Bridge`，组件会先保存物品行和桥面网格，调用 `SetChildActorClass` 换成 `BP_BridgePreview`，再恢复物品行和网格。如果已经是本 Mod 的预览，就直接更新；其他预览类会被跳过。

### 模式记忆与按键

`SessionMode` 保存在会话组件的持久临时变量节点中。收起桥梁、换成另一种桥梁或重新创建预览时，组件会把已有选择应用到新的预览；它不是写入存档的设置。

`WasInputKeyJustPressed` 检测 Z 或 `Gamepad_DPad_Left`，每次按下只切换一档。左右 Shift 的 `IsInputKeyDown` 只在 Z 被按下时决定是否使用反向循环表，因此手柄始终前进，也没有长按重复。背包打开、鼠标指针显示或视角输入被界面占用时，不处理切换。

内部编号与显示顺序分别维护：

| 循环顺序 | 模式 | 内部编号 |
| --- | --- | --- |
| 1 | 原版 | 0 |
| 2 | 下90° | 4 |
| 3 | 下45° | 2 |
| 4 | 下30° | 1 |
| 5 | 右45° | 8 |
| 6 | 右30° | 7 |
| 7 | 左45° | 6 |
| 8 | 左30° | 3 |
| 9 | 无支撑 | 5 |

正向表为 `[4,8,1,5,2,0,3,6,7]`，反向表为 `[5,2,4,6,0,3,7,8,1]`。用当前编号索引所选表，就得到下一档编号。

组件将编号转为 Name，写入预览 Actor 的 `Tags` 数组。`BP_BridgePreview` 读取第一个标签并转回整数。这是当前会话组件与预览之间传递模式的方式；写入时会替换整个 `Tags` 数组。

## 3. 预览如何计算支撑

`BP_BridgePreview` 继承原版 `DeployProxy_Bridge`，主要改动在 `AdditionalDeployCheck`。

**原版模式**直接调用父类的支撑检查，使用原版预览流程已经计算好的支撑数据。

**固定角度模式**先清空 `Supports`，将 `CurrentSupports` 和 `CurrentDeployableSupports` 归零，隐藏四个支柱预览，然后遍历原版 `SupportSocketsName` 中的四个插槽。

每根柱子的起点是桥面 `DeployableHologramMesh` 对应插槽的世界坐标。射线最大长度为 2000 个虚幻单位，角度相对于水平面。

设桥梁前向为 `F`、右向为 `R`、角度为 `θ`，水平探测方向为 `H`：

- 下30°、下45°：`H = -F`，即桥梁局部后方。
- 左30°、左45°：`H = -R`。
- 右30°、右45°：`H = R`。
- 下90°：竖直向下，不使用水平分量。

固定角度的射线位移为：

```text
位移 = H × (2000 × cos θ) + (0, 0, -2000 × sin θ)
终点 = 插槽世界坐标 + 位移
```

水平分量跟随桥梁自身方向，所以旋转桥梁后，左右方向也随之旋转；竖直分量使用世界坐标的 Z 轴。

### 命中规则

`LineTraceSingle` 使用 **DeployTrace**，即当前 Modkit 中的 `TraceTypeQuery8`，并将 **Trace Complex 设为 false**。射线忽略预览自身和所附着的玩家。

命中且不是起点已重叠时，把 `ImpactPoint` 加入 `Supports`，增加 `CurrentSupports`。如果命中的 Actor 继承 `AbioticDeployed_ParentBP`，还会增加 `CurrentDeployableSupports`。未命中时写入零向量，保留四个插槽与数组元素的一一对应关系。

这里没有植物名称或模型列表。是否阻挡由对象对 DeployTrace 的响应及简单碰撞形状决定：叶片等装饰网格可以被跳过，花盆、地面等实际碰撞仍然能成为端点。这与原版支撑探测采用相同的规则。

## 4. 支柱预览与整桥合法性

有效端点确定后，预览会把对应的 `SupporHologram_FL/FR/BL/BR` 移到插槽位置，使用 `FindLookAtRotation` 朝向端点，并按距离沿 X 轴缩放。原版支柱网格的基准长度为 24 个单位，因此长度缩放为 `距离 / 24`。

自定义端点计算完成后，仍调用父类 `AdditionalDeployCheck`。支撑数量、可放置条件和失败提示继续经过原版检查。

`TryUpdateDeployableHologram` 先运行父类更新，再调用 `RefreshSupportVisuals`。这个事件也接在预览 Tick 后面，用于同步支柱材质和显示状态：

- 四根支柱的材质跟随桥面最终材质，避免支柱显示有效而整桥不可放置。
- 切回原版模式时恢复支柱可见性；固定角度模式中，有效命中的支柱由自定义探测显示。
- 无支撑模式保持四根支柱隐藏。

无支撑模式不执行四条射线，支撑数组为空、计数为零，然后照常进入父类检查。因此沙盒要求大于零时，原版检查会拒绝放置；要求为零时，则由其余原版条件决定能否放置。

## 5. 放置结果如何进入游戏

Mod 没有另建永久桥梁类，也没有自行执行物品扣除。玩家确认放置后，原版流程使用预览中的支撑数据，生成原版 `Deployed_Bridge_Tier1` 或 `Deployed_Bridge_Tier2`。

网络上的放置请求和物品消耗同样沿用游戏流程。玩家的模式选择用于计算本地预览，实际传给原版桥梁的是支撑端点，而不是让每个客户端各自按角度重新计算。

原版部署物保存结构本身包含 `Supports` 向量数组。这个设计让桥梁结果使用原版数据表示；会话模式则只属于当前玩家控制器上的组件。

## 6. 提示界面与本地化

`WBP_BridgeHint` 主要提供布局。文字、图标和显示状态由 `BP_BridgeSession` 更新：左侧是 `TextBlock`，右侧是含九张纹理的 `WidgetSwitcher`，Z 字母和手柄方向键图标分别叠在模式图标中央，按当前设备切换显示。九张模式纹理的圆圈、方框、箭头和支柱都带有向右下偏移一个界面像素的黑色阴影。

文字沿用原版操作提示的字体和右下黑色阴影。会话组件通过玩家 HUD 找到 `W_PlayerHUD_ItemActionList.PrimaryFireKeybind`，读取其缓存几何，转换为视口坐标，再放置本 Mod 的提示，使 Z 图标跟随原版操作图标对齐。

界面通过 `AddToPlayerScreen` 添加，设为 `HitTestInvisible`，不接收鼠标。离开放置状态或打开背包时隐藏；组件结束时移除。

本地化在会话蓝图内完成：

1. `GetCurrentLanguage` 读取当前语言。
2. `GetSuitableCulture` 匹配支持的语言，默认回退英文。
3. 按“文字条目编号 × 10 + 语言编号”读取翻译字符串。
4. 转为 Text，更新提示。

支持英语、简体中文、繁体中文、德语、法语、拉美西班牙语、巴西葡萄牙语、俄语和日语；`es` 作为西班牙语别名占据第十个语言位置。

普通条目编号对应模式编号。无支撑模式被沙盒要求限制时，使用额外的第 9 号文字条目显示“无支撑 — 请将沙盒支撑要求设为 0”，并将提示染为琥珀色。这个文字条目不是额外模式。

### 自动切换按键提示

会话组件使用 `Get Local Player Subsystem` 从自己的玩家控制器取得原生 `AbioticUIActionRouter`，读取 `IsUsingGamepad`。键鼠输入时显示 `KeyLetter`，手柄输入时显示 `GamepadKey`。这复用了原生的输入设备判断规则。

`GetGamepadEnumID` 返回手柄类型；`GamepadIcons` 选择游戏自带的 Xbox、PS4、PS5 或 Steam Deck 方向键左纹理，再通过 `SetBrushFromTexture` 更新图像。默认类型使用 Xbox 图标。资源保留为原生引用。

## 7. 修改时从哪里入手

| 想修改的行为 | 蓝图中的位置 |
| --- | --- |
| 循环顺序 | `BP_BridgeSession` 的 `CycleForwardFixed`、`CycleReverseFixed`。 |
| 按键 | `ZPressed`、`GamepadCyclePressed`、`CycleLeftShift`、`CycleRightShift`。 |
| 角度或方向 | `BP_BridgePreview` 的 `DirectionTableNine` 及后续向量计算。 |
| 支撑命中规则 | `AdditionalDeployCheck` 的 `Trace` 与 `ValidHit`。 |
| 模式文字与翻译 | `BP_BridgeSession` 的 `Languages`、`Translations`。 |
| 图标与排版 | `WBP_BridgeHint` 的控件树及 `Content/UI` 纹理。 |
| 提示位置 | `NativeHintOffset` 与 `FollowNativeKey`。 |

新增模式时，模式编号需要同时对应方向表、正反向循环表、图标索引和翻译条目。

若要接入其他作者的桥梁，还需要调整物品行与预览类的接管条件。现有实现依赖四个支撑插槽、四个原版支柱预览组件，以及接收 `Supports` 的原版桥梁放置流程；仅让一个新桥梁类继承原版父类，并不会自动通过当前的物品行筛选。
