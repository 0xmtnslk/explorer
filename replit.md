# CoinHunters Explorer

A modern blockchain explorer for the Cosmos ecosystem, built with Vue 3 + Vite + TypeScript.

## Overview

CoinHunters Explorer is a customized version of Ping.pub blockchain explorer, branded for CoinHunters validator team. It provides real-time blockchain data, governance tracking, staking information, and more for Cosmos SDK-based blockchains.

## Recent Changes (December 2025)

- Complete UI redesign with modern gradient-based styling
- Navigation moved from left sidebar to top navigation bar
- Navigation shows: Dashboard, Governance, Staking, Uptime, Blocks, Transactions (icons only on mobile, with text on desktop)
- Added CoinHunters branding (logo, favicon, site name)
- Homepage redesigned with:
  - Hero section with CoinHunters branding
  - TVL stats cards (Validator name, Mainnet count, Dynamic Total Value Locked calculation)
  - TVL calculated from chain prices via CoinGecko API with $3M+ fallback
  - Mainnet/Testnet tabs with chain counts (13 mainnet, 3 testnet)
  - Modern chain cards with price data and favorites
- Added social media links (Website, X/Twitter, Telegram)
- New footer with CoinHunters branding and links
- Note: CORS errors for external APIs are expected in local development but resolve when deployed

## Project Architecture

```
/
├── chains/
│   ├── mainnet/     # Mainnet chain configurations (JSON)
│   └── testnet/     # Testnet chain configurations (JSON)
├── public/
│   ├── logo.webp    # CoinHunters logo
│   └── favicon.ico  # Favicon
├── src/
│   ├── components/  # Reusable Vue components
│   ├── layouts/     # Layout components (DefaultLayout, NavFooter, etc.)
│   ├── modules/     # Page modules (chain pages)
│   ├── pages/       # Top-level pages (index.vue - homepage)
│   ├── plugins/     # Vue plugins (i18n, router)
│   ├── stores/      # Pinia stores (useDashboard, useBlockchain, etc.)
│   └── types/       # TypeScript type definitions
├── index.html       # HTML entry point
├── vite.config.ts   # Vite configuration
└── tailwind.config.js  # Tailwind CSS configuration
```

## Key Features

- **Dashboard**: Overview of blockchain stats (height, validators, supply, etc.)
- **Governance**: Track and participate in blockchain governance proposals
- **Staking**: View validators and staking information
- **Uptime**: Monitor validator uptime
- **Blocks**: Browse recent blocks
- **Transactions**: View transaction history

## Navigation Menu Items

The top navigation bar includes:
- Dashboard
- Governance  
- Staking
- Uptime
- Blocks
- Transactions

## Social Links

- Website: https://coinhunterstr.com/
- X (Twitter): https://x.com/CoinHuntersTR
- Telegram: https://t.me/CoinHuntersTR

## Tech Stack

- Vue 3 with Composition API
- TypeScript
- Vite (build tool)
- Tailwind CSS + DaisyUI
- Pinia (state management)
- Vue Router
- Iconify (icons)

## Development

The app runs on port 5000 with Vite dev server.

```bash
npm run dev
```

## Deployment

Build for production:

```bash
npm run build
```

Output will be in the `dist/` directory for static hosting.

## Notes

- CORS warnings for CoinGecko API calls are expected in local development
- External blockchain API endpoints (like itrocket.net) may have rate limits
- Dark/Light mode is supported via theme switcher
