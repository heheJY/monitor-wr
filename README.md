# ==============================================================================
#
# THIS SOFTWARE IS PROVIDED BY CLOUDFLARE "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
# INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL CLOUDFLARE BE LIABLE FOR ANY DIRECT,
# INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES  (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR
# BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
# STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
# USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
# ==============================================================================
 
# 📊 monitor-wr — Waiting Room Live Monitor

### Real-time Dashboard for Cloudflare Waiting Rooms

Deploy a **Cloudflare Worker + Durable Object** to visualize your Waiting Room metrics in real-time without manual refreshes.

* **Real-time Metrics:** Estimated Active, Estimated Queued, and Max Wait time.
* **Live Updates:** Powered by **Server-Sent Events (SSE)**.
* **Persistence:** History is stored in a Durable Object (charts persist across page reloads).

---

## 🏗️ Architecture

* **Worker:** Serves the `/dashboard` UI and proxies requests to the Durable Object.
* **Durable Object:** * Polls the Cloudflare **Waiting Room Status API**.
* Stores rolling historical data.
* Broadcasts updates to all connected clients via SSE.



> [!NOTE]
> Metrics are **estimates** from the Status API. If **Queue-all** is enabled, "Active" users may trend toward zero as traffic is diverted from the origin.

---

## 🚀 Getting Started

### 1. Initialize the Project

Run the following commands to scaffold your environment:

```bash
npx wrangler@latest init wr-live-monitor
cd wr-live-monitor

```

### 2. Add Source Files

Replace the contents of your project folder with the three essential files from this repository:

* `wrangler.toml` — Configuration and bindings.
* `src/index.js` — Main Worker entry point.
* `src/monitor_do.js` — Durable Object logic.

---

## ⚙️ Configuration

### Step 1: Configure Environment Variables

Edit your `wrangler.toml` and provide your specific identifiers:

| Variable | Description |
| --- | --- |
| `ZONE_ID` | Your Cloudflare Zone ID |
| `WR_ID` | The specific Waiting Room ID to monitor |
| `DASH_KEY` | *(Optional)* A simple shared key. Access via `?k=YOUR_KEY` |

### Step 2: Set API Secrets

Create a Cloudflare API Token with **Waiting Rooms: Read** permissions, then securely upload it to your worker:

```bash
wrangler secret put CF_API_TOKEN

```

---

## 🚢 Deployment

Deploy your monitor to the Cloudflare edge:

```bash
wrangler deploy

```

**Access your dashboard at:**
`https://<your-worker-domain>/dashboard`

---

## 🔒 Security & Optimization

### 🛡️ Recommended: Cloudflare Access

To prevent public access to your metrics, wrap the monitor in **Zero Trust**:

1. Go to **Zero Trust > Access > Applications**.
2. Create a **Self-hosted** application for your monitor's domain.
3. Define an **Allow** policy for your team's emails or IP ranges.

## Demo
Protected by Cloudflare Access
![Access](./Access.png)

Final Product

![Monitoring Dashboard](./Dashboard.png)
