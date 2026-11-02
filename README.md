# Bridge Support Modes

[简体中文](README.zh-CN.md)

A Blueprint mod for Abiotic Factor that lets you choose the support direction of wooden and carbon bridges while placing them.

Press **Z** to cycle through:

**Vanilla → Down 90° → Down 45° → Down 30° → Right 45° → Right 30° → Left 45° → Left 30° → None**

Hold **Shift** and press **Z** to cycle backwards. On a controller, press **D-pad Left** to advance once; holding it does not repeat. Button prompts follow the active input device automatically. Each player keeps their selection for the current session. None requires the sandbox bridge support requirement to be **0**.

## How it works

The mod replaces the bridge's placement preview with a subclass of `DeployProxy_Bridge`. That preview calculates four support endpoints for the selected direction and writes them into the native `Supports` array. The game then uses its normal placement flow to create the bridge with those endpoints.

Placed bridges retain their pillar layouts after uninstalling the mod because the endpoints are stored in the native save data.

Four Blueprints implement the feature:

| Blueprint | Responsibility |
| --- | --- |
| `BP_BridgeBootstrap` | Adds a replicated session component to each player controller. |
| `BP_BridgeSession` | Handles local input, remembers the mode, replaces the preview and updates the hint. |
| `BP_BridgePreview` | Calculates support endpoints and updates the support holograms. |
| `WBP_BridgeHint` | Displays the localized label, shadowed mode icon and device-aware button prompt. |

The support traces use **DeployTrace with simple collision**, matching native bridge support detection. Decorative foliage surfaces do not become support endpoints just because they have visible triangles.

Read [Implementation](docs/IMPLEMENTATION.md) for the Blueprint flow, mode IDs, geometry calculations and UI layout.

## Open the project

1. Open the official **Abiotic Factor Modkit**.
2. In its project browser, choose [BridgeSupportModesdb1Fc.abfmod](BridgeSupportModesdb1Fc/BridgeSupportModesdb1Fc.abfmod).
3. Open `Content/Blueprints`. Start with `BP_BridgeSession` to follow a complete placement session, or `BP_BridgePreview` to study the support geometry.

The project includes the Blueprint source assets and all nine icon textures. Keep the internal project name `BridgeSupportModesdb1Fc`; asset references use this package root.

## License

[MIT](LICENSE) covers the original Blueprint logic and artwork in this repository. Referenced game and Unreal Engine content is provided by the official Modkit.
