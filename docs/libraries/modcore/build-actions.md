---
summary: ModCore build actions - a mod's own build-phase action (an offering, a bet) that travels and applies like the game's market purchase, the same on every computer
---

# Build actions

A choice that changes the simulation and must happen at once (pay gold for something during the build phase) has to reach every computer and be applied there in the
same order. The game already does this for its own build actions, such as a market purchase: the click is planned as an action, the player's own screen applies it
at once (the gold drops), the game sends it to everybody with its own messages, and every computer applies it when the build phase ends (the game waits for that).
`Core.BuildActions` lets a mod use the same path.

## Register your action

Pick a code from 0 to 63 once for your mod and register the handler in your plugin's `Load()`. The handler runs on every computer with the build data of the island
the action belongs to and the building id it was planned on.

```csharp
Core.BuildActions.Register(7, "author.bets (bet)", (data, buildingId) => {
    var stored = data.Island.statistics.Storage.stored;     // read only the game state: every computer must reach the same answer
    if (stored.gold < 3) return false;                       // false: the action is dropped, like any failed game action
    TakeGold(data.Island, 3);
    Bets.Add(data.Island, buildingId);
    return true;
});
```

Return `true` when the action happened. Read only what every computer has (the island, its buildings and storage); a value from your own screen or a random number
would give different results on different computers.

## Plan it on the player's click

During the build phase, plan the action with the local player's build data (`BuildInterface.Data`) and the building it is about. The handler runs on this screen at once
and its result is returned.

```csharp
public static void OnBetClick(BuildInterface build, int buildingId) {
    if (Core.Match.Phase != MatchPhase.Build) return;
    bool placed = Core.BuildActions.Plan(build.Data, 7, buildingId);
    build.RefreshButtons();   // the build bar shows what the player can still afford
}
```

## How it travels

The game reads only the action kinds it knows, so a mod action is sent as the game's repair action on a building id with a marker: bit 30, and the code in bits 24..29.
Building ids are `y << 16 | x` with both below 256, so they never use those bits. ModCore hooks `BuildData.DoAction` natively (it takes the action by reference, which a
Harmony patch cannot pass safely) and runs your handler for a marked action; every other action goes to the game unchanged.

Every player in the match needs the mod that registered the code, so plan actions only while your lobby rule is on (ModCore uses a rule only when everybody has the
mod). Codes in use: 0 Upkeep (Cathedral offering), 1 Casino (spin). A second registration of a code is refused and logged.
