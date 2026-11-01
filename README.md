# HBountyTracker

Finds Torn bounty targets you can hit right now. One static page, no build step, no server. Everything runs in the visitor's browser on their own API key.

A freebie by [The Great Stag](https://www.thegreatstag.com). Open source, so anyone can check exactly what happens to their key.

## What it does

1. **Bounty list.** Pulls the whole board from `GET /v2/torn/bounties`, 100 per page. Stacks multiple bounties on one head into a single row. Torn serves one shared snapshot, so if `bounties_timestamp` hasn't moved since the last search, the saved copy is reused and the sweep costs one call.
2. **Your filters.** The board shows unfiltered while it loads, then snaps to the search limits (max level, min reward, max fair fight). Everything stays editable on the board, plus name, level range and minimum account age.
3. **Fair fight.** One FFScouter call per 205 targets (`/api/v1/get-stats`). Optional, and only runs if you give it the key you registered on ffscouter.com.
4. **Live status.** One `GET /v2/user/{id}/profile` per target, in the board's current sort order. Timers (hospital, jail, protection, bounty expiry) count down locally from the last call. With live checks on, okay targets get re-checked every 90 seconds, and anyone whose timer runs out gets checked again.
5. **Ready.** The five highest rewards you can hit right now.
6. **Out of hospital next.** The five matches leaving hospital soonest, with big countdowns.

Each row links to the target's profile and attack page. Nothing is automated on torn.com.

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

## Where it lives

| | |
| --- | --- |
| The tool | https://hbountytracker.marelecerb.workers.dev, a Cloudflare Worker serving this repo as static assets. No build step. |
| The project page | https://www.thegreatstag.com/projects/hbountytracker, a Webflow page that embeds the tool in an iframe. |

Cloudflare applies `_headers` on the Worker too, so the CSP, framing and referrer rules hold there.

If the tool ever moves to a subdomain of thegreatstag.com, the iframe becomes same-site with the Webflow page. That matters for Safari, which partitions storage in cross-site iframes, so "Remember on this device" and the cached board are less reliable on workers.dev there.

## Show it on Webflow

This sits in a Code Embed element in the tracker section of the project page. If the host changes, update `src` and `ORIGIN` together.

```html
<iframe id="hbt-frame" src="https://hbountytracker.marelecerb.workers.dev/" title="HBountyTracker, a free Torn bounty tracker"
  style="display:block;width:100%;height:1100px;border:0;background:#0D0D0D"></iframe>
<script>
(function () {
  var ORIGIN = 'https://hbountytracker.marelecerb.workers.dev';
  var frame = document.getElementById('hbt-frame');
  window.addEventListener('message', function (e) {
    if (e.origin !== ORIGIN || !e.data || e.data.type !== 'hbt:height') return;
    frame.style.height = e.data.height + 'px';
  });
})();
</script>
```

The tool posts its content height to the parent whenever it changes, so the iframe grows and shrinks with the board and never shows an inner scrollbar.

`_headers` only allows framing from thegreatstag.com, www.thegreatstag.com and `*.webflow.io` (staging). If the site lives somewhere else, add that origin to `frame-ancestors`.

## Demo mode

Add `?demo` to the URL, or press **Try the demo**. Made-up players and bounties, statuses that move, new accounts under protection, no API calls. Good for the project page and for testing.

## Limits worth knowing

- Torn allows 100 calls per minute per player, shared across every tool and script they run. The default pace is 60. It's adjustable from 20 to 90 under "Pace and live checks".
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

## Local preview

Any static server works:

```bash
python -m http.server 8766
```

Then open http://localhost:8766/?demo
