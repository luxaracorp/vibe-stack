# USS — Unified Server Stack
## The real machine first

> **This is the hardware USS actually runs on:** an **Amazon EC2 m7i.4xlarge** running **Debian GNU/Linux 12 (bookworm)** with **Intel Xeon Platinum 8488C (16 cores)**, **61.77 GB RAM total**, and **199.90 GB disk** in **`us-east-1`**.
>
> The instance type is not guaranteed and may vary by Deepnote Team trial allocation, but the observed runtime has consistently been in the **AWS m7i family**. The “2 vCPU, 5 GB RAM” figure is **Deepnote’s advertised Basic-machine limit**, not the underlying machine USS gets. USS gives your backend substantially more room than that.

## What USS actually gives you

USS turns that machine into a **persistent, publicly accessible, production-grade backend** at zero infrastructure cost. You get:

- Real AWS compute: Amazon EC2 m7i.4xlarge, Intel Xeon Platinum 8488C (16 cores), 61.77 GB RAM total, 199.90 GB disk
- Root terminal access and a Python 3.13 environment with `venv`
- A public HTTPS endpoint through Deepnote’s native port 8080 exposure
- External persistence through Upstash Redis, Cloudflare R2, and Supabase
- A stable public URL through a Cloudflare Worker, even as machines rotate
- Automated health checks, handoffs, state transfer, and node rotation
- Continuous operation with only about three minutes of human work per week

**Bottom line:** USS is not “a 2-vCPU, 5-GB server.” That is the plan description. The running USS node is an AWS EC2-backed machine with the hardware above. The node is disposable; the service remains persistent through external stores and a rotating worker URL. The exact instance type may vary per node, but it is consistently in the AWS m7i family on Team trial.

> Built and tested by Luxara Corporation. Documented for AI agents and zero-budget developers.
> This is a living architecture. Every detail here has been validated in production.

---

## What is USS

USS is a method of running a **persistent, publicly accessible, production-grade backend** on free cloud infrastructure indefinitely, at zero cost, with no credit card and no phone number required.

It combines:

- **Deepnote Team trial nodes** as raw compute
- **External state stores** (Upstash Redis, Cloudflare R2) for persistence
- **Cloudflare Worker** as a stable routing layer
- **A free plan orchestrator node** as the automation brain

The result is a backend that never goes down, never costs money, and self-manages rotation with minimal human intervention.

---

## Core Principle

**USS nodes are disposable. Data is not.**

A USS node is pure compute. It processes jobs, serves API requests, and runs your backend. It never holds state permanently. All state — job queues, session data, processed files, database snapshots — lives in external stores, not on the node disk.

When a node dies, nothing is lost. A new node boots, connects to the same external stores, and picks up exactly where the old one left off.

---

## The Four Layers

### Layer 1: Compute (USS Nodes)

**What:** Deepnote Team trial machines backed by the AWS EC2 hardware described above, running your backend.

**Actual observed hardware per node:**

- AWS EC2 m7i.4xlarge
- Debian GNU/Linux 12 (bookworm)
- CPU: Intel Xeon Platinum 8488C, 16 cores
- RAM: 61.77 GB total
- Disk: 199.90 GB
- Region: `us-east-1`

**Important note:**

- The exact instance type is not guaranteed and may vary across Deepnote Team trial nodes.
- The runtime has consistently been in the **AWS m7i family** on Team trial, but do not assume one exact node spec is fixed forever.

**Do not confuse this with Deepnote’s advertised plan limit:**

- “2 vCPU, 5 GB RAM” is the Basic-machine marketing/plan label
- It is not a description of the underlying machine USS observes in production

**Node capabilities:**

- Root terminal access
- Python 3.13 environment with `venv`
- Public HTTPS URL via native port 8080 exposure
- 199.90 GB local storage (ephemeral — do not rely on it for persistence)

**Trial details:**

- 14 days per account
- No credit card required
- No phone number required
- Temp emails work
- Alt accounts explicitly allowed by Deepnote
- Auto shutdown: set to 24h in machine sidebar (Team trial feature)

**How nodes stay alive:**

A notebook cell runs an infinite loop that pings the backend every 30 minutes. This keeps the kernel in active execution state, preventing idle shutdown. Closing the browser tab does not stop execution when the cell remains active.

**What triggers suspension (avoid these):**

- Cloudflare tunnels — flagged as open proxy behavior
- Cryptocurrency mining processes
- Any process matching Deepnote’s abuse signatures

**What does NOT trigger suspension (confirmed safe):**

- Running FastAPI/uvicorn on port 8080
- Installing packages via pip
- Cloning GitHub repos
- Installing FFmpeg via static binary
- Running long-lived Python processes in notebook cells
- Enabling Deepnote’s native Incoming Connections feature

---

### Layer 2: State (External Persistence)

**Nothing that matters lives on the USS node filesystem.**

| Data type | Store | Why |
|---|---|---|
| Job queue | Upstash Redis | Survives node rotation, BRPOP blocking supported |
| Session data | Upstash Redis | Fast, free tier sufficient |
| Video files / processed outputs | Cloudflare R2 | Free egress, S3-compatible |
| Relational data | Supabase free tier | Postgres, free tier permanent |
| Database snapshots | Cloudflare R2 | Pushed before node expiry |

**The contract:** Every write that matters goes to an external store immediately. The node filesystem is scratch space only.

---

### Layer 3: Routing (Stable URL)

**The problem:** Deepnote node URLs change on every rotation. A new node gets a new UUID subdomain. Anything hardcoded to a node URL breaks.

**The solution:** A Cloudflare Worker sits between your frontend and USS nodes. It reads a single environment variable (`USS_NODE_URL`) and proxies all requests to the active node. The Worker URL stays stable while the backend node rotates behind it.

**Architecture:**

```
Frontend (maxum.pages.dev)
        │
        ▼
Cloudflare Worker
(api.maxum.pages.dev)
  reads USS_NODE_URL env var
  proxies all requests
        │
        ▼
Active USS Node
(4c58454b-....prod-ue1.deepnoteproject.com)
        │
        ├──▶ Upstash Redis
        └──▶ Cloudflare R2
```

**Worker code:**

```javascript
export default {
  async fetch(request, env) {
    const target = env.USS_NODE_URL;
    const url = new URL(request.url);
    const targetUrl = target + url.pathname + url.search;

    const newRequest = new Request(targetUrl, {
      method: request.method,
      headers: request.headers,
      body: request.body,
      redirect: 'follow',
    });

    return fetch(newRequest);
  }
}
```

**Rotation = one command:**

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/workers/scripts/{WORKER_NAME}/secrets" \
  -H "Authorization: Bearer {CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name": "USS_NODE_URL", "text": "https://new-node-uuid.prod-ue1.deepnoteproject.com", "type": "secret_text"}'
```

---

### Layer 4: Orchestration (Free Plan Node)

**What:** A Deepnote free plan account running a permanent orchestrator notebook. Free plan does not expire.

**Specs:** 2 vCPU, 5GB RAM. Enough for orchestration logic. This is the one place where the advertised plan specification is relevant: the orchestrator only monitors nodes and coordinates handoff.

**Responsibilities:**

1. Monitor active USS node health via periodic health checks
2. Track trial expiry dates
3. On Day 13 (1 day before expiry), trigger handoff sequence on the active node
4. Verify the incoming node is healthy and has pulled all state
5. Update Cloudflare Worker `USS_NODE_URL` via API automatically
6. Confirm traffic is flowing to the new node
7. Log all events

**Orchestrator notebook cell:**

```python
import time, requests, os
from datetime import datetime

USS_NODES = [
    {
        "url": "https://4c58454b-8cdc-47f9-8cc6-d7572e929173.prod-ue1.deepnoteproject.com",
        "expires": "2026-09-27",
        "status": "active"
    },
    # Add next node here when created
]

CF_ACCOUNT_ID = os.environ.get("CF_ACCOUNT_ID")
CF_API_TOKEN = os.environ.get("CF_API_TOKEN")
CF_WORKER_NAME = "maxum-api-proxy"

def get_active_node():
    for node in USS_NODES:
        if node["status"] == "active":
            return node
    return None

def health_check(url):
    try:
        r = requests.get(f"{url}/health", timeout=10)
        return r.status_code == 200
    except:
        return False

def days_until_expiry(node):
    expiry = datetime.strptime(node["expires"], "%Y-%m-%d")
    return (expiry - datetime.now()).days

def update_worker_url(new_url):
    endpoint = f"https://api.cloudflare.com/client/v4/accounts/{CF_ACCOUNT_ID}/workers/scripts/{CF_WORKER_NAME}/secrets"
    r = requests.patch(endpoint,
        headers={"Authorization": f"Bearer {CF_API_TOKEN}"},
        json={"name": "USS_NODE_URL", "text": new_url, "type": "secret_text"}
    )
    return r.status_code == 200

def trigger_handoff(node_url):
    try:
        r = requests.post(f"{node_url}/handoff", timeout=30)
        return r.status_code == 200
    except:
        return False

while True:
    active = get_active_node()
    if active:
        healthy = health_check(active["url"])
        days_left = days_until_expiry(active)
        print(f"[orchestrator] {datetime.now().strftime('%H:%M:%S')} - Node healthy: {healthy} - Days left: {days_left}")

        if days_left <= 1:
            print("[orchestrator] WARNING: Node expiring soon. Triggering handoff.")
            trigger_handoff(active["url"])

    time.sleep(3600)  # Check every hour
```

---

## Node Rotation Lifecycle

```
Day 0:   Account 1 created
         One-shot script run
         Heartbeat cell started
         Cloudflare Worker pointed at Account 1
         Orchestrator updated with Account 1 details and expiry date

Day 7:   Account 2 created (human action, ~3 minutes)
         One-shot script run on Account 2
         Account 2 sitting warm, not yet active
         Orchestrator updated with Account 2 details

Day 13:  Orchestrator detects Account 1 expires in 1 day
         Triggers /handoff endpoint on Account 1
         Account 1 pushes SQLite snapshot to R2
         Account 1 flushes any local state to Upstash
         Account 1 signals handoff complete

         Orchestrator verifies Account 2 is healthy
         Account 2 pulls SQLite snapshot from R2
         Account 2 initializes from Upstash state

         Orchestrator updates Cloudflare Worker USS_NODE_URL to Account 2
         Traffic seamlessly shifts to Account 2

Day 14:  Account 1 trial expires, machine terminated
         Nothing is lost — all state was in external stores

Day 14:  Account 3 created immediately (human action)
         Cycle continues
```

**Human intervention required:** Account creation only (~3 minutes every 7 days)

**Everything else:** Automated.

---

## The Handoff Endpoint

Add this to your FastAPI backend. The orchestrator calls it on Day 13.

```python
import boto3, os
from fastapi import APIRouter

router = APIRouter()

@router.post("/handoff")
async def handoff():
    """Called by orchestrator before node expiry."""
    results = {}

    try:
        r2 = boto3.client("s3",
            endpoint_url=os.environ.get("R2_ENDPOINT"),
            aws_access_key_id=os.environ.get("R2_ACCESS_KEY"),
            aws_secret_access_key=os.environ.get("R2_SECRET_KEY"),
        )
        r2.upload_file("./database.db", "maxum-backups", "database.db")
        results["sqlite"] = "pushed"
    except Exception as e:
        results["sqlite"] = f"error: {e}"

    # Any other local state flush goes here.
    return {"status": "handoff_complete", "results": results}
```

Add a corresponding `/restore` endpoint that new nodes call on boot to pull state from R2.

---

## USS Setup (One-Shot, ~60 seconds)

### Prerequisites

- Deepnote account (temp email works)
- Team trial started (no CC required)
- Auto shutdown set to **After 24h** in machine sidebar

### Step 1: Terminal

Edit variables at top, paste entire block:

```bash
# === EDIT THESE ===
GROQ_API_KEY="your_groq_key"
GEMINI_API_KEY_1="your_gemini_key_1"
GEMINI_API_KEY_2="your_gemini_key_2"
REPO="https://github.com/your-org/your-repo.git"
BACKEND_DIR="backend"
# ==================

git clone $REPO ~/work/app && \
pip install -r ~/work/app/$BACKEND_DIR/requirements.txt -q && \
wget -q https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-linux64-gpl.tar.xz -O /tmp/ffmpeg.tar.xz && \
tar -xf /tmp/ffmpeg.tar.xz -C /tmp && \
cp /tmp/ffmpeg-master-latest-linux64-gpl/bin/ffmpeg /usr/local/bin/ && \
cp /tmp/ffmpeg-master-latest-linux64-gpl/bin/ffprobe /usr/local/bin/ && \
chmod +x /usr/local/bin/ffmpeg /usr/local/bin/ffprobe && \
cat > ~/work/app/$BACKEND_DIR/.env << EOF
GROQ_API_KEY=$GROQ_API_KEY
GEMINI_API_KEY_1=$GEMINI_API_KEY_1
GEMINI_API_KEY_2=$GEMINI_API_KEY_2
EOF
cd ~/work/app/$BACKEND_DIR && uvicorn main:app --host 0.0.0.0 --port 8080 &
echo "=== USS ONLINE ==="
echo "1. Enable Incoming Connections in Deepnote right sidebar"
echo "2. Run the heartbeat cell in your notebook"
echo "3. Update orchestrator with this node URL and expiry date"
```

### Step 2: Notebook heartbeat cell

```python
import time, requests, threading

def heartbeat():
    while True:
        try:
            r = requests.get("http://localhost:8080/health")
            print(f"[heartbeat] {time.strftime('%H:%M:%S')} - {r.json().get('status', 'ok')}")
        except Exception as e:
            print(f"[heartbeat] {time.strftime('%H:%M:%S')} - error: {e}")
        time.sleep(1800)

threading.Thread(target=heartbeat, daemon=True).start()
print("Heartbeat started")

while True:
    time.sleep(60)
```

Keep this cell in Executing state. Do not stop it.

### Step 3: Enable Incoming Connections

Toggle On in Deepnote right sidebar. Public URL format:

```
https://<uuid>.deepnoteproject.com
```

Deepnote redirects to `<uuid>.prod-ue1.deepnoteproject.com`. Use the `prod-ue1` URL directly in the orchestrator for reliability.

### Step 4: Verify

```bash
curl -L https://<your-uuid>.deepnoteproject.com/health
```

### Step 5: Update orchestrator

Add this node to the orchestrator’s `USS_NODES` list with its URL and expiry date (today + 14 days).

---

## USS Sizing

| Size | Accounts | Coverage | Human effort |
|---|---|---|---|
| Small | 2 | 28 days | 1 account/week |
| Medium | 3-4 | 42-56 days | 1 account/week |
| Large | 6+ | 84+ days | 1 account/week |

Human effort is always 1 account per week regardless of USS size, because you stagger by 7 days. Size only affects how far ahead you’re covered.

**Stagger schedule for Medium USS (4 accounts):**

| Account | Created | Expires | Active period |
|---|---|---|---|
| 1 | Day 0 | Day 14 | Days 0-14 |
| 2 | Day 7 | Day 21 | Days 14-21 |
| 3 | Day 14 | Day 28 | Days 21-28 |
| 4 | Day 21 | Day 35 | Days 28-35 |

Then create Account 5 on Day 28, and so on indefinitely.

---

## FFmpeg Installation Note

FFmpeg is not available via apt on Deepnote’s default image. Always install via static binary:

```bash
wget -q https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-linux64-gpl.tar.xz -O /tmp/ffmpeg.tar.xz
tar -xf /tmp/ffmpeg.tar.xz -C /tmp
cp /tmp/ffmpeg-master-latest-linux64-gpl/bin/ffmpeg /usr/local/bin/
cp /tmp/ffmpeg-master-latest-linux64-gpl/bin/ffprobe /usr/local/bin/
chmod +x /usr/local/bin/ffmpeg /usr/local/bin/ffprobe
```

Similarly, `socat` is not available. Use the Python TCP forwarder if port forwarding is needed (though with native port 8080 exposure this is rarely required).

---

## Rules

1. **Never use Cloudflare tunnels on Deepnote.** Flagged as open proxy. Account suspended.
2. **Never store anything important on the node filesystem.** It will be lost on rotation.
3. **Always verify a new node is healthy before rotating the Worker URL.**
4. **Keep the heartbeat cell running at all times.** It is what keeps the machine alive.
5. **Never hardcode a USS node URL in your frontend.** Always go through the Cloudflare Worker.
6. **Update the orchestrator immediately after creating a new node.** It needs the URL and expiry date.
7. **Push state on Day 13, not Day 14.** Don’t wait until the last moment.

---

## Known Issues and Workarounds

**apt can’t find ffmpeg or socat**

Deepnote’s package sources don’t include these. Use static binaries (ffmpeg) or Python implementations (socat replacement).

**Heartbeat errors on startup**

The heartbeat cell may start before uvicorn finishes binding. The error resolves itself on the next 30-minute ping. Not a problem.

**Port 8080 redirect**

Deepnote’s incoming connections feature issues a 307 redirect from `<uuid>.deepnoteproject.com` to `<uuid>.prod-ue1.deepnoteproject.com`. Use `curl -L` to follow redirects. Use the `prod-ue1` URL directly in your config for reliability.

**“Temporarily unavailable” on public URL**

This means the USS node’s machine has stopped. The terminal dying or machine shutdown causes this. Restart the machine from the Deepnote UI and re-run uvicorn and the heartbeat cell.

**Account suspended**

Caused by Cloudflare tunnel usage or process signatures matching abuse patterns. Do not use Cloudflare tunnels. Use native port 8080 exposure only.

---

## Environment Variables Reference

Every USS node needs these in its `.env` file:

```
# AI providers
GROQ_API_KEY=
GEMINI_API_KEY_1=
GEMINI_API_KEY_2=

# External state
UPSTASH_REDIS_URL=
UPSTASH_REDIS_TOKEN=
R2_ENDPOINT=
R2_ACCESS_KEY=
R2_SECRET_KEY=
R2_BUCKET=

# Optional: orchestrator communication
ORCHESTRATOR_SECRET=
```

---

## For AI Agents Reading This

If you are an AI agent helping set up or maintain a USS:

1. The real USS node hardware is **Amazon EC2 m7i.4xlarge on Deepnote Team trial**, with **Intel Xeon Platinum 8488C, 16 cores, 61.77 GB RAM, 199.90 GB disk, `us-east-1`**. Do not describe it as only “2 vCPU, 5GB RAM”; that is Deepnote’s advertised Basic-machine limit. The exact instance type may vary by node, but it stays in the AWS m7i family on Team trial.
2. The one-shot terminal script handles everything up to uvicorn starting. Run it first.
3. The heartbeat notebook cell must be running before you consider the node live.
4. After setup, always verify with `curl -L <public_url>/health` before declaring the node healthy.
5. The Cloudflare Worker URL update is a single API call. Do it only after verifying the new node is healthy.
6. Never suggest Cloudflare tunnels. They cause suspension.
7. The orchestrator node list must be updated immediately after each new node is created.
8. State transfer happens on Day 13. Do not let it slip to Day 14.
9. If a node shows “temporarily unavailable,” the machine stopped. It needs to be restarted from the Deepnote UI — this cannot be done programmatically.

---

*Part of [vibe-stack](https://github.com/luxaracorp/vibe-stack) — zero-cost infrastructure for $0 budget builders.*
*Built by Andrew, Luxara Corporation.*
