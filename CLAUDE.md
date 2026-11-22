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

- **Never write any PIN into code, commits, or chat.** The hidden Admin login stores only `ADMIN_HASH`. PINs are checked in browser code, so they're casual protection only.
- **Never auto-fill acid from the ticket.** The acid tower runs short or over, but water is almost always exact. Water = ticket gallons × the water ratio; acid = scaled gallons − ticket water. CableShield chemicals use their exact ticket amounts and come off first. With no ticket gallons, fall back to the ratio split.
- The menu Load Weight page reuses `finSplit` / `csChems` / `ticketWater` from the end-of-load window, so changes to that math affect both.
- The Help page's pictures (`figure.hx`) are frozen copies of real app markup. When a layout they copy changes, update the matching picture too.
- Style: professional, small corner radii, thin borders, logo colors (blue `#315a8f` / `#2c6384`, orange `#eca03f`). There are Light and Dark themes plus an Auto/Light/Dark switch, so check both.
- Keep suggestions short. The lead usually picks the recommended option and often skips long idea lists.

## Data (Firebase Realtime Database)

Path: `crews/<prefix+PIN>/orders/<id>`. Fixed keys live inside `orders` and have no `prod`, so the board ignores them:

- `_roster`: the crew list.
- `_chat`: Crew Chat. Messages expire after 24 h.
- `_sched`: Schedule. `{hrs:{d,n}, w:{name:"DNO×7, Sun first"}, ov:{name:{on,u}}}`. Day shift starts at 6 AM and night shift at 6 PM. A night shift counts toward the day it started.
- `_handoff`: Shift Handoff. `{d: draft, p: {id: post}}`. Leads edit; posts are kept 7 days.

Changes to `_chat` and `_handoff` skip the board rebuild. `update` with `null` deletes a key.
