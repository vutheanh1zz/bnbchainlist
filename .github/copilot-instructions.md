## Copilot / AI Agent Instructions for bnbchainlist

SUMMARY
- Small Next.js static site that lists BNBChain networks and their RPC endpoints. The UI is data-driven: authoritative sources are `utils/chains.json` (primary) and `utils/extraRpcs.json` (optional overrides merged at build time).

BIG PICTURE
- Build-time (SSG): `pages/index.js` (getStaticProps) runs `populateChain` to merge `utils/extraRpcs.json` into `utils/chains.json`, dedupe RPCs (via `removeEndingSlash`) and filter out placeholder RPCs containing `${INFURA_API_KEY}`. Each chain may get a `chainSlug` from `components/chains.js` used for icon resolution.
- Runtime: the client polls RPC endpoints every 5s using `hooks/useRPCData.js` (react-query `useQueries`).
  - HTTP RPCs: axios POST with request/response interceptors to measure latency (requestStart → response.latency).
  - WSS RPCs: `fetchWssChain` uses a one-message WebSocket pattern (open → send body → wait for first message).
- UI/UX: `components/RPCList/index.js` sorts RPCs, computes a trust color (green/orange/red), and disables connect for WSS or API_KEY endpoints. `components/chain/chain.js` handles icon lookup and `addToNetwork` (wallet integration).

AUTHORITATIVE DATA & MUTATIONS (must preserve)
- Change chain metadata only in `utils/chains.json`. Add additional RPCs under `utils/extraRpcs.json` with shape:
  ```json
  { "56": { "rpcs": ["https://my-bsc-node.example"] } }
  ```
- `pages/index.js` behavior to note:
  - `removeEndingSlash(rpc)` normalizes RPC urls.
  - RPCs containing `${INFURA_API_KEY}` are filtered out (placeholders only).
  - `revalidate: 3600` — site rebuilds hourly (SSG).

KEY FILES / LOCATIONS
- `pages/index.js` — build-time merge & normalization (`populateChain`, `removeEndingSlash`).
- `hooks/useRPCData.js` — polling (5s), HTTP latency via axios interceptors, WSS single-message flow (`fetchWssChain`).
- `components/RPCList/index.js` — sorting, trust-scoring and `disableConnect` logic (see threshold numbers below).
- `components/chains.js` — chainId → slug mapping for icons (update when adding new icons)
- `components/chain/chain.js` — `addToNetwork` / wallet integration and `assetPrefix` usage.
- `base/env.js` & `next.config.js` — `assetPrefix` and `generateBuildId` (uses `COMMIT_SHA`).
- `Dockerfile` — accepts `ARG COMMIT_SHA` and sets `ENV COMMIT_SHA` (used for deterministic buildId).

KEY, IMPLEMENTATION & GOTCHAS
- Placeholders/API keys: any RPC URL containing `API_KEY` or `${INFURA_API_KEY}` is filtered out at build-time, ignored by the poller and **disabled in UI** — do not commit real keys.
- WSS handling: `wss://` endpoints are polled with a WebSocket and are intentionally **not** allowed for `addToNetwork` (UI disables connect).
- Trust scoring (see `components/RPCList/index.js` for exact numbers):
  - If top.height - height > 3 OR top.latency - latency > 5000 → **red** (untrustworthy)
  - If top.height - height < 2 AND top.latency - latency > -600 → **green** (healthy)
  - Else → **orange**
- First-request latency quirk: `Row` (in `components/RPCList/index.js`) calls `refetch()` on the first observed URL and records seen RPCs in `useRpcStore` — this intentionally ignores the first request (which includes DNS lookup) and refetches so latency reflects response time more consistently.
- Sorting & special-case behavior:
  - `nodereal.io` RPCs are collected into `specialChain` and prioritized ahead of normal RPCs.
  - Ethereum Mainnet rows show `Copy URL` instead of `Add to Network`.
- Wallet add: `utils/utils.js` → `addToNetwork` builds `wallet_addEthereumChain` params (chainId converted to 0x hex) and calls `window.ethereum.request`; ensure RPC used is not `wss://` and doesn't contain `API_KEY`.
- Dark mode: uses localStorage key `yearn.finance-dark-mode` (affects shimmer and table theme).

BUILD / DEV / DEBUG (practical steps)
- Common scripts: `npm run dev` (dev), `npm run build` (prod), `npm run start` (serve prod), `npm run export` (static export).
- Reproduce production asset paths: set `NODE_ENV=production` and set `COMMIT_SHA` env var (or build with the Docker ARG) so `next.config.js` uses a stable `assetPrefix` and `generateBuildId`.
- Debug tips:
  - To inspect latency/timeouts, add logs in `hooks/useRPCData.js` (axios request interceptor sets `requestStart` and response latency).
  - WSS issues: inspect `fetchWssChain` and its `createPromise()` pattern (socket opens → sends body → waits for one message). Sockets that never respond will hang until rejected.
  - Wallet flow: check `stores/accountStore.js` and `components/chain/chain.js` for `tryConnectWallet` and `addToNetwork` behavior.

TESTING, VERIFICATION & PR GUIDELINES
- No CI/unit tests currently — verify changes locally:
  - Run `npm run dev`, open UI and confirm RPC table updates (~5s), colors and `Add to Network` behavior.
  - For data changes: edit only `utils/chains.json` or `utils/extraRpcs.json`, and include a short verification note in the PR.
  - When changing scoring/polling logic, add unit tests where practical and include manual steps verifying row colors and actions.

FILES & LOCATIONS QUICK MAP (fast onboarding)
- Data: `utils/chains.json`, `utils/extraRpcs.json`
- Build: `pages/index.js`, `next.config.js`, `Dockerfile`
- Polling/latency: `hooks/useRPCData.js`
- UI/logic: `components/RPCList/index.js`, `components/chain/chain.js`, `components/chains.js`
- Stores: `stores/index.js`, `stores/accountStore.js` (flux + zustand)

CONTACT / NEXT STEPS
- If any behavior or edge case is unclear (e.g., why a row's latency is high, or how `chainSlug` maps to assets), open an issue or ask for a short example and I’ll expand the doc or add a tiny unit test or dev note.

---
(Updated: concise, actionable, and anchored to discoverable files.)
