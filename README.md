# monitor-wr — Waiting Room Live Monitor
A self-hosted, faster-updating dashboard for Cloudflare Waiting Rooms. Deploys entirely on Cloudflare Workers + Durable Objects — no external infrastructure needed.
> **Why this exists:** The Cloudflare dashboard refreshes Waiting Room metrics on a ~60 second UI cycle, on top of a ~20–50s API cache. During a live event, your team could be reacting to data that is nearly 2 minutes old. This tool cuts that to ~55 seconds by polling the Status API directly and pushing updates to your browser instantly via Server-Sent Events.
---
## What You Get
- **Faster updates** — Picks up Status API cache refreshes within 5 seconds and pushes to your browser immediately via SSE. No manual refresh needed.
- **30-minute rolling chart** — Persists across page reloads. Open the dashboard mid-event and the chart history is already there.
- **Glitch filtering** — The Status API occasionally returns transient zeros during active queueing. The dashboard holds the last known good value so your chart does not show false drops.
- **Smooth chart lines** — Exponential moving average smooths out the coarseness of the ~30s API cache cycle without hiding real trends.
- **SSE fallback** — If the live connection drops, the dashboard automatically falls back to polling until it reconnects.
- **Zero Trust ready** — Wrap with Cloudflare Access for proper identity-based protection.
---
## How It Works
Browser ──── GET /dashboard ────► Worker
               (HTML + JS)
Browser ──── GET /sse ──────────► Worker ──► Durable Object
Browser ──── GET /api/history ──► Worker ──► Durable Object
                                                   │
                                          Polls every 5s
                                                   │
                                                   ▼
                                    Waiting Room Status API
**Worker** — Serves the dashboard UI and routes live requests to the Durable Object.
**Durable Object** — The single source of truth. Polls the Waiting Room Status API every 5 seconds, stores a rolling 30-minute history in persistent storage, and broadcasts updates to all connected clients simultaneously. Only polls while someone is watching (cost-efficient).
---
## Data Freshness
This tool does not bypass Cloudflare's infrastructure. The Status API data is always ~20–50 seconds behind reality — that is a Cloudflare-side constraint. What this tool eliminates is the Cloudflare dashboard's own ~60s UI refresh delay on top of that.
| | Worst case lag |
|---|---|
| Cloudflare dashboard | ~110 seconds |
| monitor-wr | ~55 seconds |
> Polling faster than every 30 seconds yields no benefit — the API cache does not refresh faster than that.
---
## Prerequisites
- Cloudflare account on **Business or Enterprise plan** (required for Waiting Room)
- A Waiting Room already configured on your zone
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) installed
---
## Setup
### 1. Create the project
```bash
npx wrangler@latest init wr-live-monitor
cd wr-live-monitor
2. Place the files
wr-live-monitor/
├── wrangler.toml
└── src/
    ├── index.js
    └── monitor_do.js
3. Configure wrangler.toml
name = "wr-monitor"
main = "src/index.js"
compatibility_date = "2025-01-01"
[durable_objects]
bindings = [
  { name = "WR_MONITOR", class_name = "WRMonitorDO" }
]
[[migrations]]
tag = "v1"
new_sqlite_classes = ["WRMonitorDO"]
[vars]
ZONE_ID = "YOUR_ZONE_ID"
WR_ID   = "YOUR_WAITING_ROOM_ID"
# DASH_KEY = "optional-secret-key"
Variable	Where to find it
ZONE_ID	Cloudflare dashboard → your zone → Overview page (right sidebar)
WR_ID	Run the command below
DASH_KEY	Optional. Any string. If set, dashboard requires ?k=<value>
Find your Waiting Room ID:
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/waiting_rooms" \
  --header "Authorization: Bearer $CF_API_TOKEN"
Use the id field from the response.
4. Add your API token
Create a token at Cloudflare dashboard → My Profile → API Tokens with Waiting Rooms: Read permission, then:
wrangler secret put CF_API_TOKEN
5. Deploy
wrangler deploy
6. Open the dashboard
https://<your-worker-name>.<your-subdomain>.workers.dev/dashboard
With DASH_KEY set:
https://<your-worker-name>.<your-subdomain>.workers.dev/dashboard?k=<your-key>
---
## Security
### Query Key (built-in, basic)
Set `DASH_KEY` in `wrangler.toml`. All dashboard routes return `401` without the correct `?k=` parameter in the URL.
Suitable for internal use where the URL is only shared with a small trusted group. Note: the key is visible in the URL and browser history.
### Cloudflare Zero Trust Access (recommended)
For proper authentication, wrap the Worker hostname in a [Cloudflare Access self-hosted application](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/):
1. Go to **Zero Trust → Access → Applications → Add an application → Self-hosted**
2. Set the domain to your Worker's hostname
3. Create an Allow policy scoped to your team's email addresses or identity provider group
Users will be prompted to authenticate before reaching the dashboard. No API keys in URLs.
![Cloudflare Access protecting the dashboard](./Access.png)
---
Dashboard
Waiting Room Live Monitor dashboard
---
Limitations
- Metrics are estimates. The Status API returns estimated values — probabilistic aggregations across Cloudflare's global data centers, not exact counters.
- queue_all affects the active user count. When queue_all is enabled, estimated_total_active_users trends toward zero because no users are being passed through to origin — not because the room is empty.
- One Waiting Room per deployment. Each instance monitors a single WR_ID. Deploy multiple instances with different configurations to monitor multiple Waiting Rooms.
- No alerting. This is a passive dashboard. It does not send notifications when queue depth crosses a threshold.
---
Resources
- Cloudflare Waiting Room docs (https://developers.cloudflare.com/waiting-room/)
- Waiting Room Status API (https://developers.cloudflare.com/api/resources/waiting_rooms/subresources/statuses/methods/get/)
- Cloudflare Durable Objects (https://developers.cloudflare.com/durable-objects/)
- Cloudflare Zero Trust Access (https://developers.cloudflare.com/cloudflare-one/policies/access/)
---
Disclaimer
THIS SOFTWARE IS PROVIDED AS-IS. CLOUDFLARE IS NOT LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, OR CONSEQUENTIAL DAMAGES ARISING FROM ITS USE.
---
Key things fixed vs the original:
- **No emoji headings** — GitHub renders them fine but they break copy-paste into some wikis and editors
- **Screenshots placed in context** — Access screenshot sits under the Zero Trust section, Dashboard screenshot has its own clean section — both use the existing `./Access.png` and `./Dashboard.png` paths from the repo
- **Disclaimer moved to bottom** — was the first thing you saw in the original, now it's a clean single-line footer
- **Architecture is a proper code block** — the original had bullet points that mangled the DO description
- **Setup is one linear flow** — original had Steps 1/2 under Getting Started then a separate Configuration section that broke the flow
- **`wrangler.toml` example is included** — original just described variables in a table with no actual file example
