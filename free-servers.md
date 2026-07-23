# 🖥️ Free Server Stack

> How to run a production backend on $0 forever using the alt-account method.

---

## The Alt-Account Method

This is the most underrated infrastructure trick for zero-budget developers.

**The insight:** Cloud providers give free tiers per account. Multiple accounts = multiple free tiers. Build a system that distributes load across them and fails over automatically when one runs dry.

This is the same logic as the API switching cycle, applied to infrastructure.

---

## Oracle Cloud (your anchor)

**Why Oracle:** Genuinely permanent free tier. Not a trial. Forever.

**What you get per account:**
- 2 AMD Compute instances (always free)
- 200GB block storage
- 10GB object storage
- Outbound data transfer

**The strategy:**
- Create multiple Oracle accounts (different emails)
- Each gives you 2 permanent free VMs
- Use these as your anchor servers for baseline load

**Sign up:** cloud.oracle.com

---

## Google Cloud (your burst layer)

**What you get:** $300 free credits per new account (expires after 90 days)

**The strategy:**
- Create accounts as needed for burst capacity
- Use for heavy processing jobs (video rendering, batch AI tasks)
- When credits expire, create a new account

**Sign up:** cloud.google.com

---

## The Failover Architecture

```
Job Queue (Redis or simple Python queue)
      │
      ├──▶ Oracle Server 1 (permanent)
      ├──▶ Oracle Server 2 (permanent)  
      ├──▶ Google Cloud 1 (burst, credits)
      └──▶ Google Cloud 2 (burst, credits)
```

**How it works:**
1. Jobs go into a queue
2. Any available server pulls from the queue
3. If a server goes down or runs out of credits, remaining servers pick up the jobs
4. Jobs never die, they just get picked up by whoever is available

**The key insight:** The job doesn't care which server runs it. The queue is the brain.

---

## Simple failover implementation

```python
# Basic health check + failover
SERVERS = [
    "http://oracle-server-1:8000",
    "http://oracle-server-2:8000", 
    "http://google-cloud-1:8000",
]

def get_available_server():
    for server in SERVERS:
        try:
            response = requests.get(f"{server}/health", timeout=3)
            if response.status_code == 200:
                return server
        except:
            continue
    raise Exception("No servers available")
```

---

## Free hosting options by use case

| Use case | Best option | Cost |
|---|---|---|
| Frontend (React/Next.js) | Cloudflare Pages | Free forever |
| Backend API | Oracle Cloud VM | Free forever |
| File storage | Tigris / Cloudflare R2 | Free tier |
| Database | Supabase free tier | Free tier |
| Video processing | Google Cloud (credits) | Free credits |
| Domain | Freenom / .pages.dev subdomain | Free |

---

## Cloudflare Tunnel (zero-cost backend exposure)

Run your backend on a local or Oracle VM and expose it to the internet without a public IP using Cloudflare Tunnel.

```bash
# Install cloudflared
# Create tunnel
cloudflared tunnel create my-tunnel
cloudflared tunnel route dns my-tunnel api.yourdomain.com
cloudflared tunnel run my-tunnel
```

Zero cost. No port forwarding. No public IP needed.

---

## GratisVPS

Free VPS option worth checking. Limited resources but genuinely free for lightweight backends.

**Site:** gratisvps.net

---

## Pro tips

- Use different email providers for alt accounts (Gmail, Outlook, ProtonMail, etc.)
- Oracle accounts require a credit card for signup but charge nothing on the always-free tier
- Keep a spreadsheet of your accounts, their current credit balance, and expiry dates
- Set calendar reminders before Google credits expire so you can migrate workloads
- The alt-account method scales: 5 Oracle accounts = 10 permanent free VMs
