# 🖥️ Free Server Stack

> How to run a production backend on $0 using the alt-account method.

---

## USS — Unified Server Stack (Recommended)

The most powerful free backend method discovered to date. Tested and documented by Luxara.

**What it is:** Deepnote is a cloud data notebook platform. Their Team trial gives you real AWS-backed compute with no credit card required — just an email address. Temp emails work. Alt accounts are explicitly allowed by Deepnote.

**What you get per USS node:**
- 2 vCPU, 5GB RAM
- Public HTTPS URL via native port 8080 exposure
- Root terminal access
- 48-hour continuous runtime (Team trial), resettable via heartbeat cell
- Persists after closing browser tab or logging out
- Zero cost, no credit card, no phone number

**One USS node lasts 14 days** (Team trial duration). For continuous uptime, rotate accounts.

**Medium USS (30 days continuous):** 3 accounts started on days 1, 10, and 20.

---

### USS Setup (One-Shot, ~60 seconds)

**Prerequisites:**
- Create a Deepnote account (temp email works)
- Start the Team trial (no CC required)
- Set Auto shutdown to **After 24h** in the machine sidebar

**Step 1: Paste this in a new Deepnote terminal**

Edit the variables at the top, then paste the entire block:

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
```

**Step 2: Run this notebook cell**

Create a new cell in any Deepnote notebook and run it. Keep it executing — do not stop it.

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

**Step 3: Enable Incoming Connections**

Toggle **Incoming connections** On in the Deepnote right sidebar. Your public URL appears:
https://<uuid>.deepnoteproject.com

**Verify from anywhere:**

```bash
curl -L https://<your-uuid>.deepnoteproject.com/health
```

---

### USS Rules

- **Never use Cloudflare tunnels on Deepnote.** Their abuse detector flags it as an open proxy. Use the native port 8080 exposure only.
- **All persistent state must live outside the USS node.** Use Upstash Redis for queues, Cloudflare R2 for files. The Deepnote filesystem resets on node rotation.
- **FFmpeg is not in Deepnote's default apt sources.** The setup script installs it via static binary from BtbN's GitHub releases.
- **The heartbeat cell must stay in Executing state.** Closing the browser tab is fine — the machine keeps running. The cell is what keeps the kernel alive.

---

### USS Rotation

One node lasts 14 days. To run indefinitely:

1. Create a new Deepnote account before the current node expires
2. Run the one-shot setup on the new account
3. Update your frontend/client with the new public URL
4. Stagger account creation so you always have overlap

Keep a note of each node's URL and trial expiry date. Set a reminder 4 days before expiry.

---

## Google Cloud (burst layer)

**What you get:** $300 free credits per new account (expires after 90 days)

**The strategy:**
- Create accounts as needed for burst capacity
- Use for heavy processing jobs (video rendering, batch AI tasks)
- When credits expire, create a new account

**Sign up:** cloud.google.com

---

## The Failover Architecture
Job Queue (Upstash Redis)
│
├──▶ USS Node 1 (active trial)
├──▶ USS Node 2 (next trial, starts day 10)
└──▶ USS Node 3 (overlap, starts day 20)

Jobs stay in the Redis queue until a node picks them up. Node rotation is transparent to the queue.

---

## Free hosting options by use case

| Use case | Best option | Cost |
|---|---|---|
| Frontend (React/Next.js) | Cloudflare Pages | Free forever |
| Backend API | USS (Deepnote) | Free, rotating |
| File storage | Cloudflare R2 | Free tier |
| Job queue | Upstash Redis | Free tier |
| Database | Supabase free tier | Free tier |
| Video processing | USS or Google Cloud credits | Free |
| Domain | .pages.dev subdomain | Free |

---

## Pro tips

- Use different email providers for alt accounts (Gmail, Outlook, ProtonMail, temp-mail services)
- Deepnote explicitly allows alternative accounts — this is not against their rules
- The alt-account method scales: more accounts = more parallel USS nodes
- Always test a new USS node with a health check before rotating your frontend to it

⚠️ USS uses Deepnote's Team trial. This is a gray-line method — technically within their rules when used correctly (native port exposure, no tunneling) but they could change their trial policies at any time. Always have a backup node ready.
