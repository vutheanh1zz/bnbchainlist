## Copilot Instructions for bnbchainlist

### Quick Summary
Next.js static site displaying BNB Chain networks and RPC endpoint health. Data merges [utils/chains.json](utils/chains.json) (base) + [utils/extraRpcs.json](utils/extraRpcs.json) (overrides) at build time. Client continuously polls RPCs every 5s to show latency and block height.

### Architecture Overview

**Core Stack**: Next.js 12 + React 17 + Material-UI + react-query + Zustand + Web3.js

**Three-Tier Data Flow**:
1. **Build-time**: [pages/index.js](pages/index.js) `getStaticProps()` merges chains.json + extraRpcs.json, normalizes URLs, filters API key placeholders
2. **State Layer**: Hybrid stores—Zustand for UI state ([stores/index.js](stores/index.js)), Flux for account events ([stores/accountStore.js](stores/accountStore.js))
3. **Runtime**: [hooks/useRPCData.js](hooks/useRPCData.js) polls via react-query—HTTP (axios + interceptors) and WebSocket (wss://) RPC queries, measures latency, parses hex block height

### Key Integration Points

**RPC Health Polling** ([hooks/useRPCData.js](hooks/useRPCData.js) lines 10-50):
- HTTP: `axios.post()` with request/response interceptors to measure latency
- WebSocket: Custom promise wrapper for latency measurement
- **Critical**: Skips URLs containing 'API_KEY' string to block secrets
- **Result format**: `{ url, height (int), latency (ms) }` or `null` on error
- **Interval**: 5000ms, configured at line 60 `refetchInterval`

**Wallet Integration** ([utils/utils.js](utils/utils.js) lines 80+):
- Calls `window.ethereum.request()` with `wallet_addEthereumChain` RPC
- Expects chainId as hex string, RPC URLs array from component props
- Error handling via Flux emitter `ERROR` event

**Theme & Status Colors** ([theme/coreTheme.js](theme/coreTheme.js)):
- Green: Valid block height + low latency
- Orange: Valid block height + high latency  
- Red: Failed RPC or null latency

### Development Workflow

**Dev Server** (hot reload):
```bash
npm run dev
# RPC poll every 5s auto-refreshes
```

**Production Build**:
```bash
NODE_ENV=production COMMIT_SHA=$(git rev-parse HEAD) npm run build
```
Sets `next.config.js` → `assetPrefix` to `https://www.bnbchainlist.org/static` for CDN cache busting.

### Project Patterns & Conventions

**Data Structure** ([utils/chains.json](utils/chains.json)):
```json
[
  {
    "chainId": 56,
    "name": "BNB Smart Chain Mainnet",
    "rpc": ["https://bsc.nodereal.io", ...],
    "nativeCurrency": { "name": "...", "symbol": "BNB", "decimals": 18 },
    "explorers": [{ "name": "bscscan", "url": "..." }]
  }
]
```

**Override Pattern** ([utils/extraRpcs.json](utils/extraRpcs.json)):
```json
{
  "56": { "rpcs": ["https://custom-rpc.example"] },
  "97": { "rpcs": [...] }
}
```
Used at build-time merge—extraRpcs values **replace** base chains.json rpcs.

**Secret Filtering** (enforced in 3 places):
1. [hooks/useRPCData.js](hooks/useRPCData.js) line 14: Skip URLs with 'API_KEY'
2. Build process: Filter `${INFURA_API_KEY}` placeholders
3. UI: Disabled state on placeholder RPCs

**Component Hierarchy**:
- [pages/index.js](pages/index.js): Main page, fetches chains via getStaticProps
- [components/chains.js](components/chains.js): Maps chains to Card components
- [components/chain/chain.js](components/chain/chain.js): Expandable card with wallet button
- [components/RPCList/index.js](components/RPCList/index.js): RPC status rows with copy buttons

### Common Tasks

| Task | Pattern |
|------|---------|
| Add RPC to chain | Edit [utils/chains.json](utils/chains.json) → add URL to `rpc` array |
| Override RPC (no rebuild) | Add to [utils/extraRpcs.json](utils/extraRpcs.json) → merge applied at build |
| Change poll interval | [hooks/useRPCData.js](hooks/useRPCData.js) line 60: `refetchInterval: 5000` |
| Add testnet | [utils/chains.json](utils/chains.json) → new object with chainId < 1000 typically |
| Adjust theme colors | [theme/coreTheme.js](theme/coreTheme.js) → Material-UI palette |
| Handle RPC errors | [hooks/useRPCData.js](hooks/useRPCData.js) lines 35-40: catch block returns null |

### Testing & Verification

- **Dev smoke test**: `npm run dev` → expand chain → watch RPC rows update within 5s
- **Build safety**: `npm run build` → grep output for 'API_KEY' or placeholders (should find none)
- **Wallet flow**: Expand chain → test "Add to wallet" button with MetaMask
- **Docker build**: `COMMIT_SHA=abc123 docker build -t bnbchainlist:latest .`

### PR Checklist
- ✅ RPC data updated in chains.json or extraRpcs.json
- ✅ No API keys committed (lint build output)
- ✅ Dev server tested: RPCs poll, latency displays
- ✅ Wallet add flow tested (at least one chain)
- ✅ Build succeeds: `npm run build`
