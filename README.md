# @sudobility/mogulgame_lib

Pure, synchronous business-logic helpers for **MogulGame** — offer validation,
balance/offer aggregates, and country-scoped currency formatting.

There are **no stores, no hooks, no network calls, and no React** in this
package. It is a bag of pure functions: same input, same output, no I/O. View
code (`mogulgame_app`, `mogulgame_app_rn`) calls these helpers before or after
the `mogulgame_client` network hooks; this library never sits in the request
path.

- **Package**: `@sudobility/mogulgame_lib` (private, scoped)
- **Registry**: GitHub Packages — `https://npm.pkg.github.com`
- **License**: BUSL-1.1

## Installation

This package is published to **GitHub Packages**, not the public npm registry.
Consumers need an `.npmrc` that routes the `@sudobility` scope there (and a
GitHub token with `read:packages`):

```
@sudobility:registry=https://npm.pkg.github.com
```

```bash
bun add @sudobility/mogulgame_lib
```

The compiled type declarations reference `@sudobility/mogulgame_types`, so
consumers must also have that package available for types to resolve.

## Usage

```ts
import {
  calculateMaxOffer,
  validateOfferPrice,
  formatPrice,
  formatPriceShort,
} from "@sudobility/mogulgame_lib";

const balance = 100_000;

calculateMaxOffer(balance); // 500000  (5x balance)

validateOfferPrice(600_000, balance);
// "Offer exceeds maximum allowed ($500,000)"
validateOfferPrice(400_000, balance); // null  (valid)

formatPrice(1_500_000, "GB"); // "£1,500,000"
formatPriceShort(1_500_000, "US"); // "$1.5M"
```

## API

All exports live under `business/utils` and are re-exported from the package
root.

### Offers (`business/utils/offers.ts`)

| Export | Signature | Purpose |
|--------|-----------|---------|
| `MAX_OFFER_MULTIPLIER` | `5` | Max offer relative to balance |
| `calculateMaxOffer` | `(balance: number) => number` | `balance * 5` |
| `validateOfferPrice` | `(offerPrice: number, balance: number) => string \| null` | Error string, or `null` when valid |
| `calculateBalanceFromLedger` | `(transactions: Transaction[]) => number` | Sum a transaction ledger |
| `countActiveOffers` | `(offers: PretendOffer[]) => number` | Count `status === "active"` |
| `countWonOffers` | `(offers: PretendOffer[]) => number` | Count `status === "won"` |
| `totalActiveOfferValue` | `(offers: PretendOffer[]) => number` | Sum `offer_price` of active offers |
| `formatResolutionSummary` | `(resolution: PretendOfferResolution) => string` | Human-readable settlement summary |

### Currency (`business/utils/currency.ts`)

Country-scoped formatting for the six supported countries (`US`, `CA`, `GB`,
`AE`, `ES`, `AU`). The currency map itself is internal; use the functions.

| Export | Signature |
|--------|-----------|
| `formatPrice` | `(amount: number, country: CountryCode) => string` |
| `formatPriceShort` | `(amount: number, country: CountryCode) => string` — `$1.5M` / `$250K` |
| `getCurrencySymbol` | `(country: CountryCode) => string` |
| `getCurrencyCode` | `(country: CountryCode) => string` |

Types (`CountryCode`, `PretendOffer`, `PretendOfferResolution`, `Transaction`)
come from `@sudobility/mogulgame_types`.

## The 5x rule exists twice, on purpose

`validateOfferPrice` is a **pre-flight UX check** run client-side before an
offer is submitted. The authoritative check lives server-side in
`mogulgame_api` (`src/routes/offers.ts`). The duplication is deliberate: changing
one side without the other produces a UI that accepts offers the API rejects (or
vice versa).

## Development

Install dependencies first (Bun only — do not use npm/yarn/pnpm):

```bash
bun install
```

Scripts:

```bash
bun run build          # Compile ESM to dist/ (tsc + fix-esm-imports post-step)
bun run build:watch    # tsc --watch
bun run clean          # Remove dist/
bun run typecheck      # tsc --noEmit
bun run lint           # ESLint over src/
bun run lint:fix       # ESLint --fix
bun run format         # Prettier --write src/
bun run format:check   # Prettier --check src/
bun run test:unit      # vitest run  (see note below)
bun run test:watch     # vitest in watch mode
bun run verify         # typecheck + lint + test:unit + build (run before commit)
```

> **There are no test files yet.** `vitest` is configured with
> `passWithNoTests: true`, so `test:unit` prints "No test files found" and
> exits 0 — and `verify` therefore passes vacuously. Adding a `*.test.ts` file
> is what makes `verify` start protecting you.

## Build details

`bun run build` runs `tsc -p tsconfig.build.json` (declarations + source maps,
ESM) and then `scripts/fix-esm-imports.mjs`, which rewrites relative import
specifiers in the emitted `.js`/`.d.ts` to fully-qualified paths (`./x` →
`./x.js`) so Node's native ESM loader can resolve them. Only `dist/**/*` is
published (`files` in `package.json`); `prepublishOnly` cleans and rebuilds.

## Related packages

- **mogulgame_types** — Shared type definitions; this package imports `CountryCode`, `PretendOffer`, `PretendOfferResolution`, `Transaction`
- **mogulgame_client** — API client SDK. Not imported by this package (and vice versa)
- **mogulgame_api** — Backend; owns the authoritative offer validation. Must **not** import this package (React peer dep would drag React into a Bun server)
- **mogulgame_app** — Web frontend; calls these helpers directly from page components
- **mogulgame_app_rn** — React Native app

## License

BUSL-1.1
