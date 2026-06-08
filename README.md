# Polymarket CLOB Client

<a href='https://www.npmjs.com/package/@polymarket/clob-client'>
    <img src='https://img.shields.io/npm/v/@polymarket/clob-client.svg' alt='NPM'/>
</a>

Typescript client for the Polymarket CLOB

### Usage

```ts
// npm install @polymarket/clob-client
// npm install ethers
// Client initialization example and dumping API Keys

import { ApiKeyCreds, ClobClient, OrderType, Side, } from "@polymarket/clob-client";
import { Wallet } from "@ethersproject/wallet";

const host = 'https://clob.polymarket.com';
const funder = ''; // This is your Polymarket Profile Address, where you send USDC to.
const signer = new Wallet(""); // This is your Private Key. If using email login export from https://reveal.magic.link/polymarket otherwise export from your Web3 Application


// In general don't create a new API key, always derive or createOrDerive
const creds = new ClobClient(host, 137, signer).createOrDeriveApiKey();

//0: Browser Wallet(Metamask, Coinbase Wallet, etc)
//1: Magic/Email Login
const signatureType = 1; 
  (async () => {
    const clobClient = new ClobClient(host, 137, signer, await creds, signatureType, funder);
    const resp2 = await clobClient.createAndPostOrder(
        {
            tokenID: "", //Use https://docs.polymarket.com/developers/gamma-markets-api/get-markets to grab a sample token
            price: 0.01,
            side: Side.BUY,
            size: 5,
        },
        { tickSize: "0.001",negRisk: false }, //You'll need to adjust these based on the market. Get the tickSize and negRisk T/F from the get-markets above
        //{ tickSize: "0.001",negRisk: true },

        OrderType.GTC, 
    );
    console.log(resp2)
  })();
```

See [examples](examples/) for more information.

### Authentication

The client supports three authentication levels:

- **L1:** a signer only. Used to create or derive API keys and to sign orders.
- **L2:** a signer plus `ApiKeyCreds`. Required for private CLOB operations such as posting orders, RFQs, open orders, readonly keys, and heartbeats.
- **Builder:** L2 auth plus a `BuilderConfig` from `@polymarket/builder-signing-sdk`. Used for builder-authenticated order posting and builder endpoints.

`signatureType` controls how orders are signed:

```ts
import { SignatureType } from "@polymarket/clob-client";

SignatureType.EOA; // 0: browser wallet or direct EOA
SignatureType.POLY_PROXY; // 1: Polymarket proxy wallet, including Magic/email login
SignatureType.POLY_GNOSIS_SAFE; // 2: Polymarket Gnosis Safe
```

Pass `funderAddress` when the signing wallet differs from the Polymarket profile or proxy address that holds funds. See `examples/signatureTypes.ts` for end-to-end signature type examples.

### Using viem WalletClient

```ts
import { ClobClient } from "@polymarket/clob-client";
import { createWalletClient, http } from "viem";
import { polygon } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const host = "https://clob.polymarket.com";
const account = privateKeyToAccount("0x...");
const walletClient = createWalletClient({
    account,
    chain: polygon,
    transport: http(),
});

const clobClient = new ClobClient(host, 137, walletClient);
```

### ClobClient options

`ClobClient` uses positional constructor arguments. The first four cover the common L2-authenticated client; later arguments enable operational features.

| # | Argument | Purpose |
| - | - | - |
| 1 | `host` | CLOB API base URL, for example `https://clob.polymarket.com`. |
| 2 | `chainId` | Network chain ID. The client supports Polygon `137` and Amoy `80002`. |
| 3 | `signer` | Ethers `Wallet` or viem `WalletClient` used for L1 signatures and order signing. |
| 4 | `creds` | L2 API credentials: `{ key, secret, passphrase }`. |
| 5 | `signatureType` | `SignatureType.EOA`, `SignatureType.POLY_PROXY`, or `SignatureType.POLY_GNOSIS_SAFE`. |
| 6 | `funderAddress` | Profile, proxy, or Safe address that funds trades. |
| 7 | `geoBlockToken` | Adds `geo_block_token` to client requests; see `examples/geoToken.ts`. |
| 8 | `useServerTime` | Uses the server timestamp when building authenticated headers. |
| 9 | `builderConfig` | Builder signing config for builder order flow and builder endpoints. |
| 10 | `getSigner` | Lazy signer provider used by the order builder. |
| 11 | `retryOnError` | Retries transient POST failures in the HTTP helper. |
| 12 | `tickSizeTtlMs` | Tick-size cache TTL in milliseconds. Defaults to 5 minutes. |
| 13 | `throwOnError` | Throws `ApiError` for API error responses instead of returning `{ error, status }`. |

For readability, prefer naming optional values before passing them:

```ts
const useServerTime = true;
const retryOnError = true;
const tickSizeTtlMs = 300_000;
const throwOnError = true;

const clobClient = new ClobClient(
    host,
    137,
    signer,
    creds,
    SignatureType.POLY_PROXY,
    funder,
    undefined, // geoBlockToken
    useServerTime,
    undefined, // builderConfig
    undefined, // getSigner
    retryOnError,
    tickSizeTtlMs,
    throwOnError,
);
```

### RFQ (Request for Quote)

RFQ methods are exposed through `clobClient.rfq` and require L2 auth. The complete two-party flow is implemented in `examples/rfqFullFlow.ts`.

```ts
const requester = new ClobClient(host, chainId, requesterWallet, requesterCreds);
const quoter = new ClobClient(host, chainId, quoterWallet, quoterCreds);

const request = await requester.rfq.createRfqRequest(
    { tokenID, price: 0.5, side: Side.BUY, size: 40 },
    { tickSize: "0.01" },
);

const quote = await quoter.rfq.createRfqQuote(
    { requestId: request.requestId, tokenID, price: 0.5, side: Side.SELL, size: 40 },
    { tickSize: "0.01" },
);

await requester.rfq.acceptRfqQuote({
    requestId: request.requestId,
    quoteId: quote.quoteId,
    expiration: Math.floor(Date.now() / 1000) + 3600,
});

await quoter.rfq.approveRfqOrder({
    requestId: request.requestId,
    quoteId: quote.quoteId,
    expiration: Math.floor(Date.now() / 1000) + 3600,
});
```

RFQ method guide:

- Requester methods: `createRfqRequest`, `getRfqRequests`, `getRfqRequesterQuotes`, `getRfqBestQuote`, `acceptRfqQuote`, and `cancelRfqRequest`.
- Quoter methods: `createRfqQuote`, `getRfqQuoterQuotes`, `approveRfqOrder`, and `cancelRfqQuote`.
- Shared utility: `rfqConfig` returns server RFQ configuration.
- `acceptRfqQuote` and `approveRfqOrder` fetch quote details, create signed orders internally, and use the supplied `expiration` as the signed order expiration.
- For complementary matches, requester and quoter orders use opposite sides. For mint/merge matches, the requester order uses the complement token and inverse price.

RFQ list filters:

- `getRfqRequests`, `getRfqRequesterQuotes`, and `getRfqQuoterQuotes` accept `offset`, `limit`, `state`, market, size, USDC size, price, and sort filters.
- `offset` is a base64 cursor; the default is `MA==`. `limit` defaults to 50 and is capped at 100.
- `state` is `active` or `inactive`. If omitted, no state filter is applied.
- Array filters such as `requestIds`, `quoteIds`, and `markets` are sent as repeated query parameters. Market filters must be condition IDs formatted as `0x` plus 64 hex characters.
- Request sorting supports `price`, `expiry`, `size`, or `created`; quote sorting supports `price`, `expiry`, or `created`.

RFQ troubleshooting checks:

- Use a token ID from the same environment as `CLOB_API_URL`.
- Confirm the market is active and RFQ-enabled.
- Confirm the quoter is whitelisted for the flow.
- Pass an explicit `tickSize` when you already know the market tick size.

Related examples: `rfqFullFlow.ts`, `rfqCreateRequest.ts`, `rfqCreateQuote.ts`, `rfqAcceptQuote.ts`, `rfqApproveOrder.ts`, `rfqGetRequests.ts`, `rfqGetQuotes.ts`, `rfqGetBestQuote.ts`, `rfqCancelRequest.ts`, `rfqCancelQuote.ts`, and `rfqConfig.ts`.

### Builder integration

Builder key creation and lookup use L2 auth. Revocation and builder-authenticated order posting also need a `BuilderConfig`, which can use local builder credentials or a remote signer URL.

```ts
import { BuilderConfig } from "@polymarket/builder-signing-sdk";

const builderConfig = new BuilderConfig({
    localBuilderCreds: {
        key: process.env.BUILDER_API_KEY!,
        secret: process.env.BUILDER_SECRET!,
        passphrase: process.env.BUILDER_PASS_PHRASE!,
    },
});

const clobClient = new ClobClient(
    host,
    chainId,
    signer,
    creds,
    undefined,
    undefined,
    undefined,
    false,
    builderConfig,
);

const trades = await clobClient.getBuilderTrades();
```

See `examples/createBuilderApiKey.ts`, `examples/getBuilderApiKeys.ts`, `examples/revokeBuilderApiKeys.ts`, `examples/getBuilderTrades.ts`, and `examples/getBuilderOpenOrders.ts`.

### Readonly API keys

Readonly API keys let integrations read private data without permission to trade. Managing readonly keys requires a regular L2-authenticated client.

```ts
const readonlyKey = await clobClient.createReadonlyApiKey();
const keys = await clobClient.getReadonlyApiKeys();
await clobClient.validateReadonlyApiKey(address, readonlyKey.apiKey);
await clobClient.deleteReadonlyApiKey(readonlyKey.apiKey);
```

Readonly keys are not trading credentials. To use one for private read endpoints, make a direct HTTP request with `POLY_READONLY_API_KEY` and `POLY_ADDRESS`:

```ts
import axios from "axios";

await axios.get(`${host}/data/orders`, {
    headers: {
        POLY_READONLY_API_KEY: readonlyApiKey,
        POLY_ADDRESS: address,
    },
    params: { maker_address: address },
});
```

See `examples/createReadonlyApiKey.ts`, `examples/getReadonlyApiKeys.ts`, `examples/deleteReadonlyApiKey.ts`, and `examples/getOpenOrdersWithReadonlyKey.ts`.

### Balances and allowances

Use balance and allowance helpers before trading to confirm the wallet has enough USDC or conditional token approval. These endpoints require L2 auth and automatically include the client's `signature_type`.

```ts
import { AssetType } from "@polymarket/clob-client";

const usdc = await clobClient.getBalanceAllowance({ asset_type: AssetType.COLLATERAL });

const outcome = await clobClient.getBalanceAllowance({
    asset_type: AssetType.CONDITIONAL,
    token_id: tokenID,
});

await clobClient.updateBalanceAllowance({ asset_type: AssetType.COLLATERAL });
await clobClient.updateBalanceAllowance({
    asset_type: AssetType.CONDITIONAL,
    token_id: tokenID,
});
```

`AssetType.CONDITIONAL` requires `token_id`. See `examples/getBalanceAllowance.ts` and `examples/updateBalanceAllowance.ts`.

### Orders and operations

- Use `createAndPostOrder` or `createOrder` plus `postOrder` for limit orders.
- Use `createAndPostMarketOrder` or `createMarketOrder` plus `postOrder` for market-style FOK/FAK orders. For market buys, `amount` is the USDC amount to spend; for market sells, `amount` is the share amount to sell.
- Use `postOrders([{ order, orderType, postOnly }], deferExec, defaultPostOnly)` to submit a batch. Per-order `postOnly` overrides `defaultPostOnly`.
- `postOnly` is the fourth argument to `postOrder(order, OrderType.GTC, deferExec, postOnly)` and is supported for GTC/GTD orders. See `examples/postOnlyOrder.ts`.
- `postHeartbeat(heartbeatId)` keeps a heartbeat chain active. If heartbeats are started and one is not sent within 10 seconds, all orders are cancelled. Pass the previously returned `heartbeat_id` to continue the chain. See `examples/postHeartbeat.ts`.

Order management helpers require L2 auth:

```ts
await clobClient.cancelOrder({ orderID });
await clobClient.cancelOrders([orderID1, orderID2]);
await clobClient.cancelMarketOrders({ market: conditionID });
await clobClient.cancelAll();

const single = await clobClient.isOrderScoring({ order_id: orderID });
const batch = await clobClient.areOrdersScoring({ orderIds: [orderID1, orderID2] });
```

`cancelMarketOrders` accepts `market` and/or `asset_id` filters. `cancelAll` has no filters and cancels all cancellable open orders for the authenticated user.

### Market data and private reads

- Market list helpers `getMarkets`, `getSimplifiedMarkets`, `getSamplingMarkets`, and `getSamplingSimplifiedMarkets` return one `PaginationPayload` page for the supplied `next_cursor`.
- Public market data helpers such as `getOrderBook`, `getOrderBooks`, `getPrice`, `getPrices`, `getMidpoint`, `getMidpoints`, `getSpread`, and `getSpreads` do not require L2 credentials.
- Historical and activity helpers `getPricesHistory({ market, startTs, endTs, interval, fidelity })` and `getMarketTradesEvents(conditionID)` are public reads. See `examples/getPricesHistory.ts` and `examples/getMarketTradesEvents.ts`.
- `getTrades` and `getOpenOrders` require L2 auth and auto-page by default until the API returns the end cursor. Pass `only_first_page = true` to fetch only one page.
- `getTradesPaginated(params, next_cursor)` returns one page as `{ trades, next_cursor, limit, count }`.
- Pagination starts at cursor `MA==` and ends at cursor `LTE=`.
- `getOpenOrders` uses builder headers when the client has builder auth available.

```ts
import { PriceHistoryInterval } from "@polymarket/clob-client";

const history = await clobClient.getPricesHistory({
    market: tokenID,
    interval: PriceHistoryInterval.ONE_DAY,
    fidelity: 5,
});
```

Notifications are private reads. `getNotifications()` automatically includes the client's `signature_type`; `dropNotifications({ ids })` removes notifications by ID.

```ts
const notifications = await clobClient.getNotifications();
await clobClient.dropNotifications({ ids: ["notification-id"] });
```

### Rewards

Rewards helpers cover both public market reward configuration and authenticated user earnings:

```ts
const currentRewards = await clobClient.getCurrentRewards();
const marketRewards = await clobClient.getRawRewardsForMarket(conditionID);

const daily = await clobClient.getEarningsForUserForDay("2025-01-31");
const totals = await clobClient.getTotalEarningsForUserForDay("2025-01-31");
const percentages = await clobClient.getRewardPercentages();
const configured = await clobClient.getUserEarningsAndMarketsConfig(
    "2025-01-31",
    "earnings",
    "DESC",
    true,
);
```

`getCurrentRewards` and `getRawRewardsForMarket` are public. User earnings, reward percentages, and earnings-with-market-config methods require L2 auth and include the client's `signature_type`. Reward dates are UTC date strings in `YYYY-MM-DD` format. See `examples/rewards.ts`.

### Tick size cache

`createOrder`, market-order helpers, and RFQ order creation resolve a market's tick size before rounding prices and sizes. Valid tick sizes are `"0.1"`, `"0.01"`, `"0.001"`, and `"0.0001"`.

- The client caches tick sizes for 5 minutes by default; set constructor argument `tickSizeTtlMs` to change the TTL.
- Passing `options.tickSize` skips using a smaller value than the market minimum; smaller tick sizes throw `invalid tick size`.
- Call `clearTickSizeCache(tokenID)` to refresh one token or `clearTickSizeCache()` to clear all cached tick sizes.

### Examples

The `examples/` directory contains runnable scripts. Most examples load `.env` from the repository root.

| Category | Examples |
| - | - |
| Getting started | `createOrDeriveApiKey.ts`, `getMarkets.ts`, `order.ts`, `orders.ts` |
| Limit and market orders | `GTDOrder.ts`, `marketBuyOrder.ts`, `marketSellOrder.ts`, `postOnlyOrder.ts`, `matchOrders.ts` |
| Order data | `getOrder.ts`, `getOpenOrders.ts`, `getOrderbook.ts`, `getOrderbooks.ts`, `getTrades.ts`, `getTradesPaginated.ts` |
| RFQ | `rfqFullFlow.ts` and the `rfq*.ts` scripts |
| Builder | `createBuilderApiKey.ts`, `getBuilderApiKeys.ts`, `revokeBuilderApiKeys.ts`, `getBuilderTrades.ts`, `getBuilderOpenOrders.ts` |
| Readonly keys | `createReadonlyApiKey.ts`, `getReadonlyApiKeys.ts`, `deleteReadonlyApiKey.ts`, `getOpenOrdersWithReadonlyKey.ts` |
| Operations | `postHeartbeat.ts`, `geoToken.ts`, `getServerTime.ts`, `cancelOrder.ts`, `cancelOrders.ts`, `cancelMarketOrders.ts`, `cancelAll.ts` |
| Notifications and scoring | `getNofications.ts`, `dropNofications.ts`, `isOrderScoring.ts`, `areOrdersScoring.ts` |
| Rewards and analytics | `rewards.ts`, `getPricesHistory.ts`, `getMarketTradesEvents.ts`, `getLastTradePrice.ts`, `getLastTradesPrices.ts` |
| WebSocket | `socketConnection.ts` uses the `ws` dev dependency and is an example script, not a package export. |

### Error handling and retries

HTTP helpers normalize Axios failures into structured return values; they do not throw or log failed API responses by default. Applications should inspect the return value before treating a response as successful, or opt into exceptions.

By default, API errors are returned as `{ error: "...", status: ... }` objects. To have the client throw errors instead, pass `throwOnError: true` as the last constructor argument:

```ts
import { ClobClient, ApiError } from "@polymarket/clob-client";

const clobClient = new ClobClient(
    host, 137, signer, await creds, signatureType, funder,
    undefined, // geoBlockToken
    undefined, // useServerTime
    undefined, // builderConfig
    undefined, // getSigner
    undefined, // retryOnError
    undefined, // tickSizeTtlMs
    true,      // throwOnError
);

try {
    const book = await clobClient.getOrderBook(tokenID);
} catch (e) {
    if (e instanceof ApiError) {
        console.log(e.message); // "No orderbook exists for the requested token id"
        console.log(e.status);  // 404
        console.log(e.data);    // normalized error response object
    }
}
```

`throwOnError` checks the normalized response object returned by the client helpers. Responses with an `error` field become `ApiError`; successful responses are returned unchanged. Authentication and configuration failures, such as missing L1 or L2 credentials, still throw regular `Error` instances from the relevant client method.

`retryOnError` is the eleventh constructor argument and applies to POST requests made through `ClobClient`. When enabled, the HTTP helper makes one additional POST attempt after a 30 ms delay for network errors, selected network error codes, and HTTP 5xx responses. GET, PUT, and DELETE helpers are not retried automatically.

### Development

```bash
pnpm install
pnpm lint
pnpm test
pnpm build
pnpm ci
```

The package requires Node.js `>=20.10` and uses ESM with TypeScript `moduleResolution: "nodenext"`. Local source and test imports intentionally include `.ts` extensions; `rewriteRelativeImportExtensions` rewrites relative imports for the emitted `dist/` package during `pnpm build`.

When adding TypeScript files:

- Use `import type` for type-only imports because `verbatimModuleSyntax` is enabled.
- Keep runtime imports extension-qualified, matching the existing `../src/index.ts` and `./client.ts` style. Package consumers should continue importing from `@polymarket/clob-client`.
- Put package code under `src/`; `pnpm build` compiles `src/` via `tsconfig.build.json`, while `pnpm typecheck` checks tests and their imported source through `tsconfig.test.json`.
- Remember that `pnpm lint` currently checks `src/` only. Run `pnpm ci` before publishing or opening package changes.
- Use `.env.example` as a starting point when running example scripts locally.
