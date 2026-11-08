# HBountyTracker

Finds Torn bounty targets you can hit right now. One static page, no build step, no server. Everything runs in the visitor's browser on their own API key.

A freebie by [The Great Stag](https://www.thegreatstag.com). Open source, so anyone can check exactly what happens to their key.

## What it does

1. **Bounty list.** Pulls the whole board from `GET /v2/torn/bounties`, 100 per page. Stacks multiple bounties on one head into a single row. Torn serves one shared snapshot, so if `bounties_timestamp` hasn't moved since the last search, the saved copy is reused and the sweep costs one call.
2. **Your filters.** The board shows unfiltered while it loads, then snaps to the search limits (max level, min reward, max fair fight). Everything stays editable on the board, plus name, level range and minimum account age.
3. **Fair fight.** One FFScouter call per 205 targets (`/api/v1/get-stats`). Optional, and only runs if you give it the key you registered on ffscouter.com.
4. **Live status.** One `GET /v2/user/{id}/profile` per target, in the board's current sort order. Timers (hospital, jail, protection, bounty expiry) count down locally from the last call. With live checks on, okay targets get re-checked every 90 seconds. Every status check carries a `timestamp` parameter, so Torn's 30-second service cache never hands back an old answer.
5. **Fresh.** Fresh isn't a status of its own. It's how the first 10 seconds of Okay look once a hospital timer runs out. After those 10 seconds the target gets one check, since someone may have hit them in the window. Until that check lands, they show as **Okay** with a small "confirming" dot. Same for jail, without the 10 seconds.
6. **Ready.** The five highest values you can hit right now, Fresh included.
7. **Available soon.** The five matches whose hospital timer ends soonest, with big countdowns. Once a timer ends they move to Ready.

## Current value

A target can carry several bounties. One hit claims one bounty, so the board shows the **current value**: what your next hit pays. That's the highest single bounty on the target, because Torn's own board shows the highest one per target. After a hit, the next bounty moves up and becomes the new current value.

The Value column, the Ready strip, Available soon, sorting and the value filters all use the current value. The Bounties column shows the rest: "+4 more, $400k stacked".

> **To be tested and confirmed:** that a hit always claims the highest bounty first. Built on one real hit that matched it. If Torn turns out to use another order (oldest first, say), only `recalc()` changes.

Each row links to the target's profile and attack page. Once you open Attack, a **Hit?** button appears. Press it and the target gets a Hit tag and drops off the board. The **Hit** chip shows them again. The tag clears by itself the moment that target leaves hospital and turns Fresh, or when its bounty leaves Torn's board. "Bring back" in the board header clears every tag. Nothing is automated on torn.com.

## Sorting

Highest or lowest value, Okay first, lowest or highest level, lowest or highest fair fight, ending soonest, name A to Z or Z to A, checked longest ago or most recently. From the Sort menu, or by clicking the Target, Level, Status, Value, Bounties, Fair fight and Checked headers.

## API inspector

**Inspect calls** (next to the pace line, or "Watch every API call live" under the key) opens a live view of every call the tool makes. For everyone, not only devmode: transparency is the point.

- **Live log.** Newest on top. Each call expands to the full request (URL, headers), the full JSON answer, timing, size, and why the engine made it right then ("First pass, never checked", "Fresh is over, confirming Okay", "Board page 12 of 69"...). Filter by service or call type, pause the stream, clear, export as JSON.
- **Queue.** What's waiting at the pacer, and the next 10 status checks with their reason and when they're due.
- **Every call explained.** The catalog of all six calls: service, endpoint, when it runs, what it's for, the fields the tool reads, and how many ran this session.
- **Learn mode.** Adds a plain-English card to every call and view.

Keys are always masked (`••••••••••••3fA9`), in the log and in exports. The log lives in memory only: the last 500 calls, with full answers for the last 80. Gone on reload.

| Service | Call | When | What it's for |
| --- | --- | --- | --- |
| Torn | `GET /v2/key/info` | Connect | What kind of key this is, before anything else runs |
| Torn | `GET /v2/user/profile` | Connect | Who you are: name, ID, level, account age |
| Torn | `GET /v2/torn/bounties` | Start the hunt, Refetch board | The bounty board, 100 per page |
| Torn | `GET /v2/user/{id}/profile` | Status loop, Check now | One target's live status and account age |
| FFScouter | `GET /api/v1/check-key` | Connect, only with an FFScouter key | Is the key registered? |
| FFScouter | `GET /api/v1/get-stats` | After the board, Refetch fair fight | Fair fight for up to 205 targets |

Google Fonts loads the two typefaces. That's a font file, not an API call. Nothing goes to The Great Stag, and there's no analytics.

## Devmode

The **Devmode** button in the header opens a panel for the engine's internals. The choice is remembered.

- **Engine.** Run each part on its own: refetch the board, re-check every match, re-check stale only (older than 2, 5, 10 or 30 minutes), pause or resume the status loop, clear the hand-picked queue, refetch fair fight, freeze all calls, start over. Each shows its live cost in calls, and the time at your pace, before you press.
- **Board.** Sliders for max level, max fair fight, and a value range (from / up to, on a log scale from $1k to $1b). They filter the board you already have and cost no calls. Column toggles for every column, plus an Account age column that's off by default.
- **Storage.** Everything the tool keeps in your browser, with its size and a Clear button. Keys are never shown. "Wipe everything" takes two presses and disconnects. The rate-limit log can't be cleared, because it keeps you under Torn's cap.

## Alerts

- **Sound** when a matching target turns Fresh (two rising notes) or comes back Okay after a check (one note). At most one sound every 2 seconds. The **Sound** button in the header turns all sound off.
- **Browser notification**, opt-in under "Pace and live checks", for the same moments, only while the tab is in the background.
- Hit-tagged and protected targets never alert.
- Browsers block notifications inside a cross-site iframe, which is how the tool sits on thegreatstag.com. There, the setting links to the tool on its own page. Sound works in both.

## New player protection

Torn protects every account for its first 14 days, and the protection ends the moment the account turns 14 days old. Other players can't attack it in that window, unless it joins a faction war. There's no level rule: it's purely account age.

The status check uses the profile endpoint rather than basic because profile carries `age` and `signed_up` for the same single call. Protected targets get a teal "Protected" block counting down to sign-up plus 14 days, a "New player" tag, and no place in the Ready strip. The "Min account age" filter set to 14 hides them. If you're under 14 days yourself, other protected players older than a day count as fair game, which matches Torn's rule.

The API doesn't say who is in a faction war, so a protected player who is fighting one still shows as protected.

## Which key works

A dedicated **Public Only** key. That's all the tool needs: `/key/info`, `/user/profile`, `/user/{id}/profile` and `/torn/bounties` are all public.

Connect calls `GET /v2/key/info` first. It answers for any key, so the page can name the key type and stop a key that can't do the job before anything else runs. A Custom key would need **User → Profile** and **Torn → Bounties**.

The key you registered on FFScouter is a Custom key built from FFScouter's own list (battle stats, attacks, profile and so on). Torn → Bounties isn't on it, so Torn refuses the bounty board with "access level too low". It belongs in the FFScouter field, with a Public Only key in the Torn field.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole app: markup, styles, script. |
| `_headers` | Cloudflare Pages security headers (CSP, framing, referrer). |

## Demo mode

Add `?demo` to the URL, or press **Try the demo**. Made-up players and bounties, statuses that move, new accounts under protection, no API calls. Good for the project page and for testing.

## Limits worth knowing

- Torn allows 100 calls per minute per player, shared across every tool and script they run. The default pace is 60. It's adjustable from 20 to 90 under "Pace and live checks". The pace line shows both numbers apart: what you used out of your pace, and what your pace leaves for your other tools.
- The pace is a hard cap on any rolling minute, shared by every open tab of the tool and kept across reloads. The tool can't see calls from your other Torn tools, so leave them room.
- A status check is one call per target. Checking 200 targets takes about 3 to 4 minutes at 60 per minute. Narrow filters go faster.
- Re-checks pause while the tab is in the background. The first pass keeps going.
- If Torn answers "too many requests", everything waits 30 seconds, then carries on.

## Privacy (Torn API terms)

| | |
| --- | --- |
| Data storage | Only in the visitor's browser. Nothing on a server. |
| Data sharing | Nobody. The Torn key goes only to api.torn.com, in a header. The FFScouter key, if one is added, goes only to ffscouter.com. |
| Purpose of use | Finding bounty targets and showing their live status. |
| Key storage and sharing | Local storage, only if "Remember on this device" is ticked. Never sent to The Great Stag. Disconnect wipes it. |
| Key access level | Public Only. |

## License

MIT. Open Source - TGS 2026. See [LICENSE](LICENSE).
