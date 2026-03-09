# Auto-Moderation Bot with AI Scam Detection

> A production-ready Discord moderation bot with **3-layer scam detection**. Protects crypto communities from seed phrase theft, fake admin impersonation, suspicious links, wallet address harvesting, and airdrop scams. Powered by pattern matching + Claude AI as a second opinion.

![Python](https://img.shields.io/badge/Python-3.12-blue) ![discord.py](https://img.shields.io/badge/discord.py-2.7.1-7289da) ![anthropic](https://img.shields.io/badge/anthropic-0.80.0-orange)

---

## What This Bot Does

Every message posted in your Discord server goes through a 3-layer detection funnel:

1. **Layer 1 — Trusted role allowlist**: If the message author has a trusted role (e.g. Admin, Moderator), the message is immediately marked `SAFE` and skipped. Zero processing cost.
2. **Layer 2 — Pattern matching**: The message is checked against 14 high-risk phrases, 6 suspicious domains, and a regex for Ethereum wallet addresses. Fast, offline, no API calls. `HIGH_RISK` if 2+ matches. `SUSPICIOUS` if 1 match.
3. **Layer 3 — Claude AI second opinion**: Only `SUSPICIOUS` messages reach the AI. Claude classifies the message as `SAFE`, `SUSPICIOUS`, or `SCAM`. If `SCAM` → upgrade to `HIGH_RISK`. AI errors always default to `SUSPICIOUS`, never `SAFE`.

**Actions:**
- `HIGH_RISK` → message is **auto-deleted**, action logged to mod channel with embed
- `SUSPICIOUS` → message is **kept but flagged** in mod channel for human review
- `SAFE` → no action, no log

---

## Features

| Feature | Details |
|---------|---------|
| 3-layer detection funnel | Allowlist → patterns → AI (in order of cheapness) |
| 14 high-risk phrases | Seed phrases, fake admins, fake airdrops, wallet harvesting |
| 6 suspicious domain patterns | bit.ly, tinyurl.com, wallet-verify.io, claim-airdrop.xyz, etc. |
| Wallet address detection | Ethereum address regex (0x + 40 hex chars) |
| Claude AI second opinion | Only for ambiguous SUSPICIOUS messages (~10% of all traffic) |
| Auto-delete HIGH_RISK | Message removed immediately, reason logged |
| Embed logging | Rich mod channel embeds with message content, author, reason, risk level |
| Hourly hot-reload | Scam patterns reload from disk every hour — no bot restart needed |
| Fail-safe defaults | All AI/network failures → SUSPICIOUS, never SAFE |
| Pre-compiled regex | Patterns compiled at startup, not per-message — O(1) matching |
| Trusted role allowlist | Admins/mods bypass all checks — configurable via `scam_patterns.json` |

---

## How It Works

```
New Discord Message
        |
        v
+---------------------------------+
|  Layer 1: Trusted Role Check    |  Author has trusted role?
|  pattern_detector.py            |  YES -> SAFE (skip all checks)
+----------+----------------------+
           | NOT trusted
           v
+---------------------------------+
|  Layer 2: Pattern Matching      |  Check: phrases, domains, wallets
|  pattern_detector.detect_scam   |  2+ matches -> HIGH_RISK
|                                 |  1 match   -> SUSPICIOUS
|                                 |  0 matches -> SAFE
+----------+----------------------+
           | SUSPICIOUS only
           v
+---------------------------------+
|  Layer 3: Claude AI             |  "SAFE / SUSPICIOUS / SCAM?"
|  ai_classifier.py               |  SCAM -> upgrade to HIGH_RISK
|                                 |  Error -> stay SUSPICIOUS (fail-safe)
+----------+----------------------+
           |
    +------+------+
    | HIGH_RISK           | SUSPICIOUS
    v                     v
Auto-delete message   Keep message
Log to mod channel    Flag in mod channel
                      (human review)
```

### Why 3 Layers?

- **Layer 1** (free): Eliminates ~40% of messages (staff/admins)
- **Layer 2** (offline, microseconds): Eliminates ~50% of remaining with no API cost
- **Layer 3** (AI, ~$0.0001/message): Only runs on the ambiguous ~10% — keeps API costs minimal while catching sophisticated scams that pattern matching would miss

---

## Scam Patterns (`scam_patterns.json`)

This file is **hot-reloaded every hour**. Edit it without restarting the bot.

```json
{
  "high_risk_phrases": [
    "send me your seed phrase",
    "dm me for whitelist",
    "connect your wallet to claim",
    "limited time airdrop",
    "guaranteed 100x",
    "i am the admin",
    "verify your wallet",
    "free crypto giveaway",
    "click here to claim",
    "your wallet is compromised",
    "enter your private key",
    "secret alpha group",
    "double your crypto",
    "send eth to receive eth"
  ],
  "suspicious_domains": [
    "bit.ly",
    "tinyurl.com",
    "wallet-verify.io",
    "claim-airdrop.xyz",
    "free-nft.io",
    "metamask-support.com"
  ],
  "trusted_role_ids": [
    123456789,
    987654321
  ],
  "wallet_regex": "0x[a-fA-F0-9]{40}"
}
```

Add your own phrases and domains. Add your Admin/Mod role IDs to `trusted_role_ids`.

---

## File Structure

```
project3-scam-mod-bot/
├── discord_mod.py       Main bot: event handlers, actions, hot-reload task
├── config.py            Loads env vars; safe int conversion for LOG_CHANNEL_ID
├── pattern_detector.py  Layer 2: phrase/domain/wallet detection; RiskLevel enum
├── ai_classifier.py     Layer 3: Claude AI second opinion (SUSPICIOUS only)
├── scam_patterns.json   Editable pattern database (hot-reloaded hourly)
├── requirements.txt     Pinned dependencies
├── .env.example         Template for credentials
├── .gitignore           Excludes .env, venv/, __pycache__/
└── .python-version      Pins Python 3.12
```

---

## Tech Stack

| Library | Version | Purpose |
|---------|---------|---------|
| discord.py | 2.7.1 | Discord bot framework |
| anthropic | 0.80.0 | Claude AI API client |
| python-dotenv | 1.0.1 | Load `.env` file |
| regex | 2024.11.6 | Advanced regex (wallet detection) |
| loguru | 0.7.2 | Structured logging |

**AI Model:** `claude-sonnet-4-6`
**Python:** 3.12

---

## Installation

### Step 1 — Create Discord bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. Click **New Application** → give it a name
3. Go to **Bot** → click **Reset Token** → copy token
4. Under **Privileged Gateway Intents** → enable:
   - **Server Members Intent**
   - **Message Content Intent**
5. Go to **OAuth2 → URL Generator** → select `bot` scope → select permissions:
   - `Read Messages / View Channels`
   - `Manage Messages` (to delete scam messages)
   - `Send Messages`
   - `Embed Links`
6. Open the generated URL to invite the bot to your server

### Step 2 — Create a mod log channel

1. In Discord, create a private channel (e.g. `#mod-log`)
2. Right-click the channel → **Copy Channel ID** (requires Developer Mode in Discord settings)
3. Save this ID — it goes in `LOG_CHANNEL_ID`

### Step 3 — Install

```bash
cd project3-scam-mod-bot
pip install -r requirements.txt
```

### Step 4 — Configure

```bash
cp .env.example .env
```

Edit `.env`:

```env
DISCORD_TOKEN=MTIzNDU2Nzg5MDEyMzQ1Njc4.Gxxxxx.xxxxxxxxxxxxxxxxxxxxxx
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LOG_CHANNEL_ID=1234567890123456789
```

### Step 5 — Add trusted role IDs

Open `scam_patterns.json` and add your Admin/Moderator role IDs:

```json
"trusted_role_ids": [1234567890, 9876543210]
```

To find a role ID: Discord → Server Settings → Roles → right-click role → **Copy Role ID**.

### Step 6 — Start

```bash
python discord_mod.py
```

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DISCORD_TOKEN` | Yes | Bot token from Discord Developer Portal |
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key for Claude AI |
| `LOG_CHANNEL_ID` | Yes | Channel ID where mod actions are logged |

---

## Mod Channel Log Embeds

Every action is posted as a rich embed in your mod channel:

**HIGH_RISK embed (red):**
- Author username and ID
- Full message content
- Risk level: HIGH_RISK
- Detection reasons (which phrases/domains matched)
- Timestamp
- Action taken: Message deleted

**SUSPICIOUS embed (yellow):**
- Same fields as above
- Action: Flagged for human review (message not deleted)

---

## Deployment

### Railway (Recommended)

1. Push fork to GitHub
2. Railway → New Project → Deploy from GitHub → select repo
3. Add env vars in Variables tab
4. Bot stays always-on (Railway Hobby = $5/month)

### VPS with PM2

```bash
pip install -r requirements.txt
pm2 start "python discord_mod.py" --name scam-mod-bot
pm2 save && pm2 startup
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot not detecting messages | Enable **Message Content Intent** in Discord Developer Portal |
| `Missing Permissions` error | Bot needs `Manage Messages` permission in the server |
| LOG_CHANNEL_ID not working | Enable Developer Mode in Discord → User Settings → Advanced |
| Patterns not reloading | Check `scam_patterns.json` is valid JSON (use jsonlint.com) |
| Too many false positives | Add legit domains to `trusted_role_ids` or remove phrases from `high_risk_phrases` |
| AI always returns SUSPICIOUS | Check `ANTHROPIC_API_KEY` is valid — errors default to SUSPICIOUS by design |

---

## Adding Custom Scam Patterns

Edit `scam_patterns.json` at any time. The bot reloads it automatically every hour.

To force immediate reload without restarting: restart the bot process.

**Adding a new scam phrase:**
```json
"high_risk_phrases": [
  "existing phrase",
  "your new scam phrase here"
]
```

**Adding a suspicious domain:**
```json
"suspicious_domains": [
  "existing-domain.com",
  "new-scam-site.xyz"
]
```

---

## Security Notes

- All AI and network failures default to `SUSPICIOUS` — the bot never assumes safety on error
- Trusted role allowlist prevents false positives for staff
- Pattern matching runs entirely offline — no API calls for obvious scams
- `LOG_CHANNEL_ID` is validated as an integer at startup — misconfiguration is caught early
