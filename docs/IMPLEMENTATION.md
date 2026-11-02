# Implementation

[简体中文](IMPLEMENTATION.zh-CN.md) · [Back to README](../README.md)

The mod changes the **support endpoints calculated by the placement preview**. The game already knows how to build pillars from endpoints, so the custom preview supplies a different set of points and passes them to the native placement flow.

## 1. Loading the Blueprints

`Content/DataTables/DT_ClassManager/Class_BridgeSupportBootstrap` is a ClassManager row asset. Its gameplay tag is `ClassManager.EntityDirector`, and its `ObjectClass` points to `BP_BridgeBootstrap`. The game's ClassManager uses this entry to attach the bootstrap component to the world.

`BP_BridgeBootstrap` ticks once per second and runs its logic on the server:

1. Find all player controllers.
2. Check each controller for a `BP_BridgeSession` component.
3. Add a missing component and call `SetIsReplicated(true)` on it.

Repeating this check also picks up players who join later. Single-player uses the same path.

## 2. One placement session per player

`BP_BridgeSession` is an ActorComponent attached to a player controller. It checks `IsLocalController` before handling input or UI, so each player operates their own session.

Every frame, it checks whether the player is in placement mode and retrieves:

```text
Player character
  → DeployableOverlapActor (ChildActorComponent)
  → ChildActor (current placement preview)
```

It reads the preview's `LinkedItemRowHandle`. The supported item rows are `bridge_t1` for wood and `bridge_t2` for carbon.

When the preview class is exactly the native `DeployProxy_Bridge`, the component remembers its item row and deck mesh, calls `SetChildActorClass` to replace it with `BP_BridgePreview`, and restores the row and mesh. An existing custom preview is updated directly. Other preview classes are skipped.

### Remembering and switching modes

`SessionMode` is stored in a persistent temporary-variable node in the session component. It survives putting the bridge away, switching bridge types and recreating the preview because the component applies its existing selection to the new preview. It is session state, not a saved preference.

`WasInputKeyJustPressed` checks Z or `Gamepad_DPad_Left`, advancing once per press. Both Shift keys are checked only when Z is pressed to select the reverse table. Controller input always advances, with no hold repeat. Switching is ignored while the inventory is open, the mouse cursor is visible or an interface has taken over look input.

Mode IDs and display order are separate:

| Cycle position | Mode | Internal ID |
| --- | --- | --- |
| 1 | Vanilla | 0 |
| 2 | Down 90° | 4 |
| 3 | Down 45° | 2 |
| 4 | Down 30° | 1 |
| 5 | Right 45° | 8 |
| 6 | Right 30° | 7 |
| 7 | Left 45° | 6 |
| 8 | Left 30° | 3 |
| 9 | None | 5 |

The forward table is `[4,8,1,5,2,0,3,6,7]`; the reverse table is `[5,2,4,6,0,3,7,8,1]`. Indexing the selected table with the current ID gives the next ID.

The session converts the ID to a Name and writes it into the preview Actor's `Tags` array. `BP_BridgePreview` reads the first tag and converts it back to an integer. This is the current mode handoff between the two Blueprints; the write replaces the entire Tags array.

## 3. Calculating support endpoints

`BP_BridgePreview` inherits from the native `DeployProxy_Bridge`. Its main geometry change is in `AdditionalDeployCheck`.

**Vanilla mode** calls the parent support check using the support data already calculated by the native preview update.

**Fixed-angle modes** clear `Supports`, reset `CurrentSupports` and `CurrentDeployableSupports`, hide all four support holograms, and iterate the four entries in the native `SupportSocketsName` array.

Each pillar starts at the corresponding socket's world position on `DeployableHologramMesh`. Traces have a maximum length of 2,000 Unreal units. Angles are measured from the horizontal plane.

Let `F` be the bridge's forward vector, `R` its right vector, `θ` the angle, and `H` the horizontal direction:

- Down 30° and Down 45° use `H = -F`, towards the bridge's local rear.
- Left 30° and Left 45° use `H = -R`.
- Right 30° and Right 45° use `H = R`.
- Down 90° points vertically down, with no horizontal component.

The fixed-angle trace is calculated as:

```text
Displacement = H × (2000 × cos θ) + (0, 0, -2000 × sin θ)
End = Socket world position + Displacement
```

The horizontal component follows the bridge's orientation, so rotating the bridge also rotates its left and right support directions. The vertical component uses world-space Z.

### Hit selection

`LineTraceSingle` uses **DeployTrace**, exposed as `TraceTypeQuery8` in this Modkit, with **Trace Complex set to false**. It ignores the preview itself and its attached player.

A hit that does not begin inside a collider contributes its `ImpactPoint` to `Supports` and increments `CurrentSupports`. If the hit Actor inherits from `AbioticDeployed_ParentBP`, it also increments `CurrentDeployableSupports`. A miss contributes a zero vector, preserving the correspondence between the four sockets and four array entries.

There is no plant-name or model blacklist. An object blocks a support trace according to its DeployTrace response and simple collision geometry. Decorative leaves can be skipped while a solid pot or floor still provides an endpoint. This matches the native bridge's support collision policy.

## 4. Holograms and placement validity

For each valid endpoint, the preview moves the corresponding `SupporHologram_FL/FR/BL/BR` component to its socket, uses `FindLookAtRotation` to aim at the endpoint, and scales the pillar along X. The native pillar mesh has a base length of 24 units, so the length scale is `distance / 24`.

After computing the custom endpoints, the Blueprint still calls the parent `AdditionalDeployCheck`. Support counts, placement conditions and failure messages continue through the native check.

`TryUpdateDeployableHologram` runs its parent update and then calls `RefreshSupportVisuals`. The preview Tick also calls this event to synchronize materials and visibility:

- Each pillar uses the deck's final material, so a pillar cannot appear valid while the whole bridge is marked invalid.
- Returning to Vanilla restores pillar visibility. Fixed-angle modes show their valid pillars during the custom trace pass.
- None keeps all four pillars hidden.

None skips the four traces, leaving the support array empty and both counts at zero, then calls the parent check. A nonzero sandbox requirement therefore rejects placement. With a requirement of zero, the remaining native conditions determine whether placement is allowed.

## 5. Creating the actual bridge

The mod does not define a permanent bridge class or subtract inventory items itself. When the player confirms placement, the game's normal flow consumes the preview's support data and creates a native `Deployed_Bridge_Tier1` or `Deployed_Bridge_Tier2`.

Placement requests over the network and item consumption also use the native flow. The player's selected mode determines the local preview geometry; the resulting support endpoints are passed to the native bridge rather than having every client independently recalculate the selected angle.

The native deployable save structure already contains a `Supports` vector array. The placed result therefore uses the game's existing data representation, while mode selection belongs to the component on the current player controller.

## 6. Hint layout and localization

`WBP_BridgeHint` mainly defines the layout. `BP_BridgeSession` updates its text, icon and visibility. A `TextBlock` sits on the left; a `WidgetSwitcher` containing nine icon textures sits on the right. Separate Z text and gamepad button images overlay the center and switch visibility with the active input device. All nine mode textures include a black shadow offset one UI pixel down and right across the circle, key frame, arrows and pillars.

The label uses the native action hint's font and bottom-right black shadow. The session finds `W_PlayerHUD_ItemActionList.PrimaryFireKeybind` through the player's HUD, reads its cached geometry and converts it to viewport coordinates. It positions the custom hint relative to that key, keeping the Z icon aligned with the native action icons.

The widget is added with `AddToPlayerScreen` and uses `HitTestInvisible`, so it does not intercept the mouse. It is hidden when the player leaves placement mode or opens the inventory, and removed when the session component ends.

Localization is handled inside the session Blueprint:

1. Read the current language with `GetCurrentLanguage`.
2. Match it with `GetSuitableCulture`, falling back to English.
3. Look up the translation at `label entry × 10 + language index`.
4. Convert the string to Text and update the label.

The supported languages are English, Simplified Chinese, Traditional Chinese, German, French, Latin American Spanish, Brazilian Portuguese, Russian and Japanese. The `es` alias occupies the tenth language position.

Normal label entries use the mode ID. When None is blocked by the sandbox support requirement, an additional label entry at index 9 displays the localized requirement message and the hint turns amber. This label entry is not another mode.

### Device-aware button prompts

The session gets the native `AbioticUIActionRouter` through `Get Local Player Subsystem` using its own player controller, then reads `IsUsingGamepad`. Keyboard and mouse input shows `KeyLetter`; gamepad input shows `GamepadKey`. This follows the native device-detection rules.

`GetGamepadEnumID` identifies the controller family. `GamepadIcons` selects a native Xbox, PS4, PS5 or Steam Deck D-pad-left texture, and `SetBrushFromTexture` updates the image. The default family uses the Xbox icon. These textures remain references to game content.

## 7. Where to make changes

| Change | Blueprint location |
| --- | --- |
| Cycle order | `BP_BridgeSession`: `CycleForwardFixed` and `CycleReverseFixed`. |
| Input keys | `ZPressed`, `GamepadCyclePressed`, `CycleLeftShift` and `CycleRightShift`. |
| Angles or directions | `BP_BridgePreview`: `DirectionTableNine` and its vector calculations. |
| Support hit rules | `AdditionalDeployCheck`: `Trace` and `ValidHit`. |
| Labels and translations | `BP_BridgeSession`: `Languages` and `Translations`. |
| Icons and layout | The `WBP_BridgeHint` widget tree and `Content/UI` textures. |
| Hint position | `NativeHintOffset` and `FollowNativeKey`. |

Adding a mode requires a matching entry in the direction table, both cycle tables, the icon switcher and the translation entries.

Supporting another author's bridge also requires adjusting the item-row and preview-class gates. The current implementation expects four support sockets, four native pillar-preview components, and the native placement flow that consumes `Supports`. Inheriting a native bridge class alone does not make a new item pass the current row-name filter.
