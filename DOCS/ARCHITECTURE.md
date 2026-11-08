<!--
project: NATALLY
document: DOCS/ARCHITECTURE.md (clean-room rewrite)
created_at: 2026-11-08
status: frozen — architecture -> checklist -> implementation with recorded-journey proof (one coherent flow; no extra sign-off or review cycles)
authority: operator instruction (whole-app rewrite, same functionality/look/sound, reuse actual mascot/media, no layered old code)
requirements_source: original repo DOCS/ARCHITECTURE.md (read-only) as product requirements — no original source/history consulted
media_source: repo-root asset-manifest.json (assets track; byte-for-byte originals)
scope: the finished system, entire product boundary
safety: credentials only as kstore:// pointers; no secrets (including the redemption code value) in repo or docs; no placeholder functionality; no mock payments or invented data
-->

# Natally — clean-room rewrite architecture (finished-system snapshot)

This document describes the **finished system** the rewrite must deliver: every
substantive capability, entity, external record, interface and invariant. It is
a snapshot of the target, not a change history. Module layout, file names and
internal APIs are NEW; behaviour, appearance, sound, voice, copy and external
records are the original product's. Application code is written anew; the
mascot, room art, sounds, voice model, fonts, ephemeris engine and authored
words are the actual original material (asset-manifest.json + §7N, §7G).

## 0. Reading rules

1. Every table row is a deliverable. Anything absent from this document is out
   of scope; anything present is required.
2. Where this document quotes customer copy verbatim (marked **frozen copy**),
   the words are exact, including punctuation.
3. Money, identity and delivery invariants in §8 bind every module without
   exception.
4. CHECKLIST.md translates this document 1:1 into disjoint, file-owned
   implementation tasks. Nothing in CHECKLIST may widen or narrow this spec.
5. Delivery of any capability is established only by a **recorded running
   journey** (§10). Static inspection, type checks and test suites never
   establish delivery.

## 1. Product boundary

Natally is a voice-forward personal astrology studio. One product, three
surfaces, one identity:

- **Landing** (`/landing`): a working replica of the atelier with an anonymous
  example chart, authored education spoken in Natally's voice, prices, and
  entry to the app. No inference, no account required.
- **App** (`/app.html`, native shells): the signed-in atelier — the mascot
  room, the interactive 3D natal chart, onboarding intake, spoken authored
  lessons plus **paid** personalized readings and conversation, saved
  readings with retained audio, synastry, credit top-up, redemption, purchase
  history and invoices, and the purchased PDF souvenir with its hosted
  interactive atelier.
- **Native shells** (Tauri): Windows, Android, Linux packaging of the app,
  with native Google sign-in on desktop and Android.

Identity, voice and character are constant: the **Natally mascot** (metallic
silver face, amber eyes, purple celestial turban) in her atelier, speaking
with the **Supertonic voice F4** at speed 0.9, in the original factual
Gen-Z-leaning register. The chart renderer is the original plum-and-metal
vivid astrolabe. Typography centres on Playfair Display.

## 2. External systems and preserved records (unchanged)

The rewrite preserves the operator's existing external identity, payment and
hosting records exactly. **Same projects, same records, same domains.**

| System | Preserved record / value | Used by |
| --- | --- | --- |
| Firebase (existing project) | Auth users (Google / phone / email), display names; named Firestore: `users/{uid}/private/intake`, `natallyCreditAccounts/{uid}`, `natallyCreditGrants/{receiptId}`, `users/{uid}/private/souvenirs/{purchaseId}`; authorized-domains set ≤ 20 (`*.firebaseapp.com`, `*.web.app`, `localhost`, `tauri.localhost`, canonical + legacy + today's staging hosts) | identity, onboarding, billing server, souvenir |
| RevenueCat (existing project) | Sole customer balance + purchase authority. Currency `ROCHE` ("Roche Credit") as a raw integer ledger at `BILLING_CREDIT_UNITS_PER_USD = 1,000,000` units per USD (§ Money and units). Offering **`Natally`** (`ofrng352609e01b`): conversation packs → products `natally_credit_usd_500` / `natally_credit_usd_1000` / `natally_credit_usd_2500`; reading package → `natally_souvenir_reading_usd_3000` (`prod564fa6a5e4`); legacy `natally_reading_test_v3` (US$0.99) detached but **accepted** for souvenir entitlement. Web Billing (`rc_billing`, production). Customer id = Firebase uid. Existing wallet balances and purchase records are read **exactly as stored** — never migrated, re-scaled or reinterpreted (the recorded 79 raw ledger units remain 79 raw ledger units) | commerce client, billing server |
| OpenRouter (existing account) | Per-user managed keys API (create limit 0 / expiry / PATCH absolute limit / disable). Inference model chain `openrouter/free`. Operator tops the account up manually (§9) | billing server, inference |
| Netlify (existing site **`natally-hosted-chat`**) | Sole billing host: Function `billing`, 26 s timeout, `https://natally-hosted-chat.netlify.app` | billing server |
| Surge (existing domains) | Static hosting: `natally.surge.sh` and `natally.robin.mba` (canonical), plus per-run staging host; CNAME staging, served-byte verification | platform/release |
| Stack/20i shared hosting (existing account) | Souvenir delivery: SFTP/FTP upload root + SMTP for souvenir email (§7L) | billing server |
| Hugging Face | Supertonic ONNX voice model, pinned revision `cff123c84b0655d9d647641f1b532c3cbb8f7faa`, voice `F4` (§7N). Public download; no credentials | voice |
| Google OAuth | Web client (Firebase Google provider popup flow) + Android credential API client of the production upload key (§7I). Client ids in Firebase provider allowlist | identity |
| KStore (canonical credential storage) | `.env` holds **only** `kstore://` pointers. Pointers cover: RevenueCat secret v2 key, Firebase admin/CLI token (`CREDENTIALS/Natally/firebase-admin.md`), OpenRouter admin key, billing key-encryption key, the redemption code value (`BILLING_REDEMPTION_CODE` — never a literal in repo, docs or code), Android production keystore (`CREDENTIALS/prefetch-android-signing.md`), Netlify + Surge tokens, souvenir SFTP + SMTP credentials | platform/env, billing server |
| Android signing | One operator-owned production upload keystore; APK/AAB certificate SHA-256 `56C13674EF22DF95DEB1E5C468820E8CFA3EA2F522511749AB7B6E5BDE3BD943` | platform/release |

Business identity for receipts (frozen): **Roche Media Services,
HST826393555RT0001, 88 CORPORATE DR., TORONTO, ON M1H 3G6**. Natally computes
and remits its own GST/HST; no external tax service is used.

### Money and units (frozen — three distinct concepts, never conflated)

1. **Raw ledger units** — the RevenueCat `ROCHE` integer ledger on the
   authoritative wallet: `BILLING_CREDIT_UNITS_PER_USD = 1,000,000` units per
   USD (server configuration). Internal storage granularity only; never
   displayed, never converted by hand. Existing balances are read exactly as
   stored — this rewrite does **not** migrate, re-scale or reinterpret them.
2. **Spendable allowance** — the USD value available for inference spend.
   A verified purchase funds allowance = verified **pre-tax** paid USD × the
   server's configured allocation `BILLING_TOPUP_ALLOCATION_BPS / 10,000`
   (currently 4000 bps = 40%). Redemption grants follow the configured grant
   action. Usage debits the authoritative ledger; the managed-key ceiling
   derives from the authoritative balance minus unposted usage.
3. **Display (Cr)** — presentation only: a displayed USD quantity ×
   `VITE_CREDIT_DISPLAY_PER_USD = 100`, through the one shared formatter,
   ≤ 2 fractional digits, suffix `Cr`. Pack-selection labels follow the
   approved formula pack amount USD × display factor (US$5 → **500 Cr**,
   US$10 → 1000 Cr, US$25 → 2500 Cr). A funded allowance displayed to the
   customer uses the same factor applied to the allowance USD (at 4000 bps a
   US$5 pack funds US$2.00 allowance → displayed 200 Cr). Raw ledger units
   are never shown and never equated to Cr.

## 3. Entry contract, documents and PWA

| URL / document | Anonymous | Authenticated session |
| --- | --- | --- |
| `/`, `/index.html` (bare domain) | unified-entry in-place selection: the landing renders while auth is unresolved and for anonymous visitors after resolution | the app mounts in place |
| `/landing` and alias `/landing.html` | always landing (forced landing) | same |
| `/app.html`, `?app=1`, `?offer=reading\|conversation` | always app | same |
| `/marketing.html` (legacy) | static redirect to landing path, query + hash preserved | same |
| Tauri native shells | always app | same |

- Entry keys (closed set, env-driven, no routing literal hardcoded where a key
  governs it): `VITE_LANDING_PATH=/landing`, `VITE_LANDING_PATH_ALIAS=/landing.html`,
  `VITE_APP_PATH=/app.html`, `VITE_LEGACY_PATH_REDIRECTS=/marketing.html:/landing`,
  `VITE_APP_NAME=Natally`, `VITE_APP_NAMESPACE=natally`.
- Source documents: `index.html` (app; Tauri + dev source), `landing.html`
  (landing source), `marketing.html` (generated redirect), `souvenir.html`
  (static souvenir-atelier build input, §7L). Web emission: `index.html`
  emitted and copied byte-identical to `app.html`; `landing.html` emitted;
  landing document keeps its version badge.
- Bare-domain behaviour (unified entry contract): the document selects the
  experience **in place** — it awaits Firebase auth resolution, renders the
  landing while unresolved/anonymous, and swaps into the app in place on an
  authenticated session; the auth listener stays active only for that
  document. **No forced redirect is introduced.** Sign-in on the landing
  stays on the landing; the explicit `/landing`, `/app.html` and query-driven
  entries behave exactly per the table.
- PWA: manifest name/cache prefix from `VITE_APP_NAME`/`VITE_APP_NAMESPACE`;
  service-worker shell routes = `/`, `/app.html`, landing path + alias;
  installable ("Install Natally" landing entry); offline serves the shell.

## 4. Module map (clean layout, disjoint ownership)

| Module | Path | Role | Checklist owner |
| --- | --- | --- | --- |
| Contracts shell | `package.json`, `tsconfig.json`, `vite.config.ts`, `.env.example`, `index.html`, `landing.html`, `souvenir.html`, `src/shared/**`, `src/platform/{env,assets,nativeBridge,eventBus}.ts`, `scripts/environment.mjs` | frozen types, formatters, typed env, asset loader, native bridge client, event bus, kstore resolve | T0 |
| Assets | `public/**`, `asset-manifest.json` | actual original media (done; assets track) | T1 |
| Astrology core | `src/astro/**` | ephemeris, gazetteer, transits, synastry (pure, deterministic) | T2 |
| Voice engine | `src/voice/**` | Supertonic F4 synthesis, streaming speech, waiting cues, retained audio, admission bus | T3 |
| Authored content | `src/content/**` | verbatim authored lesson cores, sessions, nudges, closes, souvenir replies | T4 |
| Identity + onboarding | `src/identity/**`, `src/onboarding/**`, `firestore.rules` | Firebase init, auth modal, native auth, account storage, isolation, intake, first-light | T5 |
| Chart renderer | `src/chart/**` | Astrolabe3D vivid renderer, tiers, callouts, focus visuals | T6 |
| Billing server | `billing/**`, `netlify/functions/billing.mjs`, `netlify.toml`, `scripts/deploy-billing-netlify.mjs` | Netlify-hosted billing runtime (§7K) | T7 |
| Commerce client | `src/commerce/**` | billing API client, RevenueCat checkout, packs, history, access | T8 |
| Inference + chat engine | `src/app/inference/**` | hosted inference, turn accounting, reading composition, prompts | T9 |
| App surfaces | `src/app/**` (except `inference/`) | shell, atelier room + mascot, panes, education pipeline, chat UI, readings UI, account UI | T10 |
| Landing | `src/landing/**` | landing replica, example chart, price bindings | T11 |
| Souvenir | `src/souvenir/**` | snapshot, content generator, PDF, delivery, viewer, purchase dialog | T12 |
| Platform + release | `src/main.tsx`, `src/platform/{entry,analytics,version,pwa}.ts`, `scripts/{build-web,pwa-build,verify-offer-prices,release-state,deploy-existing}.mjs`, `scripts/build-linux.sh`, `build_all.sh`, `build_all_win.ps1`, `src-tauri/**` | entry dispatch, analytics sinks, version, PWA, builds, signing, deploy | T13 |

Rule: a module consumes another module only through the frozen contracts in
§5. Cross-module imports of internals are forbidden. CSS is owned by each
surface (`src/app/**/*.css`, `src/landing/landing.css`); shared design tokens
(`src/shared/tokens.css`) are frozen in T0.

## 5. Shared contracts (frozen; defined once in T0, imported everywhere)

### 5.1 Domain types (`src/shared/types.ts`)

```ts
type HouseSystemId = 'W' | 'P' | 'K' | 'O' | 'R' | 'C' | 'E' | 'T' | 'B';   // default 'W'
type BodyId = 'Sun'|'Moon'|'Mercury'|'Venus'|'Mars'|'Jupiter'|'Saturn'|'Uranus'
            | 'Neptune'|'Pluto'|'Chiron'|'TrueNode'|'Ascendant'|'Midheaven';
type AspectType = 'Conjunction'|'Opposition'|'Square'|'Trine'|'Sextile';

interface NatalChartInput { name: string; birthDate: string; birthTime: string;
  birthTimeUnknown: boolean; placeName: string; latitude: number; longitude: number;
  timezoneOffset: number; houseSystem: HouseSystemId }

interface NatalBodyPosition { id: BodyId; name: string; glyph: string; lon: number;
  speed: number; isRetrograde: boolean; signName: string; signGlyph: string;
  degInSign: number; minInSign: number; degText: string; fullDegText: string; house: number | null }

interface HouseCusp { num: number; title: string; signName: string; signGlyph: string;
  formattedDeg: string; isCardo: boolean }

interface AspectChord { id: string; name: string; type: AspectType; symbol: string;
  from: BodyId | 'ASC' | 'MC'; to: BodyId | 'ASC' | 'MC'; angle: number; orb: number }

interface NatalChartResult { input: NatalChartInput; julianDayUT: number;
  ascLon: number; mcLon: number; dscLon: number; icLon: number;
  ascFormatted: string; mcFormatted: string; sunFormatted: string; moonFormatted: string;
  natalBodies: NatalBodyPosition[]; houseCusps: HouseCusp[];        // 12, ordered
  aspectChords: AspectChord[];                                       // sorted by orb asc
  houseKnown: boolean; engineVersion: string }

interface TransitSnapshot { date: string; bodies: NatalBodyPosition[];   // current sky
  aspectsToNatal: AspectChord[] }
interface SynastryResult { left: NatalChartResult; right: NatalChartResult;
  crossAspects: AspectChord[] }                                        // sorted by orb

type ChartComponentId = { kind: 'body'; id: BodyId } | { kind: 'sign'; id: number }
                      | { kind: 'house'; id: number } | { kind: 'aspect'; id: string };

interface VoiceSettings { voice: 'F4'; speed: 0.9 }                     // frozen (§7N)
type SpeechSource = 'greeting'|'lesson'|'chat'|'reading'|'nudge'|'souvenir'|'waiting';
interface SpeechRequest { text: string; source: SpeechSource; ownerId: string;
  onAdmitted?: () => void }
interface SpeechHandle { stop(): void }

// Money: three distinct quantities (§2 "Money and units"). balanceCr and
// labelCr are DISPLAY projections (USD × VITE_CREDIT_DISPLAY_PER_USD); raw
// RevenueCat ROCHE ledger units (BILLING_CREDIT_UNITS_PER_USD = 1,000,000
// per USD) never cross this boundary and are never displayed.
interface BillingAccess { accountUid: string; expiresAt: string;
  state: 'ready' | 'blocked' | 'unavailable';
  source: string; provider: string;
  balanceCr: number;      // displayed Cr = available spendable allowance USD × display factor
  redemption?: { requestId: string; beforeCr: number; afterCr: number; at: string };
  ready?: { apiKey: string; apiKeyHash: string; keyGeneration: number;
    accessExpiresAt: string; limitUsd: number; usageUsd: number } }

interface CreditPack { productId: string; packageId: string; amountUsd: number;
  formattedPrice: string; labelCr: number }  // labelCr = amountUsd × display factor (US$5 → 500 Cr);
                                             // funded allowance is server-side (allocation BPS), never on this type

interface CheckoutCustomer { balanceCr: number;   // display projection, §2
  transactions: { id: string; product: string; date: string }[] }       // SDK, browser-first

interface PurchaseHistoryRow { id: string; kind: 'purchase'|'redemption'|'refund'|'grant'|'souvenir';
  date: string; description: string; amountUsd?: number; currency?: string; status: string;
  invoiceId?: string; atelierUrl?: string; pdfUrl?: string }

interface ChatMessage { id: string; role: 'user' | 'natally'; text: string;
  createdAt: string; model?: string }                                  // 'Authored guide' | hosted model id
interface ReadingRecord { id: string; kind: 'opening' | 'comprehensive' | 'synastry';
  createdAt: string; chartInputs: NatalChartInput[]; transcript: ChatMessage[] }

interface SouvenirGeneratedElement { cover: { title: string; subtitle: string;
    yourRelation: string }; deepDive: { origins: string; traditionalInterpretation: string;
    lore: string; examples: string }; model: string; generatedAt: string }
interface SouvenirGenerated { groups: { id: 'planets'|'signs'|'aspects'|'houses';
    cover: { title: string; intro: string };
    elements: Record<string, SouvenirGeneratedElement> }[] }
interface SouvenirSnapshot { schemaVersion: 2; purchaseId: string; accountUid: string;
  createdAt: string; chart: NatalChartResult; messages: ChatMessage[];
  readings: ReadingRecord[]; voiceSettings: VoiceSettings; generated?: SouvenirGenerated }

interface AnalyticsEvent { name: string; detail?: Record<string, string | number>;
  uid?: string; at: string }
```

### 5.2 Cross-module service APIs

| API | Signature (frozen) | Owner |
| --- | --- | --- |
| ephemeris | `calculateNatalChart(input: NatalChartInput): Promise<NatalChartResult>`; `currentSky(now?: Date): Promise<TransitSnapshot>` | T2 |
| gazetteer | `lookupPlace(query: string): Place[]` (`{placeName, latitude, longitude, timezoneOffset}`); backed by `public/data/gazetteer.json` | T2 |
| synastry | `calculateSynastry(a: NatalChartInput, b: NatalChartInput): Promise<SynastryResult>` | T2 |
| speech | `speak(req: SpeechRequest): Promise<SpeechHandle \| null>` (null = not admitted); `stopOwner(ownerId)`; `speakWaiting(cue)`; `retainAudioFor(readingId, chunks)`; `playRetained(readingId)` | T3 |
| lessons | `getAuthoredChartExplanation(chart, component): string`; `buildLesson(chart, component): string`; `getAuthoredConversationClose(chart): string`; `getAuthoredUpsellClose(chart, variant: 'default'\|'souvenir'): string`; `nextNudge(seed): string`; `souvenirReply(question): string` | T4 |
| education session | `createAuthoredEducationSession(roomUid): AuthoredEducationSession` (`explain(component)`, `invitations`, rotation, dedup) | T4 |
| identity | `signInGoogleWeb()`, `signInPhone(phone, recaptcha)`, `signInEmail(email, pw)`, `signInNativeGoogle(): Promise<{idToken} \| {cancelled: true}>`, `linkProvider(kind)`, `signOutAccount()`; `onUid(cb)`; `accountStorageKey(name): string \| null` | T5 |
| onboarding | `saveIntake(input)`, `loadIntake(): NatalChartInput \| null`, `fetchRemoteIntake()`, `getEntryStage(): 'onboarding' \| 'ready'` | T5 |
| renderer | `<Astrolabe3D chart onToggleComponent onActivate selection insetMode ... />` (props interface frozen in T0) | T6 |
| commerce | `getCheckoutOffers(kind: 'conversation'): Promise<CreditPack[]>`; `purchasePack(pack): Promise<{purchaseId}>`; `redeemCode(code, requestId): Promise<BillingAccess>`; `readAccess(): Promise<BillingAccess>`; `readPaidConversationAccess(uid): {canInfer, canOpenSouvenir}`; `loadHistory(): Promise<PurchaseHistoryRow[]>`; `fetchCheckoutCustomer(): Promise<CheckoutCustomer>`; `formatCr(value): string` | T8 |
| inference | `completeHosted(prompt, context, key): Promise<stream>`; `withInferenceTurn(uid, fn)`: gates + accounts a paid turn; `composeOpeningReading(chart)`, `composeComprehensiveReading(chart)`, `composeSynastryReading(syn)` → prompt/context builders | T9 |
| entry | `resolveEntryExperience(pathname, search, native, authState): 'app' \| 'landing'` (bare domain: unresolved/anonymous → landing, authenticated → app — in-place selection) | T13 (T0 types) |
| events | `recordEvent(evt: AnalyticsEvent)`; `subscribeEvents(cb)` | T0 bus, T13 sinks |
| assets | `assetUrl(id: AssetId): string`; manifest typed from `asset-manifest.json` | T0 |
| native bridge | `isTauri()`, `isNativeAndroid()`, `isNativeDesktop()`, `invoke<T>(cmd, args)`; Rust command `native_sign_in(provider) → {idToken} \| {cancelled:true}` | T0 client, T13 Rust |

### 5.3 Billing HTTP API (Netlify function)

Every authenticated route verifies the Firebase ID token's
signature/issuer/audience/subject/expiry — a body uid is never identity.

| Route | Method | Purpose |
| --- | --- | --- |
| `/billing/health` | GET | liveness (`{"service":"natally-billing","status":"ok"}`) |
| `/billing/access` | GET | access envelope (display-projected balance Cr; ready managed access) |
| `/billing/reconcile` | POST | verify RevenueCat purchases → grant once → ensure key |
| `/billing/redeem` | POST `{code, requestId}` | configured-code redemption (§7K) |
| `/billing/history` | GET | every provider purchase + receipts + links (§7K) |
| `/billing/invoices/:id` | GET | official invoice record |
| `/billing/invoices/:id/file` | GET | invoice PDF |
| `/billing/logout` | POST | disable owned key generation, clear access |
| `/billing/souvenir-deliver` | POST `{purchaseId, snapshot, pdfBase64}` | hosted souvenir delivery (§7L) |
| `/billing/webhook` | POST | RevenueCat webhook (authenticated) |

CORS: exact allowlist `https://natally.robin.mba, https://natally.surge.sh,
http://tauri.localhost, https://tauri.localhost, tauri://localhost,
http://localhost:5173` (dev origin included — documented delta, §9). Client
base `VITE_BILLING_API_URL=https://natally-hosted-chat.netlify.app`.

### 5.4 Asset contract

`asset-manifest.json` (repo root, assets track) is authoritative for every
media file: byte-for-byte originals, sha256-verified at copy. The app resolves
assets only through `src/platform/assets.ts`. Required ids (see §7N): mascot
master, atelier scene trio + scene descriptor, 9 atelier audio files, 10
waiting-voice cues + provenance, Swiss Ephemeris trio, gazetteer dataset,
Playfair Display, full branding/marketing/paywall set, native icon set
(Tauri + Android res), licence notices. The Supertonic model is an **external
runtime download** pinned by revision (never committed). No substitution,
regeneration or redrawing of any original asset is permitted anywhere in the
rewrite.

## 6. State and persistence map

| Store | Key / path | Scope | Continuity |
| --- | --- | --- | --- |
| Chat history | `astrosphere_natally_chat_history_v3:<uid>` (localStorage) | per uid | **exact original key preserved** (same-origin continuity) |
| Natal intake (local) | `astrosphere_natal_intake_v1:<uid>` (localStorage) | per uid | **exact original key preserved** |
| Intake (remote, authoritative) | Firestore `users/{uid}/private/intake` | per uid | existing records read as-is |
| App inset position | `natally.inset-position.<uid>` (localStorage) | per uid | **exact original key preserved** |
| Landing inset position | `natally.marketing.inset-position.<roomUid>` (roomUid = `__marketing_guest__` when anonymous) | per room | **exact original key preserved** |
| Phone-link prompt | `natally_link_prompt_v1:<uid>` | per uid | 30-day re-prompt |
| Soul lore/disclosures, atelier memories, onboarding-restart, saved readings, retained-audio refs, inference access cache | `natally.<store>.v1:<uid>` (localStorage, new scheme) | per uid | new keys (no legacy data existed outside chat/intake/inset) |
| Credit accounts (working copy) | Firestore `natallyCreditAccounts/{uid}` (`creditUnits` mirror, encrypted key state, usage checkpoints, leases) | per uid | existing docs preserved |
| Grant/receipt ledger | Firestore `natallyCreditGrants/{digest([uid, kind, providerId])}` | per uid | existing receipts preserved |
| Souvenir snapshots | Firestore `users/{uid}/private/souvenirs/{purchaseId}` (create-only) | per uid | new (no legacy souvenirs were ever delivered) |
| Balance + purchases (authoritative) | RevenueCat customer `<uid>`, currency ROCHE raw ledger units (§2) | per uid | existing balances/purchases read exactly as stored — never reinterpreted |
| Hosted souvenir files | `<SOUVENIR_HOST_ROOT>/<uid>/<transaction>/{index.html, snapshot.json, souvenir.pdf}` | per purchase | immutable once delivered |
| Voice model + wasm | Cache Storage (runtime download, §7N) | per device | pinned revision |
| In-memory session (messages, name, intake, onboarding flag, presentation, lessons, speech) | React state behind `sessionGeneration` guard | per uid transition | **reset on every uid change before load** (§8.1) |

Firestore client-write rules (frozen): a user may read/write only their own
`users/{uid}/private/**`; `natallyCredit*` collections are server-only (admin
SDK); everything else denied.

## 7. Grouped entity tables — the finished system

### 7A Landing (`src/landing/**`, document `landing.html`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| resolveEntryExperience | fn | §5.2; native → app; app path/`?app=1`/`?offer=` → app; landing path/alias → landing; bare domain → in-place selection (landing while unresolved/anonymous, app when authenticated) |
| EXAMPLE_CHART_INPUT | const (frozen) | `name 'Aurelia (example)'`, `birthDate '1997-11-08'`, `birthTime '19:05'`, `placeName 'Toronto, ON, Canada'`, latitude `43.6532`, longitude `-79.3832`, timezoneOffset `-5`, houseSystem `'W'`, `birthTimeUnknown: false` |
| loadExampleChart | fn | computes the example chart once per visitor session through the local ephemeris (no network, no auth, no inference) |
| displayedChart | derivation | `ownedChart ?? exampleChart` gates every chart pane exactly as an owned chart (mount, drag handle, selector) |
| Example caption (**frozen copy**) | copy | signed-out or signed-in-no-intake: “Example chart — Aurelia, an anonymous example born 8 Nov 1997, 7:05 PM, Toronto. It is not your chart.” + create-your-chart link to `${VITE_APP_PATH}?app=1`; owner-ready keeps its owner caption |
| Sticky hero replica | UI | the landing hero is a working replica of the app's atelier main+inset pane pair — identical operation (room, mascot, sounds, inset drag/zoom/activation), labelled as the example above; whole hero stays sticky while the page scrolls beneath |
| Scroll-driven inset | behaviour | when the natal-chart section scrolls into view the inset chart zooms to main; scrolling out returns it to inset; both directions by scroll position |
| Header/nav (**frozen copy**) | UI | Natally logo + name, Sign In; nav row: Your chart · How she reads · Conversation · Prices · Education · Install Natally |
| Hero affordances | UI | Room sounds, Free cookies, “Here with you”, Hear Natally (gesture-gated voice), Room selector, Natally's greeting text + Replay |
| Landing sections | content | the eight sections of the original landing, beginning “Natal astrology, read aloud …”, scrolling alongside the sticky hero (never under it) |
| Guest education | behaviour | `AuthoredEducationSession` keyed by guest roomUid; prefab lessons over `displayedChart` with Supertonic speech after explicit gesture; authored upsell close after each lesson; **no inference call ever fires on the landing document** |
| Price bindings | build+runtime | `data-bind="reading-price \| credit-price \| credit-price-min \| credit-packs"` filled from env prices (US$30 reading; US$5/10/25 packs; min = US$5) |
| Version badge | UI | shows the build's stamped version |
| Transition guards | behaviour | account/intake changes clear example-owned lesson/selection/speech exactly as an owner chart; sign-in on landing stays on landing |

### 7B App shell, atelier, room and mascot (`src/app/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| AppShell | component | auth/intake/onboarding/chat routing; uid-keyed remounts of chat, intake form, readings; `sessionGeneration` resets (§8.1) |
| Atelier room | component | the original room scene (`atelier/room.webp`, poster fallback), two ambience loops (chart / workshop) via the Room selector |
| AtelierCharacter (mascot) | component | puppet rig over the scene: animated WebP idle master + clean-plate head/forearm; mouth-open events driven by the voice engine's playback; remains visible and animated as the inset pane while the chart is focused |
| Room interactions | behaviour | interaction sounds (bowling ball, brass clatter, cookie bite v1/v2, crystal break, glass shatter, shelf wobble); Free cookies interaction; ambience follows selected room |
| Greeting | behaviour | Natally's greeting text on entry with Replay (explicit gesture) |
| Pane pair | component | main + inset panes; inset drag, snap, keyboard move, shelf avoidance; persistence keys §6; two-path activation and focus choreography with the renderer (§7C) |
| Top line | UI | compact balance pill `«N Cr»` (display projection §2, ≤ 2 fractional digits, shared formatter), **Top Up** and **Souvenir** buttons — always visible to a signed-in user, never hidden by zero/unknown balance or prior purchase; one control font size/family across the row; deliberate wrap on narrow screens, no overflow/overlap |
| Profile dropdown (first) | UI | account identity (email/phone + uid in small monospace), account & billing entry, GitHub/project link, and **Log Out as the last item** (§7I.6) |
| Account modal | UI | shared customer-value copy first (**frozen copy**: heading “Explore what your chart means for you”; “Get a personal reading from Natally, then ask follow-up questions about the placements, patterns and relationships that matter to you.”; “Cr pays for your personal readings and conversations. Your balance is used as Natally creates a reading or reply; longer replies can use more Cr.”; “Choose an amount when you're ready, then return to your reading.”), then Balance heading + Cr amount, purchases, receipts |
| Education pipeline | behaviour | selection from the inset posts the authored lesson, sets the view to chart, and on settle reverts to atelier unless the user chose a view; then the paid personalized extension if `canInfer`, then the authored close (variant per §7G); rapid reselection cancels in-flight work |
| Chat surface | component | the conversation pane: streaming replies with voice, input with guards, “Add credits for your personal reading” entry that opens the conversation packs (never the souvenir paywall) |
| Readings surface | component | ReadingStage (reading + chart panes), saved-readings list with retained-audio replay, synastry entry |
| Onboarding (first-light) | flow | intake form (§7E) on a fresh uid; completes → opening reading |
| Purchase routing | component | `?offer=conversation` → Top Up packs; `?offer=reading` → souvenir purchase dialog; souvenir is offered only from souvenir entry points |

### 7C Chart renderer and callouts (`src/chart/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Astrolabe3D | component | the vivid plum-and-metal astrolabe: torus bezels, floor disc, guide rings (structure frame), planet beads + glyphs, sign baubles + cusp ticks, house spokes, badges + sector fills, aspect lines/tubes, current-sky bodies; slow `baseSpin`; drag + momentum; wheel/pinch zoom; selection halos; annotation modes; reduced-motion honoured |
| Renderer tiers | behaviour | WebGL → low-power WebGL → CPU SVGRenderer; `ChartErrorBoundary` wraps every mount; a missing GPU never blanks the page |
| Focus dimming | behaviour | on focus, non-focused representational elements dim (CHART_FOCUS constants); `structureFrame` meshes keep their base opacity (frame never dims) |
| Two-path inset activation | behaviour | inset state: object hit → activate (focus) + deferred component toggle applied at pane transition end (immediately under reduced motion); empty-space hit → activate only; foreground clicking unchanged; `pendingInsetSelection` cleared on unmount, drag start, chart rebuild, new pointerdown |
| Hover callouts | behaviour | shared hover explanation for planet/sign/house/aspect with the real chart object's name; the **entire inset pane (incl. border) is an exclusion rectangle** — the callout renders in a viewport overlay outside transformed/clipped ancestors, at ≥ 8 px viewport margin and ≥ 12 px clearance from the pane, choosing a fitting side then above/below, wrapping in the viewport band; drag/scroll/resize/zoom/pane transitions refresh or dismiss; **no fallback over the chart** (dismiss until a readable rectangle exists); no stale content from an earlier chart or unmounted owner; endpoint label cards stay off the inset; each renderer owns its overlay + cleanup; warm restrained style; leaving/drag/dialog dismisses |
| Callout placement | fn | `chartCalloutPlacement.ts` measures pane + callout in viewport coordinates (shared by app, landing, souvenir stages) |
| Focus visuals | const | `chartFocusVisuals.ts` CHART_FOCUS constants (frozen art direction) |
| Accessibility | behaviour | inert-pane accessibility labels; keyboard chart controls; reduced-motion instant transitions |

### 7D Astrology core (`src/astro/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Swiss Ephemeris runtime | vendor | `public/vendor/swisseph/{swisseph.js, swisseph.wasm, swisseph.data}` via the asset contract; all computation local and deterministic (AGPL notice served) |
| calculateNatalChart | fn | §5.2; returns 14 bodies (incl. Ascendant, Midheaven), 12 house cusps (title, sign, degree, cardo flag), aspect chords sorted by orb, angles (ASC/MC/DSC/IC + formatted), `houseKnown` (false when birth time unknown → houses suppressed downstream), `engineVersion`, `julianDayUT` |
| House systems | const | `HouseSystemId` set, default `'W'` (whole sign) |
| currentSky / transits | fn | current-sky bodies with speeds/retrogrades and aspects-to-natal for live context in readings and lessons |
| calculateSynastry | fn | two-chart comparison: cross-aspects sorted by orb; feeds the synastry reading + conversation |
| Gazetteer | data | GeoNames-keyed city/timezone lookup (`public/data/gazetteer.json`) resolving placeName → latitude/longitude/timezoneOffset for the intake form |
| detectedAspects (12 tightest) | derivation | aspects ranked by orb; the 12 tightest (all if fewer) drive the souvenir aspects group |

### 7E Onboarding and account-scoped state (`src/onboarding/**`, `src/identity/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Intake form | component | name, birth date, birth time **or** unknown flag, place (gazetteer lookup), house system (default W); validation client-side; saved atomically local+remote |
| Intake store | service | §5.2; local key + Firestore `users/{uid}/private/intake`; remote wins on conflict; a uid with no stored intake always enters onboarding |
| getEntryStage | fn | `'onboarding'` until intake exists for the current uid, then `'ready'` |
| Onboarding restart | flag | per-uid stored restart request returns the user to intake |
| First-light flow | flow | intake → chart computed → opening reading begins (paid or authored fallback per access) |
| accountStorageKey | fn | returns null without a uid; every user-content store goes through it (§6, §8.1) |
| Session generation | guard | `sessionGeneration` token bumped on every uid transition; async results from a previous generation are discarded (§8.1) |
| Link prompt | behaviour | phone-only account (`providerData` = [phone]) sees a dismissable, non-blocking prompt (**frozen copy**: “Add Google or email to keep your history if your number changes”) calling `linkProvider('google'\|'email')`; re-shown only after 30 days (`natally_link_prompt_v1:<uid>`) |
| Account panel identity | UI | email/phone + uid in small monospace under it |

### 7F Conversation, readings and synastry (`src/app/inference/**`, `src/app/{chat,readings}/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Chat history store | service | uid-scoped persistence (continuity key §6); load on mount, save on change; remounts with uid |
| Opening reading | flow | first paid reading after onboarding: comprehensive natal interpretation streamed + spoken |
| Comprehensive reading | flow | on-demand full-chart reading |
| Follow-up conversation | flow | streamed replies with context = target name(s), natal chart facts, current-sky facts; message metadata distinguishes `'Authored guide'` from hosted-model turns |
| withInferenceTurn | fn | per-turn: read access → ensure funded key → run → account actual usage; refuses (authored fallback) when `canInfer` is false; stable turn ids; in-flight cancellation guards |
| completeHosted / streaming | fn | hosted API call on the customer's managed key; model chain `openrouter/free`; streamed text feeds speech chunk-wise |
| Synastry reading | flow | second-person intake → `calculateSynastry` → synastry reading + conversation; saved like other readings |
| Saved readings | service | per-uid reading records + retained audio refs; readings modal lists and replays them (documents remain accessible after balance is spent) |
| Retained audio | service | retained reading playback through the voice engine; replay is explicit-user and supersedes |
| Authored fallback | behaviour | when `canInfer` is false, Natally answers with authored content (lesson cores / close) — never an LLM call without access |
| Cost display | behaviour | displayed Cr balance only (§2 display projection); no token/dollar conflation in customer surfaces |

### 7G Authored content (`src/content/**` — original words, new code)

All passages are the **original authored text carried verbatim** (content
port, no code); register and tone preserved. No placeholder text anywhere.

| Entity | Kind | Role / contract |
| --- | --- | --- |
| LessonCore | type | `{ what, regimes, evolution }` — factual core, how traditions/regimes regard the element, how its meaning evolved |
| BODIES | table | authored cores for all 14 bodies (Sun…Midheaven) |
| SIGN_CORES / HOUSE_CORES / HOUSE_TOPICS | tables | 12 + 12 authored cores + house topics |
| ASPECT_CORES | table | 5 (Conjunction, Opposition, Square, Trine, Sextile) |
| GENERAL_LESSON / CONTEXT_LINE | table/const | general lesson framing + context line |
| buildLesson / getAuthoredChartExplanation | fn | compose/lookup lessons (what → regimes → evolution) |
| AuthoredEducationSession | service | per-roomUid session: framing, INVITATIONS, lesson rotation without verbatim back-to-back repetition, dedup of told lessons |
| getAuthoredConversationClose | fn | authored conversation close |
| getAuthoredUpsellClose | fn | two variants ('default': personal reading with Cr; 'souvenir': PDF + persisted interactive “vignette frozen in time” atelier), light in-file rotation; perfect-sales assertiveness with self-aware cheesiness, never self-consciousness |
| authoredNudges | table | idle-story library for admitted idle nudges (§7H) |
| authoredSouvenirReplies | fn | frozen-question replies in the souvenir viewer suggesting Top Up for a live reading; no LLM |
| Customer-value copy | const | the frozen ACCOUNT-VALUE copy of §7B, shared by both entry points from one component |

### 7H Voice and character (`src/voice/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Supertonic engine | service | ONNX synthesis via `@huggingface/transformers@4.3.1` + onnxruntime-web in a worker; model = external pinned download (§7N); voice **F4, speed 0.9** — the original voice, no substitution |
| Model download | behaviour | runtime download (~234 MB) into Cache Storage, once per device; progress surfaced honestly |
| speak / speakNatallyText | fn | §5.2; synthesis + playback; emits mouth events for the mascot |
| Streaming speech | behaviour | chunked synthesis of streamed text with gapless scheduling; retains engagement while waiting for more text |
| Waiting voice | behaviour | pre-rendered cues (**original assets**): “Let me take a closer look.” / “Give me a moment.” / “Let me follow that thread.” (+v2/v3 variants) and the closer cue, from `public/voice/waiting/` |
| Admission bus | service | single shared busy owner: a busy engine asked to admit idle speech returns **false before stopping anything** (NO idle queue); stale completion/abort cannot release or stop a newer owner; explicit user replay/selection may supersede, stopping old playback first |
| Idle nudges | behaviour | timers check shared busy + draft/input/inference/visibility/account guards before story selection, after async resolution, and at final admission; intervening activity discards the attempt and restarts quiet timing; only admitted stories append (dedup §7G) |
| Retained reading audio | service | record + replay per reading (§7F) |
| Gesture gate | invariant | no audio autostarts without an explicit user gesture |

### 7I Identity and authentication (`src/identity/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Firebase init | service | existing project; web SDK; auth + named Firestore; authorized domains §2 |
| Web sign-in | flows | Google (`signInWithPopup`), phone (`signInWithPhoneNumber` + reCAPTCHA verifier — also the Android WebView path), email/password |
| Native desktop Google | flow | `signInNativeGoogle()` → Tauri Rust `native_sign_in` (system browser/loopback PKCE, no client secret) → `{idToken}` → `signInWithCredential` |
| Native Android Google | flow | `signInNativeGoogle()` → Tauri bridge to the **supported Android native Google credential API** (Google Play-services Credential Manager / the platform's supported native Google Sign-In surface, using the Android OAuth client of the production upload key): the native provider flow returns a Google ID token (complete) or a cancellation result (cancel). No custom redirect-URI/custom-scheme OAuth is claimed or built. Complete: bridge returns `{idToken}` → `signInWithCredential`, uid + email visible. Cancel: bridge returns `{cancelled:true}` → auth modal restored, no hang, no partial state. The Android client id must be allowlisted in the Firebase Google provider |
| No silent linking | invariant | a sign-in submitted while `auth.currentUser` is non-null signs out first and awaits the auth listener; the signed-in view never renders sign-in forms; new phone sign-ins never silently link to an existing account |
| signOutAccount | fn | Log Out (dropdown bottom): revoke managed key (`POST /billing/logout`) → Firebase sign-out → clear local access/state; the **only** sign-out control |
| uid transitions | invariant | §8.1 (reset + remount + uid-scoped stores) |
| Firestore rules | file | frozen §6 client-write rules |

### 7J Commerce client surfaces (`src/commerce/**` + `src/app/account/**`)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| RevenueCat checkout | service | Web Billing SDK with the project's public web key; one-tap pack purchase returns a real purchase id; ownership/cancel/pending states authoritative |
| Packs catalog | service | `getCheckoutOffers('conversation')` → the three configured packs `natally_credit_usd_{500,1000,2500}` with the offering's real `formattedPrice`; pack label Cr = pack amount × display factor (§2: US$5 → 500 Cr, US$10 → 1000 Cr, US$25 → 2500 Cr); **Top Up always renders the packs** — never an empty panel; offer-load failure → “Prices are unavailable right now. Retry.” with Retry; zero packages (misconfigured offering) → “Credit packs are not available yet.” + redemption still present |
| Top Up dialog | component | value copy (§7B) → packs (label `${formattedPrice} — ${labelCr} Cr`; `labelCr` = amountUsd × display factor, e.g. US$5 → 500 Cr) → redemption under a `<details><summary>Have a code?</summary>` disclosure, submit button “Redeem”; no landing-explanation gate before purchase; funded allowance is never invented client-side — it is read from the access envelope after reconcile |
| Redeem | service | `/billing/redeem {code, requestId}`; success toast **“Code redeemed. N credits added.”**; balance pill updates |
| Purchase completion | flow | confirmed purchase → `/billing/reconcile` → state `ready`, balance +Cr (display projection of the funded allowance), history row + invoice access, confirmation copy; a checkout-start failure says the purchase could not start (never claims success or “not needed”) |
| History (browser-first) | service | read the SDK customer first: list `CustomerInfo.nonSubscriptionTransactions` always; then merge server receipts/amounts/redemptions/refunds; “No purchases” only when the provider confirmed zero; a failed server read shows the actual error + saved receipts; restore/reconcile run after and never hide the list |
| Invoices | service | invoice rows link `invoiceId`; PDF via `/billing/invoices/:id/file`; facts per §7K |
| Access cache | service | `/billing/access` envelope cached short; guards stale account/generation results |
| Customer language | invariant | Balance/Cr/purchases/receipts only; **no** provider names, keys, provisioning or access-control internals in customer wording |
| Cr formatter | fn | shared `formatCr`, ≤ 2 fractional digits, suffix `Cr`; applies the display factor to a USD quantity only (§2); invalid config never invents a balance |

### 7K Billing server (`billing/**` on Netlify `natally-hosted-chat`)

Deployed by `scripts/deploy-billing-netlify.mjs` (forces
`BILLING_CREDIT_BALANCE_STORE=revenuecat`, imports every env key through the
kstore resolver, health-checks). `netlify.toml` sets the function timeout to
26 s.

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Runtime | module | portable runtime behind `netlify/functions/billing.mjs`; routes §5.3; CORS §5.3; private customer responses |
| Identity | invariant | Firebase ID-token verification on every authenticated route; never a body uid |
| RevenueCat wallet (authoritative) | module | ROCHE raw integer ledger operations via RevenueCat's idempotent integer transaction interface; `BILLING_CREDIT_UNITS_PER_USD = 1,000,000` units per USD (§2); usage debits once per stable identity, fractional units carry forward; provider allowance = authoritative balance − unposted usage; existing stored balances are never migrated or reinterpreted |
| Purchase grants | module | for each owned production purchase not yet granted: record the grant with stable identity `purchase:<purchase.id>` and receipt `natallyCreditGrants/digest([uid,'payment','revenuecat:'+id])`; the funded spendable allowance = **verified pre-tax paid USD × the configured allocation** (`BILLING_TOPUP_ALLOCATION_BPS/10,000`; at 4000 bps a US$5 pack funds US$2.00 → displayed 200 Cr, while its pack label stays 500 Cr per §2), posted to the authoritative ledger in raw units; verified paid amount only; dedup by store payment identity — webhooks, restore and reconcile never double-grant |
| Redemption | module | the configured permanent code is resolved from kstore (`BILLING_REDEMPTION_CODE` — no literal value exists in repo, docs or code) with the configured allowance (`BILLING_REDEMPTION_ALLOWANCE_USD`): exact compare after outer trim; a valid request applies its **configured grant semantics** exactly once per `requestId` (before/after recorded so a retry never applies twice); it never resets, overwrites or reduces an existing larger balance; invalid code mutates nothing; durable receipt returned |
| Managed keys | module | OpenRouter per-user keys: created limit 0, `expires_at = now + 60 s`; `ensure` PATCHes an **absolute** limit = accounted usage + available allowance (idempotent; nothing increments on purchase); rotated within expiry/on logout/when disabled; AES-256-GCM at rest (`BILLING_KEY_ENCRYPTION_KEY`, AAD = uid/hash/version); unfunded orphans never become funded duplicates (stable operation names, provider key pagination reconcile, known-hash PATCH read-back); ready requires verified identity, expiry, enabled state, capacity |
| Accounts / ledger | module | `natallyCreditAccounts` (creditUnits working-copy mirror, encrypted key state, usage checkpoints, operation intents/leases via CAS); `natallyCreditGrants` receipts immutable; Firestore is a working copy — RevenueCat is authority |
| Refunds | module | deterministic reversals appended (retain original receipts), lower usable capacity, appear in history |
| History | module | §5.3 `/billing/history`: every provider purchase (any product/status/refunds) + every redemption/grant receipt + souvenir links (`atelierUrl`, `pdfUrl`); provider failure returns the actual `providerMessage`, never an empty “no purchases” |
| Invoices | module | official invoice per paid purchase from verified RevenueCat facts + Natally's manual GST/HST policy (Ontario: 13% HST; base/tax/total shown; e.g. US$10 base → US$1.30 HST → US$11.30 total; allowance funds from the pre-tax amount × configured allocation): seller block §2, unique invoice number/date, purchaser, description, currency/subtotal/tax name+rate+amount/total, payment date/terms/status; original PDF + printable invoice; HST payable recorded separately by currency; no invented or estimated tax |
| Logout | module | disables only owned matching key generation; local access cleared client-side |
| Webhook | module | RevenueCat webhook, authenticated; same grant path/idempotency |
| Souvenir delivery | module | §7L |
| Price authority | invariant | prices come from env + RevenueCat, never code; `verify-offer-prices.mjs` (§7M) gates the web build |

### 7L Souvenir system (`src/souvenir/**` + `souvenir.html` + server §7K)

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Souvenir purchase | flow | souvenir entry points only (US$30, package `natally_reading_test` → `natally_souvenir_reading_usd_3000`; legacy `natally_reading_test_v3` entitlement accepted); confirmed purchase id drives everything |
| createSouvenir | fn | frozen snapshot (schemaVersion 2, §5.1) stored create-only at `users/{uid}/private/souvenirs/{purchaseId}` |
| generateSouvenirContent | fn | runs once per purchase before PDF/atelier: sequential page-pair requests (group cover; element cover + deep dive) through the customer's managed key on `openrouter/free`; resumable (skips elements already in `snapshot.generated`); honest progress (“Writing Houses: 7 of 12”); prompts state only the input variables (chart/person/chart-facts/conversation/identity — per original frozen variable tables); customer cover details come from chart values, never invented; deep dives are traditional background; an element failing after retries is reported and the souvenir is **not delivered incomplete** |
| buildSouvenirPdf | fn | `(snapshot) → Promise<Blob>`; layout (fixed): Cover → Your calculated chart → Your calculated positions → four element groups — “Planets”, “Signs”, “Significant aspects” (12 tightest by orb), “Houses” — each = 1 group cover + 2 pages per element (cover: graphic theme + the customer's own details; deep dive: origins, traditional interpretation, lore, examples) → Your conversation with Natally; every detail kept, with space and uncluttered layout; houses without a birth time state that houses need one (education pages remain) |
| Hosted delivery | flow | `POST /billing/souvenir-deliver {purchaseId, snapshot, pdfBase64}`: verifies Firebase identity + that the purchase is the customer's owned souvenir purchase in RevenueCat; uploads the built atelier template + `snapshot.json` + `souvenir.pdf` by SFTP/FTP to `<SOUVENIR_HOST_ROOT>/<uid>/<transaction>/`; emails both links (`SOUVENIR_SMTP_*`, recipient = Firebase account email); records receipt fields `atelierUrl`, `pdfUrl`, `deliveredAt`; idempotent per purchase id; never overwrites an existing delivery; links = `SOUVENIR_PUBLIC_BASE_URL/<uid>/<transaction>/` |
| Client completion | flow | client also starts the PDF download; purchase-history row shows both links |
| Souvenir atelier (viewer) | component | `souvenir.html` static build: immutable message source (saved conversation, readings, chart data); replay, chart interaction and spoken reread of what is there only; questions beyond the conversation get authored `souvenirReply`s suggesting Top Up — never an LLM call; no sign-in, no lookups, no network beyond its own files |

### 7M Platform: env, build, release, deploy, analytics, version

| Entity | Kind | Role / contract |
| --- | --- | --- |
| Env schema | file | `.env.example` fully annotated, schema + defaults, no kstore refs, **no secret values**; effective `.env` adds only real values/`kstore://` pointers. Public keys: §3 entry set, `VITE_BILLING_API_URL=https://natally-hosted-chat.netlify.app`, `VITE_CREDIT_DISPLAY_PER_USD=100`, `VITE_REVENUECAT_UNITS_PER_USD=100`, `VITE_REVENUECAT_CONVERSATION_PACKS_JSON` (500/1000/2500 → product ids), `VITE_REVENUECAT_READING_PRODUCT_ID=natally_souvenir_reading_usd_3000` (+ legacy accepted id), `VITE_REVENUECAT_WEB_API_KEY` (public Web Billing key), `VITE_APP_VERSION` (injected). Server keys: `BILLING_CREDIT_BALANCE_STORE=revenuecat`, `BILLING_CREDIT_UNITS_PER_USD=1000000`, `BILLING_TOPUP_ALLOCATION_BPS=4000`, `BILLING_CREDIT_PACKS_USD=5,10,25`, `BILLING_OFFER_PRICE_USD=30.00`, `BILLING_REDEMPTION_CODE` (value only via kstore pointer — never a literal), `BILLING_REDEMPTION_ALLOWANCE_USD=10.00`, `BILLING_ALLOWED_ORIGIN` (§5.3), `REVENUECAT_SECRET_API_KEY`, `BILLING_KEY_ENCRYPTION_KEY`, `OPENROUTER_ADMIN_KEY`, Firebase service auth, `SOUVENIR_HOST_*`, `SOUVENIR_SMTP_*`, `SOUVENIR_PUBLIC_BASE_URL` — all secret-valued keys resolved from kstore at deploy/run time |
| kstore resolver | script | `scripts/environment.mjs resolve` — fresh, process-only resolution; failures terminate before stamping/building; resolved secrets never logged |
| Build entry points | scripts | exactly two independent coordinators: `build_all.sh` (Linux-only) and `build_all_win.ps1` (native Windows PowerShell); neither calls the other or PowerShell/Git-Bash cross-host; host mismatch fails clearly |
| Build matrix | targets | Windows standalone EXE + NSIS (+ MSI on Windows host); Android APK + AAB + matching native-debug-symbols ZIP; Linux AppImage + DEB + RPM; web ZIP + installer + PWA; every invocation auto-stamps a unique version; full matrix attempted and outcomes recorded honestly; MSI from Linux remains honestly pending (WiX needs Windows); Windows EXE/NSIS cross-compile from Linux via cargo-xwin |
| Android signing | flow | production upload keystore via kstore; newest complete `$ANDROID_HOME/build-tools` set providing apksigner/zipalign/aapt selected dynamically; verify APK + configured certificate SHA-256 (§2); 16 KiB ZIP alignment; signed AAB + symbols; never a debug fallback |
| Release framing | invariant | every canonical build is a **production release candidate**; inventories `dist/<prefix>-build-inventory.json/.xml`; evidence strings state production facts (Android: “Release profile; production upload certificate verified”; Windows cross-build: “cross-compiled from Ubuntu via cargo-xwin; Windows code-signing occurs only on the Windows host”); no “test build”/“not-a-release” artefacts or wording anywhere |
| Version discipline | flow | `version.json`/`version.txt` from the release-state stamp (MINOR auto-increment per invocation); the app badge shows the build's stamped version; **after** a successful web deploy the coordinator saves run state, stamps the next version and commits “v\<next\>: next after deployed v\<this\>” — only stamp commits may carry version numbers |
| Price gate | script | `verify-offer-prices.mjs` runs before the web build and refuses to publish prices RevenueCat does not charge (transport: RevenueCat v2 secret key from kstore); overrides never used by agents |
| Static deploy | script | `deploy-existing.mjs` → Surge: per-domain CNAME staging, completed-artifact hash + served-byte verification, domain allowlist; staging first, promote to `natally.robin.mba` explicitly; no implicit production publish |
| Billing deploy | script | §7K header |
| Analytics | module | structured event bus (T0) with sinks (T13): chart activate/inset-toggle/component-select, lesson admitted/settled, purchase started/succeeded, redeemed, sign-in/out; staging evidence overlay + dev feed; **no third-party analytics property, no campaign property**; no PII beyond uid |
| PWA | module | §3 shell; offline shell; installable |
| Tauri shells | project | `src-tauri/**`: Windows/Android/Linux config, icons from `public/native/**`, opener plugin for the desktop system-browser sign-in flow, Android native Google credential API bridge (§7I), Rust `native_sign_in` |

### 7N Media and asset identity (asset-manifest.json — byte-for-byte originals)

| Asset id | File(s) | Role |
| --- | --- | --- |
| mascot master | `public/natally-mascot.webp` | animated WebP idle animation; puppet source (metallic silver face, amber eyes, purple celestial turban) |
| atelier scene | `public/atelier/{room,poster,puppet-cleanplate}.webp` + `public/atelier/manifest.json` | room scene (1586×992 in a 2048×1280 coordinate space), static poster fallback, head/forearm clean plate for the puppet rig; scene descriptor with per-asset sha256 |
| atelier audio | `public/atelier/audio/*.mp3` (9) | interaction sounds (bowlingball, brass-clatter, cookie-bite-crunch v1/v2, crystal-break, glass-shatter, shelf-wobble) + ambience loops (chart-ambience, workshop-ambience) |
| waiting voice | `public/voice/waiting/*.wav` (10) + `provenance.json` | pre-rendered Supertonic F4 cues (look/moment/thread + v2/v3, closer) |
| voice model (external) | `onnx-community/Supertonic-TTS-ONNX@cff123c84b0655d9d647641f1b532c3cbb8f7faa`, voice **F4** speed **0.9**, runtime `@huggingface/transformers@4.3.1` | runtime download (~234 MB: config/tokenizers + latent_denoiser/text_encoder/voice_decoder onnx + data), browser-cached; never committed |
| ephemeris | `public/vendor/swisseph/{swisseph.js, swisseph.wasm, swisseph.data}` | Swiss Ephemeris engine + data (AGPL) |
| gazetteer | `public/data/gazetteer.json` | GeoNames-keyed city/timezone dataset |
| font | `public/fonts/playfair-display-latin.woff2` (+ OFL notice) | display font |
| branding | `public/branding/**` | PWA icons 180/192/512, favicon set, icon master + guide layer (pointing pose) + provenance, Google Play hero |
| marketing/paywall art | `public/marketing-assets/natally-still.webp`, `natally-paywall-original.webp`, `public/paywall/natally-original.webp` | mascot still + paywall portraits (app + marketing copies) |
| native icons | `public/native/icons/**`, `public/native/android-res/**` | full Tauri desktop/iOS icon set; Android launcher mipmaps + `natally_guide` opening-animation drawable — source of truth for `src-tauri` |
| notices | `public/notices/**` | AGPL-3.0 (project ©2026 Robin L. M. Cheung, MBA), Swiss Ephemeris + wrapper licences, third-party notices |

Rule (absolute): **actual graphics and original sound only — no substitutions,
regenerations, redrawn variants or placeholder media anywhere.** Voice = F4 at
speed 0.9 exactly.

## 8. Cross-cutting invariants (binding on every module)

1. **Privacy / uid isolation.** Every persisted user-content store is keyed by
   Firebase uid through `accountStorageKey` (null without uid). On every
   `onAuthStateChanged` uid transition the in-memory session (messages, name,
   intake, onboarding flag, presentation, lessons, speech) is reset **before**
   loading the new uid's data; uid-keyed components remount; stale-generation
   async results are discarded. A fresh uid with no intake always enters
   onboarding; the header shows the new account's name only. Switching back
   restores that uid's data. No silent provider linking (§7I).
2. **Paid inference discipline.** Inference runs only in the app, only through
   the paid-access check and turn accounting, on managed keys. The landing and
   the souvenir viewer never make LLM calls.
3. **No placeholders.** No mock customer experiences, dummy payments, invented
   balances, fabricated tax, filler text or fake history. Unknown states are
   shown truthfully (real errors, real zeroes only when confirmed).
4. **Money invariants.** Three quantities, never conflated (§2 "Money and
   units"): (a) raw RevenueCat ROCHE ledger units — 1,000,000 per USD,
   authoritative, internal, never displayed, never migrated or reinterpreted;
   (b) spendable USD allowance — purchases fund pre-tax paid USD × the
   configured allocation BPS; (c) displayed Cr — USD × display factor (100),
   shared formatter, ≤ 2 fractional digits. Pack labels use amount × display
   factor; funded allowance is server-configured and read back, never invented
   client-side. RevenueCat is the only balance authority; Firebase is a working
   copy. Grants, reversals and debits carry stable identities and are
   idempotent; refunds append reversals; usage debits once; redemption follows
   its configured grant semantics and never resets or reduces an existing
   larger balance. Custom amounts are not sold (§9).
5. **Key custody.** Managed keys: limit 0 at creation, 60 s life, absolute
   limit = accounted usage + available, AES-256-GCM with uid/hash/version AAD,
   disabled on logout, never orphan-funded.
6. **Voice admission.** One busy owner; idle admission refuses without
   stopping anything; stale owners cannot stop newer ones; explicit replay
   supersedes; no autostart without gesture.
7. **Truthful states.** Every customer surface names its real state (LP08);
   failures show the actual error; “no data” only on confirmed absence.
8. **Credentials.** Secrets (including the redemption code value) live in
   KStore; the repo and `.env` carry only `kstore://` pointers; resolved
   values are never logged or committed.
9. **Release honesty.** Production-candidate framing; honest matrix outcomes;
   no version number in non-stamp commits; deployment explicit.
10. **Identity preservation.** Original media, original voice, original copy,
    original visual identity. Any deviation is a defect.

## 9. Deliberate deltas and explicitly absent scope

Deltas from the original historical implementation (each is an operator
ruling or a consequence of this rewrite's constraints — nothing else may
deviate):

- **Billing host**: Netlify function on `natally-hosted-chat` only; the
  retired Cloud Run host is gone. Static hosting: Surge only.
- **Card rail**: RevenueCat Web Billing exclusively. The early-round direct
  Stripe Checkout path, its `/billing/checkout` endpoint and its Stripe
  webhook are superseded (operator rulings 2026-11-02/03) and do not exist
  here. Invoices are issued by Natally from verified RevenueCat facts + the
  manual GST/HST policy.
- **Credits are sold only as the configured packs** (US$5/10/25). The
  early-round custom-amount entry is superseded and absent. No USD1 product.
- `http://localhost:5173` added to the billing CORS allowlist for local
  development (documented convenience; no production effect).
- Release framing is production-candidate (operator rejected test-build
  framing); localStorage continuity keys preserved (§6); new `natally.*`
  scheme for stores that had no legacy key.
- **No unit/smoke test suites.** Delivery evidence = recorded journeys (§10).

Explicitly absent (future rails/decisions recorded, not built): pay-per-use
rails (Alby/x402/Blink), BTCPay/Blink POS top-ups, a WooCommerce billing rail,
a second inference provider (Venice), automated provider-account funding
(operator tops up OpenRouter manually), a general redemption-code table (only
the configured permanent code exists), campaign/marketing analytics, Stripe
Tax, Netlify billing alternatives.

## 10. Verification — recorded running journeys

Delivery of any row is established only by a **recorded running journey** of
the real application against the real backend, archived under
`DOCS/DEMONSTRATIONS/<journey>-<date>/` (screencast + redacted JSON/logs).
The orchestrator runs and records the journeys (CHECKLIST T14); the operator
participates only where a step is genuinely personal or consequential (a real
card purchase, a refund). The journey matrix (CHECKLIST T14) includes:
purchase E2E, redemption + idempotent retry + no-reduction, privacy isolation
across two accounts, Android Google sign-in complete + cancel, Log Out with
key revocation, souvenir E2E (generation → PDF → hosted links → email →
viewer), voice busy/waiting behaviour, landing replica + zero-inference proof,
education branching (prefab → paid extension → both close variants),
invoice/refund/restore, PWA offline + install, and the full release build +
staging + promotion. Static checks may supplement, never substitute.

## Attestation

This snapshot was written from the original `DOCS/ARCHITECTURE.md` (read-only,
as product requirements), the operator's rewrite instructions, and the
completed asset manifest. It specifies the complete intended finished system;
it certifies no implementation. CHECKLIST.md translates it 1:1 into disjoint
owned tasks; a task is done only when its recorded journey passes.
