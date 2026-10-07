# QR Slate

iPhone web app that shows a QR slate (episode, scene, setup, take, usage, camera, notes) for VFX
photo and witness-camera slating on set, with a start-of-day page, slate history, corrections and
a day log.

Version **0.10.0**. The data inside the QR codes and the day log is described in
[`docs/qr-format.md`](docs/qr-format.md), so any sorting tool can read it.

## Install on the iPhone

The app is a plain static site, so any HTTPS host works. With GitHub Pages:

1. In this repo on GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, branch `main`, folder `/ (root)`.
2. After a minute the app is at `https://darabal.github.io/QR-Slate/`.
3. Open that link in **Safari** on the iPhone, tap **Share → Add to Home Screen**.
4. Open it once from the home screen while online. After that it works offline (flight mode, no signal on location).

> GitHub Pages on a **private** repo needs a paid GitHub plan (Pro, Team or Enterprise). On a free plan, either make the repo public or host the files elsewhere (Netlify, Cloudflare Pages).

**Versions:** `major.minor.patch`, shown on the start page and written into the day log (`app_version`). Still `0.x` until the app has been used on a real shooting day; then `1.0`. Small fixes bump the patch (0.8.1 → 0.8.2), new features the minor (0.8 → 0.9).

**Updating:** after changing any app file, set the same number in `APP_VERSION` (`index.html`) and `VERSION` (`sw.js`) so phones fetch the new version. The phone picks it up the next time the app is opened online.

## How it works on set

- **Start page:** shooting day / block (one free-text field, e.g. `D12_B3`), unit, and a message of the day. Shown when no day is open. On a new day the day/block field starts empty (the last value shows as a hint), the unit is kept, and the slate form starts empty. **Skip for now** starts without info; add it later from the Day log. The **Home** button (house icon, top left on every screen) comes back here mid-day; **Continue** returns to the form. A new day only starts after Wrap.
- **Form:** episode, scene, setup, take, usage (set_ref, hdri, photogrammetry + object, scene_ref, witness_cam, custom), camera name, and one note (300 characters; write per-take remarks in it, e.g. `T1: boom in shot · T2: good`). It fits on one screen.
- **Takes:** `witness_cam` counts takes from 1. Photo usages (set_ref, hdri, photogrammetry, scene_ref, custom) start without a take: one slate per folder. Take + still adds one if needed.
- **hdri:** no note, and the smallest possible QR (21×21): only the slate ID, e.g. `3-42A`. A second HDRI of the same setup: Take + gives `3-42A-1`, `3-42A-2`, and the sorter puts them in `hdri/3042A/1`, `hdri/3042A/2`. Day and unit come from the day log.
- **360:** on a witness_cam slate, tap **360** to switch to a tiny QR (like hdri, e.g. `3-21B-T4`) that 360 and fisheye cameras can read. It stays on until tapped again. Camera, note and time then come from the day log and the clip.
- **Sync:** on the slate screen, **Sync** shows a QR of the phone's clock that changes every frame (`SYNC-142605.120`) and the time in big digits. Film it for 2 seconds on each camera; the sorter uses it to line up camera timecode with the slate times. **Done** goes back to the slate.
- **Big QR:** tap the QR to fill the whole screen; tap again to go back.
- **False start:** the take was cut and rolls again on the same number. Tap **False start**: the slate stays on the same take and shows **ROLL 2** (QR `roll: 2`); film it at the start of the new roll.
- **New-day check:** if a day is still open, the date has changed **and** the last slate is more than 6 hours old, the app asks *New day* or *Same day*. New day wraps the old day and opens the start page with the day number +1 (same block, same unit).
- **Slate previous take:** if a take was shot without a slate, tap *Slate previous take*, pick the take with Take −/+, record a short clip of the blue slate, then **Done**. The sorter applies it to the clip before.
- **Slate:** big QR, big slate ID and usage, live date and time, and four buttons: setup −/+, take −/+. **New** goes back to the form for the next slate; **Slates** fixes past ones. The note shows under the slate with a pencil (or **+ note**); tap it to edit just the note. The rotate button (top left) turns the slate sideways inside the app. Setup runs `3_21 → 3_21A → … → 3_21Z → 3_21AA` (no I or O). Moving to a setup that already exists today goes back to it (its note and last take); only a name that doesn't exist yet makes a new slate, with an empty note and the take reset (1 for witness_cam, empty otherwise). Making a slate in the form with an existing name and a different note asks: **Go to existing**, **Overwrite** or **Cancel**.
- **Slates:** one row per slate with its takes as circles (dashed = slated after). The camera belongs to each take, so an empty or different camera name no longer splits a slate into two rows; **Show again** on a row lands on its last take. Select a slate or a take, then **Show again**, **Edit** or **Delete** (delete always asks to confirm).
  - Editing only the camera name or notes just saves. No new slate is needed, so you can slate with a general name like `witness` and fill in `blackmagic_A` later.
  - Editing episode, scene, setup, usage, object or take makes a **correction slate** (WAS → IS) with an orange border around the screen and a full-size QR. Photograph it; the sorter applies it.
- **Day log:** a read-only report of the day: totals and a compact list (to change a slate, open Slates). **Wrap** closes the day; **Send** shares the day log as `.json` (for the sorter) and `.txt` (for people) through the iPhone share sheet.

Everything is stored on the phone only (browser storage). A future version may sync day logs to a server.
