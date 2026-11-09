# MogulGame Lib

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

Pure, synchronous business-logic helpers for MogulGame.

**Package**: `@sudobility/mogulgame_lib` (private, BUSL-1.1). Published to **GitHub Packages**
(`npm.pkg.github.com`), not the public npm registry — see `.npmrc` and `publish-target: github`
in `.github/workflows/ci-cd.yml`.

> **No stores, no hooks, no network, no React.** The `package.json` description still says "with
> Zustand stores" and `zustand` is still a peer dependency, but `zustand` is imported in **zero**
> files and `src/business/stores/index.ts` / `src/business/hooks/index.ts` are **0 bytes**. Treat
> this package as a bag of pure functions until someone deliberately changes that.

## Tech Stack

- **Language**: TypeScript (strict mode)
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Build**: TypeScript compiler (ESM)

## Project Structure

```
src/
├── index.ts
└── business/
    ├── index.ts
    ├── stores/index.ts               # EMPTY (0 bytes)
    ├── hooks/index.ts                # EMPTY (0 bytes)
    └── utils/
        ├── index.ts
        ├── offers.ts                 # Offer validation, balance and offer aggregates
        └── currency.ts               # Per-country price formatting
```

## Commands

```bash
bun install            # Required first (Bun only — never npm/yarn/pnpm)
bun run build          # tsc -p tsconfig.build.json + scripts/fix-esm-imports.mjs
bun run clean          # Remove dist/
bun run test:unit      # vitest run -- there are NO test files; exits 0 vacuously
bun run test:watch     # vitest in watch mode
bun run typecheck      # tsc --noEmit
bun run lint           # Run ESLint over src/
bun run verify         # typecheck + lint + test:unit + build (use before commit)
bun run prepublishOnly # Clean + build (runs on publish)
```

There is **no `bun run test` script** and `bun test` runs Bun's own runner, not vitest.
Use `test:unit` / `test:watch`.

## Public API

### `business/utils/offers.ts`

| Export | Purpose |
|--------|---------|
| `MAX_OFFER_MULTIPLIER` | `5` |
| `calculateMaxOffer(balance)` | `balance * 5` |
| `validateOfferPrice(offerPrice, balance)` | Returns an error string, or `null` when valid |
| `calculateBalanceFromLedger(transactions)` | Sum a transaction ledger |
| `countActiveOffers(offers)` | — |
| `countWonOffers(offers)` | — |
| `totalActiveOfferValue(offers)` | — |
| `formatResolutionSummary(resolution)` | Human-readable settlement summary |

### `business/utils/currency.ts`

`formatPrice(amount, country)`, `formatPriceShort(amount, country)` (`1.5M` / `250K`),
`getCurrencySymbol(country)`, `getCurrencyCode(country)`. Countries: `US CA GB AE ES AU`.
(`CURRENCY_MAP` is an internal const — **not** exported.)

## The 5x Rule Exists Twice, On Purpose

`validateOfferPrice` is a **pre-flight UX check** run in the browser before submitting. The
authoritative check is server-side in `mogulgame_api/src/routes/offers.ts`. Changing one without the
other creates a UI that accepts offers the API rejects, or vice versa. This duplication is deliberate,
not an accident to be "fixed" by deleting either side.

## Dependencies

Peer (from `package.json`):

- `@sudobility/types` — **not actually imported** (the real type import is `@sudobility/mogulgame_types`, see below)
- `react` (>=18) — **not actually imported**
- `@tanstack/react-query` (>=5) — **not actually imported**
- `zustand` (>=5) — **not actually imported**

The only module actually imported is `@sudobility/mogulgame_types` (`CountryCode`, `PretendOffer`,
`PretendOfferResolution`, `Transaction`). It is declared only as a **devDependency**, yet the emitted
`dist/**/*.d.ts` reference it — so consumers must also have `@sudobility/mogulgame_types` installed
for types to resolve.

The React and react-query peers are why `mogulgame_api` **must not import this package**: doing so
would drag React into a Bun server. Shared logic that both the API and the app need (e.g. `isCrawler`)
belongs in `mogulgame_types`, which has no React dependency.

## Related Projects

- **mogulgame_types** — Shared type definitions; this package imports `CountryCode`, `PretendOffer`, `Transaction`
- **mogulgame_client** — API client SDK. This package does **not** import it, and it does not import this one
- **mogulgame_api** — Backend. Owns the authoritative offer validation. **Cannot import this package** (React peer dep)
- **mogulgame_app** — Web frontend; calls these functions directly from page components
- **mogulgame_app_rn** — React Native app

## Coding Patterns

- Everything here is a **pure function**: same input, same output, no I/O, no React, no globals
- View code calls these helpers before or after the `mogulgame_client` hooks; this package never sits in the network path
- Currency and formatting are always country-scoped -- take a `CountryCode`, never assume USD

## Gotchas

- **There are no tests.** `bun run test:unit` prints "No test files found, exiting with code 0" (vitest `passWithNoTests: true`), so `bun run verify` passes vacuously. Adding a `*.test.ts` file is the only way `verify` starts protecting you
- `formatResolutionSummary`, `calculateBalanceFromLedger`, and `MAX_OFFER_MULTIPLIER` are exported but imported by nothing. Don't assume an export is load-bearing
- The `package.json` description and the `zustand` peer dependency are stale template residue -- see the note at the top
- This is a published npm package -- editing `src/` has no effect on `mogulgame_app` until published and the dep bumped; use `bun link` for local iteration

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
