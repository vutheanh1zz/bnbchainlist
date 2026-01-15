## Copilot / AI Agent Instructions for bnbchainlist

Summary
- Small Next.js static site (SSG) listing BNBChain networks and their RPC endpoints.
- Authoritative data: `utils/chains.json`. Add or override RPCs with `utils/extraRpcs.json` (merged at build time).

Quick architecture
- Build (SSG): `pages/index.js` (getStaticProps) runs `populateChain` to merge `extraRpcs`, normalize RPCs with `removeEndingSlash`, and filter placeholder RPCs (e.g., `${INFURA_API_KEY}`).
- Runtime: client polls RPCs every 5s via `hooks/useRPCData.js` (react-query `useQueries`).
  - HTTP RPCs: axios POST + interceptors measure latency (`requestStart` → response.latency).
  - WSS RPCs: `fetchWssChain` uses a one-message pattern (open → send → wait for first message).

Authoritative edits & examples
- Edit chain metadata only in `utils/chains.json`.
- To add RPCs, update `utils/extraRpcs.json` using:
  ```json
  { "56": { "rpcs": ["https://my-bsc-node.example"] } }
  ```
- Do NOT commit real keys. RPC strings containing `API_KEY` or `${INFURA_API_KEY}` are filtered at build-time and disabled in the UI.

Key behaviors & gotchas
- First-request latency: `Row` in `components/RPCList/index.js` calls `refetch()` on first observation to avoid DNS cold-start impacting latency metrics.
- Trust scoring: color rules (green/orange/red) are implemented in `components/RPCList/index.js` (compare height and latency deltas before changing logic).
- Special sorting: `nodereal.io` RPCs are grouped as `specialChain` and prioritized.
- WSS endpoints are polled but blocked from wallet add (`addToNetwork`).
- Wallet add flow: `utils/utils.js` → `addToNetwork` builds `wallet_addEthereumChain` params (chainId hex) and calls `window.ethereum.request`.

Files to inspect for common changes
- Data/build: `pages/index.js`, `utils/chains.json`, `utils/extraRpcs.json`
- Polling/latency: `hooks/useRPCData.js` (`fetchWssChain`, axios interceptors)
- UI/rules: `components/RPCList/index.js`, `components/chain/chain.js`, `components/chains.js`
- Wallet/account: `stores/accountStore.js`, `utils/utils.js`
- Build config: `next.config.js`, `base/env.js`, `Dockerfile` (ARG `COMMIT_SHA` → deterministic `assetPrefix`/buildId)

Build / dev / verify
- Dev: `npm run dev` (hot reload); expect RPC rows to refresh ~every 5s.
- Prod build (locally): set `NODE_ENV=production` and `COMMIT_SHA=<sha>` before `npm run build` (or pass `--build-arg COMMIT_SHA=<sha>` to Docker build).
- Verify changes:
  - Data edits: update `utils/chains.json`/`extraRpcs.json`, run dev, confirm new RPC appears and follows filters.
  - Scoring/polling changes: add temporary logs in `hooks/useRPCData.js` to observe latency/timeout behavior.

PR Guidance
- Add a short verification note to PRs (how you tested: e.g., "ran dev, confirmed RPC shows and row color is green").
- Avoid committing secrets and API keys; rely on `extraRpcs` for opt-in node additions.
- For behavior changes (scoring/sorting/polling), include a manual test plan and unit tests when practical.

If unclear
- Open an issue and reference the files above; small tests or dev notes can be added.

---
(Concise, actionable, and rooted in discoverable files.)
