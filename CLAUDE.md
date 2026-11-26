# Tarquin Coolpro Loader Hub

A single-file PWA used by the loader crew at Tarquin Coolpro. GitHub Pages serves `main`, so **every push to main goes live on every crew phone right away.**

Files: `index.html` (the whole app: HTML, CSS, one global script), `config.js`, `sw.js` (service worker), `manifest.webmanifest`, and the icons `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`.

## Release process (every change to the app)

1. Work directly on `main`. Don't use branches.
2. Bump `const V="tclo-vNN"` in `sw.js` by one. Each release gets exactly one new number.
3. Check for duplicate function names. Everything is in one global script, so a reused name silently replaces the earlier function. (A second `hm()` in v37 broke the time chips until v41.)
   ```
   grep -o 'function [A-Za-z_$0-9]*(' index.html | sort | uniq -d
   ```
   This must print nothing.
4. Test offline (see below), then show the lead what changed and **ask before committing and pushing.**
5. Commit with a message like `vNN: short description`, then push. Retry if the push hits a connection reset.
6. Poll the live service worker until it shows the new version:
   ```
   curl -s "https://cam2024official-dotcom.github.io/load-log/sw.js?x=$RANDOM" | grep -o 'tclo-v[0-9]*'
   ```
7. Remind the lead to close and reopen the app on each phone.

Changes that aren't app files (like this one) don't need a version bump.

## Testing: never touch the live Firebase

Copy `index.html` and `config.js` into a temp folder. Write a `stub.js` that sets `window.__DL` to a fake Firebase (setCode, exists, createMeta, subscribe, onConn, newId, set, update, remove) that keeps its data in sessionStorage, and put a crew PIN in localStorage. Load it after config.js:

```
sed 's|<script src="config.js"></script>|<script src="config.js"></script><script src="stub.js"></script>|' index.html > tmp/index.html
```

Serve the folder on localhost and drive it in a browser: a PowerShell HttpListener on Windows, or `python3 -m http.server` on Linux.

## Rules

- **Never write any PIN into code, commits, or chat.** The hidden Admin login stores only `ADMIN_HASH`, and the backup lead PIN only `LEAD_HASH` (`pinHash("Backup lead", pin)`, v79). PINs are checked in browser code, so they're casual protection only.
- **Never auto-fill acid from the ticket.** The acid tower runs short or over, but water is almost always exact. Water = ticket gallons × the water ratio; acid = scaled gallons − ticket water. CableShield chemicals use their exact ticket amounts and come off first. With no ticket gallons, fall back to the ratio split.
- The menu Load Weight page reuses `finSplit` / `csChems` / `ticketWater` from the end-of-load window, so changes to that math affect both.
- The Help page's pictures (`figure.hx`) are frozen copies of real app markup. When a layout they copy changes, update the matching picture too.
- Style (industrial look since v64): Barlow Condensed for headings, titles, buttons and big numbers (`--display`, often uppercase with letter-spacing), Barlow for body text (`--body`). Charcoal header (`--head`) with a solid safety-orange stripe, steel-gray backgrounds, white cards (near-black in Dark), safety-orange buttons and highlights (`--acc`, `--brand`). Small corner radii, thin borders. The look lives in one block at the end of the `<style>` ("Industrial look (v64)"); change colors through its tokens, not one-off hex values. Status tags keep their own colors: NM yellow, Third Party purple, CS red, HAL orange, Management blue `#315a8f`. Printed sheets keep their paper-form colors. There are Light and Dark themes plus an Auto/Light/Dark switch, so check both.
- **Demo Board** (menu, Christian and Admin only) swaps `DL` for an in-memory `DEMO_DL` and holds back live data (`LAST_REAL`) until Exit demo. Any new code that writes must go through `DL`, never a saved copy of the real database, or the demo could write to live Firebase.
- Keep suggestions short. The lead usually picks the recommended option and often skips long idea lists.

## Data (Firebase Realtime Database)

Path: `crews/<prefix+PIN>/orders/<id>`. Fixed keys live inside `orders` and have no `prod`, so the board ignores them:

- `_roster`: the crew list. `{names, colors, pins, leads, perms}` keyed `n0`, `n1`… in name order. `perms` holds what a lead granted a crew member (`add`, `rm`, `ho`, `sched`, `rep`, `paper`); check them with `perm(k)`, which is always true for leads, managers and Admin. `saveRoster` rebuilds the whole object, so any new roster field must be added there too.
- `_chat`: Crew Chat. Messages expire after 24 h.
- `_sched`: Schedule. `{hrs:{d,n}, w:{name:"DNO×7, Sun first"}, ov:{name:{on,u}}}`. Day shift starts at 6 AM and night shift at 6 PM. A night shift counts toward the day it started.
- `_handoff`: Shift Handoff, laid out as the Loader Shift Turnover sheet. `{d: draft, p: {id: post}}`. A draft with `v:2` is the turnover sheet (fields listed in the comment above `HO_ID` in index.html); posts without `v` use the old layout. Posting signs as outgoing (`sg.o`); the next lead signs as incoming (`sg.i`). Leads edit; posts are kept 7 days.
- `_recipes`: saved Custom blend recipes. `{id: {n: name, b: base product, f: base share of the total}}`. A custom blend order stores `prod` as `x:<base>:<share>` (e.g. `x:cs100:0.25`) plus `cname`, so every calculation scales from the key.

- `_mgmt`: Management logins. `{mID: {n: name, p: pin hash}}`. Managers sign in under Management, get full lead rights (`ME.mgr`, `ME.lead`), and are kept off the crew board, schedule and loader lists; starting a load doesn't make them the loader.
- `_hist`: History. `{orderId: <the finished order> + clearedAt, clearedBy}`, written by Clear finished orders (instead of deleting) and kept 7 days; the first phone that can write removes older entries. The Shift Report reads it through `allDone()`. Remove on a load still deletes outright.
- `_presence`: who's online. `{deviceId: {n: name, v: page, t: ms}}`, written by `DL.presence` with a Firebase onDisconnect remove; the phone also removes it when the app goes to the background, and entries older than 3 minutes are ignored.
- `_daily`: Daily Checklist & Assignment (v86, under Lead, `perm("dc")`). `{eq:{eID:{n,t,os}}, pt:{<skey type>:"Tires|Lights|..."}, it:{iID:{n}}, s:{<shift start ms>:{c:{key:{n,t}}, sp:{spID:{n,by,t,dn}}, as:{<skey name>:{n,e,by,t}}, in:{eID:{n,t,ok,p:{i0:{n,ok}},nt}}}}}`. Equipment, inspection points per type (defaults in `DC_PTS`), the lead's own checklist items, and per shift: items checked, special tasks, one truck per loader, inspections. Writes use slash paths (`s/<start>/in/<eID>`) so phones don't overwrite each other; shifts are kept 7 days. Turnover checks itself from the incoming signature, Equipment Inspection when every assigned truck is inspected. Loaders inspect from the strip on the Load Log (`myEq`, `inspOpen`).
- `_rail`: Railcar timers (v88), on the Load Log under the summary tiles. `{c:{rID:{car, tk ("East"|"West"), by, at (start ms), min (minutes from start to done), done:{t,by,e}}}}`. A countdown that ends at `at + min`; `e` true means the car was marked empty, false means stopped, not empty. Finished cars are kept 7 days for the Shift Handoff's Fill from Load Log (South rail car status, Rail cars moved to empty line). Writes use slash paths (`c/<rID>/min`).

Pausing a load (v85) is saved on the order itself: `pauseAt` (start of the pause going now; null while loading), `pauseWhy`, `pauseBy`, `pausedMs` (total of finished pauses) and `pz` (`{pKEY:{a,b,w,by}}`, finished pauses; missing when empty). A paused load still has `status()` "loading"; check `isPz(o)`. Use `loadMs(o)` for load time (paused time left out) and `wEnd(o)` for when the water timer ends (it freezes while paused). Ending a paused load closes the pause through `pzClose`.

Changes to `_chat`, `_handoff`, `_presence` and `_hist` skip the board rebuild. `update` with `null` deletes a key.
