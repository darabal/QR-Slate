# QR Slate data format (v1)

This is the contract between the **QR Slate** app and the **sorter**. Both sides must follow it.

## 1. Slate QR

A QR code. Two formats exist:

- **Slate code** (17 digits, see 1c): hdri and witness_cam always, set_ref while it has no note.
  The smallest QR there is, so fisheye / 360 lenses and far cameras can still read it.
- **JSON** (photogrammetry, custom usages, set_ref with a note, correction slates from versions before 1.0.1, and any slate whose
  values don't fit the slate code): one JSON object, UTF-8, error correction level Q, described below.
- The **sync clock** (see 1d) is its own short code.

The sorter should use a robust QR decoder (e.g. ZXing / OpenCV's `QRCodeDetectorAruco`); the basic OpenCV detector misses dense codes that stronger decoders read fine.

| Key | Type | Always | Meaning |
|---|---|---|---|
| `app` | `"qrslate"` | yes | Marks this QR as ours. Ignore QR codes without it. |
| `v` | `1` | yes | Format version. |
| `episode` | string | yes | e.g. `"3"`. May be `""` since 1.0 (episode is optional). |
| `scene` | string | yes | e.g. `"21"`. Since 1.0 it may start with a scene marker: `X` (second unit), `A`/`B`/`C` (a connected scene) or `XA`/`XB`/`XC`, e.g. `"A38"`. |
| `setup` | string | no | `""`, `"A"` … `"Z"`, `"AA"` … Movie alphabet: no `I`, no `O`. Omitted for `scene_ref`. |
| `take` | integer | no | `witness_cam` counts takes from 1. For every other usage the take is optional: **no `take` = one slate for the whole folder/setup**. Also omitted on a whole-slate correction. |
| `usage` | string | yes | `set_ref`, `hdri`, `photogrammetry`, `witness_cam`, or a custom value. Lower-case, `a-z0-9_`. (`scene_ref` left the app in 1.0; old slates may still carry it.) |
| `object` | string | no | Only with `photogrammetry`: the scanned object, e.g. `hero_chair`. |
| `camera` | string | no | Camera name. The form no longer asks for it since 1.0; older slates may carry it. |
| `day` | string | no | Shooting day as one code since 1.0: `B01D56` = block 1, day 56. Either part may be missing (`D56`, `B02`). 1.0 test builds wrote a plain number and a separate `block`; before that free text like `D12_B3`. |
| `unit` | string | no | `main`, `second`, `splinter`, `vfx`, `other` or a custom value. |
| `id` | string | yes | Readable slate ID: `episode_scene+setup`, e.g. `3_21B`; without an episode just `A38A`. |
| `time` | string | yes | ISO 8601 local time with offset when the slate was shown, e.g. `2026-09-29T11:42:10+02:00`. |
| `note` | string | no | Slate note, max 300 characters. Only set_ref, photogrammetry and custom usages take a note. |
| `correction` | object | no | Present only on a correction slate. See below. |
| `roll` | integer | no | **Roll again** (was "False start" in the app): the take was cut and rolled again. `roll: 2` (or 3…) = this is a re-roll of the same take; the last roll is the real take. Absent = roll 1. |
| `rehearsal` | `true` | no | The clip is a rehearsal of that take, not the take. |
| `previous_clip` | `true` | no | **Slate-only clip:** the clip that contains this QR is only a slate. It labels the clip **before** it with this slate and take. See below. |

Example:

```json
{"app":"qrslate","v":1,"episode":"3","scene":"A38","setup":"B","take":2,"usage":"set_ref","day":"B01D56","unit":"main","id":"3_A38B","time":"2026-10-13T11:42:10+02:00","note":"Rain rig on"}
```

### 1b. Short hdri QR (until 0.11)

`3-21B` / `3-21B-2` (`episode-scene+setup[-number]`). Since 1.0 hdri slates use the slate code (1c).
Readers should still accept these, and the 0.9.1 form `QS1|3_21B|<take>|hdri|<yymmddhhmmss>`.

### 1c. Slate code (17 digits)

```
03203801042628610
```

**17 digits** in fixed-width fields, no separators. QR numeric mode, error correction level H:
always a 21×21 QR (version 1, which holds exactly 17 digits).

| Digits | Field | Values |
|---|---|---|
| 1–2 | `EE` episode | `00` none, `01`–`99` |
| 3 | `M` scene marker | `0` none, `1` X, `2` A, `3` B, `4` C, `5` XA, `6` XB, `7` XC (`8`, `9` reserved) |
| 4–6 | `SSS` scene number | `000`–`999` |
| 7–8 | `UU` setup | `00` none, `01` A … `24` Z, `25` AA … `96` CZ (movie alphabet, no I or O) |
| 9–10 | `TT` take | `00` no take, `01`–`99` |
| 11–15 | `YYDDD` date | year (2 digits) and day of the year `001`–`366`, phone date when the slate was shown |
| 16 | `N` unit | `0` not set, `1` main, `2` second, `3` splinter, `4` vfx, `5` other |
| 17 | `F` flag | `0` normal, `1` previous-take slate (see `previous_clip`), `2` rehearsal, `3` roll again (the same take rolled again after a cut; this clip is the real take), `4`–`9` reserved |

Example above: episode 3, scene A38, setup A, take 4, 13 Oct 2026 (day 286), main unit, normal → slate `3_A38A` take 4.
The scene marker is part of the scene: `A38` is the scene the sorter files as `1038A_A` for episode 1, setup A.

- The usage is not in the code: hdri, witness_cam and set_ref without a note all show it. Take the usage
  from the offload folder / camera. Block and shooting-day number aren't in it either; the date is.
- If a value doesn't fit (episode with letters or over 99, a scene in another shape, setup past CZ,
  take over 99), that slate shows the JSON QR instead.
- Older digit codes are **witness_cam only** and readers should still accept them:
  1.0 test builds wrote `DDD B N EE SSS s UU TT R F` (17 digits, starts with the day) and 0.11 wrote 15 digits
  (`DDD EE SSS s UU TT R F`); 0.10 had `3-21B-T4` / `3-21B-T4R2` / `3-21B-T4P` codes.
  Tell them apart by the QR Slate version of the day, or by the date field: a slate code always has a valid
  `YYDDD` at digits 11–15 close to the clip date.

### 1d. Sync QR (live clock)

```
SYNC-142605.000     the phone's local clock: 14:26:05
```

`SYNC-HHMMSS.000`, local time of day (same clock and time zone as the slate `time` values), alphanumeric
mode, level H, 25×25. While **Sync** is open on the slate screen the code changes **once a second, exactly
on the second** (the `.000` is always zero; 0.10.0 changed it every screen frame with real milliseconds,
which camera shutters could not catch).

- Film it for a few seconds on each camera, any time in the day (ideally at the start, after any
  timecode jam). The **first frame** showing a new code is that whole second on the phone clock:
  `offset = phone time − clip TC of that frame`, accurate to about one frame. Frames in the middle of a
  second only say "within this second"; use a change frame when there is one.
- With that offset, every slate `time` (or a QR's place in the footage) maps to a timecode on that camera, so takes
  can be found from timecode alone (cameras whose QR can't be read in the footage, e.g. some RAW formats).
- No date in the code: take it from the clip. A frame caught mid-change can fail to decode; skip it.

### Previous-take slates

Used when a take was shot without a slate. The operator records a separate short clip showing this slate (blue on the phone). `previous_clip: true` means:

- the clip with this QR is **only a slate** → `_SLATES`;
- the clip **directly before it** (same camera, by time / clip number) gets this slate's ID and `take`.

In the slate code it is flag `1`. Up to 0.11 the day log marked such takes with `"slated_after": true`.

### Correction slates (up to 1.0.0)

QR Slate 1.0.1 no longer makes correction slates (they came from the slate list, which was removed). Readers may still find them in footage slated with older versions.

A correction slate says: *everything that was slated as `was` is really what this slate says.*

```json
"correction": {
  "scope": "take",            // "take": one take, "slate": every take of that slate
  "was": {
    "id": "3_21A", "episode": "3", "scene": "21", "setup": "A",
    "usage": "set_ref", "object": "...", "camera": "witness",
    "take": 2,                // only when scope is "take"
    "time": "2026-09-29T11:40:02+02:00"   // time of the wrong slate (scope "slate": its first take)
  }
}
```

- `scope: "slate"`: relabel every take of the `was` slate. Take numbers stay the same. The correction QR has no `take`.
- `scope: "take"`: relabel only take `was.take`; its new take number is `take`.
- Match the wrong slate by `was.id` + `was.usage` (+ `object`, `camera`), and use `was.time` to pick the right one if the same ID was slated more than once.
- The correction slate itself is not a take. The sorter puts its files in `_SLATES`.
- Priority when slates disagree: **correction slate (shot later) > tail slate > head slate**.

## 2. Day log (removed in 1.0)

QR Slate 1.0 has no day log: everything the sorter reads is in the QR codes. Older versions shared one at wrap, described below, and readers may still get those files.

Shared at wrap as `daylog_<date>_wrap-<HHMM>.json` (and a readable `.txt`). This is the phone's final, corrected view of the day. Edits in Slates that don't need a new slate (camera name, notes) appear only here.

Since 0.9 a slate is identified by episode + scene + setup + usage (+ object). The slate's `camera` is the default for its takes; a take with its own `camera` overrides it (e.g. takes shot on another camera).

```json
{
  "app": "qrslate-daylog", "v": 1, "app_version": "0.9.1",
  "day": "D12_B3", "unit": "main",
  "start": "2026-09-29T07:58:12+02:00",
  "wrap":  "2026-09-29T19:31:40+02:00",
  "slates": [
    {
      "id": "3_21B", "episode": "3", "scene": "21", "setup": "B",
      "usage": "witness_cam", "camera": "blackmagic_A",
      "note": "Rain rig on",
      "was": "3_21A witness_cam · witness",        // present if this slate was corrected
      "takes": [
        {"take": 1, "time": "2026-09-29T11:42:10+02:00"},          // photo slates without a take: {"time": ...} only
        {"take": 2, "time": "2026-09-29T11:47:31+02:00", "note": "Late start", "was": "3_21A T2 set_ref"},
        {"take": 3, "time": "2026-09-29T11:52:02+02:00", "camera": "insta360_B", "slated_after": true},
        {"take": 4, "time": "2026-09-29T11:58:40+02:00", "rolls": 2},      // false start: 2 clips for T4, the last is real
        {"take": 5, "time": "2026-09-29T12:03:12+02:00", "rehearsals": 1}  // 1 rehearsal clip before T5 (witness flag 2)
      ]
    }
  ]
}
```

## 3. What the sorter does with it (plan)

- **Camera = offload folder.** Each card is offloaded into its own folder (e.g. `insta_B/`); that folder is the main source for the camera. A `camera` in the QR or day log is only extra information; if they disagree, the folder wins.
- **Cameras are lined up by time.** Witness cameras roll together, so clips are matched across cameras by time (after measuring each camera's clock offset from its QR shots). A camera that didn't roll simply has no clip for that take. An extra clip on one camera between two slates becomes `T3_extra` instead of shifting the numbering.
- **False starts:** for a take with `rolls` > 1 (or QR `roll`), the last clip is the real take; earlier clips become `T3_false1`, `T3_false2`… in the same folder and are listed in the report.
- Folder names include the camera name; file names are scene + take (exact pattern still to be decided).
- Notes: the slate note (and any old take notes) are written as a `.txt` next to the files of that slate.
- Double check after sorting, against the day log:
  - **Match**: in the log and found on the card.
  - **Missing**: in the log, not found on the card.
  - **Unknown**: found on the card, not in the log.
  - **Mismatch**: same slate, but the time differs a lot (check the camera clock).
- `_UNSORTED` is created only when a file can't be placed.
