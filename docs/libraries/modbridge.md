---
summary: ModBridge - shows ModCore's lobby rules, compatibility warnings and civilizations on UiKit's screens; when a mod needs it and how to add a new bridge
---

# ModBridge

Connects [ModCore](modcore/index.md) (data, no screens) with [UiKit](uikit/index.md) (screens, no game logic). It has no API: install it and these appear, with nothing for a
mod author to call. Source and releases: [DiceKingdoms-ModBridge](https://github.com/DiceModders/DiceKingdoms-ModBridge).

- **Lobby rules**: every rule registered with `Core.Lobby.Rules` gets a row in the Settings box of the lobby screen, grouped per mod. The host edits it before the match, everybody
  else sees the host's value read-only. Rows follow the lobby live (a member joins, the host changes a value, the match starts and rows turn read-only).
  Rules under a heading ([rule groups](modcore/lobby.md#rule-groups-per-civilization)) keep the list short: the heading's first rule, when it is an on/off tick, is its switch
  and gets a row in the list; its other rules get rows too when there is only one, and when there are two or more, a *&lt;heading&gt;: settings* row with an *Open* button
  opens them in their own window (ticks, drop-downs, numbers with - / + and *Default*, each with its description).
- **Compatibility warnings**: `Core.Lobby.Compatibility.Warnings` are listed under the rules.
- **Civilizations**: every civilization added with `Core.Civilizations.Add` gets an entry in the Rules drop-down.
- **Mod keys**: every key added with `Core.Hotkeys.Add` ([Hotkeys](modcore/hotkeys.md)) gets a row in Settings > Input, one heading per mod. The button shows the key; click it and press the new key (Escape cancels). *Default* puts the mod's own key back.

So a mod that registers a rule needs no UI code at all:

```csharp
[BepInPlugin("author.my-mod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.ModCore")]
public class MyModPlugin : BasePlugin {
    public override void Load() {
        var storms = Core.Lobby.Rules.Toggle("author.my-mod.storms", "Storms", true,
                                             description: "Storms can sink ships.");   // ModBridge draws the row
        Core.Match.PhaseEntered += phase => { if (storms.Value && phase == MatchPhase.Disasters) MaybeStorm(); };
    }
}
```

A mod with a switch and a few settings puts them under its own heading, the switch first. The list then shows one tick and one *Open* row, and the settings stay in
their window:

```csharp
var tolls = Core.Lobby.Groups.Get("author.tolls", "Tolls");
var on    = Core.Lobby.Rules.Toggle("author.tolls.on", "Tolls", false, description: "Bridges cost gold to cross.").InGroup(tolls);   // the tick in the list
Core.Lobby.Rules.Number("author.tolls.price", "Price", 2, 1, 10, 1, visible: () => on.Live).InGroup(tolls);                           // in the window
Core.Lobby.Rules.Number("author.tolls.free", "Free crossings", 1, 0, 5, 1, visible: () => on.Live).InGroup(tolls);                    // in the window
```

The *Open* row is shown only while one of the window's rules is visible, so here it appears when the host ticks *Tolls*.

A heading whose first rule is not a tick has no switch, and a mod with its own settings window can keep its rules out of the list:

```csharp
// no tick first: the list shows one "Building limits: settings" row, the numbers are in the window
var limits = Core.Lobby.Groups.Get("author.limits", "Building limits");
Core.Lobby.Rules.Number("author.limits.FARM", "Farm", 0, 0, 50, 1).InGroup(limits);
Core.Lobby.Rules.Number("author.limits.HOUSE", "House", 0, 0, 50, 1).InGroup(limits);

// no row in the list: the mod's own window edits it
var wheel = Core.Lobby.Rules.Toggle("author.wheel.on", "Wheel", false, visible: () => false);
window.Body.Toggle(wheel.Title, wheel.Live, v => wheel.TrySet(v));
```

A rule with `visible: () => false` still works and is sent to everybody; only its row is gone. In your own window only the host's `TrySet` is accepted, so show the
others the host's `Live` value read-only and update it on the rule's `Changed`.

Mods that use ModCore do not depend on ModBridge: without it their rules still work (with their defaults, or the host's value set by another tool); there are just no rows.
List it as an optional dependency, or required if your mod is useless without the screens.

## Adding a bridge

When a new ModCore feature needs a screen, the screen goes into ModBridge so that ModCore and UiKit stay independent: one file per screen with an `Install(UiScope scope)`
method that reads `Core.*`, draws with the scope's handles and subscribes to ModCore's events to update them, called from `WhenReady`.

```csharp
internal static class KillCounterScreen {
    public static void Install(UiScope scope) {
        var column = scope.Native.Scoreboard.AddColumn("modbridge.kills", "Kills");
        Core.Match.PhaseSynced += _ => {
            foreach (var p in Core.Players.All)
                if (p.Island != null) column?.SetValue(p.Island, CountKills(p).ToString());
        };
        Core.Match.Ended += () => column?.ClearValues();
    }
}

// Plugin.Load():  scope.WhenReady(s => { LobbyRulesScreen.Install(s); CivilizationEntries.Install(s); KillCounterScreen.Install(s); });
```
