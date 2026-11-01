# HBountyTracker

Finds Torn bounty targets you can hit right now. One static page, no build step, no server. Everything runs in the visitor's browser on their own API key.

A Great Stag project.

## What it does

1. **Bounty list.** Pulls the whole board from `GET /v2/torn/bounties`, 100 per page. Stacks multiple bounties on one head into a single row. Torn serves one shared snapshot, so if `bounties_timestamp` hasn't moved since the last search, the saved copy is reused and the sweep costs one call.
2. **Your filters.** The board shows unfiltered while it loads, then snaps to the search limits (max level, min reward, max fair fight). Everything stays editable on the board.
3. **Fair fight.** One FFScouter call per 205 targets (`/api/v1/get-stats`). Optional. Needs a key registered on ffscouter.com.
4. **Live status.** One `GET /v2/user/{id}/basic` per target, in the board's current sort order. Timers (hospital, jail, bounty expiry) count down locally from the last call. With live checks on, ready targets get re-checked every 90 seconds, and anyone whose timer runs out gets checked again.

Each row links to the target's profile and attack page. Nothing is automated on torn.com.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole app: markup, styles, script. |
| `_headers` | Cloudflare Pages security headers (CSP, framing, referrer). |

## Deploy on Cloudflare Pages

1. Pages → Create a project → connect this repo.
2. Framework preset: **None**. Build command: **empty**. Build output directory: **/**.
3. Add a custom domain. Use a subdomain of thegreatstag.com, for example `tools.thegreatstag.com`. Inside the Webflow iframe, that keeps the page same-site with the parent, so Safari doesn't wall off its local storage (saved key, cached board).

## Show it on Webflow

Drop this into an Embed element on the project page. Swap `ORIGIN` and `src` for the real domain.

```html
<iframe id="hbt-frame" src="https://tools.thegreatstag.com/" title="HBountyTracker"
  style="display:block;width:100%;height:1400px;border:0;background:#0D0D0D"></iframe>
<script>
  (function () {
    var ORIGIN = 'https://tools.thegreatstag.com';
    var frame = document.getElementById('hbt-frame');
    window.addEventListener('message', function (e) {
      if (e.origin !== ORIGIN || !e.data || e.data.type !== 'hbt:height') return;
      frame.style.height = e.data.height + 'px';
    });
  })();
</script>
```

The page posts its content height to the parent whenever it changes, so the iframe grows and shrinks with the board and never shows an inner scrollbar.

`_headers` only allows framing from thegreatstag.com, www.thegreatstag.com and `*.webflow.io` (staging). If the site lives somewhere else, add that origin to `frame-ancestors`.

## Demo mode

Add `?demo` to the URL, or press **Try the demo**. Made-up players and bounties, statuses that move, no API calls. Good for the project page and for testing.

## Limits worth knowing

- Torn allows 100 calls per minute per player, shared across every tool and script they run. The default pace is 60. It's adjustable from 20 to 90 under "Pace and live checks".
- A status check is one call per target. Checking 200 targets takes about 3 to 4 minutes at 60 per minute. Narrow filters go faster.
- Re-checks pause while the tab is in the background. The first pass keeps going.
- If Torn answers "too many requests", everything waits 30 seconds, then carries on.

## Privacy (Torn API terms)

| | |
| --- | --- |
| Data storage | Only in the visitor's browser. Nothing on a server. |
| Data sharing | Nobody. The Torn key goes only to api.torn.com, in a header. The FFScouter key (or the Torn key, if that field is empty) goes only to ffscouter.com. |
| Purpose of use | Finding bounty targets and showing their live status. |
| Key storage and sharing | Local storage, only if "Remember on this device" is ticked. Never sent to The Great Stag. Disconnect wipes it. |
| Key access level | Public. |

## Local preview

Any static server works:

```bash
python -m http.server 8766
```

Then open http://localhost:8766/?demo
