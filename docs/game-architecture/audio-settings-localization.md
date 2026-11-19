---
summary: Audio (sound pool, music selection, mixer and volume curve), saved settings and Steam Cloud, the tutorial, and the localization system with its language quirks
---

# Audio, settings, localization and tutorial

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Audio

Sounds and the player for them:

<!-- code: Sound | *fields -->
```csharp
namespace DiceKingdoms.Interface;

public class Sound : ScriptableObject
{
    public AudioClip[] clips;           // 0x18
    public AudioMixerGroup mixerGroup;  // 0x20
    public float minVolume;             // 0x28
    public float maxVolume;             // 0x2C
    public float minPitch;              // 0x30
    public float maxPitch;              // 0x34
    public bool loop;                   // 0x38
}
```
<!-- /code -->

<!-- code: AudioSources | Play -->
```csharp
namespace DiceKingdoms.Interface;

public class AudioSources : MonoBehaviour
{
    public AudioSources.AllocatedSource Play(Sound sound, Nullable<Vector3> positionOpt);  // RVA 0x580C20
}
```
<!-- /code -->

**Patch:** `AudioSources.instance.Play(sound, null)` plays any `Sound`.

| Piece | What it is |
|---|---|
| `Sound` (ScriptableObject, 48 assets) | `clips[]`, `mixerGroup`, `minVolume` / `maxVolume`, `minPitch` / `maxPitch`, `loop` |
| `AudioSources` (static `instance`) | a pool of **2048** AudioSources |
| `AudioSources.AllocatedSource` | the handle `Play` returns: source + **ticket**; an old handle can never stop a sound that reused the source |
| `Main` mixer | Master › Main › Effects, Music, Ambient, Interface; exposed `Main_Volume`, `Effects_Volume`, `Music_Volume`, `Ambient_Volume`, `Interface_Volume` |

`AudioSources.Play(sound, position?)` (`0x580C20`) takes the first free source (none free → error, no sound), 3D at the position or 2D without one, random pitch and volume in the sound's ranges,
**a random clip**, and plays it.

### Music

`Music` (match scene): calm track per civilization (Europe 6 clips, Middle East 3, Viking 9, Ancient 9, Maya 6), combat, decisive combat, winning and losing tracks. `Music.get_Track`
(`0x590F70`), every frame:

The music component:

<!-- code: Music | *fields, *methods -->
```csharp
namespace DiceKingdoms.Interface;

public class Music : MonoBehaviour
{
    private Sound europeTrack;                                                     // 0x18
    private Sound middleEastTrack;                                                 // 0x20
    private Sound vikingTrack;                                                     // 0x28
    private Sound ancientTrack;                                                    // 0x30
    private Sound mayaTrack;                                                       // 0x38
    private Sound combatTrack;                                                     // 0x40
    private Sound combatDecisiveTrack;                                             // 0x48
    private Sound winningTrack;                                                    // 0x50
    private Sound loosingTrack;                                                    // 0x58
    private float transitionDuration;                                              // 0x60
    private AudioSources audioSources;                                             // 0x68
    private bool someoneWinning;                                                   // 0x78
    private Sound currentTrack;                                                    // 0x80
    private AudioSources.AllocatedSource currentSource;                            // 0x88
    private readonly List<AudioSources.AllocatedSource> dyingSources;              // 0xA0

    protected void Awake();                                                        // RVA 0x590090
    protected void OnEnable();                                                     // RVA 0x590290
    protected void OnDisable();                                                    // RVA 0x590180
    protected void Update();                                                       // RVA 0x590B40
    private void OnEnterPhase();                                                   // RVA 0x5903A0
    private void UpdateVolumes();                                                  // RVA 0x5908B0
    private void UpdateVolume(AudioSources.AllocatedSource source, float factor);  // RVA 0x5907B0
    private void UpdateTrack();                                                    // RVA 0x5906B0
    private void Transition(Sound track);                                          // RVA 0x590440
}
```
<!-- /code -->

**Patch:** postfix `Music.get_Track` to play your own track.

1. Win phase with a result → winning if the winner is on your team, else losing.
2. Combat phase, 5 s into the fight → **decisive** if somebody could win this turn (`WinPhase.CheckWin()` at phase start), else combat.
3. Otherwise the calm track of **your** civilization. When a song ends, a new random clip of the same track starts (a shuffled playlist).

Changes crossfade over **2 s**. The main menu has its own `menuMusic`.

### Volume

Five sliders (0–100) in the settings. Each writes its mixer parameter as `dB = clamp(20 × log10(value / 100) + offset, −80, 0)`:

| Slider | Offset | Default |
|---|---|---|
| Main | 0 | 70 |
| Effects | +6 | 100 |
| Music | −8 | 50 |
| Ambient | 0 | 100 |
| Interface | +5 | 100 |

Ambience: the sea, and a night-nature loop that only plays at night (see [Rendering](rendering.md#day-and-night)).

For mods: `AudioSources.instance.Play(sound, null)` plays any `Sound`; keep the handle to stop it. Music can be replaced by changing the `clips` of the Music-* assets or by a postfix on `Music.get_Track`.

## Settings and saved data

The game saves only **four settings files** (`video`, `audio`, `input`, `game`, JSON) and the flag file **`GameSettings/.did-tutorial`**. There are no save games, no PlayerPrefs and no Steam
stats, achievements or leaderboards.

The settings sets:

<!-- code: SettingSets | *fields, *methods -->
```csharp
namespace DiceKingdoms.Meta;

public class SettingSets : MonoBehaviour
{
    private string localFolder;                                          // 0x18
    public VideoSettings videoSettings;                                  // 0x20
    public AudioSettings audioSettings;                                  // 0x28
    public InputSettings inputSettings;                                  // 0x30
    public GameSettings gameSettings;                                    // 0x38
    private SettingSet[] settingSets;                                    // 0x40

    protected void Awake();                                              // RVA 0x54F2A0
    protected void OnDestroy();                                          // RVA 0x54F8D0
    private string Path(SettingSet settingSet);                          // RVA 0x54F9A0
    public static void Store(bool remote, string path, byte[] bytes);    // RVA 0x54FB20
    public static bool Get(bool remote, string path, out byte[] bytes);  // RVA 0x54F6C0
    public void Save();                                                  // RVA 0x54FA30
    private void Load();                                                 // RVA 0x54F740
}
```
<!-- /code -->

`SettingSets` (`localFolder = "GameSettings"`, path relative to the game folder):

| Set | Stored |
|---|---|
| video | always the local file `GameSettings/video.json` |
| audio, input, game | **Steam Cloud** when Steam runs, otherwise the local file |

- **Save** when the settings window closes (`SettingSetsInterface.OnDisable`): each set is exported and stored.
- **Load** at start: a cloud set with Steam running is read from Steam Cloud only (missing → defaults, no fallback to the local copy); otherwise the local file or defaults.
- Defaults: video quality Normal, full screen mode 2, 1920×1080, frame limit 144, VSync on; audio as above; input bindings (rotate `/Keyboard/r`, mirror `/Keyboard/t`, keep placing
  `/Keyboard/shift`, scoreboard `/Keyboard/tab`, …); game language 0.

For a mod's own settings a BepInEx config file is easiest. To use Steam Cloud like the game: `SettingSets.Store(true, "GameSettings/mymod.json", bytes)` / `SettingSets.Get(true, ...)`.
How the settings window is built: [Interface behaviour](interface.md#settings-widgets).

### Saving when the window closes

The settings window:

<!-- code: SettingSetsInterface | *fields, OnEnable, OnDisable -->
```csharp
namespace DiceKingdoms.Meta;

public class SettingSetsInterface : ToggleableWindow
{
    private SettingSetInterface video;       // 0x18
    private SettingSetInterface audio;       // 0x20
    private SettingSetInterface input;       // 0x28
    private SettingSetInterface game;        // 0x30
    private BaseButton closeButton;          // 0x38
    private Action onClose;                  // 0x40
    public InputActionAsset actionsAssets;   // 0x48

    protected void OnEnable();               // RVA 0x54EFD0
    protected void OnDisable();              // RVA 0x54EEA0
}
```
<!-- /code -->

What the two do:

```c#
protected void OnEnable()
{
    actionsAssets.Disable();
}

protected void OnDisable()
{
    actionsAssets.Enable();
    var sets = SettingSets.instance;
    foreach (SettingSet set in sets.settingSets)
    {
        bool remote = set.remote && SteamClient.IsValid;
        SettingSets.Store(remote, sets.Path(set), set.Export());
    }
}
```

**Logic.** Opening the window switches the game's input actions off, so keys pressed in the key boxes do nothing in the game; closing switches them back on and writes every
set. There is no Save button: closing the window is the save. Nothing is checked for null, so a missing `actionsAssets`, `SettingSets.instance` or a null entry in `settingSets`
stops the method with a NullReferenceException, and the sets after that point are not written.

**Patch:** a setting changed from code is not written until the window closes; call `SettingSets.instance.Save()` (`0x54FA30`) to write it at once.

## Tutorial ("Training")

Main menu → Training → `MainMenu.LoadTraining`: writes `.did-tutorial` to Steam Cloud **as soon as it starts** (the "recommend tutorial" hint is hidden from then on), adds **two Passive
AIs** (the only use of that difficulty) and marks the match as training.

`Training` (in the match scene) has `TrainingStep` children with `TrainingObjective`s; all the data is in the scene:

| # | Step | Objectives |
|---|---|---|
| 1 | Initial buildings | Castle, 3 Cottages, Farm, Dock |
| 2 | Get more dice | 2 Cottages, 2 Houses |
| 3 | Stable food supply | 2 Farms |
| 4 | Culture | a Church; 12 culture |
| 5 | See opponents' progress | open the scoreboard |
| 6 | Military | Barracks; 8 military |
| 7 | Attack | a Dock; press Ready in combat; pillage once |
| 8 | Research | University; research Storage; build a Storage |
| 9 | Win | military win |

At the start every building button and Ready are hidden and **combat phases are skipped**; each step shows the buttons it needs, and the Pillage step turns combat on. One-time hints
(`ConfirmModal.OpenInformation`) explain the harvest, disasters, attack, pillage, starvation and the University.

## Localization

Unity **Localization** with SmartFormat, string tables in Addressables bundles, 421 keys × 6 languages. Game code goes through `StructuredTextStaticReferences.instance`
(`LocalizeString(key)`).

The language selector:

<!-- code: SteamLocaleSelector | *methods -->
```csharp
namespace DiceKingdoms.Meta;

public class SteamLocaleSelector : IStartupLocaleSelector
{
    public Locale GetStartupLocale(ILocalesProvider availableLocales);  // RVA 0x550170
}
```
<!-- /code -->

**Language at start** (`Localization.asset` startup selectors):

1. `SteamLocaleSelector.GetStartupLocale` (`0x550170`), only with Steam: english → en; schinese, tchinese → zh; french → fr; russian → ru; **ukrainian → ru**; anything else, **German included → en**.
2. Fallback `en`.

So a German Steam client starts in English, and a Ukrainian one in Russian although a Ukrainian table exists.

**Language setting** (`GameSettings.language`, Steam Cloud): 0 Steam default (selectors run again), 1 English, 2 French, 3 Chinese (simplified), 4 Russian, 5 German. There is **no Ukrainian**
entry; only a mod can select it (`LocalizationSettings.SelectedLocale`).

The setting and its values:

<!-- code: GameSettings | *fields -->
```csharp
namespace DiceKingdoms.Meta;

public class GameSettings : SettingSet
{
    public SettingSet.Setting<GameSettings.Language> language;   // 0x28

    public enum Language
    {
        STEAM_DEFAULT = 0,
        ENGLISH = 1,
        FRENCH = 2,
        CHINESE_SIMPLIFIED = 3,
        RUSSIAN = 4,
        GERMAN = 5,
    }
}
```
<!-- /code -->

What happens when it changes (`GameSettings.<>c.<.ctor>b__2_0`, `0x5597A0`, the action the constructor gives the setting):

```c#
static void OnLanguage(GameSettings.Language value)
{
    if (value == GameSettings.Language.STEAM_DEFAULT)
    {
        LocalizationSettings.Instance.ResetState();   // the startup selectors run again
        return;
    }
    SystemLanguage system = value switch
    {
        GameSettings.Language.ENGLISH            => SystemLanguage.English,   // 10
        GameSettings.Language.FRENCH             => SystemLanguage.French,    // 14
        GameSettings.Language.CHINESE_SIMPLIFIED => SystemLanguage.Chinese,   // 6
        GameSettings.Language.RUSSIAN            => SystemLanguage.Russian,   // 30
        GameSettings.Language.GERMAN             => SystemLanguage.German,    // 15
        _ => throw new ArgumentOutOfRangeException("value", value, null),
    };
    LocalizationSettings.SelectedLocale = LocalizationSettings.AvailableLocales.GetLocale(system);
}
```

**Logic.** Every value maps to a `SystemLanguage`, and the locale for it becomes the selected one; any value above 5 throws. The menu (`SettingInterfaceEnumLanguage`, a
`SettingInterfaceEnum<GameSettings.Language>`) fills its drop-down from the enum through `Interface.Dropdown.SetEntries` (`0x58B4C0`): the entries are
`(StructuredText, object)` value tuples with the boxed enum value as the object, and picking one sets the setting to that value.

**Patch:** a sixth language needs both a drop-down entry and a prefix on this handler for the new value. Add the entry by changing the drop-down after `SetEntries` has run
(a postfix) rather than by building new value tuples: creating generic structs from the interop side is unreliable and can crash the game without a log line.

Texts are `StructuredText` with `ITextElement.Localized(key, args)`: the translation can contain the same icon tokens and `{a.N}` argument slots (see
[Interface behaviour](interface.md#text-structuredtext)). Rendered text is cached per locale and labels redraw when the language changes.

### How a text is drawn

The references the texts use:

<!-- code: StructuredTextStaticReferences | *fields, LocalizeString -->
```csharp
namespace DiceKingdoms.Interface;

public class StructuredTextStaticReferences : MonoBehaviour
{
    private BuildingTypes buildingTypes;                 // 0x18
    private SkinRegistry skinRegistry;                   // 0x20
    private PlayerColors playerColors;                   // 0x28
    private Disasters disasters;                         // 0x30
    private LocalizationSettings localizationSettings;   // 0x38
    private TableReference localizationStringTable;      // 0x40
    public static StructuredTextStaticReferences instance;

    public string LocalizeString(string key);            // RVA 0x5953B0
}
```
<!-- /code -->

A localized part of a text:

<!-- code: ITextElement.Localized | *fields, get_Render -->
```csharp
namespace DiceKingdoms.Interface;

public struct ITextElement.Localized : ITextElement
{
    public string key;                 // 0x0
    public ITextElement[] arguments;   // 0x8

    public string get_Render();        // RVA 0x58FE00
}
```
<!-- /code -->

`Localized` is a **struct**. Its `Render`:

```c#
public string get_Render()
{
    var refs = StructuredTextStaticReferences.instance;
    string text = refs.localizationSettings.GetStringDatabase()
                      .GetLocalizedString(refs.localizationStringTable, key);
    var builder = new StructuredText.Builder();
    builder.Add($"{text}", arguments);
    return builder.Build().Render();
}
```

**Logic.** Every text is in one string table (`localizationStringTable`), looked up in the selected language. The translation is not used as it is: it becomes the only part of
a `"{0}"` format and is read again as a `StructuredText` with the element's `arguments`, which turns icon tokens (`{r.food}`) and argument slots (`{a.0}`) into the final text.
`LocalizeString(key)` is the same lookup without that last step.

**Patch:** a prefix on `Localized.get_Render` that returns its own text must do the same `Builder` step, or the tokens in it stay raw.

### Texts placed in the scene

Not every text goes through `Localized`. Many fixed texts in the menus are TextMeshPro objects with Unity's `GameObjectLocalizer` (Localization property variants) next to a
`Label`: the localizer writes the table's text straight into the TMP `text` field when the object is enabled and when the language changes. 131 objects work this way, among them:

| Where | Texts |
|---|---|
| Rules window, in the main menu and in the match | the title, the six tab buttons and every page |
| Lobby (`StartGame`) | the player slots, Start Game, Invite Friends, AI |
| Main menu | the left buttons (Training, Rule Book, Quick Match, Single Player, Custom Game, Daily Reward), the Custom Game panel |
| Settings | the tab titles and the Close button |
| Others | the store's confirm dialog, the phase banners, the combat plan, the win conditions on the scoreboard |

These texts never pass through `Localized.get_Render`, so a hook there does not change them: change the table entry instead (below).

### When the language can be read

Unity's `LocalizationSettings.SelectedLocale` is only ready once Unity's localization has finished starting. Reading it before that makes Unity wait for that start on the main
thread (`WaitForCompletion`). From code that runs inside the game's own loading step (while the game data is loaded at start) the wait does not end: the game stops responding
on a black screen before the intro. The same goes for anything that looks a text up (`Localized.Render`, `GetTable`).

Read the language after that step has returned (one frame later, or once the main menu is up), and follow changes with `LocalizationSettings.SelectedLocaleChanged`.

### The Ukrainian table

The game ships a Ukrainian locale (`uk`) and its table, but the language is not finished:

- 415 entries, of which 308 are the English text copied in and 15 the Russian text;
- the key `settings.values.Language.UKRAINIAN` exists, the `GameSettings.Language` enum has no value for it;
- a Ukrainian Steam client starts in Russian (see the startup selectors above).

It can be selected from code: `LocalizationSettings.SelectedLocale = LocalizationSettings.AvailableLocales.GetLocale("uk")`.

### Changing a text

```c#
var refs = StructuredTextStaticReferences.instance;
StringTable table = LocalizationSettings.StringDatabase.GetTable(refs.localizationStringTable, LocalizationSettings.SelectedLocale);
StringTableEntry entry = table.GetEntry("rules.title");
if (entry != null) entry.Value = "Rules";
else table.AddEntry("my.key", "My text");
```

**Logic.** Both routes read this table, so a changed entry shows in `Localized` texts and in the scene texts. A table belongs to one locale and is loaded when that locale is
selected, so do it again after every language change, and never inside the loading step (see above). Texts already on screen keep their old text until they are drawn again:
a `StructuredText` caches its render per locale, and a `GameObjectLocalizer` applies when its object is enabled. A window opened afterwards shows the new text.

Own texts can also be written as a `StructuredText`; icon tokens work without localization. The startup quirks can be fixed with a postfix on `SteamLocaleSelector.GetStartupLocale`.
