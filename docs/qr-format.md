# QR Slate data format (v1)

This is the contract between the **QR Slate** app and the **sorter**. Both sides must follow it.

## 1. Slate QR

A QR code. Two formats exist:

- **JSON** (all usages except hdri, and every correction / previous-take slate): one JSON object, UTF-8, error correction level Q, described below.
- **Short text** (hdri, see 1b, witness_cam with the app's **360** switch on, see 1c, and the sync clock, see 1d): the slate ID only, so the code is the smallest QR there is and fisheye / 360 lenses can still read it.

The sorter should use a robust QR decoder (e.g. ZXing / OpenCV's `QRCodeDetectorAruco`); the basic OpenCV detector misses dense codes that stronger decoders read fine.

| Key | Type | Always | Meaning |
|---|---|---|---|
| `app` | `"qrslate"` | yes | Marks this QR as ours. Ignore QR codes without it. |
| `v` | `1` | yes | Format version. |
| `episode` | string | yes | e.g. `"3"` |
| `scene` | string | yes | e.g. `"21"` (upper-case if it has letters) |
| `setup` | string | no | `""`, `"A"` … `"Z"`, `"AA"` … Movie alphabet: no `I`, no `O`. Omitted for `scene_ref`. |
| `take` | integer | no | `witness_cam` counts takes from 1. For every other usage the take is optional: **no `take` = one slate for the whole folder/setup**. Also omitted on a whole-slate correction. |
| `usage` | string | yes | `set_ref`, `hdri`, `photogrammetry`, `scene_ref`, `witness_cam`, or a custom value. Lower-case, `a-z0-9_`. |
| `object` | string | no | Only with `photogrammetry`: the scanned object, e.g. `hero_chair`. |
| `camera` | string | no | Camera name, e.g. `witness`, `blackmagic_A`, `insta360_B`. `A-Za-z0-9_-`. Often a general name on set, fixed later in the day log. |
| `day` | string | no | Shooting day / block as typed on the start page, e.g. `D12_B3`. Free text; only the sorter interprets it. |
| `unit` | string | no | `main`, `second`, `splinter`, `vfx` or a custom value. |
| `id` | string | yes | Readable slate ID: `episode_scene+setup`, e.g. `3_21B`. |
| `time` | string | yes | ISO 8601 local time with offset when the slate was shown, e.g. `2026-09-29T11:42:10+02:00`. |
| `note` | string | no | Slate note, max 300 characters. Carries over through the takes; per-take remarks are written inside it, e.g. `T1: boom in shot · T2: good`. (v5 also had `take_note`; old QR codes may still carry it.) |
| `correction` | object | no | Present only on a correction slate. See below. |
| `roll` | integer | no | **False start:** the take was cut and rolled again. `roll: 2` (or 3…) = this is a re-roll of the same take; the last roll is the real take. Absent = roll 1. |
| `previous_clip` | `true` | no | **Slate-only clip:** the clip that contains this QR is only a slate. It labels the clip **before** it with this slate and take. See below. |

Example:

```json
{"app":"qrslate","v":1,"episode":"3","scene":"21","setup":"B","take":2,"usage":"set_ref","camera":"witness","id":"3_21B","time":"2026-09-29T11:42:10+02:00","note":"Rain rig on"}
```

### 1b. Short hdri QR

```
3-21B        one HDRI for setup 3_21B
3-21B-2      HDRI number 2 of that setup (the slate's take)
```

`episode-scene+setup[-number]`, upper case, only `0-9`, `A-Z` and `-`. The app draws it in the QR
alphanumeric mode with error correction level H: up to 10 characters is a 21×21 QR (version 1),
the smallest there is. Readers should also accept `_` instead of `-` and lower case.

- Usage is always `hdri`. Day, unit, camera and time are not in the QR: the sorter takes day and
  unit from the day log (or from the other slates of the day) and the time from the photo.
- The number is the slate's take. No number = one HDRI for the setup; with numbers the sorter
  writes `hdri/<setup>/1`, `hdri/<setup>/2`.
- HDRI slates carry no note. Correction, previous-take and re-roll slates of an hdri stay JSON.
- QR Slate 0.9.1 wrote `QS1|3_21B|<take>|hdri|<yymmddhhmmss>`; the sorter still reads it.

### 1c. Short witness_cam QR (360 switch)

```
3-21B-T4      witness_cam, slate 3_21B, take 4
3-21B-T4R2    take 4, roll 2 (false start, see `roll`)
3-21B-T4P     previous-take slate for take 4 (see `previous_clip`)
```

`episode-scene+setup-T<take>[R<roll>|P]`, same character set, QR mode and error correction as 1b.
Up to 10 characters is a 21×21 QR; longer IDs (e.g. `12-105AA-T12`) give 25×25.

- Shown only when the operator turns on **360** on a witness_cam slate, so 360 / fisheye witness
  cameras can read it. Every camera rolling on that take sees the same short code.
- The `T` tells it apart from an hdri code, whose third part is a plain number.
- Usage is always `witness_cam`. Camera, note, day, unit and time are not in the QR: camera from the
  offload folder (or the day log), day and unit from the day log, time from the clip.
- Correction slates stay JSON.

### 1d. Sync QR (live clock)

```
SYNC-142605.120     the phone's local clock: 14:26:05.120
```

`SYNC-HHMMSS.mmm`, local time of day (same clock and time zone as the slate `time` values), alphanumeric
mode, level H, 25×25. The app redraws it on every screen frame (about 60 times a second) while **Sync**
is open on the slate screen.

- Film it for a couple of seconds on each camera, any time in the day (ideally at the start, after
  any timecode jam). One decoded frame pairs that clip's timecode (frame time) with the phone clock:
  `offset = phone time − clip TC`. Use the median over the frames that decode.
- With that offset, every slate `time` in the day log maps to a timecode on that camera, so takes
  can be found from timecode alone (cameras whose QR can't be read in the footage, e.g. some RAW formats).
- No date in the code: take it from the clip. A frame can land between two codes and fail to decode; skip it.

### Previous-take slates

Used when a take was shot without a slate. The operator records a separate short clip showing this slate (blue on the phone). `previous_clip: true` means:

- the clip with this QR is **only a slate** → `_SLATES`;
- the clip **directly before it** (same camera, by time / clip number) gets this slate's ID and `take`.

The day log marks such takes with `"slated_after": true`.

### Correction slates

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

## 2. Day log

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
        {"take": 4, "time": "2026-09-29T11:58:40+02:00", "rolls": 2}       // false start: 2 clips for T4, the last is real
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
