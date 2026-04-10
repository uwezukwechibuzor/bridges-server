# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DefiLlama Bridges Server — aggregates cross-chain bridge transaction data from 100+ protocols. Fetches on-chain events via adapters, stores in PostgreSQL, aggregates into hourly/daily volumes, and exposes a REST API via Fastify.

## Commands

- **Type check:** `npm run ts` (runs `tsc --noEmit`)
- **Build:** `npm run build` (Vite → `dist/`, CJS output)
- **Dev server:** `npm run dev` (requires `.env` file; tsx watch on port 3000)
- **Production server:** `npm run start` (runs `dist/index.js`)
- **Cron worker:** `npm run start:cron` (runs `dist/startCron.js`)
- **Test adapter:** `npm run test <bridgeName> [numBlocks]` — runs adapter against recent blocks
- **Test historical:** `npm run test-txs <startTs> <endTs> <adapter>` — backfill test
- **Backfill adapter:** `npm run adapter <startTs> <endTs> <bridgeName> [chain]` (requires `.env`)
- **Aggregate:** `npm run aggregate <startTs> <endTs> <bridgeName>` (requires `.env`)
- **Formatting:** Prettier with `printWidth: 120`, `tabWidth: 2`

## Architecture

### Data Pipeline Overview

The system follows a three-stage pipeline: **Fetch → Store → Aggregate**.

1. **Fetch**: Adapters pull raw bridge transaction events from on-chain logs or APIs
2. **Store**: Raw transactions are inserted into `bridges.transactions` with block timestamps
3. **Aggregate**: Raw txs are priced via DefiLlama's price API (`coins.llama.fi`), then rolled up into hourly and daily volume tables

### Adapter System (`src/adapters/`)

Each bridge protocol has a directory under `src/adapters/` exporting a default `BridgeAdapter`:
```ts
{ [chainName: string]: (fromBlock: number, toBlock: number) => Promise<EventData[]> }
```

Two main patterns:
1. **EVM event log pattern** (most adapters) — define `ContractEventParams`/`PartialContractEventParams` arrays, pass to `getTxDataFromEVMEventLogs()` from `src/helpers/processTransactions.ts`. Each params object specifies a `target` contract address, `topic` (event signature), `abi`, and key mappings (`logKeys`, `argKeys`, `txKeys`, `fixedEventData`) that map event fields to `EventData` fields. See `src/adapters/celer/index.ts` for a representative example.
2. **API-based pattern** — query bridge indexer APIs directly, construct `EventData` objects manually (e.g., `src/adapters/across/`)

Some adapters use `AsyncBridgeAdapter` (with `isAsync: true` and a `build()` method) when they need async initialization before returning the chain function map.

Adapters are registered in `src/adapters/index.ts` keyed by `bridgeDbName`.

### Adapter Execution Flow (`src/utils/adapter.ts`)

**`runAdapterToCurrentBlock(bridgeNetwork)`** — the main entry point for running a single adapter:

1. Resolves the adapter (handling async adapters via `build()`)
2. Ensures `bridges.config` entries exist for each chain the adapter supports
3. For each chain in parallel (staggered by 200ms):
   - Looks up the `bridgeID` from the config table
   - Calls `getBlocksForRunningAdapter()` to determine the block range:
     - Gets the latest on-chain block number
     - Finds the last recorded block from DB (`lastRecordedBlocks` query) or Redis progress cache
     - Redis progress (`adapter_progress:{name}:{chain}`, 7-day TTL) tracks the last successfully processed block, allowing resumption after crashes
     - `startBlock = lastRecordedEndBlock + 1`
   - Processes blocks in chunks of `maxBlocksToQueryByChain[chain]` (defined in `src/utils/constants.ts`, ~1.5-2 hours of blocks per chain)
   - Each chunk calls `runAdapterHistorical()`

**`runAdapterHistorical(startBlock, endBlock, ...)`** — processes a block range:

1. Checks Redis progress to skip already-processed ranges
2. Fetches event logs by calling the adapter's chain function (with `async-retry`, 4 retries)
3. Estimates timestamps by sampling 10 blocks across the range from the provider (Solana uses `getBlockTime` per tx instead)
4. Filters out tx groups with 100+ events per txHash
5. Inserts each event into `bridges.transactions` within a SQL transaction (with retries)
6. Updates Redis progress on success
7. On failure after 3 attempts, logs error to DB and sends Discord notification

**Skipped bridges**: Certain high-volume bridges (`bridgesToSkip` in `adapter.ts`: wormhole, layerzero, hyperlane, mayan, relay, cashmere, teleswap, intersoon, ccip) are excluded from the general `runAllAdapters` job and run via dedicated handlers with separate timeouts in the cron schedule.

### `getTxDataFromEVMEventLogs()` (`src/helpers/processTransactions.ts`)

The core EVM adapter helper. For each `ContractEventParams` in the array:

1. If `isTransfer: true`, auto-generates ERC20 Transfer event params targeting the contract address
2. Calls `@defillama/sdk` `getLogs()` to fetch raw event logs for the block range
3. Processes each log with concurrency 20 via `PromisePool`:
   - Extracts values via `logKeys` (from raw log fields), `argKeys` (from parsed event args), `txKeys` (from transaction data)
   - Applies filters: `includeArg`/`excludeArg`, `includeTxData`, `functionSignatureFilter`, `custom` filter functions
   - Optionally extracts token from receipt (`getTokenFromReceipt`) or input data (`inputDataExtraction`)
   - Applies `fixedEventData` overrides and `mapTokens` token address remapping
4. Returns accumulated `EventData[]` after applying address-level `excludeFrom`/`excludeTo`/`excludeToken` filters

### Cron System (`src/server/cron.ts`)

The cron worker (`npm run start:cron`) is **not** a traditional cron scheduler — it runs all jobs once with staggered `setTimeout` delays and then self-terminates after 54 minutes (designed to be restarted by PM2 or an external orchestrator).

**Timeline of a single cron cycle:**
- **+5 min**: `aggregateLayerZero` (last 2 days), `aggregateAll` (last 36 hours), `aggregateHourly`, `aggregateDaily`, `runAllAdapters`
- **+25 min**: Individual heavy adapters — `runWormhole`, `runMayan`, `runLayerZero`, `runHyperlane`, `runInterSoon`, `runRelay`, `runCashmere`, `runTeleswap`, `runCCIP`, `runSnowbridge`
- **+54 min**: Process exits (prints getLogs usage summary, closes DB connections)

Each job runs with `withTimeout()` — a `Promise.race` against a timeout (5-40 min depending on job). Set `NO_CRON=1` to disable.

**`runAllAdapters` job** (`src/server/jobs/runAllAdapters.ts`):
1. Queries max `tx_block` per `bridge_id` from `bridges.transactions` to get last recorded blocks
2. Shuffles all bridge networks randomly (load distribution)
3. Processes up to 10 adapters concurrently via `PromisePool`
4. Skips bridges in `bridgesToSkip` (they have dedicated handlers)

### Aggregation Pipeline (`src/utils/aggregate.ts`)

**`aggregateData(timestamp, bridgeDbName, chain, hourly, largeTxThreshold)`**:

1. Determines the time window (previous hour or previous day relative to `timestamp`)
2. Queries all raw transactions from `bridges.transactions` for that bridge/chain/time window
3. Collects unique token addresses and fetches batch prices from `coins.llama.fi` via `getLlamaPrices()`
4. For each transaction:
   - Converts raw token amount to USD using price + decimals (respects `transformTokens` and `transformTokenDecimals` mappings)
   - If `is_usd_volume` flag is set, uses amount directly as USD value
   - Accumulates per-token and per-address deposit/withdrawal totals
   - Flags transactions above `largeTxThreshold` for separate storage
   - Skips values over $10B as likely errors
5. Inserts aggregated row into `bridges.hourly_aggregated` or `bridges.daily_aggregated`
6. Inserts qualifying rows into `bridges.large_transactions`

**`aggregateHourlyVolume` / `aggregateDailyVolume`** (`src/server/jobs/`): Secondary aggregation that copies data from `hourly_aggregated` into simplified `hourly_volume`/`daily_volume` tables joined with chain info from `bridges.config`.

### Bridge Network Registry (`src/data/bridgeNetworkData.ts`)

Each bridge is registered as a `BridgeNetwork` with a unique `id`, `bridgeDbName`, `chains[]`, `largeTxThreshold`, and optional `chainMapping`/`destinationChain`. Lookup helpers in `src/data/importBridgeNetwork.ts`.

### EventData (`src/utils/types.ts`)

Core return type from all adapters: `blockNumber`, `txHash`, `from`, `to`, `token`, `amount` (ethers BigNumber), `isDeposit`, plus optional `chain`, `chainOverride`, `isUSDVolume`, `txsCountedAs`.

### Key Helpers (`src/helpers/`)

- `processTransactions.ts` — `getTxDataFromEVMEventLogs()`: core EVM log fetching/parsing
- `eventParams.ts` — `constructTransferParams()` for ERC20 Transfer events
- `bridgeAdapter.type.ts` — `BridgeAdapter`, `AsyncBridgeAdapter`, `ContractEventParams` types
- `tokenMappings.ts` — token address transforms (`transformTokens`) and decimal overrides (`transformTokenDecimals`); also `chainMappings` for chain name aliasing
- Chain-specific helpers: `solana.ts`, `sui.ts`, `tron.ts`, `stellar.ts`

### Database (`sql/data.sql`)

PostgreSQL under `bridges` schema. Key tables:
- `bridges.transactions` — raw tx records (unique on bridge_id, chain, tx_hash, token, tx_from, tx_to)
- `bridges.hourly_aggregated` / `bridges.daily_aggregated` — pre-aggregated volumes with token_total[] and address_total[] arrays
- `bridges.hourly_volume` / `bridges.daily_volume` — simplified volume tables (joined with chain from config)
- `bridges.large_transactions` — transactions above `largeTxThreshold`
- `bridges.config` — maps (bridge_name, chain) → UUID

DB access via `src/utils/db.js` using the `postgres` npm package. Two connection pools: `sql` (max 10) for writes, `querySql` (max 6) for reads.

### Server (`src/server/`)

- `index.ts` — Fastify server with Redis caching (70-min TTL, 10-min warming interval)
- `cron.ts` — Delay-based job scheduler (see Cron System above)
- Handlers in `src/handlers/` use `wrap()` from `src/utils/wrap.ts` for Lambda/Fastify compatibility

### Utilities (`src/utils/`)

- `adapter.ts` — adapter runner: `runAdapterToCurrentBlock()`, `runAdapterHistorical()`, `runAllAdaptersToCurrentBlock()`; defines `bridgesToSkip`
- `aggregate.ts` — fetches prices from `coins.llama.fi`, computes USD volumes, inserts aggregated rows
- `blocks.ts` — `getLatestBlockNumber()`, `getBlockByTimestamp()` (multi-chain: EVM, Solana, Sui, Stellar, Tron, IBC)
- `prices.ts` — DefiLlama price API integration
- `cache.ts` — Redis wrapper with getLogs tracking per adapter:chain
- `constants.ts` — `maxBlocksToQueryByChain` per-chain block query limits (~1.5-2 hours of blocks)

## Adding a New Adapter

1. Create `src/adapters/<bridge-name>/index.ts` exporting a default `BridgeAdapter`
2. Add a `BridgeNetwork` entry in `src/data/bridgeNetworkData.ts` with a new unique `id`
3. Import and register in `src/adapters/index.ts`
4. Test with: `npm run test <bridge-name> <numBlocks>`

## Environment Variables

Required: `DB_URL` (or `PSQL_USERNAME` + `PSQL_PW` + `PSQL_URL`). Optional: `REDIS_URL`, `PORT` (default 3000), `DISCORD_WEBHOOK`, `ALLIUM_API_KEY`, per-chain RPC vars (e.g. `ETHEREUM_RPC`).

## Deployment

Push to `master` triggers CI: Node 18, `npm ci`, `npm run ts`, then `sls deploy --stage prod` (AWS Lambda via Serverless Framework).
