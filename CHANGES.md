# Remaining English UI fixes

## Baseline and analysis scope

- Upstream baseline: `xiaoye97/SunHavenLocalization` `master` at `349b8d6d090394d789178187066f009e25edcac2` (`1.3.8`).
- Screenshot input: `images/` contains 11 PNG files. The numbered sequence has no `04`; `速度药水描述.png` and `游戏加载.png` are additional captures. All 11 files are covered below.
- Localization data was read and written with the repository's `LocSheet` model and the same GZip + `BinaryFormatter` format used by `SunHavenLocalizationExcelTool`.
- Game-code evidence came from decompiling the publicly available 3.0.2 `SunHaven.Core.dll`, `SunHaven.External.dll`, and `SunHaven.Loading.dll`. Patches for game-owned methods are registered through `AccessTools` and skipped with a warning when a later game version no longer exposes the same method.

## Screenshot-by-screenshot changes

| Screenshot | English or broken text | xyloc result | Cause | Fix |
| --- | --- | --- | --- | --- |
| `01_痒痒挠任务Faeyon对话.png` | `Oh finally! I've been looking for Faeyon...`, `Thanks.`, `See ya!` | Existing keys: `Quest.YouScratchMyBack.DialogueOverride`, `Thanks`, `SeeYaExcited` | `Wish.QuestDialogueOverride.CheckQuestForDialogueOverride` builds a `DialogueNode` from the serialized `overrideText` and literal response strings without calling I2. | Patch `Wish.DialogueHelper.Replace` before the dialogue typewriter starts and resolve exact strings to their existing xyloc keys. |
| `02_理发店修理对话.png` | `I'm in need of a new makeover, but then again, so does this salon.` | No matching original or translation existed. Added `Hardcoded.RepairSign.BarberSalon`. | `Wish.RepairSign.Interact` calls `LocalizeText.TranslateText(textKey, text)`, but this asset text has no usable exported translation, so the English default is returned. | Add an internal xyloc key and map the exact fallback text to it in the dialogue patch. |
| `03_NelVari捐款提示.png` | `Donate`; donation question; `Donate 1/10/100 mana orb(s)`; `See ya!` | The donation prompt and options had no matching entries. Added `Hardcoded.DonationBox.Interaction`, `.Prompt`, and `.Option`; `SeeYaExcited` already existed. | `Wish.DonationBox.InteractionPoint`, `Interact`, `GetResponses`, and `RefreshDialogue` construct all strings directly. The donated amount and option amount are dynamic. | Add xyloc templates using `1111` as the numeric placeholder. A `Contains` prefilter gates two compiled regexes that capture the number and rebuild the localized prompt/option while retaining the orb sprite and disabled-option color tag. |
| `05_BUFF_未知UI截图1.png` | `Increases Win Area and Sweet Spot during fishing minigame by 99% for 1 hour.` | Existing key: `LegendaryFishBait.Buff.Description`. | `Wish.Food` copies `statBuff.description` directly into `StatBuff.buffDescription`; `Wish.StatBuff.StartBuff` then assigns it to `Popup.description` without using I2. | Patch `Wish.StatBuff.StartBuff` and `Wish.HourlyBuff.StartBuff` before the Popup fields are assigned, resolving known names/descriptions through xyloc. |
| `06_任务UI截图2.png` | `How do I do that?` | Existing key: `HowDoIDoThat`. | Several cutscenes create this response as a literal tuple instead of calling `LocalizeText`. | Resolve it in the `DialogueHelper.Replace` patch through the existing key. |
| `07_任务追踪图书馆Amanda.png` | `Return to Amanda at the library` | Existing key: `Quest.ReturnToAmanda`; its exported original uses uppercase `Library`. | Quest progress uses an asset key/default pair. When the key misses, the fallback differs from the xyloc original by case, so the exact original-text cache also misses. | Use a fast phrase prefilter and replace this fallback fragment with `Quest.ReturnToAmanda`, including when it is embedded in quest color markup. |
| `08_战斗UI怪物信息.png` | `叶包 Haloopbut`, `叶包 Squawkmouth Bass` | Existing rows `LeafWrappedHaloopbut.Name` and `LeafWrappedSquawkmouthBass.Name` contained incomplete Chinese translations. | This is a localization-data defect, not a missed Harmony path. The UI correctly displays the stored mixed-language value. | Update the rows to `叶包哈鲁普巴特` and `叶包嘎嘴鲈鱼`. |
| `09_房屋升级_UI截图3.png` | `You can't upgrade a house with animals inside!` | No exact entry existed. Added `Hardcoded.PlayerHouse.AnimalsInside`. | `Wish.PlayerHouse.Upgrade` passes the literal sentence to `Player.PausePlayerWithDialogue`. | Add an internal xyloc key and resolve the literal through the central dialogue patch. |
| `10_任务UI截图4.png` | `Don't mention it.` | Existing key: `DontMentionItPeriod`. | Cutscene response lists contain the sentence directly, bypassing I2 even though the same sentence exists in xyloc. | Resolve it in `DialogueHelper.Replace` through the existing key. |
| `速度药水描述.png` | Literal `<sprite="jump_height_icon" index=0>` shown in an otherwise Chinese tooltip | Existing keys: `SpeedPotion.HelpDescription` and the same pattern in `MovementSpeed.Buff.Help.02`. | The second sprite asset tag is not supported in this tooltip path, so TMP exposes the tag as text. | Keep the supported movement-speed sprite and replace the jump-height sprite with readable Chinese text in both xyloc rows. |
| `游戏加载.png` | `Loading Scenes...` | No `Loading Scenes` entry existed. Added `Hardcoded.LoadingScenes`; `LoadingWorld` and `LoadingWorldFromHost` already existed. | `Wish.ScenePortalManager.StartGameRoutine` assigns the literal `Loading Scenes`; `LoadTextRoutine` writes it to TMP and appends dots without I2. | Add an exact xyloc-key mapping handled by the existing TMP setter. The first assignment becomes Chinese, and subsequent appended dots remain localized. |

## Code and performance notes

- Exact hardcoded source-to-key matching uses `Dictionary<string, string>` and the success/failure log caches now use `HashSet<string>`.
- Dialogue, buff, TMP, and legacy `UI.Text` patches return immediately for unloaded localization, non-Chinese languages, empty strings, or text without a known marker.
- Dynamic donation regexes only run after case-sensitive `Contains` checks for the relevant phrases.
- Optional game-method patches log and skip missing types/methods instead of preventing the rest of the plugin from loading.

## Validation performed

- Deserialized the original and updated `zh-CN.xyloc`; dictionary count changed from 47,564 to 47,570 (six new internal keys).
- Re-read and checked all six new rows and four updated rows after serialization.
- Decompiled the referenced game assemblies and confirmed the target method names and field names used by the Harmony patches.
- Ran a Roslyn compiler pass; it reported the expected reference-framework incompatibility before emit and no C# syntax diagnostics. A full project build was intentionally not required, and the locally available public 3.0.2 Unity assemblies target `netstandard 2.1` while this repository project targets .NET Framework 4.7.2. Runtime verification in the current game remains required during manual review.

## Manual runtime checks

1. Start the latest game with Simplified Chinese selected and confirm the log contains successful patches for `Wish.DialogueHelper.Replace`, `Wish.StatBuff.StartBuff`, and `Wish.HourlyBuff.StartBuff` (or a clear version-mismatch warning).
2. Reproduce each screenshot state and confirm no target English remains.
3. Check donation options both enabled and disabled, and donate once to confirm the dynamic total refreshes in Chinese.
4. Check the fishing buff Popup, quest tracker, house-upgrade warning, speed-potion tooltip, and scene-loading animation for formatting regressions.
5. Switch to a non-Chinese language and confirm these hardcoded replacements are not applied.
