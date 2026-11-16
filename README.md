# HYPE deposits into Elysium

A [HyperIndex](https://docs.envio.dev/docs/HyperIndex/overview) indexer for every message HyperEVM sends to [Elysium](https://elysium.kinetiq.xyz/docs), Kinetiq's Hyperliquid L2. Elysium settles to HyperEVM, so every deposit into Elysium passes through its Bridge contract on HyperEVM. HyperEVM is on [HyperSync](https://docs.envio.dev/docs/HyperSync/overview), so a full backfill of 2.6M bridge messages takes about a minute.

From the Bridge's first event on 19 Oct 2026 (HyperEVM block 48,956,737) to HyperEVM block 51,087,189 (about 18:03 UTC on 12 Nov 2026), **28.4K wallets made 47K native HYPE deposits into Elysium**. **16.1K of those deposits, from 10.9K first-time wallets, landed on 9 Nov**, the day Elysium opened its public mainnet.

| | Count |
| --- | --- |
| Messages from HyperEVM to Elysium | 2,614,447 |
| Native HYPE deposits | 46,963 |
| Wallets that deposited HYPE | 28,382 |
| HYPE in deposits sent straight to the Inbox | 381,459.67 |
| Blocks covered | 48,956,737 to 51,087,189 |

Busiest UTC days for deposits:

| Day | Deposits | New wallets |
| --- | ---: | ---: |
| 7 Nov | 3,808 | 2,271 |
| 8 Nov | 6,066 | 3,447 |
| 9 Nov | 16,126 | 10,883 |
| 10 Nov | 3,618 | 2,340 |

A separate HyperSync script that decoded every `MessageDelivered` log itself found 46,956 deposits from 28,378 wallets at block 51,086,631, a few hundred blocks earlier, with the same deposit count for every UTC day. The same logs fetched with plain `eth_getLogs` through [HyperRPC](https://docs.envio.dev/docs/HyperRPC/overview-hyperrpc) gave matching counts. For 9 Nov we also pulled the Bridge logs from a public HyperEVM RPC, which returned the same 16,126 native deposits.

## Why it matters

A rollup's deposits are recorded on the chain it settles to. Elysium isn't on HyperSync yet, but HyperEVM is, so you can follow onboarding into Elysium from day one by indexing one contract on HyperEVM. You don't need the L2 itself.

Elysium is built on the Arbitrum Nitro stack, so the Bridge emits Nitro's `MessageDelivered` event with a `kind` field. Kind 12 is a native deposit, which on Elysium is HYPE. The same approach works for any Nitro-based rollup, as long as its parent chain is indexable.

## What it indexes

| Contract | Chain | Address |
| --- | --- | --- |
| Bridge | HyperEVM (999) | [`0x503D572157113fAaaD4DbeD2647Bb10b949216B6`](https://hyperevmscan.io/address/0x503D572157113fAaaD4DbeD2647Bb10b949216B6) |
| Inbox | HyperEVM (999) | [`0xce59f8B65ad41C9DAE4c07A0571a39824bD92cff`](https://hyperevmscan.io/address/0xce59f8B65ad41C9DAE4c07A0571a39824bD92cff) |

Both addresses come from Elysium's [contract addresses page](https://elysium.kinetiq.xyz/docs/contract-addresses). The indexer listens to `MessageDelivered` on the Bridge and uses the Inbox address to tell which deposits were sent to it directly. The handler is [`src/handlers/Bridge.ts`](src/handlers/Bridge.ts).

Entities in [`schema.graphql`](schema.graphql):

- `Totals` keeps messages, HYPE deposits, depositing wallets and HYPE sent straight to the Inbox
- `MessageKind` counts messages per Nitro kind (3 is an L2 message, 9 a retryable ticket used for token bridging, 12 a native deposit)
- `Depositor` stores one row per wallet with its deposit count and first UTC day
- `DailyDeposits` counts deposits and first-time wallets per UTC day

A deposit's wallet is the `sender` on the event, the wallet that called the Inbox on HyperEVM. `hypeDepositedDirect` sums the transaction value only for deposits whose transaction went straight to the Inbox. Deposits routed through another contract still count in `hypeDeposits`.

## Run it

You need Node 22+, pnpm and Docker, plus a free Envio API token from [envio.dev/app/api-tokens](https://envio.dev/app/api-tokens).

```bash
echo 'ENVIO_API_TOKEN=<your-token>' > .env
pnpm install
pnpm codegen
pnpm dev
```

This repo has no tests yet. The agent prompt further down asks your agent to add them, after which `pnpm test` runs them.

GraphQL is at http://localhost:8080 (local admin secret `testing`).

```graphql
{
  Totals { messages hypeDeposits depositors hypeDepositedDirect lastBlock }
  MessageKind(order_by: { messages: desc }) { id messages }
  DailyDeposits(order_by: { id: asc }) { id deposits newDepositors }
}
```

`hypeDepositedDirect` is in wei, so divide by 10^18 for HYPE.

### Make it live

There's no `end_block` in `config.yaml`, so once the backfill is done the indexer keeps following HyperEVM's head and your numbers keep growing.

### Deploy it

Push your fork to GitHub and deploy it on [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service) to get a hosted GraphQL endpoint. Indexers on Envio Cloud don't need your own API token. The [deployment guide](https://docs.envio.dev/docs/HyperIndex/hosted-service-deployment) covers connecting the repo.

## Build your own with an AI agent

This repo doesn't ship Envio's agent skills yet. To add them, run `pnpx envio init` in an empty folder and copy the `.claude/skills/` folder it creates into this repo, or follow [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai). Then paste a prompt like this into Claude Code or Cursor from the repo root.

```text
This is an Envio HyperIndex indexer on HyperEVM (chain 999) for Elysium's Bridge,
which emits Arbitrum Nitro MessageDelivered events. Read README.md, config.yaml,
schema.graphql, src/handlers/Bridge.ts and any skills in .claude/skills, then
https://docs.envio.dev/docs/HyperIndex/multichain-indexing and
https://docs.envio.dev/docs/HyperIndex/testing.

Add a second Nitro-based rollup: its Bridge <BRIDGE ADDRESS> and Inbox
<INBOX ADDRESS> on <PARENT CHAIN>, from the Bridge's first event. Key Totals,
MessageKind, Depositor and DailyDeposits by rollup so the numbers stay separate.
Add an hourly deposits rollup next to the daily one. Add vitest with a "test"
script and tests using createTestIndexer for both rollups, run `pnpm codegen` and
`pnpm test` until they pass, then run `pnpm dev` and show me the GraphQL query
for deposits per day per rollup.
```

Swap in any Nitro rollup whose parent chain HyperSync supports. On a chain where the native token is ETH, kind 12 means an ETH deposit.

## Links

- [HyperIndex overview](https://docs.envio.dev/docs/HyperIndex/overview)
- [HyperSync supported networks](https://docs.envio.dev/docs/HyperSync/hypersync-supported-networks)
- [HyperRPC](https://docs.envio.dev/docs/HyperRPC/overview-hyperrpc)
- [Multichain indexing](https://docs.envio.dev/docs/HyperIndex/multichain-indexing)
- [Testing HyperIndex indexers](https://docs.envio.dev/docs/HyperIndex/testing)
- [Envio Cloud](https://docs.envio.dev/docs/HyperIndex/hosted-service)
- [Quickstart with AI](https://docs.envio.dev/docs/HyperIndex/quickstart-with-ai)
- [Elysium docs](https://elysium.kinetiq.xyz/docs)
- [Elysium contract addresses](https://elysium.kinetiq.xyz/docs/contract-addresses)
- [Token bridging on Elysium](https://elysium.kinetiq.xyz/docs/token-bridging-on-elysium)
- [Kinetiq](https://kinetiq.xyz)
- [HyperEVM docs](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperevm)
- [Every Uniswap v4 pool on Elysium](https://github.com/enviodev/elysium-uniswap-v4-indexer), a sister indexer on the Elysium side
- [Envio Discord](https://discord.gg/envio)
