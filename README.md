# HBountyTracker

Finds Torn bounty targets you can hit right now. One static page, no build step, no server. Everything runs in the visitor's browser on their own API key.

A freebie by [The Great Stag](https://www.thegreatstag.com). Open source, so anyone can check exactly what happens to their key.

## What it does

1. **Bounty list.** Pulls the whole board from `GET /v2/torn/bounties`, 100 per page. Stacks multiple bounties on one head into a single row. Torn serves one shared snapshot, so if `bounties_timestamp` hasn't moved since the last search, the saved copy is reused and the sweep costs one call.
2. **Your filters.** The board shows unfiltered while it loads, then snaps to the search limits (max level, min reward, Fair Fight Score as the fair fight cap). Everything stays editable on the board, plus name, level range and minimum account age.
3. **Fair fight.** One FFScouter call per 205 targets (`/api/v1/get-stats`). Optional, and only runs if you give it the key you registered on ffscouter.com. It scouts every target your search allows, past the fair fight cap too, so the strips and the check order can see both sides of your score. Rows FFScouter has no estimate for get a gray background.
4. **Live status.** One `GET /v2/user/{id}/profile` per target. With fair fight and a score, the first pass goes: your band first (within 1.5 of your score, either side), then everyone under it, then everyone over it, biggest bounty first in each. Targets with no fair fight go last. Without a score, it follows the board's sort. Timers (hospital, jail, protection, bounty expiry) count down locally from the last call. Every status check carries a `timestamp` parameter, so Torn's 30-second service cache never hands back an old answer. Re-checks are covered in **Smart re-checks** below.
5. **Fresh.** Fresh isn't a status of its own. It's how the first 10 seconds of Okay look once a hospital timer runs out. After those 10 seconds the target gets one check, since someone may have hit them in the window. That check jumps to the front of the queue, ahead of the first pass, and runs in a background tab too (not with live checks off). Until it lands, they show as **Okay** with a small "confirming" dot. Same for jail, without the 10 seconds.
6. **High value.** Every Okay target within 1.5 of your **Fair Fight Score**, either side, biggest bounty first. Each card gets a value border (see below). Needs an FFScouter key and a score.
7. **Easy fights.** Every Okay target, closest to your **Fair Fight Score** first, on either side of it. At the same distance, the one under your score wins, then the bigger bounty. Both strips look past the board's fair fight filter on purpose. With an FFScouter key, targets FFScouter has no estimate for stay out. Without one, Easy fights is simply the highest bounties you can hit.
8. **Fresh out of the hospital.** Everyone who turned Fresh in the last 5 minutes, newest first: the strip that tells you what the Fresh sound was about. They stay the full 5 minutes with their latest status, even back in hospital. It shows up when the other strips do, and once it's there it stays on screen even when empty, so the layout never jumps.
9. **Available soon.** Every match whose hospital timer is still running, soonest out first, with big countdowns, level and fair fight.

10. **Personal targets.** Anyone you tagged by hand, bounty or not, no cap. Add them from any card or row, or by player ID (or a profile link) with **Add personal target** on the board. Only ever removed by you (two presses). Checked first at the start of every hunt. The section is hidden while it's empty, and sits at the very top.
11. **Pinned.** Bounty targets you pinned, 10 at most. Always first on the board, with a pink stripe and tint. See **Pins and personal targets** below.

Every strip is a carousel with every qualifying target: swipe on touch, drag with a mouse, or use the arrows. With a mouse, the list stays exactly where you let go, and a fast drag keeps gliding and slows to a stop (not with "reduce motion" on). The arrows move by whole cards.

## Pins and personal targets

| | Pinned | Personal targets |
| --- | --- | --- |
| Who | A bounty target you pinned | Anyone you tagged |
| Needs a bounty | Yes | No |
| Cap | 10 | None |
| Removed | Automatically, 2 days after being marked for deletion | Only by you |

A pin is **marked for deletion** when its bounty is gone: not on the board after a refetch, expired, or you marked a hit on their last bounty. The card and the row say so, with a countdown. A new bounty during those 2 days lifts the mark (after a hit, only a board read newer than the hit can show that). **Keep as personal** moves the target to Personal targets, where nothing removes it.

Both run on their own clock, from their last known status:

| Status | Re-checked |
| --- | --- |
| Okay | Every minute |
| Hospital, jail | Every 5 minutes, in case a revive or a bail ends it early, plus the usual check when the timer ends |
| Traveling, abroad | Every 5 minutes |
| Protected | Once, when protection ends |

These re-checks have their own budget: never more than 15% of your pace in any minute, on top of the 25% for everyone else. Personal and pinned targets also get fair fight, in the same FFScouter calls.

## Section controls

Every strip has **Hide** and, except Pinned, **Pin to board**.

- **Hide.** The strip goes, and its fast re-checks stop with it (High value and Easy fights lose their 90-second top 10, Pinned and Personal lose their own clock). Hidden Fresh out of the hospital means no Fresh sound. Hidden strips line up in a "Hidden:" bar; one click brings each back.
- **Pin to board.** The strip's targets go to the top of the board, in the strip's own order. Pin several and they stack in the order you pinned them, with the number in the circle. Unpin one and the rest move up. A target in two pinned strips shows once, in the first.
- On the board: Pinned targets first, always. Then each pinned strip. Then everyone else, in the board's sort. Each block gets a header. The board's filters and status chips apply to every block, so High value and Easy fights, which look past the fair fight filter in their strips, are filtered on the board.

Hidden and pinned sections are remembered on the device.

## Fair fight colors

The fair fight number sits on a colored pill. The colors follow Torn's own formula, FF = 1 + 8/3 × (their battle stat score ÷ yours), so every anchor is a real stat ratio:

| FF | Their battle stat score vs yours | Label | Color |
| --- | --- | --- | --- |
| 1.00 to 1.50 | 0% to 19% | Free win | Blue |
| 1.50 to 2.25 | 19% to 47% | Easy | Teal |
| 2.25 to 3.00 | 47% to 75% | Fair fight | Teal to amber |
| 3.00 to 3.67 | 75% to 100% | Even | Amber to orange |
| 3.67 to 4.50 | 100% to 131% | Stronger than you | Orange to red |
| 4.50 and up | over 131% | May be impossible | Dark red |

The color slides smoothly between anchors. Hover a pill for the ratio and the label. Torn's in-game number caps at 3.00; FFScouter's doesn't. Fair fight changes respect, not bounty money, so here it's a difficulty meter.

## Value borders

One scale, used by the High value strip and every row of the board:

| Current value | Border |
| --- | --- |
| Under $50k | Gray |
| $50k to $500k | Dark green to gold, on a log scale (halfway is about $160k) |
| $500k to $1m | Gold |
| $1m to $2m | Gold to purple, on a log scale |
| $2m and up | A color cycle with a golden glow |

On the board, neighbors in the same tier merge into one shape, each row keeping its own shade. On phones the cards glue together. With "reduce motion" on in your system settings, the color cycle stops on plain gold.

## Smart re-checks

Once everyone has had a first check, the tool only spends calls where they matter:

| Target | Re-checked |
| --- | --- |
| Pinned or personal | Their own clock and their own 15%, see **Pins and personal targets** |
| Okay, in the top 10 of High value or Easy fights | Every 90 seconds, stretched so these alone never pass 25% of your pace. Not while that strip is hidden |
| Okay, active in the last hour | Every 10 minutes |
| Okay, offline an hour or more | Every 20 minutes |
| Okay, offline a day or more | Every 30 minutes |
| In hospital | Never while the timer runs. One check 10 seconds after it ends |
| Marked hit | Never, until the hospital timer from the hit ends |
| Jail, and anything else | Every 10 minutes. Travelers every 5 |

"Offline" comes from `last_action` in the same profile call, so it costs nothing extra. Re-checks never use more than 25% of your pace in any minute. With nothing due, the loop goes quiet until a timer ends.

## Current value

A target can carry several bounties. One hit claims one bounty, so the board shows the **current value**: what your next hit pays. That's the highest single bounty on the target, because Torn's own board shows the highest one per target. After a hit, the next bounty moves up and becomes the new current value.

The Value column, the Ready strip, Available soon, sorting and the value filters all use the current value. The Bounties column shows the rest: "+4 more, $400k stacked".

> **To be tested and confirmed:** that a hit always claims the highest bounty first. Built on one real hit that matched it. If Torn turns out to use another order (oldest first, say), only `recalc()` changes.

Each row links to the target's profile and attack page. Nothing is automated on torn.com.

## Hits and gains

1. Open **Attack** on a target. A **Hit?** button appears.
2. Press it once you've hospitalized them. For one minute it turns into **Undo**, in case you misclicked.
3. The tool checks them once, about 5 seconds later, to pick up the hospital timer from your hit.
4. After the minute, they're hidden everywhere, with a "Just hit" tag, and get **no status checks** until that timer ends.
5. When the timer ends, they're back everywhere with a "Hit" tag, and get checked like everyone else.

If the check after a mark shows they never went to hospital, the tag clears right away.

Every mark goes in a ledger in your browser, at the value shown when you pressed Hit?. That's what the banner under "Pace and live checks" adds up: *"This tracker gained $X for you in N hits"*, based on your Hit marks. A Public key can't read your attack log, so the tool can only count what you mark.

The **Hit** chip shows every target in the ledger, even ones whose bounty left the board, with **Gained** and **Times hit** columns, and a **Remove** button (two presses) that takes a target out of the ledger. Devmode Storage can erase all gains (two presses). Gains live in local storage, so clearing your browser's site data erases them too, and the embedded page and the standalone page keep separate ledgers.

## The board buttons

- **Refetch board.** Reads the bounty board again. If Torn's board hasn't changed, page 1 says so and that's the only call.
- **Refresh list.** Status checks only, no board. High value first, then Easy fights, then everyone else the loop walks. One call per target; the button's tooltip shows the count and the time at your pace.

Drag the edge of any column header to resize it, like a spreadsheet. Widths are saved on the device. **Reset columns** goes back to automatic.

## Sorting

Highest or lowest value, Okay first, lowest or highest level, lowest or highest fair fight, ending soonest, name A to Z or Z to A, checked longest ago or most recently. From the Sort menu, or by clicking the Target, Level, Status, Value, Bounties, Fair fight and Checked headers.

## API inspector

**Inspect calls** (next to the pace line, or "Watch every API call live" under the key) opens a live view of every call the tool makes. For everyone, not only devmode: transparency is the point.

- **Live log.** Oldest at the top, newest at the bottom; the view follows new calls while you're at the bottom. Calls are grouped in colored blocks, one per job the engine was doing, each with a header that says what the job is: Connect (gray), Board sweep (gold), Fair fight (blue, FFScouter), Initial profile fetch (green: first status and timers for each target), Re-checks (teal), Timer ended, confirming (orange), Manual checks (purple), Refresh list (pink). The response code is colored too: green when it worked, red when Torn or FFScouter refused (with Torn's error code), orange when no answer came back, gray while waiting. Each call expands to the full request (URL, headers), the full JSON answer, timing, size, and why the engine made it right then ("Initial profile fetch, never checked", "Fresh is over, confirming Okay", "Board page 12 of 69"...). Filter by service or call type, pause the stream, clear, export as JSON.
- **Queue.** What's waiting at the pacer, and the next 10 status checks with their reason and when they're due.
- **Calls left.** The summary line says how many calls the first pass still needs and how long that takes at your pace. Once it's done: how many re-checks are due now.
- **Every call explained.** The catalog of all six calls: service, endpoint, when it runs, what it's for, the fields the tool reads, and how many ran this session.
- **Learn mode.** Adds a plain-English card to every call and view, each linking to its Codex chapter.
- **Codex.** The long version, in its own tab: the big picture, a real request from your session taken apart piece by piece, how the board is paged, the three layers of caching, rate limits and how tabs share them, the scheduler, fair fight math with a calculator, keys and the walls around them, sections and pins, how the tool got here, and a glossary. The numbers in it are live: your pace, your budget, your calls.

Keys are always masked (`••••••••••••3fA9`), in the log and in exports. The log lives in memory only: the last 500 calls, with full answers for the last 80. Gone on reload.

| Service | Call | When | What it's for |
| --- | --- | --- | --- |
| Torn | `GET /v2/key/info` | Connect | What kind of key this is, before anything else runs |
| Torn | `GET /v2/user/profile` | Connect | Who you are: name, ID, level, account age |
| Torn | `GET /v2/torn/bounties` | Start the hunt, Refetch board | The bounty board, 100 per page |
| Torn | `GET /v2/user/{id}/profile` | Status loop, Refresh list, Check now | One target's live status, last action and account age |
| FFScouter | `GET /api/v1/check-key` | Connect, only with an FFScouter key | Is the key registered? |
| FFScouter | `GET /api/v1/get-stats` | After the board, Refetch fair fight | Fair fight for up to 205 targets |

Google Fonts loads the two typefaces. That's a font file, not an API call. Nothing goes to The Great Stag, and there's no analytics.

## Devmode

The **Devmode** button in the header opens a panel for the engine's internals. The choice is remembered.

- **Engine.** Run each part on its own: refetch the board (every page, forced), re-check every match, re-check stale only (older than 2, 5, 10 or 30 minutes), pause or resume the status loop, clear the hand-picked queue, refetch fair fight, freeze all calls, start over. Each shows its live cost in calls, and the time at your pace, before you press. Plus **Never stop for inactivity**, remembered on the device: the tool then keeps calling at your pace for as long as the tab is open. Leave it off unless you're testing.
- **Board.** Sliders for max level, max fair fight, and a value range (from / up to, on a log scale from $1k to $1b). They filter the board you already have and cost no calls. Column toggles for every column, plus an Account age column that's off by default.
- **Storage.** Everything the tool keeps in your browser (pins and personal targets included), with its size and a Clear button. Keys are never shown. Clearing Gains takes two presses. "Wipe everything" takes two presses, disconnects, and erases the gains too. The rate-limit log can't be cleared, because it keeps you under Torn's cap.

## Alerts

- **Sound** when a matching target turns Fresh (a board match, or anyone in your band): two rising notes. Not while Fresh out of the hospital is hidden. The target shows up there.
- **Pinned sound** for pinned and personal targets: three lower notes, when a hospital or jail timer runs out. Flights have no timer in the API, so for travelers it plays when a check finds them back in Torn.
- At most one sound every 2 seconds. The **Sound** button in the header turns them all off.
- **Browser notification**, opt-in under "Pace and live checks", for the same moment, only while the tab is in the background.
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
- A status check is one call per target. Checking 200 targets takes about 3 to 4 minutes at 60 per minute. With fair fight and a score, the first pass also covers targets above your score, so it's longer. Narrow filters go faster.
- Re-checks never use more than 25% of your pace in any minute (see Smart re-checks). Confirm checks, Refresh list and Check now skip the line.
- Re-checks pause while the tab is in the background. The first pass and confirm checks keep going.
- **Inactivity stop.** After 30 minutes with no click, key press, scroll or touch, every call stops and a banner says so. Timers keep counting. Press Resume to carry on. The first pass is exempt: you pressed Start to get the whole list. Mouse movement doesn't count, since a bumped mouse isn't someone using the tool.
- If Torn answers "too many requests", everything waits 30 seconds, then carries on.
- Torn warns that repeated requests with invalid keys can get an IP banned for a while. So a key Torn refuses (wrong key, owner in federal jail, inactive, paused) is never sent again: it's dropped from storage, and only a fingerprint is kept so the tool recognises it. A saved key that was refused is not sent on the next visit either. Paste a different key to carry on. If a paused or inactive key works again, clear "Refused keys" in devmode Storage.
- If Torn blocks your IP (error 8), nothing goes to Torn for 10 minutes, in every tab. Torn doesn't say how long a block lasts, so the tool waits on the safe side.
- "Wipe everything" keeps the rate limit log, the refused keys and the IP block cooldown. They are what keeps you inside Torn's rules.

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
