# JMT Tracker — complete setup guide

A public web page that shows your position, progress, daily stats, messages, and weather on the John Muir Trail, updated automatically whenever your ZOLEO sends a check-in.

How it flows: **ZOLEO check-in → email → Pipedream (free) parses it → writes to `checkins.json` in your GitHub repo → GitHub Pages serves the page.** Nothing runs on a server you maintain, every piece is free, and the whole thing keeps working unattended for the length of the hike.

## The files

- `index.html` — the tracker page. This is what the world sees.
- `checkins.json` — the data file the email parser appends to. Ships with demo entries placed on your route.
- `pipedream-parser.js` — code you paste into Pipedream (Part 3).
- `demo.html` — a self-contained preview with your actual route embedded. Open it anywhere to see how the page behaves; map tiles, elevation, and weather only load on the live site. **Do not upload this to the repo** — it's just for looking.
- You supply: `route.gpx` — your `JMT_TM.gpx`, renamed (Part 2 first).

Your route, as measured on the GPX: Tuolumne Meadows (Lyell Canyon) → Whitney Portal, **212.4 miles**, including the Onion Valley resupply out-and-back over Kearsarge Pass.

## Part 1 — Prepare the route file (do this first)

Your onX export has no elevation data in it, which the elevation profile, gain/loss stats, and altitude-correct weather all use. You have two options:

**Option A (recommended): bake elevation in once.**
1. Go to gpsvisualizer.com/elevation
2. Upload `JMT_TM.gpx`, output format **GPX**, and let it add DEM elevation data.
3. Download the result and rename it `route.gpx`.

This makes the page load instantly with no dependence on any outside service for elevation.

**Option B: do nothing.** Rename `JMT_TM.gpx` to `route.gpx` as-is. The page detects the missing elevations and fetches them automatically from Open-Meteo's free elevation service each time it loads (takes a second or two, invisible to viewers). If that service is ever unreachable, the elevation profile renders as a plain progress band and gain/loss figures hide rather than showing zeros — the rest of the page is unaffected.

Either way works; A is sturdier.

## Part 2 — The website (GitHub Pages)

1. Create a free account at github.com if you don't have one.
2. Create a new **public** repository (name it anything, e.g. `jmt-tracker`).
3. Upload `index.html`, `checkins.json`, and your `route.gpx` (Add file → Upload files).
4. In the repo: Settings → Pages → under "Branch" choose `main` and `/ (root)` → Save.
5. After a minute your site is live at `https://YOURNAME.github.io/jmt-tracker/`.

Open it now — you should see the topo map with your route, the waypoint markers, the elevation profile (if you did Option A, or once the auto-fetch completes), and the demo check-ins in Lyell Canyon and at Thousand Island Lake.

## Part 3 — The email parser (Pipedream)

1. Create a free account at pipedream.com.
2. New Workflow → trigger: **New Emails**. Pipedream shows you a unique address like `random-string@pipedream.email`. Copy it — this is the "contact" your ZOLEO will email.
3. Add a step → **Node.js → Run Node code** → paste in the entire contents of `pipedream-parser.js`.
4. Edit the two lines at the top of that code: `OWNER` (your GitHub username) and `REPO` (the repo name).
5. Make a GitHub token: github.com → Settings → Developer settings → Fine-grained personal access tokens → Generate new token. Scope it to **only this repository**, with Repository permissions → **Contents: Read and write**. Set the expiration past your finish date.
6. In Pipedream: Settings → Environment Variables → add `GITHUB_TOKEN` with that token as the value.
7. **Deploy** the workflow.

## Part 4 — ZOLEO

1. In the ZOLEO app, add the Pipedream email address as a contact.
2. Add it as a **check-in recipient** (Settings → Check-In) so pressing the check-in button on the device updates the site. Your human contacts stay on the list too — the parser address just rides along.
3. When you send a typed message from the trail and want it on the site, include the Pipedream contact as a recipient and make sure **location sharing is on for messages** — the parser needs the coordinates ZOLEO embeds as a map link.

## Part 5 — Test before you go (do not skip)

1. Send a real check-in from the device.
2. In Pipedream, open the workflow — the event should arrive and the code step should export a `written` result. If it exports `skipped: No coordinates found`, open the event, look at the email body, and check that location sharing was on.
3. Reload your site — the pin should appear within a couple of minutes (GitHub Pages caches briefly).
4. Send one **typed message** with a few words and confirm the text shows up cleanly in the message log. If ZOLEO boilerplate leaks into the message, tell me the offending line — the filter list in the parser takes one more pattern.
5. Once testing looks right, clear the demo data: edit `checkins.json` on GitHub and replace its entire contents with `[]`.

## Part 6 — Configure the page

At the top of the script in `index.html` is a CONFIG block. Before the hike:

- **`startDate`** — set to your permit date (`"YYYY-MM-DD"`) so "days on trail," pace, and the projected finish are right from day one. Left blank, counting starts at your first check-in.
- **`hikerName`** — optional; puts your name in the header.
- **`timeZone`** — leave as `America/Los_Angeles` for the JMT. Controls where "midnight" falls for the daily figures.
- **`waypoints`** — already measured on your GPX: Donohue 13.9, Thousand Island Lake 20.0, Red's Meadow 36.4, MTR 85.1, Muir Pass 106.2, Mather 131.0, Pinchot 137.7, Glen 153.8, Kearsarge junction 157.1, Onion Valley 163.7, Forester 181.1, Whitney summit 201.9, Portal 212.3. **One to verify: "VVR turnoff" at mi 66 is an estimate, not a measurement** — check it against your maps and adjust. Waypoints show as tappable markers on the map, hairlines on the elevation profile, and drive the "Next:" strip with distance and estimated days at your current pace.

## What the page shows and how the numbers work

**Progress** (% and miles) comes from snapping your check-ins to the route. Snapping is sequential — it knows you move forward — which matters on the Onion Valley out-and-back where the trail doubles back on itself: return-leg positions count forward correctly. Side trips more than 6 miles off-route neither advance nor roll back progress.

**Days on trail** counts calendar days since `startDate`, partial days included. Zero-days count — pace and the projected finish will dip after a rest day and recover as you hike.

**Daily stats** in the message log (each day's miles, gain/loss, hours) are estimates whose quality scales with your check-in habits. Miles are camp-to-camp along the route, so a single evening check-in still gets full credit measured from the previous night's position. Gain/loss reads off the route's elevation profile. Hours run from the day's first to last check-in — **check in when you break camp and again when you stop** if you want that column populated; a one-check-in day shows no hours. Hours include breaks; it's "hours out," not moving time — that's the best a few satellite points a day can honestly support.

**Weather** is a 4-day Open-Meteo forecast at your last check-in's coordinates and elevation — a camp at 10,400 ft gets a 10,400 ft forecast. No configuration or key needed.

**The map** uses OpenTopoMap terrain tiles and falls back to standard OpenStreetMap automatically if the topo server has an outage. The page follows each viewer's light/dark system setting, works on phones (most of your followers will open it from a text), and shows a "no recent check-in" note after 48 quiet hours phrased so nobody panics about a zero-day.

## Sharing

Your URL is `https://YOURNAME.github.io/REPONAME/` — text it, email it, group-chat it. No account or app needed on the viewer's end, the link never changes, and it unfurls with a title when pasted into messaging apps. Send it out once before you leave with a note that gaps of a day or two are normal. Optional: a URL shortener for something easier to say out loud, or a custom domain via repo Settings → Pages.

## Failsafe notice (for your home contact)

If the tracker misbehaves mid-hike, a designated contact can put a reassurance banner across the top of the site by email — no GitHub or code involved. They send an email to the Pipedream address with **`FAILSAFE ON`** as the subject (or the first line of the body), and the site displays **"MAP TEMPORARILY BROKEN BUT EVERYTHING IS OK"** prominently above everything. `FAILSAFE ON` followed by text shows that text instead (e.g. `FAILSAFE ON Site's acting up, he called from VVR and is fine`), and **`FAILSAFE OFF`** clears it.

Setup: in `pipedream-parser.js`, edit the `NOTICE_SENDERS` list to the email address(es) allowed to do this — commands from anyone else are ignored and treated as ordinary email. If you add `"zoleo"` to that list, you can toggle the notice yourself from the trail by starting a satellite message with `FAILSAFE ON`. The command must start the subject or a line, so a message merely mentioning the word won't trigger it. Have your contact test it once before the hike, same session as Part 5.

## Deleting or fixing a check-in

Every entry lives in `checkins.json`, so removing one is a 30-second edit: open the repo on github.com (works from a phone browser), click `checkins.json`, click the pencil icon, delete the entry's `{ ... }` block — watch the commas so the JSON stays valid — and commit. The page reflects it within a minute or two. The same edit fixes a wrong coordinate or a typo. A home contact with access (repo → Settings → Collaborators) can do this for you mid-hike.

## Previewing vs. live

Opening `index.html` by itself — locally or in a sandboxed preview — can't reach `route.gpx`, `checkins.json`, map tiles, elevation, or weather, so you'll see sample data with a banner and a blank map background. That's expected; use `demo.html` for offline looking (it has your route embedded — vector route, waypoints, and pins render; the tile background still needs the live site). The real data and full map only exist at the github.io address.

## Privacy

The page is public: anyone with the link sees your near-real-time position. Be deliberate about who gets it. If you'd rather not broadcast your campsite each night, send your evening check-in the next morning — or ask and I'll add a publish delay to the parser so positions post N hours late.

## Costs

$0 beyond your existing ZOLEO plan. GitHub Pages, Pipedream's free tier, OpenTopoMap tiles, and Open-Meteo weather/elevation are all free at this usage level, with plenty of headroom.

## If something breaks mid-hike

Check Pipedream first (did the email arrive? what did the step export?), then the repo's commit history (is `checkins.json` updating?). Worst case, your home contact keeps the site alive manually: edit `checkins.json` on GitHub and paste in a new entry following the pattern of existing ones — time in UTC, lat, lon, text. The page needs nothing else to keep working.
