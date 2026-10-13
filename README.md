# QR Slate

iPhone web app that shows a QR slate (block, day, unit, episode, scene, setup, take, usage, notes)
for VFX photo and witness-camera slating on set. Two screens: the form that makes the slate, and the slate
that shows the QR.

Version **1.0.0**. The data inside the QR codes is described in
[`docs/qr-format.md`](docs/qr-format.md), so any sorting tool can read it.

## Install on the iPhone

The app is a plain static site, so any HTTPS host works. With GitHub Pages:

1. In this repo on GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, branch `main`, folder `/ (root)`.
2. After a minute the app is at `https://darabal.github.io/QR-Slate/`.
3. Open that link in **Safari** on the iPhone, tap **Share → Add to Home Screen**.
4. Open it once from the home screen while online. After that it works offline (flight mode, no signal on location).

> GitHub Pages on a **private** repo needs a paid GitHub plan (Pro, Team or Enterprise). On a free plan, either make the repo public or host the files elsewhere (Netlify, Cloudflare Pages).

**Versions:** `major.minor.patch`, shown on the form. Still `0.x` until the app has been used on a real shooting day; then `1.0`. Small fixes bump the patch (0.8.1 → 0.8.2), new features the minor (0.8 → 0.9).

**Updating:** after changing any app file, set the same number in `APP_VERSION` (`index.html`) and `VERSION` (`sw.js`) so phones fetch the new version. The phone picks it up the next time the app is opened online.

## How it works on set

- **Form:** shooting day (number), block (0–9) and unit at the top; they stay until you c## How it works on set

- **Form:** block and day typed as one code (`B01D56`; `B1D56`, `D56` and `B02` work too) and unit at the top; they stay until you change them. Unit: main, second, splinter, VFX or other. The slate as it will read (e.g. `3_A38A T4`) shows next to the Slate heading. Then episode (optional), scene (tap **Add marker** to add `X` second unit, `A`/`B`/`C` connected scene, `XA`… in front; the same button then reads **Remove marker**), setup dropdown (A … Z, AA … CZ, no I or O), take (photogrammetry has no take: the field and the slate's Take buttons are locked) and usage (set_ref, hdri, photogrammetry + object, witness, custom).
- **Notes:** only set_ref, photogrammetry and custom take a note. **+ Add note** first warns that a note makes the QR denser (set_ref switches from the small code to the big one), then opens the note box (300 characters).
- **Takes:** witness counts takes from 1. Photo usages start without a take: one slate per folder. Take + still adds one if needed.
- **Slate code:** hdri, witness and set_ref without a note show a 17-digit code in the smallest QR there is (21×21), so 360, fisheye and far cameras read it: episode, scene marker, scene, setup, take, date, unit and a flag (normal, previous take, rehearsal, roll again). Values that don't fit fall back to the full QR.
- **Rehearsal:** on a witness slate, tap **Rehearsal** to mark the next clip of this take as a rehearsal (flag 2, **REH** under the take). It turns off on the next take or setup, or tap it again.
- **Sync:** **Sync** at the top of the form or the slate screen shows a QR of the phone's clock that changes once a second, on the second (`SYNC-142605.000`), and the time in big digits. Film it for a few seconds on each camera; the sorter uses it to line up camera timecode with the slate times. **Done** goes back to where you opened it from. **Done** goes back to the slate.
- **Big QR:** tap the QR to fill the whole screen; tap again to go back.
- **Roll again:** the take was cut and rolls again on the same number. Tap **Roll again**: the slate stays on the same take and shows **ROLL 2** (flag 3); film it at the start of the new roll.
- **Slate previous take:** if a take was shot without a slate, tap *Slate previous take* (it slates the current take), record a short clip of it, then **Done**, which moves on to the next take. The sorter applies it to the clip before (flag 1).
- **Slate:** big QR across the top, big slate ID and usage, the shooting day (`B01D56 · Main unit`), a big clock with the date, and four buttons: setup −/+, take −/+. **Slate** goes back to the form. The note shows under the slate with a pencil (or **+ note**). The rotate button (top left) turns the slate sideways inside the app. Moving to a setup that already exists on the same shooting day goes back to it (its note and last take); only a name that doesn't exist yet makes a new slate, with an empty note and the take reset. Making a slate in the form with an existing name and a different note asks: **Go to existing**, **Overwrite** or **Cancel**.
- **Slates list:** hidden for now while it is reworked. Every slate is still saved on the phone.

codes carry everything the sorter reads.
