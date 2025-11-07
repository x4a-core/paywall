# Multi-Chain Upgrade Guide

Complete guide for upgrading X4A Marketplace to support multiple blockchain networks (Solana + Base).

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture Changes](#architecture-changes)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Database Migration](#database-migration)
- [Testing](#testing)
- [Deployment](#deployment)
- [Rollback Plan](#rollback-plan)

---

## 🎯 Overview

### What's New?

The multi-chain upgrade enables X4A Marketplace to accept payments on **multiple blockchain networks** while maintaining full **X402 Protocol compliance**.

**Supported Networks:**
- 🟣 **Solana** (Original) - SPL USDC
- 🔵 **Base** (New) - ERC20 USDC

### Key Benefits

✅ **Broader User Base** - Accept payments from EVM wallet users  
✅ **Lower Fees** - Base offers cheaper transaction costs than Ethereum  
✅ **Full X402 Compliance** - Same protocol across all chains  
✅ **Backward Compatible** - Existing Solana-only setups continue working  

---

## 🏗️ Architecture Changes

### Before (v1.x)
```
┌─────────┐
│ Client  │
└────┬────┘
     │ Solana USDC Only
     ▼
┌─────────────┐
│   Server    │
│  (X402)     │
└─────────────┘
     │
     ▼
┌─────────────┐
│  Database   │
└─────────────┘
```

### After (v2.0)
```
┌─────────┐
│ Client  │
└────┬────┘
     │ Chain Selection
     ├─────────┬─────────┐
     ▼         ▼         ▼
  Solana    Base     (Future)
     │         │         │
     └────┬────┴────┬────┘
          ▼         ▼
    ┌──────────────────┐
    │   Chain Layer    │
    │   (chains.js)    │
    └────────┬─────────┘
             ▼
    ┌──────────────────┐
    │   X402 Server    │
    └────────┬─────────┘
             ▼
    ┌──────────────────┐
    │    Database      │
    │  (+ network col) │
    └──────────────────┘
```

### New Components

#### 1. **Chain Abstraction Layer** (`chains.js`)
```javascript
export const CHAINS = {
  solana: {
    name: 'solana',
    network: 'solana-mainnet',
    rpc: process.env.SOLANA_RPC,
    usdcMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
    decimals: 6,
    verifyPayment: async (txSig, transfers) => { /* ... */ }
  },
  base: {
    name: 'base',
    network: 'base-mainnet',
    rpc: process.env.BASE_RPC,
    usdcContract: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
    decimals: 6,
    verifyPayment: async (txHash, transfers) => { /* ... */ }
  }
};
```

#### 2. **Chain Selector UI** (`index.html`)
```javascript
function showChainSelector(chains) {
  // Modal that lets users pick Solana or Base
  // Returns selected chain name
}
```

#### 3. **Network Tracking** (`db.js`)
```sql
-- New columns in existing tables
ALTER TABLE purchases ADD COLUMN network TEXT DEFAULT 'solana';
ALTER TABLE payments ADD COLUMN network TEXT DEFAULT 'solana';
ALTER TABLE secondary_purchases ADD COLUMN network TEXT DEFAULT 'solana';
```

---

## 📦 Prerequisites

### Required

- Node.js v18+ (LTS)
- npm v9+
- Existing X4A Marketplace v1.x
- Solana wallet with keypair
- **NEW:** EVM wallet (MetaMask/Coinbase Wallet) for Base

### Environment Setup

#### Solana (Existing)
```bash
SOLANA_RPC=https://api.mainnet-beta.solana.com
USDC_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
SERVER_WALLET_PRIVATE_KEY=[...]
MARKET_ADMIN_WALLET=your_solana_wallet
```

#### Base (New)
```bash
ENABLE_BASE=true
BASE_RPC=https://mainnet.base.org
BASE_USDC_CONTRACT=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
BASE_PAYMENT_WALLET=0xYourBaseWallet
MARKET_ADMIN_WALLET_BASE=0xYourAdminWallet
```

---

## 🚀 Installation

### Step 1: Backup
```bash
# Backup your database
cp data/x402.db data/x402.db.backup

# Backup your .env
cp .env .env.backup

# Commit current state
git add -A
git commit -m "Pre multi-chain upgrade backup"
```

### Step 2: Install Dependencies
```bash
npm install ethers@6 viem @coinbase/onchainkit
```

### Step 3: Add New Files

Create `chains.js`:
```bash
touch chains.js
```

Copy the content from the upgrade package.

### Step 4: Update Existing Files

Replace sections in:
- `server.js` - Paywall & Tools handlers
- `index.html` - Payment handlers
- `db.js` - Add network columns
- `bot.js` - Network badges

---

## ⚙️ Configuration

### Option 1: Solana Only (No Changes Needed)
```bash
# .env (no changes)
ENABLE_SOLANA=true
ENABLE_BASE=false
```

Your marketplace continues working as before.

### Option 2: Solana + Base
```bash
# .env (add these)
ENABLE_BASE=true
BASE_RPC=https://mainnet.base.org
BASE_USDC_CONTRACT=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
BASE_PAYMENT_WALLET=0x742d35Cc6634C0532925a3b844Bc9e7595f0bEcD
MARKET_ADMIN_WALLET_BASE=0xAdminWalletForFees
```

### Base Wallet Setup

#### For Payment Wallet:
```bash
# Generate new Base wallet (or use existing)
# Option 1: MetaMask
# Create new account -> Copy address

# Option 2: Command line (ethers)
node -e "const ethers = require('ethers'); const w = ethers.Wallet.createRandom(); console.log('Address:', w.address); console.log('Private:', w.privateKey);"
```

#### Fund with Base ETH (for gas):
```bash
# Bridge ETH to Base via:
# - https://bridge.base.org
# - Or buy directly on Base
```

---

## 🗄️ Database Migration

### Automatic Migration

The database migration runs automatically on startup:
```bash
npm start
```

Output:
```
Created data directory at: /path/to/data
Database initialized at: /path/to/data/x402.db
✅ Migrated: added purchases.network
✅ Migrated: added payments.network
✅ Migrated: added secondary_purchases.network
```

### Manual Migration (Optional)

If you prefer manual control:
```sql
-- Connect to database
sqlite3 data/x402.db

-- Add network columns
ALTER TABLE purchases ADD COLUMN network TEXT DEFAULT 'solana';
ALTER TABLE payments ADD COLUMN network TEXT DEFAULT 'solana';
ALTER TABLE secondary_purchases ADD COLUMN network TEXT DEFAULT 'solana';

-- Create indexes
CREATE INDEX IF NOT EXISTS idx_purchases_network ON purchases (network, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_payments_network ON payments (network, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_secondary_purchases_network ON secondary_purchases (network, created_at DESC);

-- Verify
SELECT name FROM pragma_table_info('purchases') WHERE name='network';
-- Should return: network

.exit
```

### Verify Migration
```bash
npm run verify-db
```

Or manually:
```bash
sqlite3 data/x402.db "SELECT COUNT(*) FROM purchases WHERE network IS NOT NULL;"
```

---

## 🧪 Testing

### Test Suite

#### 1. Database Migration Test
```bash
npm test -- db-migration
```

#### 2. Chain Initialization Test
```bash
npm test -- chains
```

#### 3. Solana Payment Test
```bash
curl -X GET "http://localhost:3001/buy/test-listing-id?wallet=YOUR_SOLANA_WALLET&network=solana"
# Should return 402 with Solana challenge
```

#### 4. Base Payment Test
```bash
curl -X GET "http://localhost:3001/buy/test-listing-id?wallet=0xYourBaseWallet&network=base"
# Should return 402 with Base challenge
```

#### 5. Paywall Multi-Chain Test
```bash
curl "http://localhost:3001/paywall?tier=premium&wallet=YOUR_WALLET&network=solana"
curl "http://localhost:3001/paywall?tier=premium&wallet=0xYourWallet&network=base"
```

#### 6. Tools Multi-Chain Test
```bash
# Solana
curl "http://localhost:3001/weather?city=Paris&network=solana"

# Base
curl "http://localhost:3001/weather?city=Paris&network=base"
```

### Integration Test Script

Create `test-multi-chain.js`:
```javascript
import fetch from 'node-fetch';

const BASE_URL = 'http://localhost:3001';

async function testChainSelector() {
  console.log('Testing chain selection...');
  
  const config = await fetch(`${BASE_URL}/config`).then(r => r.json());
  console.log('✅ Available chains:', config.chains.map(c => c.name));
  
  const solanaChallenge = await fetch(`${BASE_URL}/paywall?tier=basic&wallet=test&network=solana`);
  const baseChallenge = await fetch(`${BASE_URL}/paywall?tier=basic&wallet=0xtest&network=base`);
  
  console.log('✅ Solana challenge status:', solanaChallenge.status);
  console.log('✅ Base challenge status:', baseChallenge.status);
}

testChainSelector().catch(console.error);
```

Run:
```bash
node test-multi-chain.js
```

### Manual Testing Checklist

- [ ] Solana marketplace purchase
- [ ] Base marketplace purchase
- [ ] Solana paywall unlock
- [ ] Base paywall unlock
- [ ] Solana tool payment
- [ ] Base tool payment
- [ ] Chain selector modal displays
- [ ] Network badges in Telegram
- [ ] Explorer links work (Solscan/Basescan)
- [ ] Inventory shows correct network
- [ ] Secondary market on both chains

---

## 🚢 Deployment

### Development
```bash
# Start dev server
npm run dev

# Test both chains
# Visit http://localhost:3001
```

### Staging
```bash
# Set staging env vars
export NODE_ENV=staging
export ENABLE_BASE=true

# Deploy to staging
npm run deploy:staging
```

### Production
```bash
# 1. Set production env vars
export NODE_ENV=production
export ENABLE_BASE=true
export BASE_RPC=https://mainnet.base.org
# ... (all other vars)

# 2. Build
npm run build

# 3. Database backup
cp data/x402.db data/x402.db.pre-multichain

# 4. Deploy
npm run deploy:production

# 5. Verify
curl https://x4a.app/config | jq '.chains'
```

### Docker Deployment
```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production

COPY . .

# Expose port
EXPOSE 3001

CMD ["npm", "start"]
```

Deploy:
```bash
docker build -t x4a-marketplace:2.0.0 .
docker run -d \
  --name x4a-marketplace \
  -p 3001:3001 \
  -v $(pwd)/data:/app/data \
  --env-file .env \
  x4a-marketplace:2.0.0
```

---

## 🔄 Rollback Plan

### If Issues Arise

#### Step 1: Stop Server
```bash
pm2 stop x4a-marketplace
# or
docker stop x4a-marketplace
```

#### Step 2: Restore Database
```bash
cp data/x402.db.backup data/x402.db
```

#### Step 3: Restore Code
```bash
git checkout v1.x
npm install
```

#### Step 4: Restart
```bash
npm start
```

### Partial Rollback (Disable Base Only)
```bash
# .env
ENABLE_BASE=false

# Restart
pm2 restart x4a-marketplace
```

Solana payments continue working. Base disabled.

---

## 📊 Monitoring

### Health Checks
```bash
# Check both chains are responding
curl http://localhost:3001/health

# Expected response:
{
  "status": "OK",
  "chains": [
    {"name": "solana", "status": "connected"},
    {"name": "base", "status": "connected"}
  ]
}
```

### Metrics to Track

- Payment success rate per chain
- Average transaction time per chain
- Failed verification reasons
- Chain selection distribution
- Gas costs (Base)

### Logging

Add chain-specific logging:
```javascript
// server.js
console.log(`✅ Payment verified on ${chain.network}: ${verified.txSig}`);
console.log(`⚠️ Payment failed on ${chain.network}: ${verified.reason}`);
```

---

## 🆘 Support

### Common Issues

See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)

### Get Help

- **GitHub Issues**: [Report a bug](https://github.com/yourusername/x4a-marketplace/issues)
- **Discord**: [Join community](https://discord.gg/x4a)
- **Email**: ops@x4a.app

---

## ✅ Post-Upgrade Checklist

- [ ] Database migration successful
- [ ] Both chains show in `/config`
- [ ] Solana payments working
- [ ] Base payments working
- [ ] Telegram notifications show network
- [ ] Explorer links correct
- [ ] All tests passing
- [ ] Monitoring dashboards updated
- [ ] Documentation updated
- [ ] Team trained on new features

---

## 🎉 Congratulations!

Your X4A Marketplace now supports multi-chain payments while maintaining full X402 Protocol compliance. Users can seamlessly pay with USDC on either Solana or Base!

**Next Steps:**
- Monitor payment distribution across chains
- Gather user feedback on chain preference
- Plan for additional chain support (Ethereum, Polygon, etc.)
- Optimize gas costs and transaction speeds

---

**Last Updated**: 2025-01-XX  
**Version**: 2.0.0  
**Author**: X4A Team
