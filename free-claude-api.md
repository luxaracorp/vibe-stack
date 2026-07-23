# 🔑 Free Claude API Methods

> Real Claude models. No credit card. Updated as methods change.

The core trick: Claude Code lets you swap its API endpoint via `settings.json`. Point it at any service that mimics the Anthropic API format and Claude Code thinks it's talking to Anthropic directly.

---

## How the base URL swap works

Your `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "your-key-here",
    "ANTHROPIC_BASE_URL": "https://the-service.com",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "model": "claude-sonnet-4-6",
  "effortLevel": "medium"
}
```

That's it. Change the URL and key, keep everything else. Claude Code routes all requests through the new endpoint.

---

## The API Switching Cycle

The single most important method in this repo.

**The problem:** Free API providers run out of credits, go down, or get rate limited.  
**The solution:** Never depend on one source. Stack multiple providers and rotate.

When Provider A fails → switch to Provider B → when B fails → switch to C → cycle back.

Same settings.json edit each time. Takes 30 seconds to switch.

This is how I've maintained a zero-cost Claude Code setup across multiple projects.

---

## Providers (ranked by reliability)

### 🥇 FreeModel.dev
- **URL:** `https://cc.freemodel.dev`
- **Models:** Claude Sonnet 5, Opus 4.8, Fable 5
- **Free tier:** Weekly credits on signup + Telegram verification
- **Reset:** Weekly
- **Notes:** Best model selection. Occasionally pauses new account promos during infrastructure stress. Check their Discord/Telegram for status. The gold standard when working.

### 🥈 NaraRouter
- **URL:** `https://router.bynara.id`
- **Models:** Claude Sonnet 5, Opus 4.7/4.8, Fable 5 + 30+ others
- **Free tier:** Daily token reset (check dashboard)
- **Reset:** Daily
- **Key format:** `sk-nry-xxxxxxxxxx`
- **Notes:** OpenAI compatible format. Use exact model IDs from `/v1/models`. Good daily backup.

### 🥉 freeclaude.org
- **URL:** `https://api.freeclaude.org`
- **Models:** Claude Sonnet 4.6, Opus 4.5–4.8, Haiku 4.5
- **Free tier:** $50 credits on signup + phone verification
- **Notes:** Anthropic native API format. Requires Chinese phone number for full credits (see tip below). Can get overloaded (503 errors) during peak hours.

**💡 Tip:** Free Chinese phone numbers for verification are available through certain online services. Search for "free Chinese virtual number" — this is legal and widely used.

### Aerolink.lat
- **URL:** Check their site for current endpoint
- **Models:** Claude models
- **Free tier:** Monthly plan on signup
- **Notes:** Similar model to FreeModel. Good backup when FreeModel is down.

### OpenAPIs (when active)
- **URL:** `https://api.openapis.online/anthropic`
- **Key:** `admin` (yes literally)
- **Models:** Claude Opus 4.7, Sonnet 4.6, Haiku 4.5
- **Notes:** Zero signup. Occasionally under maintenance. Check GitHub repo status before relying on it.

### Anthropic Console (emergency backup)
- **URL:** Default Anthropic
- **Free tier:** $5 credits on new account signup, no credit card required
- **Notes:** Real Anthropic. Small credits but zero instability. Use for critical work when everything else is down.

---

## Getting model names right

Every provider uses slightly different model ID formats. Always check:

```powershell
# PowerShell
Invoke-WebRequest -Uri "https://provider-url/v1/models" -Headers @{"Authorization" = "Bearer your-key"} | Select-Object -ExpandProperty Content
```

Use the exact `id` field from the response in your settings.json `model` field.

---

## When nothing works

Stack in this order:
1. Try each provider above in sequence
2. Switch to GLM 5.2 free on NaraRouter for non-critical tasks (`"model": "glm-5.2-free"`)
3. Use Gemini CLI with Google AI Studio free key as last resort (not Claude but capable)
4. Create a new account on any provider with a fresh email

---

## Pro tips

- **Never depend on one provider.** Ever.
- Keep your settings.json backed up with each provider's config commented out. Switching takes 10 seconds.
- Free providers go down on weekends and evenings (high traffic). Do critical work on weekday mornings.
- If a provider gives 503, wait 30 minutes and try again before switching. Often temporary.
- The model quality gap between Sonnet 4.6 and GLM 5.2 is real but GLM is surprisingly capable for codebase analysis tasks.
