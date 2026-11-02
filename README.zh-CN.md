# 桥梁支撑模式

[English](README.md)

这是一个使用蓝图实现的《非生物因素》Mod，让玩家在放置木板桥和碳素桥时选择支柱的方向。

按 **Z** 依次切换：

**原版 → 下90° → 下45° → 下30° → 右45° → 右30° → 左45° → 左30° → 无支撑**

按住 **Shift** 再按 **Z** 反向切换。手柄按 **方向键左** 前进一档，按住不会重复切换。提示会自动跟随当前输入设备。每位玩家独立记住本次会话的选择；使用无支撑模式时，沙盒的桥梁支撑要求需要设为 **0**。

## 实现思路

Mod 用 `DeployProxy_Bridge` 的子类替换桥梁放置预览。这个预览根据所选方向计算四个支撑端点，写入原生 `Supports` 数组。之后游戏沿用正常的放置流程，用这些端点生成桥梁和支柱。

已放置桥梁的支柱数据由原生存档保存，卸载 Mod 后仍保持原来的布局。

功能由四个蓝图组成：

| 蓝图 | 职责 |
| --- | --- |
| `BP_BridgeBootstrap` | 为每个玩家控制器添加可复制的会话组件。 |
| `BP_BridgeSession` | 处理本地按键、记忆模式、替换预览并更新提示。 |
| `BP_BridgePreview` | 计算支撑端点，更新支柱全息预览。 |
| `WBP_BridgeHint` | 显示本地化模式文字、带阴影的模式图标，以及随输入设备切换的按键提示。 |

支撑射线使用与原版一致的 **DeployTrace 通道和简单碰撞**，因此不会仅仅因为植物叶片存在可见网格，就把叶片表面当成支撑端点。

[实现机制](docs/IMPLEMENTATION.zh-CN.md)详细介绍蓝图流程、模式编号、几何计算和提示界面的布局。

## 打开工程

1. 启动官方 **Abiotic Factor Modkit**。
2. 在工程浏览器中打开 [BridgeSupportModesdb1Fc.abfmod](BridgeSupportModesdb1Fc/BridgeSupportModesdb1Fc.abfmod)。
3. 进入 `Content/Blueprints`。想了解完整放置过程，可以从 `BP_BridgeSession` 开始；想研究支柱几何，则先看 `BP_BridgePreview`。

工程包含蓝图源资产和全部九个图标纹理。内部工程名保留为 `BridgeSupportModesdb1Fc`，资源引用使用这一包路径。

## 许可证

仓库中的自有蓝图逻辑和图标采用 [MIT](LICENSE) 许可证。被引用的游戏及虚幻引擎内容由官方 Modkit 提供。
