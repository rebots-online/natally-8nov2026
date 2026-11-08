# Natally — CHECKLIST (clean-room rewrite implementation)

Translation of `DOCS/ARCHITECTURE.md` 1:1 into disjoint, file-owned
implementation tasks. No task may widen or narrow the architecture. All
behaviour, copy, media, voice and record references are ARCH sections
(`§` = DOCS/ARCHITECTURE.md).

## Governance

- **G1** — `DOCS/ARCHITECTURE.md` published → **HOLD for operator sign-off.**
- **G2** — this CHECKLIST published → **HOLD for operator sign-off.** No
  source edit before both sign-offs.
- Dispatch: T0 first; then T1–T9 in parallel (disjoint files, frozen
  contracts only); then T10–T12; then T13; T14 throughout/final.
- One coder per task; the same coder fixes its deficiencies. One commit (+
  push) per completed task; commit messages carry no version numbers.
- Status legend: ⬜ not started · 🔧 in progress · ✅ operator-verified via the
  task's recorded journey · 🛑 blocked (state blocker in Notes).
- Every task: implement only its owned files; consume other modules only via
  ARCH §5 contracts; no placeholder functionality; no secrets in repo; no
  test suites — acceptance is the recorded journey.

| ID | Task | Owned files (exclusive) | Depends | Order | Status |
| --- | --- | --- | --- | --- | --- |
| T0 | SHELL-CONTRACTS | `package.json`, `tsconfig.json`, `vite.config.ts`, `.env.example`, `index.html`, `landing.html`, `souvenir.html`, `src/shared/**`, `src/platform/{env,assets,nativeBridge,eventBus}.ts`, `scripts/environment.mjs` | — | 0 | ⬜ |
| T1 | ASSET-BINDING | `public/**`, `asset-manifest.json` (assets track — complete; row records the binding contract) | — | 0 | ✅ |
| T2 | ASTRO-CORE | `src/astro/**` | T0 | 1 | ⬜ |
| T3 | VOICE-ENGINE | `src/voice/**` | T0, T1 | 1 | ⬜ |
| T4 | AUTHORED-CONTENT | `src/content/**` | T0 | 1 | ⬜ |
| T5 | IDENTITY-ONBOARDING | `src/identity/**`, `src/onboarding/**`, `firestore.rules` | T0 | 1 | ⬜ |
| T6 | CHART-RENDERER | `src/chart/**` | T0, T2 (types only) | 1 | ⬜ |
| T7 | BILLING-SERVER | `billing/**`, `netlify/functions/billing.mjs`, `netlify.toml`, `scripts/deploy-billing-netlify.mjs` | T0 | 1 | ⬜ |
| T8 | COMMERCE-CLIENT | `src/commerce/**` | T0 (runtime: T7 deployed) | 2 | ⬜ |
| T9 | INFERENCE-CHAT | `src/app/inference/**` | T0 (types: T2, T8 frozen) | 2 | ⬜ |
| T10 | APP-SURFACES | `src/app/**` except `src/app/inference/**` | T2–T9 | 3 | ⬜ |
| T11 | LANDING | `src/landing/**` | T2, T3, T6 | 3 | ⬜ |
| T12 | SOUVENIR | `src/souvenir/**` | T2, T4, T7, T8, T9 | 3 | ⬜ |
| T13 | PLATFORM-RELEASE | `src/main.tsx`, `src/platform/{entry,analytics,version,pwa}.ts`, `scripts/{build-web,pwa-build,verify-offer-prices,release-state,deploy-existing}.mjs`, `scripts/build-linux.sh`, `build_all.sh`, `build_all_win.ps1`, `src-tauri/**` | all | 4 | ⬜ |
| T14 | JOURNEY-PROOF | `DOCS/DEMONSTRATIONS/**` (operator + main) | all | 5 | ⬜ |

---

## T0 — SHELL-CONTRACTS

**Provides.** `src/shared/types.ts` = ARCH §5.1 exactly (all types frozen);
`src/shared/credits.ts` `formatCr` (≤2 fractional digits, suffix `Cr`);
`src/shared/tokens.css` (Playfair Display via `public/fonts/playfair-display-latin.woff2`,
plum/brass palette tokens frozen to the original look); `src/platform/env.ts`
(typed access to every `VITE_*` key of §7M, defaults in code); `src/platform/assets.ts`
(typed loader over `asset-manifest.json` at repo root: `assetUrl(id)`, integrity
check on boot); `src/platform/nativeBridge.ts` (`isTauri`, `isNativeAndroid`,
`isNativeDesktop`, `invoke`); `src/platform/eventBus.ts` (`recordEvent`,
`subscribeEvents`); `scripts/environment.mjs` (`resolve` subcommand: fresh,
process-only kstore:// resolution; prints public values, never resolved
secrets; exit ≠ 0 on missing pointer).
**Consumes.** `asset-manifest.json` (read-only).
**Implement exactly.**
1. Vite project: React + TypeScript strict; build inputs frozen: `index.html`
   (app), `landing.html` (landing), `souvenir.html` (souvenir atelier), web
   workers under `src/voice/`; dev server on 5173.
2. `.env.example` = every key of §7M with annotation + default; no kstore refs.
3. Document skeletons `index.html` (`#root`, app mount), `landing.html`
   (`#landing-root` + section scaffolding per §7A), `souvenir.html`
   (`#souvenir-root`) — no content beyond mounts/scaffolding.
4. `vite.config.ts` declares the emission contract for T13: `app.html` copy,
   landing emission, `marketing.html` redirect generation, price `data-bind`
   fill hook, version injection (`VITE_APP_VERSION`, default `dev`).
**Forbidden.** Any business logic, UI beyond document scaffolding, media
copies, guessing manifest ids (read them from the manifest).
**Accept (journey).** `npm run dev` serves `/`, `/landing`, `/app.html` mounts
without console errors; `node scripts/environment.mjs resolve --public` prints
the public schema; every asset id of §7N resolves through `assets.ts`.

## T1 — ASSET-BINDING (assets track — COMPLETE)

Media imported byte-for-byte; `asset-manifest.json` (repo root) is the
binding contract (§5.4, §7N): mascot master, atelier scene trio + descriptor,
9 atelier audio files, 10 waiting cues + provenance, Swiss Ephemeris trio,
gazetteer, Playfair Display, branding/marketing/paywall art, native icon sets,
notices, external pinned Supertonic model reference (F4, speed 0.9).
**Accept (journey).** Manifest sha256 integrity re-verified; all §7N ids load.
**Note.** No coder touches `public/**` or `asset-manifest.json`; defects in
media go back to the assets track.

## T2 — ASTRO-CORE

**Provides.** §5.2 astrology APIs on the frozen §5.1 types.
**Consumes.** `assets.ts` (swisseph trio, gazetteer dataset).
**Implement exactly.**
1. Load `public/vendor/swisseph/*` wasm engine; deterministic local
   computation; AGPL notice linked in-app (§7N).
2. `calculateNatalChart`: 14 bodies incl. Ascendant/Midheaven; 12 cusps with
   titles/cardo flags; aspects sorted by orb; angles + formatted strings;
   `houseKnown=false` when `birthTimeUnknown`; `engineVersion`, `julianDayUT`.
3. House systems per §7D, default `'W'`.
4. `currentSky` transit snapshot (bodies + aspects-to-natal).
5. `calculateSynastry` cross-aspects sorted by orb.
6. `lookupPlace` over `public/data/gazetteer.json`.
**Forbidden.** Network calls, persistence, UI, ephemeris re-implementation or
substitution of the vendored engine.
**Accept (journey).** Frozen example input (§7A EXAMPLE_CHART_INPUT) computes
once per session, stable across reloads: all 14 bodies, 12 cusps, aspect list
rendered in a dev harness; birth-time-unknown run suppresses houses.

## T3 — VOICE-ENGINE

**Provides.** §5.2 speech APIs + admission bus + retained audio.
**Consumes.** `assets.ts` (waiting cues, provenance), external model contract
(§7N), `eventBus`.
**Implement exactly.**
1. Supertonic ONNX worker on `@huggingface/transformers@4.3.1`; model fetched
   at runtime from the pinned revision; voice **F4**, speed **0.9**; cached in
   Cache Storage; honest download progress.
2. `speak()` synthesis + playback; mouth-event stream for the mascot.
3. Streaming chunked synthesis with gapless scheduling (waiting cues from
   `public/voice/waiting/` while text is pending).
4. Admission bus per §7H: busy idle admission returns false before stopping
   anything (NO idle queue); stale completion/abort cannot release/stop a
   newer owner; explicit replay/selection supersedes, stopping old playback
   first; no autostart without gesture (API refuses until unlocked).
5. Retained-audio record/replay per reading id.
**Forbidden.** Voice/provider substitution, speed or voice changes, queueing
idle speech, touching mascot rendering (T10 owns it; consume mouth events).
**Accept (journey).** Slow streaming reply speaks a waiting cue, then speech
continues gaplessly; busy engine refuses an idle nudge while a reply streams
(nudge discarded, quiet timing restarted); explicit Replay stops the old
playback and starts the new; second visit uses the cached model (no
re-download).

## T4 — AUTHORED-CONTENT

**Provides.** §5.2 lesson/session/close/nudge/souvenir-reply APIs.
**Implement exactly.**
1. Port the **original authored passages verbatim** (content-only port from
   the original content modules; new code around them): LessonCore
   `{what, regimes, evolution}`; BODIES (14), SIGN_CORES (12), HOUSE_CORES
   (12) + HOUSE_TOPICS, ASPECT_CORES (5), GENERAL_LESSON, CONTEXT_LINE,
   INVITATIONS, conversation close, upsell closes (both variants, light
   rotation), idle nudge library, souvenir frozen-question replies.
2. `buildLesson` composes what → regimes → evolution; session rotation without
   verbatim back-to-back repetition; told-lesson dedup.
3. Register the frozen customer-value copy constant (§7B) for the account UI.
4. A dev-only coverage harness renders every key's full text (no empty rows).
**Forbidden.** Rewriting, modernizing or shortening any passage; placeholder
strings; generated text; network calls.
**Accept (journey).** Coverage harness shows authored text (what+regimes+
evolution) for every body/sign/house/aspect key, all closes rotate, nudges and
souvenir replies return original wording.

## T5 — IDENTITY-ONBOARDING

**Provides.** §5.2 identity + onboarding APIs; `AuthModal` component;
`firestore.rules`.
**Consumes.** T0 bridge/env; T4 value copy (display only via T10).
**Implement exactly.**
1. Firebase init (existing project; authorized domains §2). Sign-in flows:
   Google popup (web), phone + reCAPTCHA (web + Android WebView), email;
   native `signInNativeGoogle()` via `invoke('native_sign_in')` for
   `isNativeDesktop() || isNativeAndroid()`, resolving `{idToken}` →
   `signInWithCredential`, `{cancelled:true}` → modal restored (no hang).
2. No silent linking: any sign-in submitted while `auth.currentUser != null`
   signs out and awaits null first; signed-in view never renders sign-in
   forms; phone-only accounts get the 30-day link prompt (§7E frozen copy).
3. `accountStorageKey` (null without uid) + the §6 key scheme (continuity keys
   exact); `onUid` + `sessionGeneration` isolation reset contract (§8.1) for
   T10; `signOutAccount()` = `/billing/logout` → Firebase sign-out → clear
   local access (Log Out button itself is T10's dropdown).
4. Intake form + store (local key + `users/{uid}/private/intake`, remote
   wins), gazetteer place picker, `getEntryStage`, restart flag, first-light
   hand-off.
5. `firestore.rules` frozen per §6 (own `users/{uid}/private/**` only;
   `natallyCredit*` server-only; souvenir docs create-only for the owner).
**Forbidden.** Balance/purchase logic; touching billing endpoints beyond
`/billing/logout`; storing user content outside the §6 scheme.
**Accept (journey).** Each method signs in/out on web; two accounts A/B: B
sees empty chat/intake + onboarding, A's data returns on switch-back; uid shown
in panel; phone-only account sees the link prompt once (30-day suppression);
stale-generation late result visibly discarded.

## T6 — CHART-RENDERER

**Provides.** `Astrolabe3D` (+ frozen props interface), tiers/boundary,
callout placement, focus visuals.
**Consumes.** §5.1 chart types (T2 supplies values at runtime).
**Implement exactly.**
1. Vivid renderer per §7C: plum-and-metal structure frame, bodies, signs,
   houses, aspects, current-sky bodies, baseSpin, drag+momentum, wheel/pinch
   zoom, halos, annotation modes, reduced motion, inert-pane a11y.
2. Tiers WebGL → low-power WebGL → CPU SVGRenderer + `ChartErrorBoundary`.
3. Focus dimming: representational elements dim; `structureFrame` meshes
   never dim.
4. Two-path inset activation + `pendingInsetSelection` clearing rules.
5. Hover callouts: outside-inset exclusion rectangle, viewport overlay, 8 px
   margin / 12 px clearance, side-then-above/below placement, refresh/dismiss
   on drag/scroll/resize/zoom/transition, no-over-chart fallback (dismiss),
   no stale content, endpoint cards off the inset, per-renderer overlay
   ownership.
6. `recordEvent` for activate/toggle/select.
**Forbidden.** Tooltips, canvases or overlays shared across renderer
instances; changing CHART_FOCUS art direction; network/inference.
**Accept (journey).** Screenshot proof: callout never overlaps the inset
(incl. border) at four viewport edges; inset object-tap focuses + toggles the
component after transition; empty-space tap focuses; GPU-blocked boot renders
the SVG tier without a blank page.

## T7 — BILLING-SERVER

**Provides.** §5.3 endpoints on Netlify `natally-hosted-chat`; §7K modules.
**Consumes.** kstore env via `scripts/environment.mjs` (T0).
**Implement exactly.**
1. Portable runtime + `netlify/functions/billing.mjs` wrapper; CORS §5.3;
  `netlify.toml` `[functions.billing] timeout = 26`.
2. Firebase ID-token verification on every authenticated route (never body
   uid).
3. RevenueCat ROCHE wallet (authority): idempotent integer transactions,
   `BILLING_CREDIT_UNITS_PER_USD=100`, usage debits once with fractional
   carry-forward, allowance = balance − unposted usage.
4. Grants: `round((gross−tax)×100×0.40)` ROCHE per owned production purchase,
   identity `purchase:<id>`, receipt digest keys (§7K), webhook/restore/
   reconcile never double-grant; refunds append reversals.
5. Redemption: configured code (default `Ch4n3l.C3351`, allowance `10.00`):
   exact compare after outer trim; SET available allowance to target,
   idempotent per `requestId` with before/after records; invalid mutates
   nothing; durable receipt; success message “Code redeemed. N credits
   added.”
6. Managed keys (§7K): limit-0 creation, 60 s expiry, absolute-limit idempotent
   `ensure`, rotation, AES-256-GCM custody (uid/hash/version AAD), logout
   disable, orphan/reconcile rules.
7. `/billing/history` (§7K.6 all-records + honest provider errors),
   invoices + PDF (§7K.7 seller block, manual HST policy, real facts only),
   `/billing/logout`, `/billing/webhook`.
8. `/billing/souvenir-deliver` (§7L): entitlement verification, SFTP/FTP
   upload trio, SMTP email, receipt links, idempotent + never overwriting.
9. `deploy-billing-netlify.mjs`: forces `BILLING_CREDIT_BALANCE_STORE=revenuecat`,
   imports all env keys via kstore, deploys to `natally-hosted-chat`,
   health-checks `/billing/health`.
**Forbidden.** New payment rails; balance authority outside RevenueCat;
client-trusted facts; test/dummy purchases; logging resolved secrets;
deleting/rewriting existing Firestore/RevenueCat records (read + append only,
working-copy mirror updates excepted).
**Accept (journey).** Deployed function: health 200; OPTIONS preflight from
`https://natally.robin.mba` returns the origin; real signed-in redeem on
staging sets 1000 Cr with receipt, retry same requestId → same result no
double-apply; reconcile after a real US$5 pack purchase grants 200 Cr once and
raises the managed key's absolute limit; refund path appends reversal and
lowers capacity; logout disables the key; invoice PDF carries the §7K.7 facts.

## T8 — COMMERCE-CLIENT

**Provides.** §5.2 commerce APIs.
**Consumes.** T5 identity (ID token), T7 endpoints (contract §5.3), env keys
§7M.
**Implement exactly.**
1. `billingApi` typed client with stale account/generation guards.
2. RevenueCat Web Billing SDK init (public web key); `purchasePack(pack)` →
   real purchase id; `fetchCheckoutCustomer()` projecting
   `CustomerInfo.nonSubscriptionTransactions` → `transactions[]` + balance.
3. `getCheckoutOffers('conversation')`: the three packs with real
   `formattedPrice`; credits = amountUsd × 40; failure → retry error state;
   zero packages → “Credit packs are not available yet.”.
4. `redeemCode`, `readAccess` (+ `readPaidConversationAccess`), `loadHistory`
   (browser-first merge per §7J; never “no purchases” on server failure).
5. `formatCr` usage rules; no provider/key/provisioning wording in any string
   this module owns.
**Forbidden.** Inventing prices client-side; SDK price overrides; purchase
completion inference from redirects/callbacks; USD1 or custom amounts.
**Accept (journey).** On staging with real backend: packs list with live
prices; US$5 one-tap purchase completes → reconcile `ready` → +200 Cr →
history row + invoice id; kill the network before history → SDK transactions
still listed with the actual error surfaced for the server part.

## T9 — INFERENCE-CHAT

**Provides.** §5.2 inference APIs.
**Consumes.** T8 access/key envelope, T2 chart facts, T3 streaming speech
contract, T4 fallback content.
**Implement exactly.**
1. `completeHosted` streaming on the managed key; model chain `openrouter/free`.
2. `withInferenceTurn`: access check → key ensure → run → account usage;
   refuses with authored fallback when `canInfer` false; stable turn ids;
   cancellation guards for rapid reselection.
3. Prompt/context builders: opening reading, comprehensive reading, synastry
   reading, follow-up conversation (context = resolved target names, natal
   facts, current-sky facts); message metadata `'Authored guide'` vs hosted
   model.
4. Chat-history persistence via T5 `accountStorageKey` (continuity key §6).
**Forbidden.** Inference without the paid-access gate; balance mutation
(server owns accounting); calls from landing/souvenir contexts.
**Accept (journey).** Funded account: a follow-up streams + speaks and the Cr
balance drops by actual usage after reconcile; depleted account: authored
fallback answers, zero network inference attempts (network log proof); cancel
mid-stream leaves no partial message.

## T10 — APP-SURFACES

**Provides.** the signed-in app per §7B.
**Consumes.** T2–T9 APIs (frozen), T1 assets.
**Implement exactly.**
1. `AppShell`: auth/intake/onboarding routing, uid-keyed remounts,
   `sessionGeneration` resets, onboarding gate (`getEntryStage`), header name
   = Firebase displayName.
2. Atelier: room + ambience selector, mascot puppet rig with mouth events,
   interaction sounds, Free cookies, “Here with you”, greeting + Replay,
   pane pair with inset drag/snap/keyboard/shelf-avoidance + persistence
   (`natally.inset-position.<uid>`).
3. Education pipeline (§7B): inset-originating selection → authored lesson
   (view → chart) → paid extension if `canInfer` → close (souvenir variant iff
   preceding visible turn was paid and access still true, else default) →
   revert to atelier unless manual view; Natally stays visible in the inset.
4. Chat surface incl. “Add credits for your personal reading” → conversation
   packs; readings surface (ReadingStage + saved readings + retained replay +
   synastry entry).
5. Account UI: top line (compact `N Cr` pill + Top Up + Souvenir always
   visible, one control typeface, narrow-screen wrap), profile dropdown with
   **Log Out as the last item** (`signOutAccount`), account modal (shared
   value copy first → Balance → packs dialog (T8 catalog; labels
   `${formattedPrice} — ${credits} Cr`) → “Have a code?” disclosure with
   Redeem), AccountCreditsEntry, PurchaseAccess routing (`?offer=…`),
   purchase confirmation states with honest failure copy.
6. Purchase history view per §7J (browser-first rows, invoice links, souvenir
   links, real errors).
**Forbidden.** Sign-out anywhere else; hiding Top Up/Souvenir by state; any
provider internals in customer wording; reading `src/app/inference` internals
(API only).
**Accept (journey).** Full atelier pass recorded: greeting speaks after
gesture, room switch swaps ambience, cookie bite plays; lesson → paid
extension → close choreography with view revert; Log Out (last item) revokes
key (balance pill resets, key disabled server-side) and signs out; top line
wraps cleanly at 360 px; fresh uid lands in onboarding.

## T11 — LANDING

**Provides.** the landing per §7A.
**Consumes.** T2 (example chart), T3 (speech after gesture), T6 (renderer),
T4 (lessons/closes), T1 assets.
**Implement exactly.**
1. Document + sections: header/nav frozen copy, hero replica (room + mascot +
   inset pair, Room sounds, Free cookies, “Here with you”, Hear Natally, Room
   selector, greeting + Replay), sticky hero with copy scrolling alongside,
   scroll-driven inset zoom both directions, eight sections from “Natal
   astrology, read aloud …”, Prices section with `data-bind` fills, Install
   Natally, version badge.
2. `EXAMPLE_CHART_INPUT` + `loadExampleChart` (once per session);
   `displayedChart` gates panes exactly as owned charts; frozen example
   caption + create-your-chart link.
3. Guest education session keyed `__marketing_guest__`; prefab lessons +
   Supertonic speech after explicit gesture; upsell close after each lesson;
   **zero inference/auth network calls** on the landing document.
4. Inset persistence `natally.marketing.inset-position.<roomUid>`; account/
   intake transitions clear guest-owned lesson/selection/speech.
**Forbidden.** Signed-in app mounting on `/landing`; any LLM/billing call;
substitute art or copy.
**Accept (journey).** Anonymous visit: hero sticky, scroll zooms inset in and
out, caption exact, lesson speaks after tap, close follows; network log shows
no inference/auth/billing requests; signed-in visit to `/landing` stays
landing; `/marketing.html?q` redirects preserving query.

## T12 — SOUVENIR

**Provides.** the souvenir system per §7L incl. the reading purchase dialog.
**Consumes.** T8 (purchase/entitlement), T9 (generator turn), T7
(`/billing/souvenir-deliver`), T2/T4 (facts/authored bases), T1 (theme art).
**Implement exactly.**
1. Purchase dialog for `?offer=reading` / souvenir entry points (US$30
   package; legacy product id accepted for entitlement); confirmed purchase id
   drives everything.
2. `createSouvenir` frozen snapshot (schemaVersion 2) create-only at
   `users/{uid}/private/souvenirs/{purchaseId}`.
3. `generateSouvenirContent`: sequential page-pair requests via
   `withInferenceTurn` on `openrouter/free`; resumable; honest progress
   (“Writing Houses: 7 of 12”); incomplete souvenirs are never delivered.
4. `buildSouvenirPdf` fixed layout (§7L: cover → chart → positions → 4 groups
   × [group cover + 2 pages/element] → conversation; houses-unknown caveat;
   generous whitespace); client download after confirmed delivery.
5. `deliverSouvenir` → server upload + email; history row `atelierUrl` +
   `pdfUrl`.
6. `souvenir.html` static atelier: immutable snapshot source, replay/chart/
   spoken reread, authored frozen-question replies, no LLM/sign-in/network
   beyond own files.
**Forbidden.** Delivering without verified entitlement; regenerating existing
`generated` content; placeholder lore; LLM calls in the viewer.
**Accept (journey).** Real purchase on staging: progress reaches 12/12 houses,
PDF downloads with the full layout, SFTP trio appears under
`<root>/<uid>/<transaction>/`, email with both links arrives, history row
links work, viewer replays conversation + chart with speech and answers an
out-of-conversation question with the authored Top Up reply.

## T13 — PLATFORM-RELEASE

**Provides.** entry dispatch, analytics sinks, version/PWA, builds, signing,
deploy (§7M).
**Implement exactly.**
1. `resolveEntryExperience` + `src/main.tsx` dispatcher (§3 table incl. bare-
   domain redirect rule); dynamic app/landing imports preserve the bundle
   split.
2. PWA: manifest (name/cache from env), SW shell routes `/`, `/app.html`,
   landing path + alias; offline shell.
3. Analytics sinks: staging evidence overlay + dev feed on the T0 bus; no
   third-party property.
4. Version: badge source `VITE_APP_VERSION`; `release-state.mjs` stamp
   (MINOR auto-increment), post-deploy stamp commit `v<next>: next after
   deployed v<this>`; inventories `dist/<prefix>-build-inventory.json/.xml`;
   production evidence strings (§7M).
5. `build_all.sh` (Linux-only) + `build_all_win.ps1` (native Windows
   PowerShell-only), independent coordinators; full matrix per §7M with
   honest outcomes; Android signing chain (dynamic build-tools selection,
   SHA-256 verify §2, 16 KiB alignment, signed AAB + symbols); cargo-xwin
   cross EXE/NSIS; MSI honestly pending on Linux; no installs during compile;
   environment failures abort pre-stamp.
6. `verify-offer-prices.mjs` price gate before the web build (kstore v2 key);
   `deploy-existing.mjs` Surge deploy (CNAME staging, hash + served-byte
   verification, domain allowlist, explicit promotion).
7. `src-tauri/**`: Windows/Android/Linux shells, icons from `public/native/**`,
   deep-link + opener plugins, Rust `native_sign_in` (desktop loopback PKCE +
   Android deep-link PKCE, `{idToken}` / `{cancelled:true}`).
**Forbidden.** Implicit production publish; test-build/not-a-release wording;
version numbers in non-stamp commits; debug keystore fallback; logging
resolved secrets; cross-host coordinator calls.
**Accept (journey).** Linux host `build_all.sh` run: full matrix built or
honestly reported, APK verified against the §2 SHA-256, inventories
production-named; price gate blocks a wrong-price build; staging deploy
screencast; explicit promotion to `natally.robin.mba`; post-deploy stamp
commit exists and no other commit carries a version.

## T14 — JOURNEY-PROOF (operator + main; evidence directory)

Records under `DOCS/DEMONSTRATIONS/<journey>-<date>/` (screencast + redacted
JSON). Matrix (each = one recorded journey against the real backend):

| # | Journey | Proves |
| --- | --- | --- |
| J1 | Purchase E2E | Top Up always shows packs → US$5 real purchase → +200 Cr → history + invoice PDF → chat turn on provisioned key |
| J2 | Redemption | code sets 1000 Cr; retry idempotent; invalid code mutates nothing |
| J3 | Privacy isolation | two accounts: no leak of name/chat/intake; onboarding on fresh uid; restore on return; phone-only link prompt once |
| J4 | Android Google | production-signed APK: browser flow **completes** (signed in) and **cancels** (modal restored); phone sign-in works |
| J5 | Log Out | dropdown last item → key revoked (server) → signed out |
| J6 | Souvenir E2E | §7L/T12 accept journey |
| J7 | Voice busy | waiting cue, idle suppression, replay supersedes, model cache |
| J8 | Landing | replica + scroll zoom + caption + price binds + zero-inference network log |
| J9 | Education branch | prefab → paid extension → both close variants |
| J10 | Invoice/refund/restore | invoice facts; refund reversal; restore repairs exactly once |
| J11 | PWA | offline shell, install, version badge |
| J12 | Release | build matrix + signed Android + staging screencast + promotion + stamp |

A task flips ✅ only when its accept-journey (and any matrix rows it owns) is
recorded and operator-verified. Static checks supplement, never substitute.
