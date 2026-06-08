# PWRUniversalArmorSystem

Modular, plug & play, **server-authoritative** armor & weapon equipment system for **UE5 (4.27+ portable)**.
Works with UE4 Mannequin, UE5 Manny/Quinn, MetaHuman/MetaWoman and custom skeletons.
Fully Blueprint-friendly. Configured entirely via **Enum + Struct + DataTable** (no Data Assets).

---

## 1. What you get

| Layer | Class / Asset | Role |
|-------|---------------|------|
| Component | `UPWRArmorEquipmentComponent` | Add to any Pawn/Character. Equip/unequip/swap, replication, visuals, body-hiding, stats, save/load, debug. |
| Brain | `UPWRArmorCompatibilityLibrary` | Stateless skeleton detection + compatibility validation. Callable from BP and the editor tool. |
| Data | `FPWRArmorItemData` (DataTable row) | One row per armor piece. |
| Data | `FPWRSkeletonCompatibilityProfile` (DataTable row) | One row per skeleton family (Manny, MetaHuman, custom…). |
| Data | `FPWRArmorStats`, `FPWRArmorEquipState`, `FPWRCompatibilityResult` | Stats, replicated state, validation result. |
| Enums | `EPWRArmorSlot`, `EPWRSkeletonType`, `EPWRArmorAttachMode`, `EPWRArmorCompatibilityMode`, `EPWRBodyHideMode`, `EPWRArmorEquipResult` | All Blueprint-exposed. |
| Editor | **PWR Armor** top-bar button → *PWR Armor Compatibility Setup* tab | Pick character + armor mesh, detect skeletons, recommend attach mode, copy a DataTable snippet. |

### Folder layout
```
Plugins/PWRUniversalArmorSystem/
├─ PWRUniversalArmorSystem.uplugin
├─ Resources/Icon128.png
└─ Source/
   ├─ PWRUniversalArmor/          (Runtime — ships in the game)
   │  ├─ Public/Types/            (enums + structs)
   │  ├─ Public/Components/       (equipment component)
   │  ├─ Public/Library/          (compatibility library)
   │  └─ Private/...
   └─ PWRUniversalArmorEditor/    (Editor-only — top bar + setup tool)
```

---

## 2. First build

1. Close the editor. The project is now a **code project** (it has a C++ plugin).
2. Right-click `PWR_Tools.uproject` → **Generate Visual Studio project files**.
3. Open the `.sln` and build **Development Editor**, *or* just reopen the `.uproject` and click **Yes** when prompted to rebuild missing modules.
4. After launch you'll see a **PWR Armor** button in the Level Editor top toolbar (also under **Tools → PWR Armor**).

> No C++ knowledge is required afterwards — everything below is done in Blueprint + DataTables.

---

## 3. Create the two DataTables

### A) Armor items — `DT_PWR_ArmorItems`
Content Browser → **Miscellaneous → Data Table** → pick row struct **`PWRArmorItemData`**.
Add one row per piece. Key fields:

| Field | Meaning |
|-------|---------|
| `ItemID` | Unique name (also the row name). Used by `EquipArmorByID`. |
| `ArmorSlot` | Head/Helmet/Chest/… |
| `SkeletalMesh` / `StaticMesh` | The visual. Rigid props use StaticMesh. |
| `bAutoSelectAttachMode` | **true** = system picks the best mode automatically (recommended). |
| `AttachMode` | Used only when auto-select is off, or as a hint. |
| `SocketName` | For rigid attaches (helmets/back items). |
| `bUseCustomOffset` + `RelativeLocation/Rotation/Scale` | Manual placement. |
| `BodyHideMode` + `bHideBodySections` + sections/bones | Anti-clipping (hide skin under armor). |
| `ArmorStats`, `GameplayTags`, `Icon`, `DisplayName`, `Description` | Gameplay/UI metadata. |

### B) Skeleton profiles — `DT_PWR_SkeletonProfiles`
Row struct **`PWRSkeletonCompatibilityProfile`**. One row per skeleton family you support
(see ready-made examples in §6). Optional but recommended — gives per-skeleton default sockets/offsets.

---

## 4. Add the system to any character (Blueprint)

1. Open your Character/Pawn BP (any class — no base class requirement).
2. **Add Component → PWR Armor Equipment Component**.
3. Select it and set:
   - `ArmorDataTable` = `DT_PWR_ArmorItems`
   - `CompatibilityProfileTable` = `DT_PWR_SkeletonProfiles` (optional)
   - `SupportedSlots` = leave empty (all) or restrict.
4. In **BeginPlay**, call `RegisterCharacterMesh(Mesh)` passing the character's main `Mesh` component.
   *(If you skip this, the component auto-grabs the first skeletal mesh on the owner.)*
5. Equip: `PWRArmorComponent → EquipArmorByID("KnightHelmet")`.
6. Unequip: `UnequipArmorSlot(Helmet)`.

That's the entire integration. No GameMode, PlayerController, Enhanced Input, or AnimBP dependency.

---

## 4b. Weapons (swords, shields, bows, holstered items)

Weapons reuse the same DataTable (`FPWRArmorItemData`) and the same server-authoritative flow.
For a weapon row:
- set **`WeaponSlot`** (MainHand / OffHand / TwoHand / Shield / Ranged / Sidearm / SheathHip* / Back*),
- leave `ArmorSlot = None`,
- assign `SkeletalMesh` **or** `StaticMesh`,
- optionally set `SocketName` (else a profile `WeaponSocketMapping` entry, else a sensible default like `hand_r`/`hand_l`).

Equip in Blueprint:
```
PWRArmorComponent → Equip Weapon By ID ("KnightSword")
PWRArmorComponent → Unequip Weapon Slot (MainHand)
```
Rules handled for you: a **TwoHand** weapon clears MainHand+OffHand; equipping a one-hander clears any TwoHand.
`SupportedWeaponSlots` works like `SupportedSlots` — **empty = all allowed**, or restrict per character in the component details.

Default weapon sockets (override via item `SocketName` or profile `WeaponSocketMapping`):
MainHand/TwoHand/Ranged → `hand_r`, OffHand/Shield → `hand_l`, Sidearm/SheathHipRight → `thigh_r`,
SheathHipLeft → `thigh_l`, Back*/Ammo → `spine_03`.

## 5. Multiplayer (already wired)

Flow is **server-authoritative** with zero mesh replication:

```
Client calls EquipArmorByID
        │
        ▼
Server_EquipArmor (RPC, reliable)
        │
   ServerValidateAndEquip  ← validates item, slot, compatibility, requirements
        │
   EquippedArmor[] changes (replicated, compact: slot + ItemID only)
        │
        ▼
OnRep_EquippedArmor on EVERY client → RefreshArmorVisuals()  (rebuilds meshes locally)
```

- Only the tiny `slot→ItemID` list crosses the wire. Meshes are rebuilt locally from the DataTable on each client → **no visible delay, no double components**.
- The owning actor must replicate: on your Character set **Replicates = true** (Characters already do).
- Listen-server / standalone apply visuals immediately on authority too.

---

## 6. Skeleton profile examples

Create these as rows in `DT_PWR_SkeletonProfiles`.

### UE5 Manny / Quinn
```
ProfileID            : UE5_Manny
SkeletonType         : UE5_Manny
MainMeshComponentName: CharacterMesh0
RequiredBones        : [pelvis, spine_05, hand_l, hand_r, head]
SignatureBones       : [thigh_twist_01_l, spine_04, spine_05]
SocketMapping        : { Helmet: head, Back: spine_05, Hands: hand_r }
SupportedSlots       : [Helmet, Chest, Arms, Hands, Legs, Feet, Back, Cloak, Accessory01]
```
Armor authored on the **same** SK_Mannequin skeleton → `bAutoSelectAttachMode=true` resolves to **Leader Pose** (zero setup).

### UE4 Mannequin
```
ProfileID            : UE4_Mannequin
SkeletonType         : UE4_Mannequin
MainMeshComponentName: CharacterMesh0
RequiredBones        : [pelvis, spine_01, spine_03, hand_l, hand_r, head]
SocketMapping        : { Helmet: head, Back: spine_03 }
SupportedSlots       : [Helmet, Chest, Arms, Hands, Legs, Feet, Back]
```
UE4 armor on a UE5 character: skeletons differ → the system picks **Copy Pose From Mesh** if bone names overlap ≥75%, otherwise it tells you to retarget.

### MetaHuman / MetaWoman
```
ProfileID            : MetaHuman_Male      (make a _Female row too)
SkeletonType         : MetaHuman_Male
MainMeshComponentName: Body                (MetaHuman body component is usually named "Body")
RequiredBones        : [pelvis, spine_05, head, FACIAL_C_FacialRoot]
SocketMapping        : { Helmet: head }
SupportedSlots       : [Helmet, Chest, Arms, Hands, Legs, Feet, Cloak, Back]
```
MetaHuman bodies come as separate Body/Torso/Legs/Feet skeletal meshes sharing the MetaHuman skeleton.
- Hard-surface armor on the MetaHuman skeleton → **Leader Pose** off the `Body` mesh.
- Clothing meshes from a different rig → **Copy Pose** or retarget.
- Use `BodyHideMode = HideMaterialSection` on torso/leg material slots to remove skin clipping.

### Custom skeleton
Leave `SkeletonType = CustomHumanoid`. Set `RequiredBones` to whatever your rig guarantees.
The setup tool will report overlap and the recommended mode.

---

## 7. Using the editor tool (top bar)

**PWR Armor** button → tab opens:
1. Pick **Character Body Mesh** and **Armor Mesh**.
2. **Check Compatibility**. You get:
   - detected skeleton type for each,
   - same-skeleton yes/no,
   - compatibility mode (Direct / RetargetCompatible / SocketOnly / NeedsRetarget / NeedsManualSetup / NotCompatible),
   - **recommended attach mode**,
   - any missing bones.
3. **Copy Snippet to Clipboard** → paste the suggested values into your DataTable row.

This is the workflow for making "foreign" marketplace armor compatible without guessing.

---

## 8. Attach-mode decision (what the system does for you)

| Situation | Result | Attach mode |
|-----------|--------|-------------|
| Same `USkeleton` | DirectSameSkeleton | **Leader Pose** (free) |
| Different skeleton, bone-name overlap ≥ 75% | RetargetCompatible | **Copy Pose From Mesh** |
| Overlap 35–75% | NeedsRetarget | tells you to build an IK Retargeter |
| Overlap < 35% | NeedsManualSetup | suggests rigid Socket Attach |
| Rigid item / StaticMesh / socket set | SocketOnly | **Socket / Static Mesh Attach** |
| Full-body mesh, compatible | DirectSameSkeleton | **Full Mesh Replacement** |

> **No magic.** When a retarget or manual setup is genuinely required, the system says so and points you to the next step instead of silently producing a broken result.

---

## 9. Blueprint API quick reference

**Setup:** `RegisterCharacterMesh`, `GetCharacterMesh`, `RegisterSkeletonProfile`
**Equip (armor):** `EquipArmorByID`, `UnequipArmorSlot`, `SwapArmorItem`
**Equip (weapons):** `EquipWeaponByID`, `UnequipWeaponSlot`
**Query:** `GetEquippedArmor`, `IsArmorSlotOccupied`, `GetEquippedWeapon`, `IsWeaponSlotOccupied`, `CanEquipArmor`, `ValidateArmorItem`, `GetArmorStats`, `GetTotalArmorStats`, `GetArmorItemData`
**Visual:** `RefreshArmorVisuals`, `PreviewArmor`, `ClearArmorPreview`
**Save/Load:** `SaveEquippedArmor`, `LoadEquippedArmor`
**Debug:** `DebugArmorSystem`, `DebugCompatibility`, `DebugReplication`, `DebugVisualComponents`
**Library (static):** `DetectSkeletonType`, `HaveSameSkeleton`, `MeshHasBone`, `GetMissingBones`, `GetBoneNameOverlap`, `ValidateArmorCompatibility`, `ValidateMeshes`

**Events:** `OnArmorEquipped`, `OnArmorUnequipped`, `OnArmorChanged`, `OnArmorEquipFailed`, `OnArmorCompatibilityWarning`
**Requirements hook:** override `OnCheckRequirements` (BlueprintNativeEvent) and set `bEnforceRequirements=true` to gate equips on level/stats.

---

## 10. Test checklist

- [ ] Plugin compiles; **PWR Armor** button visible in top bar.
- [ ] Component added; `RegisterCharacterMesh` called; `EquipArmorByID` shows the mesh.
- [ ] Same-skeleton armor animates 1:1 (Leader Pose).
- [ ] Rigid helmet attaches to `head` socket with correct offset.
- [ ] `UnequipArmorSlot` removes mesh and restores hidden body sections.
- [ ] FullBody equip clears other slots; equipping a slot clears FullBody.
- [ ] **Dedicated server + 2 clients:** equip on client → appears on both clients with no delay.
- [ ] Late-joining client sees already-equipped armor (OnRep on BeginPlay).
- [ ] `OnArmorEquipFailed` fires with a clear `EPWRArmorEquipResult` on bad item / incompatible skeleton.
- [ ] Setup tool detects Manny / UE4 / MetaHuman correctly and recommends the right mode.
- [ ] `GetTotalArmorStats` sums equipped pieces.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Armor invisible | Character mesh not registered | Call `RegisterCharacterMesh(GetMesh())` in BeginPlay. |
| Armor in T-pose / not animating | Different skeleton, fell back from Leader Pose | Tool says "Copy Pose" → add a Copy-Pose AnimBP to the armor mesh, or retarget. |
| `ItemNotFound` | Row name ≠ `ItemID` | Match the DataTable **row name** to the ID you pass. |
| Helmet at character's feet | Missing/empty `SocketName` | Set `SocketName` (e.g. `head`) or a profile `SocketMapping` entry. |
| Skin pokes through armor | No body hiding | Set `BodyHideMode` + `bHideBodySections` + sections/bones on the item. |
| Works on server, not clients | Owning actor not replicated | Set actor **Replicates = true**. |
| MetaHuman armor wrong mesh picked | Multiple body meshes | Set profile `MainMeshComponentName` = `Body` (or call RegisterCharacterMesh explicitly). |
| Setup tool: "Needs Retarget" | < 75% bone overlap | Build an IK Retargeter (armor skeleton → character skeleton) and bake the mesh. |

---

## 12. Design notes / guarantees

- **No hardcoded references** to ThirdPersonCharacter, input, GameMode, or a specific skeleton — detection is by bone signature, not asset name.
- **No mesh replication** — only `slot→ItemID` replicates; visuals rebuild locally → minimal bandwidth, no double components, no race on visuals.
- **Server validates** item, slot, compatibility and optional requirements before any state change.
- Visual components are created at runtime into `TMap<EPWRArmorSlot, …>` and destroyed cleanly on unequip / EndPlay.
- Soft object pointers keep meshes out of memory until equipped.
