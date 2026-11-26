---
summary: ModCore texts - a mod's texts in the game's languages from one texts.json, translated lobby rules, headings and hotkeys, the game's own texts, and redrawing when the language changes
---

# Texts

`Core.Texts` shows your mod's texts in the language the player picked in the game. A mod keeps all its texts in one `texts.json` built into its DLL; a player can put a
`<DllName>.texts.json` next to the DLL to fix a line or add a language without rebuilding, and that file wins line by line.

```json
{
  // English is needed, other languages fall back to it.
  "window.title": { "en": "My window", "de": "Mein Fenster", "fr": "Ma fenêtre", "ru": "Моё окно", "uk": "Моє вікно", "zh-CN": "我的窗口" },
  "status.count": { "en": "{0} islands", "ru": "Островов: {0}" }
}
```

Each key has one text per language code: `en`, `de`, `fr`, `ru`, `uk`, `zh-CN` (the game's locales). Lines starting with `//` are comments. A missing language falls back to
English.

## Loading the texts

Build the file into the DLL and load it once, first thing in `Load()`:

```xml
<EmbeddedResource Include="texts.json" LogicalName="texts.json" />
```

```csharp
internal static class Tx
{
    internal static TextSet Set;
    internal static void Load() => Set = Core.Texts.Load();
    internal static string T(string key) => Set?.Get(key) ?? key;
    internal static string T(string key, params object[] values) => Set?.Get(key, values) ?? key;
}

public override void Load()
{
    Tx.Load();
    // ...
}
```

`Core.Texts.Load()` finds the calling mod's DLL by itself. A small `Tx` class like this keeps the calls short: `Tx.T("window.title")`, `Tx.T("status.count", 12)`.

## Reading a text

`Get(key)` answers in the current language. `Get(key, values...)` fills `{0}`, `{1}` ... like `string.Format`.

```csharp
string title = Tx.T("window.title");             // "Моє вікно" in Ukrainian
string count = Tx.T("status.count", 12);         // "12 islands" / "Островов: 12"
string english = Tx.Set.Get("window.title", "en");   // one language on purpose
```

Read a text when you draw it, not once in `Load()`: before the game has started its localization every text is English (see
[When the language can be read](../../game-architecture/audio-settings-localization.md#when-the-language-can-be-read)).

!!! warning "One string argument"
    `Get(key, "text")` with exactly one `string` picks the overload `Get(key, language)`. To fill `{0}` with a single string, pass it as an `object`:
    `Get(key, (object)name)`.

## Lobby rules, headings and hotkeys

Titles given in code are looked up in your texts by key, so the code keeps its English and `texts.json` translates it. No code change is needed.

| Text | Key in `texts.json` |
|---|---|
| A lobby rule's title / description | `rule.<rule key>.title`, `rule.<rule key>.description` |
| One choice of a choice rule | `rule.<rule key>.choice.<n>` (from 0) |
| A heading made with `Core.Lobby.Groups.Get` | `group.<group key>.title` |
| A hotkey's name | `hotkey.<hotkey id>.title` |

```csharp
var rule = Core.Lobby.Rules.Toggle("mymod.fastBuild", "Fast building", false, "Buildings are built in one turn.");
Core.Hotkeys.Add(Config, "openWindow", "Open my window", "Ctrl+M", MyWindow.Toggle);
```

```json
{
  "rule.mymod.fastBuild.title":       { "en": "Fast building", "de": "Schnelles Bauen" },
  "rule.mymod.fastBuild.description": { "en": "Buildings are built in one turn.", "de": "Gebäude entstehen in einem Zug." },
  "hotkey.openWindow.title":          { "en": "Open my window", "de": "Mein Fenster öffnen" }
}
```

A civilization's heading always shows the civilization's own name in the current language.

## The game's own texts

`Core.Texts.Game(key, fallback)` gives one of the game's texts in the current language, drawn the way the game draws it (icons included):

```csharp
string name = Core.Texts.Game("building.ARCHERY_RANGE.name", "Archery Range");
string unit = Core.Texts.Game("unit.ARCHER.name", "Archer");
```

Use it for unit, building, technology and civilization names in your screens instead of writing your own translations.

While the game is still loading it returns the fallback: reading the game's localization then can make the game wait for itself and hang. Once the texts
can be read, `LanguageChanged` follows, so names you read early (in `GameDataLoaded`, for example) can be read again there:

```csharp
Core.Civilizations.GameDataLoaded += ReadNames;   // may still get the fallbacks
Core.Texts.LanguageChanged += ReadNames;          // the game's names, in the player's language
```

## Fixing the game's texts

Two kinds of keys change texts of the game itself:

| Key | Changes |
|---|---|
| `gamekey:<game key>` | the game's text for that key, in the languages you give (others keep the game's text) |
| `game:<format>` | a text the game writes in its code, such as `game:{0} tiles` |

```json
{
  "gamekey:rules.title": { "uk": "Правила" },
  "game:{0} tiles":      { "de": "{0} Felder", "ru": "{0} клеток" }
}
```

`gamekey:` lines reach every text drawn through the game's `Localized` element. Texts placed in the scene with a `GameObjectLocalizer` (the Rules window, the lobby buttons,
see [Texts placed in the scene](../../game-architecture/audio-settings-localization.md#texts-placed-in-the-scene)) read the string table directly; for those, write the table
entry as shown in [Changing a text](../../game-architecture/audio-settings-localization.md#changing-a-text), from `LanguageChanged` (below).

## When the language changes

`Core.Texts.Language` is the current language code. `Core.Texts.LanguageChanged` fires after the player picks another language, and once at start when texts were asked before
the game's localization was ready. It fires one frame after the change, never inside the game's loading or inside a text lookup, so a handler may rebuild screens.

```csharp
Core.Texts.LanguageChanged += () => Log.LogInfo("language: " + Core.Texts.Language);
```

Screens that read their texts when they are drawn need nothing. Anything that keeps a text it was given must be built again: UiKit settings rows, for example, keep the label
they were made with. Build a settings block from a function and make it again when the language changes:

```csharp
internal static SettingsBlockHandle Block(UiScope ui, Func<SettingsBlockSpec> spec)
{
    var handle = ui.Native.Settings.AddBlock(spec());
    Core.Texts.LanguageChanged += () =>
    {
        handle?.Dispose();
        handle = ui.Native.Settings.AddBlock(spec());
    };
    return handle;
}

Block(ui, () => new SettingsBlockSpec
{
    Id = "author.mymod.settings",
    Page = SettingsPage.Game,
    Title = Tx.T("settings.title"),
    Build = block => block.AddToggle(Tx.T("settings.fast"), Options.Fast.Value, v => Options.Fast.Value = v),
});
```

`spec` runs again for every build, so every `Tx.T` in it is read again. Windows built when they open need nothing: they open in the current language. A top-bar button's tooltip
can follow with `button.SetTooltip(Tx.T(...))` in the same event.

!!! warning "Do not read the language while the game loads"
    Never read `LocalizationSettings.SelectedLocale` or look up texts from code that runs inside the game's loading step (for example `WhenGameDataLoaded`): the game stops
    responding. `Core.Texts` waits for the frame after that step by itself; `LanguageChanged` is the safe place for your own table work.
