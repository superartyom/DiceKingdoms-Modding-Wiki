---
summary: ModCore building limits - how many buildings of one kind an island may have, suggested by the mod that adds the building, changeable by the host through a lobby mod, checked the same way on every computer
---

# Building limits

`Core.Buildings` keeps every island to a number of buildings of one kind. The mod that adds a building suggests its limit, a lobby mod can let the host change
the numbers, and ModCore checks them inside the game's own build checks, so every computer refuses the same building. 0 means no limit, the default of every
building. An upgraded building (the upgraded Guard Tower) counts with its base building, and only standing buildings count (ruins do not).

## Suggest a limit for your building

Call it in your plugin's `Load()`. From then on an island can have at most that many, whether or not a lobby mod is installed.

```csharp
public override void Load() {
    MyBuilding.Install();
    Core.Buildings.SuggestLimit(MyBuilding.Name, 1);   // one per island; 0 = no limit
}
```

The building at its limit is greyed out in the build bar and its tooltip says how many the island has of how many it may have. A lobby mod that lets the host
change the limits ([below](#let-the-host-change-the-limits)) reads the suggestion with `Core.Buildings.SuggestedLimit(type)` and uses it as the default of its number.

## Let the host change the limits

A mod that holds the host's numbers gives ModCore a function from a base building to its limit. `null` keeps the suggested limit.

```csharp
var heading = Core.Lobby.Groups.Get("author.limits", "Building limits");
int farmDefault = Core.Buildings.SuggestedLimit(BuildingTypeName.FARM);
var farms = Core.Lobby.Rules.Number("author.limits.FARM", "Farm", farmDefault, 0, 50, 1).InGroup(heading);

Core.Buildings.SetLimits("My limits", type => type == BuildingTypeName.FARM ? farms.Value : null);
```

The answer must be the same on every computer during the match: a lobby rule's `Value` is (it is frozen when the match starts). Use the suggested limit as the
rule's default. Then a host who changes nothing gives the same limits as a player without your mod, and ModCore requires your mod from everybody only once a number
differs from its default ([Lobby and rules](lobby.md)). The function is asked with base buildings only. There is one source: a second `SetLimits` replaces the first.

## Read the limits

```csharp
int limit = Core.Buildings.LimitOf(BuildingTypeName.GUARD_TOWER_UP);    // the Guard Tower's; 0 = none
int count = Core.Buildings.CountOn(island, BuildingTypeName.GUARD_TOWER);   // plain and upgraded ones
bool full = limit > 0 && count >= limit;
var baseType = Core.Buildings.BaseOf(BuildingTypeName.GUARD_TOWER_UP);  // GUARD_TOWER
```

`BaseOf` comes from the game's `ReplaceBuildings` technologies (`baseBuildingType` is replaced by `upgradedBuildingType`), so it knows the upgrades once the game's
data is loaded.

## How it is checked

ModCore patches these game functions when the first mod calls `SuggestLimit` or `SetLimits`; without one the game is unchanged.

- **`BuildData.CanBuildType`** (postfix): the check that counts. The game runs it in `DoAction` when a planned building is placed
  ([Build phase](../../game-architecture/build-phase.md#checks-and-effects)), on every computer at the same step and with the buildings placed earlier in the phase
  already on the island, so all computers refuse the same one. Computer players' buildings go through it too.
- **`BuildData.PlaceBuildingBulk`** (prefix): walls come in groups of ten and the game checks the type once per group, so a group that would go over the limit is
  refused when it is planned.
- **`BuildInterface.RefreshBuildingButtons`** (prefix and postfix): a building at its limit is greyed out (`buildingButtons[i].button.Interactable = false`).
- **`BuildData.CanAfford`** (postfix): false at the limit, so placing with Shift held stops there. It is left alone while the build bar is redrawn: there a false
  result would make the tooltip list missing resources ([Interface behaviour](../../game-architecture/interface.md#build-bar-buttons)).
- **`BuildData.CanAffordBulk`** (postfix): the wall placer turns red when the dragged walls would go over.
- **`BuildInterface.TooltipText`** (postfix): adds "On this island: n of m" in the game's language.

The screen counts the buildings once per island and frame (the wall placer asks every frame); the check in `CanBuildType` counts afresh every time.
