# X402 Facilitator (Solana + USDC)  
![X4A Banner](X4Abanner.png)

> **Pay-to-access & digital/physical marketplace on Solana using USDC**  
> Token-gated hybrid access • SQLite-backed catalog • Telegram notifications • Key rotation • Webhook events  

---

## Features

| Feature | Description |
|-------|-----------|
| **X402 Paywall** | Secure, signed 402 Payment Required flow for tiered access (`basic`, `premium`, `ai-agent`) |
| **Marketplace** | Full catalog + checkout system with stock tracking and order history |
| **Digital & Physical Items** | Sell services, access, or real-world goods (with optional shipping prefill) |
| **Token Gating** | Free access or discount for holders of a specific SPL token |
| **Key Rotation** | Multiple signing keys with hot-swap via `/admin/rotate` |
| **SQLite Persistence** | Users, payments, entitlements, products, orders — all in `data/x402.db` |
| **Telegram Bot** | Inline notifications for payments, unlocks, and order confirmations |
| **Webhook Events** | Real-time `x402.paid`, `x402.freeUnlock`, `x402.checkout.paid` |
| **Admin Panel** | Add products directly from `/admin` (password-protected) |
| **Client SDK Ready** | `/config`, `/paywall`, `/checkout/:sku` — ready for wallet integrations |

---

## Quick Start

```bash
# Clone & install
git clone https://github.com/yourname/x402-facilitator.git
cd x402-facilitator
npm install

# Copy env template
cp .env.example .env
```

### Configure `.env`

```env
# Server
PORT=3001

# Solana
SOLANA_RPC=https://api.mainnet-beta.solana.com
USDC_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v

# Wallet (REQUIRED)
SERVER_WALLET_PRIVATE_KEY=your_base58_or_json_secret_key

# OR use multiple rotating keys
# ROTATING_KEYS_JSON='["key1", "key2", ...]'

# Tiers (USDC base units, 6 decimals)
PRICE_BASIC_BASE=500        # 0.0005 USDC
PRICE_PREMIUM_BASE=1000000  # 1.0 USDC
PRICE_AI_AGENT_BASE=10000000 # 10 USDC

# Tier durations (seconds)
TIER_DURATION_BASIC=86400
TIER_DURATION_PREMIUM=604800
TIER_DURATION_AI_AGENT=2592000

# Token Gating (optional)
GATE_MINT=your_nft_or_token_mint
GATE_MIN_BALANCE=1
GATE_FREE=false
GATE_DISCOUNT_BPS=2500  # 25% off

# Webhook (optional)
WEBHOOK_URL=https://yourapp.com/webhooks/x402
X402_WEBHOOK_SECRET=your_secret

# Admin
ADMIN_TOKEN=your_admin_token_for_rotation
ADMIN_WEB_TOKEN=your_web_admin_token

# Telegram Bot (optional)
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_BOT_ADMIN=123456789
TELEGRAM_BOT_NAME=@X4AFacilitatorBot

# Metadata
FACILITATOR_NAME=X4A Agent Facilitator
PROVIDER_SITE=https://x4a.app
PROVIDER_CONTACT=ops@x4a.app
```

> **Warning: Never commit `.env` or private keys**

---

## Run

```bash
npm start
```

Server starts at: [`http://localhost:3001`](http://localhost:3001)

---

## Endpoints

| Path | Method | Description |
|------|--------|-----------|
| `GET /` | HTML | Landing page (`templates/index.html` or `index.html`) |
| `GET /config` | JSON | Public config for wallets |
| `GET /paywall?tier=premium&wallet=So1...` | 402/200 | Tier access paywall |
| `GET /api/catalog` | JSON | List all products |
| `GET /api/catalog/:id` | JSON | Get one item |
| `GET /checkout/:sku?wallet=So1...&ref=abc&return=https://...` | 402/200 | Buy item |
| `POST /checkout/:sku/prefill` | JSON | Prefill shipping (physical) |
| `GET /api/status/:wallet` | JSON | Check entitlement |
| `GET /health` | JSON | Health check |
| `GET /x402-meta` | JSON | Metadata for explorers |
| `POST /admin/add-product` | JSON | Add product (via admin UI) |
| `GET /admin` | HTML | Admin panel (add products) |
| `POST /admin/rotate` | JSON | Rotate signing key |
| `POST /webhooks/x402` | JSON | Receive external events |

---

## Admin Panel

1. Set `ADMIN_WEB_TOKEN=yourpassword` in `.env`
2. Go to [`/admin`](http://localhost:3001/admin)
3. Enter token → Add products instantly

> Products appear in `/api/catalog` and `/catalog` (if frontend exists)

---

## Telegram Bot

Link wallet:
```
/link So11111111111111111111111111111111111111112
```

Check status:
```
/status
/tier
```

Receive instant confirmations on payment & delivery.

---

## Database (`data/x402.db`)

```sql
users         → wallet ↔ telegram_id
payments      → payment history
entitlements  → active tier + expiry
products      → catalog items
orders        → purchase records
```

Back up `data/` folder regularly.

---

## Webhook Events

Sent to `WEBHOOK_URL` with `X-X402-Token` header.

```json
{
  "event": "x402.paid",
  "tier": "premium",
  "amountBase": 1000000,
  "wallet": "So1...",
  "receipt": { ... }
}
```

Supported events:
- `x402.paid`
- `x402.freeUnlock`
- `x402.checkout.paid`

---

## Security

- **Key Rotation**: Rotate via `POST /admin/rotate` with `ADMIN_TOKEN`
- **Admin UI**: Protected by `ADMIN_WEB_TOKEN`
- **Webhook Auth**: `X-X402-Token`
- **SQLite WAL**: Safe concurrent access
- **Helmet + CSP**: Secure headers

---

## Folder Structure

```
├── data/                  → SQLite DB
├── templates/             → index.html (optional)
├── static/                → CSS/JS/images
├── X4Abanner.png          → Banner
├── index.js               → Main server
├── .env                   → Config
└── package.json
```

---

## Customize

### Add Frontend
Place `index.html` in:
- `templates/index.html` (recommended)
- or project root

Use `/api/catalog` and `/checkout/:sku` in your UI.

### Custom Tiers
Update `.env`:
```env
PRICE_MYTIER_BASE=5000000
TIER_DURATION_MYTIER=2592000
```
Then request: `/paywall?tier=mytier`

---

## License

MIT © X4A

---

**Built for the on-chain access economy**  
*Pay once. Access forever.*
