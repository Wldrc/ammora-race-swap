# Race Swap — Ammora Season 2 Builder

Minimal ETH → $tAMM swap on **GIWA Sepolia** testnet (chain 91342), built with
[`@ammora-protocol/sdk` 0.3.9](https://www.npmjs.com/package/@ammora-protocol/sdk).
Live demo: **https://wldrc.github.io/ammora-race-swap/** (GitHub Pages).

Testnet only. No real funds. No private key is ever stored or sent anywhere —
your own wallet signs every transaction.

## What it does

- **Verifies the deployment on load** — the V5 deployment manifest (factory,
  pool implementation, fee controller, fee vault, token policy, fee policy) is
  pinned by runtime code hash and re-verified on-chain by the SDK
  (`ArlV5FactoryClient.verifyDeploymentAtBlock` / `verifyPoolAtBlock`) before
  anything else happens.
- **Live quote** — reads the pool's `quoteExactIn` through the SDK's
  `arlV5PoolAbi` (no mock data, no hardcoded price).
- **One-transaction native swap** — ETH straight in, $tAMM straight out, via
  Ammora's V5 native router. Calldata is built by the SDK
  (`buildV3NativePoolExactInput` → `swapPoolExactInNative`), 0.5% slippage
  floor via `minOutFromSlippage`.
- **Balances** — ETH / WETH / $tAMM of the connected wallet (balance reads are
  retried after a swap: GIWA's public RPC replicas can lag a fresh head).

Pool: `0x1EF92A6a9BB4bBed2b385Dd5E688F88e7965BCf8` ($tAMM / WETH, 1% fee) ·
$tAMM: `0x10B3Cc2fc981277Ada576891CF649D0ABA768B06` (6 decimals) ·
Native router: `0x4FC91b90087288520A7bB4aA7Ab5441bb4E9C496`.

## The V5 manifest (`src/v5-manifest.json`)

SDK 0.3.9 ships no catalog entry for this pool's factory generation, so the
manifest was derived from on-chain reads on 2026-11-22 and is re-verified by
the SDK at runtime — a wrong pin fails closed:

- Factory `0xC0Fc168DeD39DA53C11D5082D9303216C3c37375` (the pool's own
  `factory()` return; created block 36388500 by Ammora deployer
  `0x13cd27a6e337097db858c9c4fa472fbad2f2c637`).
- The other five roles come from the factory's own getters
  (`poolImplementation`, `feeController`, `feeVault`, `tokenPolicyRegistry`,
  `feePolicy`); each pin is `keccak256` of the contract's live runtime code,
  and the SDK additionally checks the wiring between them
  (`controller.factory`, `implementation.authorizedFactory`, `vault.owner`,
  factory/ controller versions, fee delay/window).
- `factoryVersion` on-chain matches the SDK's `ARL_V5_VERSION`
  (`0x9a3b94…`), `MINIMUM_FEE_DELAY` 86400, `FEE_EXECUTION_WINDOW` 604800.

## Verified end-to-end (race wallet `0x9be74E6B7fc07d2Ec4d534cAD5A65335Dc60F90c`)

- SDK manifest + pool verification: `verifyDeploymentAtBlock` ✓,
  `verifyPoolAtBlock` ✓, fee 100 bps, authority `0x13cD27a6…` (probe:
  `src/probe-v5.mjs`).
- Live test swap, 2026-11-22, 0.0004 ETH → **0.125397 $tAMM**:
  tx [`0x589a4cc8417b15b9d6a50c838aef40fe7f27be675f8442f2b133ec52edc19c5f`](https://sepolia-explorer.giwa.io/tx/0x589a4cc8417b15b9d6a50c838aef40fe7f27be675f8442f2b133ec52edc19c5f)
  (block 41980230, status 1). Wallet $tAMM 0.105937 → 0.231334, exactly the
  quoted amount; 0.0000008 ETH fee to the fee vault.
- Same wallet bridged 0.04 ETH Sepolia → GIWA the same morning:
  [`0x9394c0d1c8d10eb30dd990de65419c5c28d57f9464d5ae2408238f8c99420161`](https://sepolia.etherscan.io/tx/0x9394c0d1c8d10eb30dd990de65419c5c28d57f9464d5ae2408238f8c99420161)
  via the official GIWA `L1StandardBridge`
  (`0x77b2ffc0F57598cAe1DB76cb398059cF5d10A7E7`, verified against
  docs.giwa.io + on-chain `MESSENGER()`/`OTHER_BRIDGE()`).

## Run locally

```sh
npm install
npm run build   # esbuild -> dist/
npx serve dist  # or: python3 -m http.server -d dist
```

Open the page, connect a wallet on GIWA Sepolia (chain 91342,
RPC `https://sepolia-rpc.giwa.io`), get test ETH from
[faucet.giwa.io](https://faucet.giwa.io) or the
[official bridge](https://sepolia-bridge.giwa.io) (Sepolia → GIWA).

## Known notes

- The app swaps the **V5** $tAMM pool. The SDK's shipped V4/auto-route
  catalogs do not recognise this pool's factory, which is why the manifest
  above is bundled and verified instead of relying on the catalog.
- The race wallet's daily Ammora tasks (check-in, vault, 5 swaps, stake) were
  already completed for 2026-11-22 (900 AP) before this build; the test swap
  above is a 6th swap and does not change that.
