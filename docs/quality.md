# How the driver app was kept correct

A record of the practices and the release checks, from the project's own documentation at release time (Oct 2026). Internal hosts, ticket numbers and the client's API document are left out.

## Contract first

The app was built against a written API document. Instead of generated types (the shared schema package did not exist yet), one definition of every payload lives in four files: `endpoints` (routes), `transport` (auth, refresh, idempotency, errors), `mappers` (the only code that reads server JSON) and `errors` (code → driver-facing text). A matrix tracked every documented endpoint against the app with a status of MATCH, DIFF, GAP or DONE-UNIT (wrapper exists and is unit-tested, not yet run live), so nothing was assumed to work because it compiled.

## The test pyramid

| Layer | Runs in | What it proves |
|---|---|---|
| **Unit** (243 at release) | `node --test`, milliseconds, no React Native | Domain rules (swap gate, step view, validation, money), mappers against the response examples from the contract, transport behaviour (single-flight refresh, idempotency key reuse, error mapping) |
| **API end-to-end** (16 journeys) | the real transport + endpoints + mappers against a steerable mock backend | Sign-in, KYC, stations, the full swap, dues, history, notifications, support, profile, and the failure modes (503, latency, expiry, station blocked, no charged battery, swap faults) |
| **Device** (16 steps) | an emulator or a phone in Expo Go, driven by a runner that starts the mock and Metro | The real UI: sign-in, stations, a swap, receipt, dues, history, notifications, profile, sign-out; PASS/FAIL per step with screenshots of failures |
| **Live smoke** | read-only calls against the real server | Reachability and contract shape in each environment; never creates or changes data |

The mock backend is a test double that follows the contract and can be steered through a QA endpoint. It proves the app matches the contract; it proves nothing about the real server, and the documentation says so. Production code has no mock mode.

## Release readiness, as a checklist

Before a build was called releasable:

- `npm test` green (243/243) and `npm run test:e2e` green (16/16), run and recorded with the date.
- Device run on an Android emulator against the mock, all 16 steps passing.
- Frame-by-frame comparison with the Figma design (about 50 frames), each frame marked Match, Intended difference, Fixed, Gap or Code only. Intended differences were product decisions written down with the reason (for example: the design shows UPI and auto-pay screens; the contract excludes payments, so those stay non-functional and the Dues screen says so).
- Security pass: tokens in the OS keystore only; KYC photos re-encoded without metadata and deleted after upload; screenshot blocking on KYC screens; Android backup off; no sensitive values in logs; test coordinates disabled in release builds.
- Environment safety: the API address comes from the run command or build profile; a missing or misspelt environment falls back to development, never production.
- Known gaps listed explicitly (for example: push notifications built but untested with a real token; a flow verified only against the mock because the live station was not available), so nobody mistook "tested" for "tested live".

## Things that bit, and what changed

- **Expo SDK upgrade and multipart uploads.** The runtime `fetch` on the new SDK rejected the older `{ uri, name, type }` form part. The upload moved to the platform `File` object. Found by the e2e layer, fixed the same day, covered by a test.
- **Parallel calls and token refresh.** Two requests hitting a 401 at once used to trigger two refreshes. The transport now runs one refresh and lets the loser retry with the winner's token.
- **Retrying a swap authorisation.** A retry after a dropped reply could start two swaps. One idempotency key per attempt, reused on retry, closed that.
- **Clock drift on cheap phones.** Step countdowns follow the server's `Date` header, not the phone clock.

## Design decisions kept on record

- Dues are read-only in the app by contract; no payment SDK is bundled and no payment code is added without a documented contract.
- Reservation code paths stay behind a flag until reservations are in scope.
- The vehicle and battery screens were removed by a product decision on a dated note, not silently.
