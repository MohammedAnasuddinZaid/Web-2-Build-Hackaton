# CareSpeak — Complete Technical Deep Dive
## From Physics to Business Model: How Every Single Thing Works

> **Purpose:** This document is a self-contained, copy-pasteable reference. If you know nothing about CareSpeak, start at Part 0. If you know the basics, jump to Part 2 (HLD) or Part 4 (Math). Every formula, flow, and threat is written so it can be lifted verbatim into a report, slide, or exam answer.

> **Currency:** Updated 2026-11-11. Verified against the committed tree (commit `136c7f9`, pushed to `voicelink/main`). Test counts: **754 TS tests / 44 files** (`npx vitest run`) and **534 Python tests** (`pytest -q`); `tsc --noEmit` and `next build` both clean. All values below are read from source, not invented.

---

# PART 0 — Glossary & Units

| Term | Meaning |
|---|---|
| CareSpeak Core | On-device assistive communication: webcam gestures/eyes → speech + nurse dashboard + IoT vitals + escalation. |
| Web Voice Call — Demo | Browser-to-browser AI healthcare voice call layer added for the hackathon. |
| VoiceLink | Local Python (FastAPI) service: ASR + TTS + LLM over WebSocket, plus telephony. |
| Session | One care interaction between a nurse and (optionally) a patient, keyed `wc_<hex>` (`newId("wc")`). |
| Pair token | Cryptographic per-bed token baked into a QR code; proves a scanner is the bed's device. |
| Participant token | Short-lived token minted at join time; proves "patient" role. |
| Nurse cookie | Signed `care_session` cookie, 8 h TTL, backed by persistent store. |
| Passage | A small labelled text chunk extracted from the chart for retrieval (MiniSearch). |
| Scope | Question classification: `record` \| `general` \| `mixed` \| `chitchat` \| `meta` \| `emergency`. |
| Finding | A live reading that crossed a threshold (e.g. low alertness). |
| TwiML | Twilio's XML instruction language for phone calls. |
| KVDriver | Storage abstraction: `redis` \| `file` \| `memory`. |
| PortalAnswer | The server's structured reply `{ answer, scope, grounded, citations, engine, ... }`. |

**Units used throughout:**
- Time: milliseconds (ms) unless stated. 1 s = 1000 ms.
- Audio: samples/second (Hz). Target PCM rate = 16000 Hz (16 kHz); telephony = 8000 Hz; TTS = 22050 Hz.
- Storage: bytes; transcript cap 4000 chars/entry; record cap 40 000 chars; event cap 2000/session.
- Concurrency: requests/second (RPS), sessions.

---

# PART 1 — Product, Personas, Market, Business Model

## 1.1 The problem (quantified)
- ICU/ward patients who cannot speak or move have no self-service channel to express need.
- Nurse-to-patient ratio in Indian public hospitals ≈ 1:6 to 1:12 in wards; ICU ≈ 1:2 (NHM norms; varies by state).
- Missed escalation cost: a single delayed response to respiratory distress can convert a reversible event into an ICU transfer (cost ~₹1–3 lakh incremental + morbidity).
- Documentation burden: nurses spend 25–40% of shift on charting (industry literature; planning assumption).

## 1.2 Personas
| Persona | Need | CareSpeak answer |
|---|---|---|
| Patient (non-verbal/motor-impaired) | Express need, be heard, feel safe | Gesture/eye → speech, SOS, nurse reply banner |
| Bedside nurse | Know patient state without visiting | Dashboard, risk ring, alerts, SSE push |
| Remote/on-call nurse | Answer questions, dictate messages | Web Voice Call, portal assistant, phone call |
| Ward in-charge | Multi-patient overview, audit | Ward view, reports, CSV/JSON export |
| Hospital admin | Compliance, cost, outcomes | Audit trail, shift reports, analytics |
| Family | Reassurance, updates | Family panel, messages |

## 1.3 Value proposition (one line)
> "Give every patient a voice — on-device, offline-capable, audit-ready, and honest about what it knows."

## 1.4 Business model (detailed)
### 1.4.1 Revenue streams
1. **SaaS subscription** (monthly/annual): Starter (1 ward, 5 beds, 500 AI turns, 7-day retention) ₹25,000/mo; Professional (5 wards, 50 beds, 10,000 turns, 90 days, citations + SBAR) ₹1,20,000/mo; Enterprise (unlimited wards, SSO, data residency, SLA 99.9%, FHIR) ₹6,00,000+/mo.
2. **Usage-based AI overage**: beyond included turns, ₹0.50–₹2.00 per AI turn (cost to serve ≈ ₹0.038/turn → margin 92–98%).
3. **Hardware kit**: ESP32 wearable (HR/SpO₂ + SOS) + pairing stand + QR cards. BOM ≈ ₹1,100; sell ₹2,500–₹3,500; margin ≈ 55%.
4. **White-label / API licensing**: telehealth platforms embed voice+assistant; revenue share or per-call fee.
5. **Professional services**: on-prem private deployment (VoiceLink + local LLM), EHR/HL7/FHIR integration, multilingual packs, training. Day-rate ₹25,000–₹60,000 per consultant-day.
6. **Compliance add-on**: extended retention (7 years), regulatory export (DPDP/HIPAA/GDPR), advanced audit analytics. +20% on subscription.

### 1.4.2 Unit economics (illustrative)
- CAC = ₹1,50,000; ARPU (ward) = ₹60,000/mo; gross margin 78%; churn 8%/yr → monthly churn 0.00667; L = 1/m ≈ 150 months (capped at 60 for planning).
- LTV (60 mo) = 60,000 × 0.78 × 60 = ₹2,80,80,000; LTV/CAC ≈ 187×. Conservative (12 mo): 60,000 × 0.78 × 12 = ₹5,61,600 → 3.7× (healthy, >3×).
- Break-even = 1,50,000 / (60,000 × 0.78) = 1,50,000 / 46,800 = **3.2 months**.

### 1.4.3 Cost to serve (per 1,000 AI turns)
- Gemini turns (≈20%): 200 × ₹0.15 = ₹30.
- Retrieval + store + compute (Vercel): ≈ ₹8.
- Total ≈ ₹38 / 1,000 turns → **₹0.038/turn**. Sold at ₹0.50–₹2.00 → margin 92–98%.

### 1.4.4 Market sizing (planning)
- India hospital beds ~1.9 million. Initial target: ICU + step-down + long-stay wards in tier-1/2 cities ≈ 1,20,000 beds.
- SAM: 1,20,000 beds ÷ 10 beds/ward = 12,000 wards × ₹60,000/mo = ₹72 crore/mo = ₹864 crore/yr.
- SOM (3-year, 2% penetration): 240 wards → ₹14.4 crore/yr ARR; +30% hardware/services → ~₹19 crore ARR by year 3.
- Global (aging, disability care): 10× India over 7–10 years (planning horizon only).

### 1.4.5 Go-to-market
Hackathon → credibility + pilot LOIs → 2–3 ward free 90-day pilots (measure nurse time saved + escalation speed) → case study → regional chains (Apollo, Fortis, Max, Manipal) → national telehealth (Practo, 1mg, Apollo 24/7) → channel via medical-device distributors (hardware) + system integrators (on-prem).

---

# PART 2 — System Design (HLD)

## 2.1 Context diagram (text)
```
┌─────────────────────────────────────────────────────────────────────┐
│                         CARESPEAK PLATFORM                          │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐     │
│  │ Patient      │   │ Nurse        │   │ Admin / Ward         │     │
│  │ Browser      │   │ Browser      │   │ Console              │     │
│  │ (gesture/eye │   │ (dashboard,  │   │ (multi-patient,      │     │
│  │  AI, WebRTC) │   │  voice call, │   │  reports, audit)     │     │
│  │              │   │  portal)     │   │                      │     │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘     │
│         │                  │                      │                 │
│         │  WebRTC P2P (media) + HTTP signalling     │                 │
│         └────────────┬─────┴──────────────────────┘                 │
│                      ▼                                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Next.js 15 App (Vercel) — API routes + SSR pages             │   │
│  │  • /api/care/*  (sessions, signal, portal, voice, transcript)│   │
│  │  • /api/health, /api/ingest, /api/sync, /api/stream (SSE)    │   │
│  └──────┬───────────────────────┬───────────────────────┬────────┘   │
│         │                       │                       │            │
│         ▼                       ▼                       ▼            │
│  ┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│  │ KV Store    │     │ Twilio (PSTN)    │     │ Gemini (opt.)    │   │
│  │ Redis/File/ │     │ voice webhook    │     │ fast/deep models │   │
│  │ Memory      │     │                  │     │                  │   │
│  └─────────────┘     └──────────────────┘     └──────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Optional: Python VoiceLink (on-prem)                         │   │
│  │  FastAPI :8000 — WebSocket ASR/TTS/LLM, telephony, SQLite     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  IoT: ESP32 wearable (HR/SpO₂ + SOS) → /api/sync            │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 2.2 Component responsibilities
| Component | Responsibility | State |
|---|---|---|
| Patient browser | Gesture/eye classification, TTS, WebRTC peer, metrics beacon | Local (MediaPipe models cached by SW) |
| Nurse browser | Dashboard, voice call UI, TTS, WebRTC offerer, portal assistant | Local + server session |
| Next.js API | Auth, sessions, signalling, RAG orchestration, Twilio webhook, reports | Serverless (stateless) |
| KV store | All persistent state | Redis/File/Memory |
| Twilio | PSTN dialing, TwiML fetch, speech/DTMF capture | External |
| Gemini | Optional answer phrasing | External |
| VoiceLink (Python) | Local ASR/TTS/LLM voice calls, telephony | On-prem process + SQLite |

## 2.3 Deployment topologies
- **Topology A — Cloud (Vercel):** Next.js + Upstash Redis. Browser WebRTC for calls. Gemini optional. Twilio for phone. Best for SaaS.
- **Topology B — On-prem (VoiceLink):** Python FastAPI on a hospital LAN machine; local Whisper + sherpa-onnx + local LLM; SQLite; no cloud. Best for air-gapped/privacy-first.
- **Topology C — Hybrid:** Vercel frontend + on-prem VoiceLink for ASR/TTS (browser points `NEXT_PUBLIC_VOICELINK_BACKEND` at LAN). Best for low-connectivity wards with privacy needs.

## 2.4 Key data flows
### Flow F1 — Session creation & join
1. Nurse POST `/api/care/sessions` { patientId, nurseId, nurseName, ttlMinutes }.
2. `createSession`: generate token + code; store session; index token hash; append events.
3. Response includes `joinToken` (raw, only here) and `joinCode`.
4. Nurse renders QR: `makeQrDataUrl(joinUrl?session=X&pair=TOKEN)` + code.
5. Patient scans → `/care/join?session=X&pair=TOKEN` → POST join with code/token.
6. `joinSessionAtomic` claims slot (CAS). Mint participant token. Status → active.
7. Both sides poll `/api/care/signal?to=...&sinceIdx=...`.

### Flow F2 — WebRTC call
1. Nurse `useCareRtc` (role=nurse, start=true) creates `RTCPeerConnection`, adds local tracks, creates offer, sends `offer` signal.
2. Patient polls, receives offer, sets remote desc, creates answer, sends `answer`.
3. ICE candidates flow both ways as `ice` signals; pending queue drains after remote desc set.
4. Data channel opens → status `connected`. Media flows P2P.
5. Nurse adds AI TTS track via `addTrack` → renegotiate (new offer/answer).
6. On end, `bye` signal → close.

### Flow F3 — AI turn (browser call)
1. Speech recognition → transcript → `askPortal(sessionId, text, { voice: true })`.
2. Server: load context + extras; classify route; retrieve passages; build Gemini prompt or deterministic answer; cache; persist transcript entry; return `PortalAnswer`.
3. Client speaks answer via TTS (chunked); persists turn to localStorage; appends transcript entry.

### Flow F4 — Twilio phone call
1. `placeCall` REST → Twilio dials.
2. Twilio GET/POST webhook URL (with secret `k`).
3. Turn 0: speak opening + DTMF menu, gather.
4. Caller speech/digits → POST → `askPortal` → TwiML prompt → gather. Repeat until hang-up or 12 turns.

### Flow F5 — Report
1. Nurse clicks "SBAR handoff" → `generateSbar` → `putReport` (draft).
2. Nurse reviews/edits → approve → status `approved`, `reviewedAt`, `reviewedBy`.
3. Report includes transcript ref + full transcript + provenance sections.

## 2.5 Non-functional requirements (targets)
| NFR | Target | Mechanism |
|---|---|---|
| AI turn latency (record) | < 1.5 s p95 | Local retrieval + cached patient; Gemini Flash-Lite |
| AI turn latency (voice, spoken) | < 2.5 s to first audio | Sentence chunking; fast model |
| WebRTC connect time | < 4 s on LAN | HTTP polling 1.2 s; ICE gathering |
| Availability | 99.9% (cloud) | Vercel + Redis; stateless API |
| Consistency | Strong per session | CAS joins; atomic mutateDoc; monotonic seq |
| Durability | Survives instance recycling | Redis + durable pair tokens |
| Privacy | Video never uploads | On-device MediaPipe; on-prem option |
| Offline | Core gestures work offline | SW caches models; outbox syncs later |

## 2.6 Capacity math
Assume N = 1,000 concurrent sessions (a large hospital). Each session: 1 nurse client polling signals every 1.2 s + metrics every 10 s + context refresh.
- Signal polls: 1,000 / 1.2 ≈ 833 RPS. Each hits Redis `lrange` + filter → ~2 ms → Redis load ≈ 1,666 ops/s. Plan: Upstash pay-as-you-go or self-hosted Redis (free tier 10k commands/day is too small for prod).
- AI turns: 1,000 sessions × 4 turns/hour × 8 h = 32,000 turns/day. Gemini calls (≈20%) = 6,400/day ≈ 4.4/min, within Flash-Lite free tier. Cost ≈ 6,400 × ₹0.15 = ₹960/day ≈ ₹28,800/month for 1,000 sessions → ≈0.5% of revenue at ₹60,000/ward/mo. Healthy.
- Storage: transcript 4,000 entries cap × ~200 bytes = 0.8 MB/session; 1,000 sessions = 800 MB/month. Redis cost ≈ ₹4,000/month. Fine.
- Vercel functions: 833 RPS × ~50 ms = 41,650 function-ms/s ≈ 2.5 M function-ms/min → within Pro plan (100 GB-hrs = 3.6e8 ms). Fine.

**Conclusion:** Architecture scales to 1,000 concurrent sessions on standard paid tiers with < 5% of revenue going to infra.

---

# PART 3 — Low-Level Design (LLD)

## 3.1 Entity-relationship (text)
```
Patient (1) ──< SessionPatientLink (1) ── Session (1) ──< CareEvent (N)
                                             │
                                             ├──< CareTranscriptEntry (N)
                                             ├──< CareSignal (N) [to nurse/patient]
                                             └──< CareReport (0..1)

Patient (1) ──< PatientRecord (N)
Patient (1) ──< PatientMessage (N)
Patient (1) ──< CareTask (N)
Session (1) ──< CareTask (0..1)

Nurse (LoginRecord) ──< Session (N) [nurseId]
```
All stored as KV docs/lists; no SQL joins. Relationships resolved by id lookup.

## 3.2 Store layer (KVDriver)
Interface (`src/lib/care/store.ts`):
| Method | Semantics |
|---|---|
| `getDoc(key)` | Read one document; null if missing/corrupt. |
| `setDoc(key, value)` | Overwrite. |
| `setDocIfAbsent(key, value)` | Conditional write; true if this call wrote. |
| `incr(key, by=1)` | Atomic increment; returns new value. |
| `mutateDoc(key, fn)` | Atomic read-modify-write; fn returns null to abort. |
| `appendList(key, value, cap=5000)` | Append; trim to cap from front. Returns new length. |
| `getList(key, from=0)` | Slice from index. |
| `setList(key, values)` | Replace entire list. |
| `del(key)` | Delete. |

**Driver implementations:**
- **MemoryDriver**: `Map`-backed; critical section is synchronous JS (no await between read and write within one tick). `mutateDoc` runs synchronously. Used when `NODE_ENV === "test"` or forced.
- **FileDriver**: single JSON document under `.care-data/care-store.json`; all ops serialized via a promise chain (`run`). Atomic write: write to `.tmp` then `rename`.
- **RedisDriver**: Upstash REST; `mutateDoc` uses a distributed lock (`key:__mutate_lock`, NX, PX 5000 ms) with 80 retries, backoff `10 + min(attempt,20)*3` ms. `appendList` uses `RPUSH` + `LTRIM`.

**Driver selection order:** forced env (`CARE_STORE`) → Redis if configured → memory if test/forced → file (unless Vercel without `CARE_DATA_DIR` → memory). Singleton per process; `resetDriverForTests()` clears it.

**Typed key namespace (`CareKeys` / `K`):** `care:session:<id>`, `care:session:index`, `care:session:<id>:seq`, `care:session:<id>:events`, `care:session:<id>:transcript`, `care:session:<id>:signal:<to>`, `care:session:<id>:signal-seq:<to>`, `care:report:<id>`, `care:report:index`, `care:audit`, `care:patient:<id>`, `care:patient:index`, `care:patient:seeded`, `care:patient:seed-version`, `care:patient:deleted:<id>`, `care:tasks`, `care:messages`, `care:record:<id>`, `care:records`, `care:records:seed:v1`, `care:sessionlink:<session>`, `care:sessionunlinked:<session>`, `care:ai:cache:<hash>`, `care:ai:cache:index`, `care:seed:v1`, `care:login:<hash>`, `care:jointoken:<hash>`, `care:participant:<hash>`, `care:pair:<session>`.

**Caps:** events 2000/session; transcript 4000 entries; signal queue 1000/side; reports index 1000; audit 5000; tasks 2000; messages 2000; records index 1000; session index 500; patient index 2000; AI cache index 1000.

## 3.3 Session state machine
States: `created → waiting → active → ending → ended`; also `expired`, `error`.
- `created`: just created; no join link yet (legacy).
- `waiting`: join link issued, awaiting scan.
- `active`: participant joined; `startedAt` set.
- `ending`: nurse initiated end (transient).
- `ended`: terminal; `endedAt`, `durationMs`, `endReason`.
- `expired`: join window passed without join (auto).
- `error`: unrecoverable.

Guards:
- `joinSessionAtomic` rejects if `joinedAt` set, or status in {ended, expired}, or join expired.
- `expireIfStale` re-checks inside atomic write (join may have won race).
- `endSession` is idempotent (returns existing if already ended); computes duration inside atomic update so concurrent `startedAt` set isn't lost.

## 3.4 Signalling protocol
- Two queues per session: `signal:<session>:nurse` and `signal:<session>:patient` (append-only lists).
- Signal: `{ idx, at, from, kind, data }`. `kind ∈ {offer, answer, ice, ready, bye, beacon}`.
- Counter `signal-seq:<session>:<to>` seeded from existing list length once (legacy migration), then `INCR`. Index = counter-1. Guarantees unique, gap-free indexes even on concurrent posts.
- Client polls `listSignals(session, to, sinceIdx)` → returns signals with `idx >= sinceIdx`; client advances `sinceRef = max(sinceRef, idx+1)`.
- Poll interval 1.2 s (configurable). 404/500 tolerated (keep polling).

## 3.5 WebRTC negotiation (algorithm)
Nurse (offerer):
1. `createPc()`: `RTCPeerConnection({ iceServers })`; wire `onicecandidate` → send `ice` signal; `ontrack` → remote stream; `onconnectionstatechange` → status.
2. If nurse role: `createDataChannel("care", { ordered: true })`.
3. Add local tracks.
4. `negotiate()`: `createOffer({ iceRestart: false })` → `setLocalDescription` → send `offer` signal.
5. On `answer` signal: `setRemoteDescription` → `drainPendingIce`.
6. On `ice` signal: add candidate now or queue.

Patient (answerer):
1. `createPc()` up-front (so first offer handled).
2. On `offer` signal: `setRemoteDescription` → drain pending ICE → `createAnswer` → `setLocalDescription` → send `answer`.
3. `ondatachannel` → attach.

Renegotiation: `addTrack` (nurse) adds track + calls `negotiate()` if `localDescription` exists. ICE candidate handling: queue until `remoteDescription` set, then drain. Null candidate signals end-of-gathering (ignored).

## 3.6 QR pairing (durable + ephemeral)
- In-memory `Map<session, PairRecord>` (globalThis, survives HMR) with 24 h TTL, max 20 scans/session, max 2000 sessions.
- Durable layer: `care:pair:<SESSION>` doc stores `{ token, registeredAt }`.
- `registerPairTokenPersistent`: if existing token differs → false (conflict); else store + warm in-memory.
- `verifyPairTokenPersistent`: in-memory first; on miss, read store; on match, warm in-memory.
- `rotatePairTokenPersistent`: requires prior registration; writes new token; old QR instantly invalid.
- `recordScan`: requires valid token; records `{ at, ip, device, role }`; IP from `x-forwarded-for` first hop.
- `getPairSummary`: unique IPs + last scan.

## 3.7 Auth flows
**Nurse login:** POST `/api/care/auth/login` { username, password } → `login()`:
- Find credential by username (case-insensitive).
- `safeEqual(hashToken(password), hashToken(match.password))` (constant-time).
- Mint token (32 bytes base64url, `randomBytes(32).toString("base64url")`); store `LoginRecord` at `care:login:<hash>` with 8 h expiry (`SESSION_MINUTES = 8 * 60`).
- Set cookie `care_session=<token>` (httpOnly, secure in prod, sameSite lax, path /).
- Return `{ ok, user }`.

**Nurse from request:** read cookie → `getLogin(hashToken(token))` → check expiry → delete if expired.

**Participant:** token in `x-care-participant` or Bearer → `findSessionByParticipantToken` → verify `participantTokenHash` matches.

**Credentials:** env `CARE_NURSE_USER/PASS/NAME` (defaults `nurse.demo`/`nurse.demo`/"Nurse Demo"); optional admin via `CARE_ADMIN_USER/PASS`. Demo-credentials warning surfaced.

## 3.8 RAG pipeline (detailed)
### 3.8.1 Passage construction
`buildPatientPassages(patient, extras)` in `retrieval.ts`:
- Profile passage: name, age, sex, room, concern, care plan.
- One passage per condition, medication, allergy.
- Observation passages grouped by label (latest + trend).
- Encounter passages.
- Care plan passage.
- Inserted records: chunked by paragraph (~1200 chars) with title/source/date header.
- Transcripts: per session, chunked by turns.
- Reports: summary + sections.
- Tasks, messages.

Each passage: `{ id, kind, title, text, source, at, patientId }`. General knowledge: `CARE_KNOWLEDGE` array (26 passages) with same shape, `kind: "knowledge"`.

### 3.8.2 Tokenizer & expansion
- `tokenize(text)`: lowercases; applies `COMPACT_TERMS` (SpO2/SaO2→oxygen saturation, O2→oxygen, CO2, BP→bp blood pressure, HR→hr heart rate, RR→rr respiratory rate, temp→temperature, wt→weight, HbA1c, eGFR→renal kidney, U&E→urea electrolytes, FBC/WCC/WBC→full blood count haemoglobin white cell, LFT, CRP, ESR, INR, X-ray, SOBOE/DOE→breathless exertion); splits letters/digits; strips non-alphanumerics; drops stopwords and length-1 tokens.
- `stem(term)`: light conservative stemmer (ies→y, ing, ed, trailing s).
- `SYNONYMS`: clinical synonym map (bp, hr, spo2, sats, sugar, breath/dyspnoea, renal/creatinine/eGFR, haemoglobin/HbA1c, sepsis/UTI, AF/HF/CVA/MI/DVT/PE/COPD, pain/ache/sore, mood, sleep, appetite, walk, fluid, thin/warfarin, etc.).
- `expandTerms(question)`: base terms first (longest-first), then synonyms and stems, deduped, capped at `MAX_QUERY_TERMS = 16`. Longest-first ordering protects meaningful words in dictated paragraphs.
- `wordSets(text)`: whole-word sets (raw + stemmed) for substring-free matching.
- `hasTerm(words, term)`: `words.raw.has(term) || words.stemmed.has(stem(term))`.

### 3.8.3 MiniSearch index
```ts
const mini = new MiniSearch<Passage>({
  fields: ["title", "text", "keywords"],
  storeFields: ["id"],
  tokenize,
  processTerm: stem,
  searchOptions: {
    boost: { title: 2.4, keywords: 1.35, text: 1 },
    fuzzy: (term) => (term.length >= 5 ? 0.2 : false),
    prefix: (term) => term.length >= 3,
    maxFuzzy: 2,
    weights: { fuzzy: 0.7, prefix: 0.85 },
    combineWith: "OR",
  },
});
```
- Index cached per passage signature (`passagesSignature`) in `INDEX_CACHE`, capped at `RETRIEVAL_CACHE_MAX = 40` (LRU-ish refresh on hit). Test hooks: `clearRetrievalCache()`, `retrievalCacheSize()`.
- `passagesSignature` covers id/kind/title/text/at/source/keywords/weight, so any indexed field change invalidates.

### 3.8.4 Ranking & match-strength gate
For each passage `p` and question `q` (tokenized, expanded):
- MiniSearch returns hits with `score`, `terms` (index terms matched), and `queryTerms` (the query terms that produced the match, including fuzzy/prefix hits).
- `matchedTerms = terms.filter(t => fuzzyMatched.has(stem(t)) || hasTerm(hay, t))` where `fuzzyMatched = new Set(hit.queryTerms.map(stem))`. This is the fix that lets STT slips like `potasium`/`furosemid` survive the literal word check.
- `minMatched` gate (default 1): passages with fewer than `minMatched` matched terms are dropped.
- `titleFocus`: true if any index term or focus term matches a whole word of the title.
- Post-boosts: clinical-fact kinds (condition, medication, allergy, observation) +0.4; recency `max(0, 1 - ageDays/180)` linear over 180 days; title exact match +0.3.
- Match-strength gate: `strong` requires top score ≥ 0.25 × max score AND ≥1 matched term. If not strong, answer is "not in record" rather than a weakly-matched passage.
- Excerpt: centred on the first matched term, ≤ 320 chars, with `…` ellipses.

### 3.8.5 Routing decision
`classifyQuestion(question, patient)` returns `{ scope, reason, patientSpecific }`. Scope ∈ {record, general, mixed, chitchat, meta, emergency}.

Key regexes:
- `EMERGENCY_SYMPTOM`: chest pain, crushing chest, heart attack, can't/cannot breathe, not/stopped breathing, gasping, collapsed, unconscious/unresponsive, stroke, face droop, slurred speech, severe/heavy bleeding, suicide, overdose, seizure, choking, anaphylaxis, swollen tongue, throat closing.
- `NEGATED`: no/never/without/denies/denied/denying/negative for/free of/no longer, or "does not/do not/did not/is not/are not/…", or contractions. Applied to the 56-char window before the symptom.
- `RESOLVED`: gone/resolved/settled/eased/subsided/improved/better now/no longer. Applied to the 48-char window after the symptom.
- `REPORTING_SYMPTOM`: "she/he/they/the patient … has/have/had/is/are/feels/reports/complains of/describes/says". A reported symptom wins over an education tail ("she has chest pain, help me understand" still escalates).
- `EMERGENCY_NOW`: right now/now/currently/suddenly/just started/had/happened/started having/is having/collapsed/unresponsive/not breathing/stopped breathing/bleeding heavily/a lot/emergency/immediately/quickly.
- `HISTORICAL`: when did/when was/history/previous/past medical/last time/in the past/documented/recorded.
- `EDUCATION`: what's/is/are, what should/do/would, how do/should, define/definition/meaning of, tell me about, difference between, warning signs/red flags, protocol/guideline, teach/explain/understand, signs/symptoms of, first aid for, management of, causes/triggers, risk factors for/of.
- `GENERIC_CONDITION_WORDS`: a set of words (heart, failure, kidney, liver, lung, blood, pressure, disease, disorder, syndrome, chronic, acute, cardiac, renal, hepatic, respiratory, infection, cancer, pain, anxiety, depression, essential, primary, secondary, left, right, lower, upper, small, large, cell, mellitus, stage, recovery) skipped when matching a condition name, so "heart attack" (education) is not mistaken for "heart failure" (chart).

Decision: if `EMERGENCY_SYMPTOM` matches and not negated/resolved, escalate when `happeningNow || (!historical && (reported || !educational))`. Then META, CHITCHAT, small-talk/arithmetic, mixed (record + general nouns), record-only, general-only.

### 3.8.6 askPortal flow (with cache)
`askPortal(sessionId, question, opts)` in `portalBrain.ts`:
1. `buildPortalContext(sessionId)` → patient + extras + findings.
2. `classifyQuestion(question, ctx.patient)`.
3. Emergency path → deterministic emergency script + citations [].
4. Chitchat/meta path (if small talk) → deterministic reply.
5. Retrieve general passages (`CARE_KNOWLEDGE`, limit 3, floor 0.22) and, for record/mixed scope, record passages (limit 6, floor 0.20).
6. General-only + strong general hits → rules answer or Gemini.
7. Mixed with no record hits but strong general hits → Gemini on general.
8. Record path: `rulesAnswer` (deterministic from chart) → if confident, return grounded.
9. Cache check (voice turns and dictation bypass): key = `SHA256(parts.join("\u0000")).slice(0,40)` with parts `["answer:v2", patient.id, metricsBucket, findingsKeys, question, historySignature, detail, scope]`; TTL `CACHE_TTL_MS` (default 10 min); stored at `care:ai:cache:<key>`.
10. Gemini path: `geminiGenerate` → cache put → return `{ answer, scope, grounded, citations, engine: "gemini" }`.
11. Fallback: deterministic fallback answer, `grounded: false`.

### 3.8.7 Gemini client
- Model lists: `FAST_MODELS = [gemini-flash-lite-latest, gemini-2.5-flash-lite, gemini-2.0-flash-lite]`; `DEEP_MODELS = [gemini-2.5-flash, gemini-flash-latest, gemini-2.0-flash, ...FAST_MODELS]`.
- Env: `GEMINI_API_KEY` (or `GOOGLE_GENERATIVE_AI_API_KEY`/`GOOGLE_API_KEY`), `GEMINI_MODEL` (primary), `GEMINI_FAST_MODEL`, `GEMINI_DEEP_MODEL`.
- Request: POST `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` with `x-goog-api-key` header, `systemInstruction`, `contents`, `safetySettings` (HARM_CATEGORY_* = BLOCK_ONLY_HIGH), `generationConfig { temperature 0.2, maxOutputTokens 420, topP 0.9, thinkingConfig { thinkingBudget: 0 } }`.
- Retry: on 400, retry same model without thinking config; on 429/503, sleep 400 ms then retry without thinking; on 401/403 return null (auth won't improve); default timeout 14 s.
- Spoken mode appends "SPOKEN OUTPUT: plain speech only…" to system; maxTokens often 120.

### 3.8.8 Deterministic quick answers
`quickAnswer(input)` intents (in order): `closest` (nearest threshold), `changed` (since last call), `documents` (list on file, unless `DOCUMENT_CONTENT` asks what a note says), `lastCall` (recap, unless a document question), `tasks` (open follow-ups), named-series trend/latest (if the question names a series via `labelMatches`), `live` (device readings, guarded by `DETAIL_NOUN`), `trend` (any series).
- `DETAIL_NOUN` guards live-reading answers for medication/allergy/condition/task/wound/pain/symptom/breathing questions.
- `DOCUMENT_CONTENT`/`DOCUMENT_NOUN`/`CALL_NOUN` guards prevent document questions being answered as a document list or last-call recap.
- `LABEL_ALIASES` maps nurse terms ("bp", "hr", "spo2", "sugar", "temp", "weight", "inr", …) to chart series labels, so "how is her bp trending" answers only the blood-pressure series.
- `RE.live` deliberately excludes "observations" so chart-observation questions reach the record engine.
- Findings wording special-cases `signal_stale` ("no fresh device readings… check the patient device") and `sos`.

## 3.9 Browser TTS (`browserAI.ts` / `src/lib/tts.ts`)
- `speakText(text, opts)`: chunk into ≤ ~180-char sentence chunks; pick voice (Samantha/Karen/Moira/Tessa/Fiona → en-* female → en-* → first); cancel in-flight; `setTimeout 45 ms` → speak; per-chunk watchdog `max(12000, chunk.length*130 + 6000)` ms; on watchdog fire cancel + advance. Mitigates Chrome's ~15 s synthesis stall.
- `src/lib/tts.ts`: multi-language speech output with a serialized promise queue, per-gesture cooldown (max 3 plays / 10 s), config persistence (`LANG_KEY`, `CONFIG_KEY`), `MAX_QUEUE=4`.

## 3.10 Twilio flow (LLD)
`handle(req, speech, digits)` in `src/app/api/care/voice/route.ts`:
1. Parse `session`, `turn` from URL. Validate secret `k`.
2. Normalize session (regex `^[A-Z0-9_-]{3,32}$`).
3. If digits present (or spoken digit on turn 1): menu branch — 9 hangup, 1 repeat opening, 2 live readings, 3 append audit `care.voice_urgent_flag` + flag + 112 reminder, else "did not recognise"; return `twimlPrompt(answer, actionUrl)` (gather).
4. Turn 0: speak opening + menu, gather (menu mode).
5. No speech: "did not catch", gather (no menu).
6. Hang-up regex matches: hangup.
7. Else: `askPortal(session, speech, { voice: true })`; execute action if present; if turn ≥ 12 → hangup with answer; else prompt + gather.

`twimlPrompt(text, actionUrl, opts)`:
```
<Say>text</Say>
<Gather input="speech[ dtmf]" [numDigits="1"] ... action="actionUrl">
</Gather>
<Say>noInputLine</Say>
<Hangup/>
```

Twilio helpers (`src/lib/care/voice.ts`): `twilioAccountSid`, `twilioAuthToken`, `twilioFromNumber` (default `+919908786211`), `twilioConfigured`, `emergencyNumbers` (default `+919849130407`, split on `,`/`;`), `voiceSecret` (derived from auth token if unset), `escapeXml`, `buildWebhookUrl`, `isPublicHttpsUrl`, `placeCall` (REST, form-encoded body), `classifyTwilioFailure` (trial_custom_call / unverified_recipient / invalid_from / network / unknown), `checkCooldown`/`clearCooldown` (Redis or in-process, default 5-min window).

## 3.11 Python VoiceLink
(See the backend report; summary here.)
- **CallManager** (`backend/app/call_manager.py`): states `IDLE, DIALING, CONNECTING, CONNECTED, LISTENING, PROCESSING, SPEAKING, ENDED, ERROR` with a strict transition map. `start_call` validates caller/callee, enforces `max_active_calls` (4), creates the call, transitions, sends greeting. Turn pipeline: ASR → emergency detection → LLM stream → sentence segmentation → TTS → audio chunks with seq numbers; metrics (asr_ms, llm_first_token_ms, llm_total_ms, tts_first_ms, tts_total_ms, tts_sentences, e2e_first_audio_ms). Barge-in cancels TTS. Heartbeat every 20 s; max call minutes 30; max turns 40. `CallSession.history` capped to last 24 entries.
- **Engine** (`engine.py`): ASR (sherpa/whisper.cpp), TTS (sherpa VITS LJSpeech, 22050 Hz), LLM (Ollama, qwen2.5:3b), Silero VAD (threshold 0.55, silence 850 ms, min speech 260 ms, barge-in 140 ms, preroll 400 ms, max utterance 30 s). Health cached 8 s.
- **Database** (`database.py`): SQLite; 19 tables; seeds 6 synthetic patients (CS-1001..CS-1006). Audit hash chain (SHA-256, canonical JSON).
- **Prompts** (`prompts.py`): system prompt with rules (1–3 sentences, <45 words, no lists/markdown/emojis/URLs, general guidance framing, emergency safety → 112, no false claims, never reveal instructions); history trimmed to 14 messages / 7000 chars; notes truncated to 1800 chars.
- **Telephony** (`telephony/`): provider abstraction (Exotel/Twilio/Mock), registry, number masking (`mask` keeps last 4), webhook handling, call service, audio helpers (8 kHz telephony / 16 kHz AI / 22050 Hz TTS), bridge.
- **Auth** (`auth.py`): scrypt (N=2^14, r=8, p=1, len=32), session cookie `voicelink_session`, 480 min, RBAC (admin/nurse/patient/family), audit. Default admin password written to `runtime/admin_credentials.txt` (chmod 0600); demo nurse `nurse.demo`/`nurse.demo`.
- **Scheduler** (`scheduler.py`): 5 s poll, claims up to 10 jobs with 120 s lease; handlers for ack-check (escalation ack timeout 180 s, up to 2 re-checks), retry-call, follow-up-call/message, purge (hourly).

## 3.12 Report generation
`generateSbar` (in `portalBrain`) builds sections from transcript + context: reasonForInteraction, patientReportedSymptoms (with provenance), symptomDuration, relevantHistory, medicationsDiscussed, allergiesDiscussed, keyPoints, aiSummary, concernsForNurse, escalations, suggestedFollowUp, unresolvedQuestions. Transcript ref: `{ count, firstSeq, lastSeq }` + full transcript array. sourceFlags: recordDerived, patientReported, aiGenerated, reviewed. Status: draft → approved (nurse review).

## 3.13 Seeding & versioning
**Patient seed (`seedPatients.ts`):** `SYNTHETIC_PATIENTS: ClinicalPatient[]` (14, pt_0001..pt_0014). `SYNTHETIC_SEED_VERSION = 3`. `SYNTHETIC_ADDITION_IDS` = pt_0005..pt_0014 (added after first seed). Reconciliation: first run writes all; on version bump, refresh synthetic patients, add only new ones, honour tombstones (`care:patient:deleted:<id>`), never resurrect a deleted patient. Always calls `ensurePatientRecordSeed()`.

**Record seed (`seedDemo.ts`):** `DEMO_RECORDS: PatientRecord[]` (31). `RECORDS_SEED_VERSION = 2`. Guard: `care:records:seed:v1` doc `{ version, seeded }`. For each patient in index (not tombstoned): insert records not already present (by id), skipping deleted records. Also seeds demo sessions, reports, tasks, messages on first run (guarded by `care:seed:v1`).

---

# PART 4 — Mathematics, Physics, Chemistry Analogies

## 4.1 Risk score & Shapley attribution (base product)
Risk model: additive score `S = Σ_i w_i · f_i(x)`, where `f_i` are clinical factors (alertness, blink rate, movement, HELP frequency, inactivity, etc.) and `w_i` learned/heuristic weights. Shapley value for factor i:
```
φ_i = Σ_{S ⊆ N\{i}} [ |S|! (|N|-|S|-1)! / |N|! ] · (v(S∪{i}) - v(S))
```
Since the model is additive, `v(S) = Σ_{j∈S} w_j f_j`, so `v(S∪{i}) - v(S) = w_i f_i` (constant), and the Shapley sum telescopes to `φ_i = w_i f_i`. Test asserts `Σ_i φ_i = S` (zero approximation). Rendered as signed contribution bars.

Risk factors (`src/lib/risk.ts`, `computeRisk`): HR (≤40 or >130 → 25; <50 or >110 → 12), SpO₂ (≤91 → 30; ≤93 → 18; ≤95 → 8), alertness (<25 → 30; <45 → 18; <65 → 8), blink (abnormal if <6 or >40; <3 or >55 → 15 else 8), movement (stillness ≤0.02 → 12 "Prolonged stillness"; agitated ≥0.95 → 8 "Agitated movement"), help frequency (≥2 recent HELP in 10 min → min(30, count×10)), unacknowledged critical (min(25, count×25)). `healthyBaseline` = { alertness 80, blink 15, movement 0.3, HR 75, SpO₂ 97 }. Score = clamp(round(rawSum), 0, 100); when clamped, each `φ *= score/raw` to preserve the efficiency invariant. Bands: ≥70 critical, ≥45 high, ≥20 moderate, else low. Citations: RCP-NEWS2 §hr, §spo2-scale1; CS-ADVISORY §alertness, §blink-rate, §movement, §help-frequency; NICE-CG50 §monitoring.

Escalation triggers (rule engine, `src/lib/escalation.ts` + `ESCALATION_RULES`):
- HELP frequency ≥ 3 in 120 s → escalate.
- Alertness score < 25 for ≥ 30 s → escalate.
- No movement/activity for ≥ 60 s → escalate (note: user removed the "no movement for 60 s" alert/sound per their constraint; the rule remains but is silenced in UI).
- SOS button → immediate escalate.

## 4.2 Retrieval ranking (detailed formula)
MiniSearch uses a TF-IDF variant. Conceptually, for query terms `q_1..q_m` and passage `d`:
```
score(d,q) = Σ_{t ∈ q} idf(t) · [ tf(t,d) · (k1+1) ] / [ tf(t,d) + k1·(1 - b + b·|d|/avgdl) ]
```
with `k1 = 1.2`, `b = 0.75` (BM25 defaults; MiniSearch approximates). Post-boosts (clinical facts +0.4, recency linear over 180 days, title exact match +0.3) and the match-strength gate (`strong` ⇔ top ≥ 0.25 × max AND ≥1 matched term) are described in 3.8.4.

## 4.3 Recency weighting (chart evidence)
For inserted records/transcripts, `listRecords` returns newest-first; `buildRecordContext` packs until `maxChars` (24,000). So older content is omitted by budget, not by score — retrieval then ranks the packed passages.

## 4.4 Cache key (hash)
```
H = SHA-256( s_1 ‖ "\x00" ‖ s_2 ‖ "\x00" ‖ ... ‖ s_n )
key = H[0:40]   (hex, 40 chars = 160 bits)
```
Collision probability for 10^6 keys ≈ 10^12 / 2^161 ≈ 10^-37 (negligible). Namespace `"answer:v2"` bumps on schema change. TTL = `CACHE_TTL_MS` (default 10 min). Stored at `care:ai:cache:<key>`.

## 4.5 Cooldown (Twilio protection)
```
allowed(key, t) = (t - last(key) ≥ W)  where W = windowMs (default 5 min)
```
Redis: `SET care:callcooldown:<key> <t> PX <W>` (get-then-set). Memory: `Map<key, t>`. Failed dials call `clearCooldown(key)` to release. Expected dials/hour per key = 3600000/W = 12 (for W=5 min).

## 4.6 Session TTL & expiry
Join window `T_join = ttlMinutes × 60 s` (default 30 min). Expiry check:
```
expired(s) ⇔ s.status ∈ {waiting, created} ∧ now ≥ parse(s.joinExpiresAt)
```
Auto-expire runs on read (lazy) via `expireIfStale`, atomic to avoid join race. Session duration: `durationMs = max(0, endedAt - (startedAt ?? createdAt))` [ms].

## 4.7 WebRTC ICE/SDP (conceptual)
Offer/answer exchange establishes a session description. ICE gathers candidate paths (host, srflx via STUN, relay via TURN). Connectivity checks select the best working path.
```
P(connect) = 1 - P(no host works) × P(no srflx works) × P(no relay works)
```
With TURN relay (configured via `CARE_TURN_*`), P(relay works) ≈ 0.99, so P(connect) ≈ 0.999 even on symmetric NAT. Without TURN, symmetric-NAT pairs fail (~5–10% of consumer networks). Recommendation: configure TURN for production.

## 4.8 Audio: sample-rate conversion
`downmixToPcm16(input, sourceRate)`:
```
if sourceRate == 16000:
  out[i] = clamp(input[i], -1, 1) * (input[i]<0 ? 32768 : 32767)
else:
  outLen = floor(input.length * 16000 / sourceRate)
  for i in 0..outLen-1:
    pos = i * sourceRate / 16000
    lo = floor(pos); hi = min(lo+1, n-1); frac = pos - lo
    val = clamp(input[lo]*(1-frac) + input[hi]*frac, -1, 1)
    out[i] = val * (val<0 ? 32768 : 32767)
```
Linear interpolation (first-order hold). SNR of resampling ≈ 6 dB/bit-equivalent; for voice, adequate. For music, use sinc; not needed here.

## 4.9 TTS watchdog timeout
```
T_watchdog(chunk) = max(12000, 130·|chunk| + 6000)   [ms]
```
where `|chunk|` = character count. For a 180-char chunk: max(12000, 23400+6000=29400) = 29.4 s. If synthesis hasn't fired `onend` by then, cancel + advance. Prevents indefinite stalls (Chrome bug where synthesis hangs on long utterances).

## 4.10 Capacity/latency budget (Little's Law)
Little's Law: `L = λ · W`, where L = average items in system, λ = arrival rate, W = average time in system.
- Signal poll path: λ = 833 RPS (1,000 sessions), W = 5 ms (Redis round-trip) → L = 4.2 concurrent requests. Vercel handles this trivially (concurrency limit ~1,000 per function instance).
- AI turns: λ = 32,000/86,400 = 0.37 RPS, W = 1,200 ms → L = 0.44. No queueing concern.
- Queueing (M/M/1) for Gemini: if arrival λ = 0.075 RPS (6,400/day) and service μ = 1/0.8 = 1.25 RPS (800 ms/turn), utilization ρ = λ/μ = 0.06. Average wait ≈ ρ/(μ(1-ρ)) ≈ 0.06/(1.25×0.94) ≈ 51 ms. Negligible. So Gemini is never the bottleneck.

## 4.11 Unit economics (re-derived)
```
Contribution per ward per month = ARPU × grossMargin = 60,000 × 0.78 = ₹46,800
Break-even months = CAC / contribution = 1,50,000 / 46,800 = 3.20
LTV (36 months) = 46,800 × 36 = ₹16,84,800
LTV/CAC = 16,84,800 / 1,50,000 = 11.2
```
Healthy SaaS benchmarks: LTV/CAC ≥ 3, break-even ≤ 12 months, gross margin ≥ 70%. All satisfied.
AI cost per ward per month (100 turns/day, 20% Gemini): `= 100×30 × (0.20 × 0.15 + 0.80 × 0.00) = 3,000 × 0.03 = ₹90/month`. AI cost = 0.15% of ARPU. Extremely healthy.

## 4.12 Physics/Chemistry analogies (for intuition only)
| System concept | Physical/chemical analog | Formula/intuition |
|---|---|---|
| Request queue | Chemical reaction kinetics: rate = k[A][B]; saturation at high concentration | Michaelis-Menten: v = Vmax·[S]/(Km+[S]); latency saturates like enzyme saturation |
| Backpressure | Fluid pressure in a pipe: ΔP = ρ·g·h; flow limited by narrowest section | Throughput ≤ min(capacity_i); bottleneck = narrowest pipe |
| Caching | Thermal equilibrium: hot cache = high-energy state; TTL = decay to equilibrium | Newton's cooling: T(t)=T_env+(T_0-T_env)e^{-kt}; cache hit rate decays with TTL |
| Consistency (CAS) | Chemical equilibrium: reaction proceeds until Q=K; CAS = ensuring no two reactions consume same reactant | Le Chatelier: system resists perturbation; CAS restores equilibrium |
| WebRTC ICE | Percolation: find a connected path through a random medium | Probability of connection = 1 - product of failure probabilities (see 4.7) |
| Entropy of tokens | Thermodynamic entropy S = k·ln(W); token entropy = log2(62^L) bits | 32-byte token = 256 bits → 2^256 possible tokens; brute force infeasible (2^256 ≈ 10^77) |
| Load balancing | Diffusion: particles spread from high to low concentration | Fick's law: J = -D·∇C; requests diffuse to least-loaded instance |
| Retry/backoff | Damped harmonic oscillator: retries with exponential backoff = critical damping | x(t) = (C1+C2·t)e^{-ζωt}; prevents oscillation (retry storms) |
| TTL expiry | Radioactive decay: N(t)=N0·e^{-λt}; half-life t½=ln2/λ | Cache entry "half-life" = TTL·ln2 for exponential decay models |
| Rate limiting | Electrical current limiting: I = V/R; rate limit = max current | Token bucket = capacitor charging: tokens accumulate at rate r, burst capacity b |

These analogies are for intuition only; the actual implementations use the discrete formulas in sections 4.2–4.10.

---

# PART 5 — Security (Deep)

## 5.1 Threat model
Assets: patient clinical data, session state, nurse credentials, pairing tokens, Twilio secret, Gemini key.
Threats:
- T1: Unauthorized read of patient data (session id guessing).
- T2: Session hijacking (token theft).
- T3: Replay of join link / QR.
- T4: Webhook abuse (Twilio endpoint).
- T5: Prompt injection via patient data / records.
- T6: Data exfiltration via AI answers.
- T7: Denial of service (Twilio balance drain).
- T8: Cross-session data leakage (multi-tenant).
- T9: Stored XSS via patient messages/records.
- T10: Formula injection in CSV export.

## 5.2 Mitigations (mapped)
| Threat | Mitigation | Code location |
|---|---|---|
| T1 | Session id alone not a capability; nurse cookie or pair token required | `sessionAccess.ts` |
| T2 | Tokens hashed (SHA-256) at rest; raw only in URL/memory; httpOnly cookie | `crypto.ts`, `auth.ts` |
| T3 | Join token hash stored; regeneration invalidates old; atomic join (single slot) | `repo.ts` joinSessionAtomic, regenerateJoinToken |
| T4 | `voiceSecret()` in URL param `k`; derived from auth token if unset | `voice.ts` buildWebhookUrl, route.ts |
| T5 | System prompt forbids revealing instructions; user input treated as data, not instructions; retrieval only quotes passages | `portalBrain.ts` ANSWER_SYSTEM |
| T6 | Scope labels + citations; AI never invents values; grounded flag | `portalBrain.ts` |
| T7 | Cooldown (5 min default) + clearCooldown on failure; max 12 turns; trial classification | `voice.ts` checkCooldown |
| T8 | All keys namespaced by session/patient; no global queries except audit (admin) | `store.ts` K.* |
| T9 | React auto-escapes; no dangerouslySetInnerHTML on user content; XML escape in TwiML | `voice.ts` escapeXml |
| T10 | CSV export prefixes `=`, `+`, `-`, `@` with tab/apostrophe | (report export code) |

## 5.3 Token entropy
- Join/participant token: 32 bytes = 256 bits from `crypto.randomBytes(32).toString("base64url")`. Space = 2^256 ≈ 1.16 × 10^77. Brute force at 10^9 guesses/s → 3.7 × 10^59 years. Infeasible.
- 6-digit join code: 10^6 space. Brute-forceable in ~1 s at 10^6 guesses/s, but rate-limited by session existence + expiry + atomic join (one attempt consumes the slot). Acceptable as human fallback, not a secret.
- Pair token: `crypto.randomUUID()` (122 bits) or 32-byte. Same entropy class.
- Nurse password: compared via `safeEqual(hashToken(a), hashToken(b))` — constant-time over equal-length hex digests (64 chars). Timing attack mitigated.

## 5.4 Distributed lock (Redis mutateDoc)
```
lockKey = key + ":__mutate_lock"
token = pid-timestamp-random
for attempt in 0..79:
  got = SET lockKey token NX PX 5000
  if got: break
  sleep(10 + min(attempt,20)*3) ms
if not locked: throw "Care store is busy; please retry."
try: read; mutate; write
finally: if GET lockKey == token: DEL lockKey
```
Worst-case wait = 80 × (10+60) ms ≈ 5.6 s before throwing. In practice, lock held < 1 ms, so contention is rare. Lock auto-expires at 5 s (crash safety).

## 5.5 CSP & headers
`next.config.js`: CSP restricts script sources to self + MediaPipe CDN; `X-Frame-Options: SAMEORIGIN`; `X-Content-Type-Options: nosniff`; `Referrer-Policy: strict-origin-when-cross-origin`; `Permissions-Policy: camera=(self), microphone=(self), geolocation=()`. Care routes add `Cache-Control: no-store` to prevent sensitive data caching. `/sw.js` gets `Cache-Control: public, max-age=0, must-revalidate` + `Service-Worker-Allowed: /`.

---

# PART 6 — Operations, Deployment, Runbooks

## 6.1 Environment variables (key ones)
| Var | Purpose |
|---|---|
| `UPSTASH_REDIS_REST_URL/TOKEN` | Persistent store |
| `CARE_STORE` | Force driver: redis/file/memory |
| `CARE_DATA_DIR` | File driver path |
| `CARE_SEED_DEMO` | false to disable seeding |
| `CARE_NURSE_USER/PASS/NAME` | Nurse credentials (defaults nurse.demo/nurse.demo) |
| `CARE_ADMIN_USER/PASS` | Admin credentials |
| `GEMINI_API_KEY` (or `GOOGLE_GENERATIVE_AI_API_KEY`/`GOOGLE_API_KEY`) | Optional Gemini |
| `GEMINI_MODEL/FAST_MODEL/DEEP_MODEL` | Model overrides |
| `TWILIO_ACCOUNT_SID/AUTH_TOKEN/FROM_NUMBER` | Phone calls (default from +919908786211) |
| `CARE_EMERGENCY_PHONE` | Dial targets (comma/semicolon; default +919849130407) |
| `CARE_VOICE_SECRET` | Webhook secret (derived if unset) |
| `CARE_VOICE_BASE_URL` | Public HTTPS for Twilio webhook |
| `CARE_TURN_URL/USERNAME/CREDENTIAL` | TURN server for WebRTC |
| `NEXT_PUBLIC_VOICELINK_BACKEND` | Python VoiceLink URL |
| `CARE_MAX_SESSION_MINUTES`, `CARE_JOIN_TTL_MINUTES` | Limits (default 30) |
| `CARE_TURN_URL`, `CARE_TURN_USERNAME`, `CARE_TURN_CREDENTIAL` | TURN relay |
| `CARE_ASR_MODEL`, `CARE_LLM_MODEL`, `CARE_TTS_ENGINE` | In-browser model/engine overrides |
| `NEXT_PUBLIC_CARESPEAK_URL`, `CARESPEAK_PUBLIC_URL` | Public base for QR/join links |

(Backend VoiceLink env vars: see `backend/app/config.py` — 60+ fields; key ones listed in the backend report.)

## 6.2 Deployment steps (cloud)
1. `npm install && npm run build` (verify clean).
2. Set env vars in Vercel (Redis, Gemini, Twilio, voice base URL).
3. Deploy: `vercel --prod` (or GitHub integration).
4. Verify `/api/health` → ok.
5. Create a session → scan QR from a phone on cellular (not same LAN, to test TURN).
6. Place a test Twilio call to a verified number (trial) → expect trial preamble + menu.

## 6.3 Deployment steps (on-prem VoiceLink)
1. `cd backend && python -m venv .venv && .venv\Scripts\pip install -r requirements.txt`.
2. Download models (sherpa-onnx, whisper) into `backend/models/`.
3. `python -m app.main` (or uvicorn) → listens on :8000.
4. Set `NEXT_PUBLIC_VOICELINK_BACKEND=http://<lan-ip>:8000` in frontend.
5. Verify `/health` → status ready.
6. Run `pytest -q` → 534 passing.

## 6.4 Runbook: "AI answers are wrong/off-topic"
1. Check `scope` in the answer (should be record/general/etc.).
2. Inspect citations; if empty for a record question, retrieval missed → check passage construction (`buildPatientPassages`).
3. Check `CACHE_TTL_MS` and cache key (stale cache?) → bump namespace.
4. If Gemini involved, check model (health endpoint) and safety filters (may block).
5. Reproduce with `tests/care-rag.test.ts` / `tests/care-rag-edge.test.ts` patterns; add a regression test.

## 6.5 Runbook: "WebRTC won't connect"
1. Check ICE state in `useCareRtc` (exposed as `iceState`).
2. If `failed`: missing TURN → configure `CARE_TURN_*`.
3. If signalling not flowing: check `/api/care/signal` (auth, session exists).
4. Check browser console for `getUserMedia` permission.
5. Test on same LAN first (host candidates), then cellular (relay).

## 6.6 Runbook: "Twilio call fails"
1. Check `twilioConfigured()` (all three envs).
2. Read `classifyTwilioFailure` kind: `trial_custom_call` → upgrade Twilio or use browser call; `unverified_recipient` → verify number in console; `invalid_from` → fix `TWILIO_FROM_NUMBER`; `network` → check `CARE_VOICE_BASE_URL` is public HTTPS.
3. Verify webhook reachable: `curl https://<base>/api/care/voice?session=TEST&turn=0` → should return TwiML.
4. Check secret `k` matches.

---

# PART 7 — API Reference (selected)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/care/auth/login` | none | Nurse login |
| POST | `/api/care/auth/logout` | cookie | Logout |
| GET | `/api/care/auth/me` | cookie | Current nurse |
| POST | `/api/care/sessions` | nurse | Create session |
| GET | `/api/care/sessions/:id` | nurse/pair | Get session |
| POST | `/api/care/join` | code/token | Join session |
| GET/POST | `/api/care/signal` | nurse/patient | Poll/push signals |
| GET | `/api/care/portal` | nurse/pair | Build context |
| POST | `/api/care/assistant` | nurse/pair | askPortal (JSON) |
| GET/POST | `/api/care/voice` | secret | Twilio webhook |
| GET/POST | `/api/care/transcript` | nurse/patient | Append/list transcript |
| POST | `/api/care/report` | nurse | Generate report |
| GET | `/api/care/report/:id` | nurse | Get report |
| GET/POST | `/api/care/patients` | nurse | List/create patients |
| GET/PUT/DELETE | `/api/care/patients/:id` | nurse | Patient CRUD |
| GET/POST | `/api/care/records` | nurse | List/add records |
| GET/DELETE | `/api/care/records/:id` | nurse | Get/delete record |
| GET/POST | `/api/care/tasks` | nurse | Tasks |
| GET/POST | `/api/care/messages` | nurse | Messages |
| GET | `/api/care/audit` | nurse | Audit log |
| GET | `/api/care/config` | none | Public config (ICE servers, mode) |
| GET | `/api/health` | none | Health |
| GET | `/api/ingest` (POST) | device token | Wearable vitals/SOS ingest |
| GET | `/api/network-info` | none | LAN IPv4 |
| POST | `/api/notify` | none | Emergency SMS/WhatsApp dispatch |
| GET/POST | `/api/pair` | token-gated | Pairing register/verify/scan |
| GET | `/api/stream` | none | SSE push |
| GET/POST | `/api/sync` | none | REST reconcile |
| GET | `/api/ward` | none | Ward board |

---

# PART 8 — Testing strategy

## 8.1 Coverage
TypeScript (Vitest, `environment: "node"`, 44 files, 754 tests):
- `tests/care-domain.test.ts` (29): patients, sessions, join atomicity, transcripts, reports, tasks, messages, seeding.
- `tests/care-voice.test.ts` (28): Twilio helpers, webhook menu, spoken digits, silence retry, cooldown, failure classification.
- `tests/care-rag.test.ts` (12): routing, retrieval, general answers, emergency, greetings, jokes/math, dataset integrity.
- `tests/care-rag-edge.test.ts` (22): normalisation, typos, cache, routing, quick answers, typo-safe portal answer.
- `tests/care-routing-deep.test.ts` (21): safety-first routing, historical/educational, mixed.
- `tests/care-knowledge.test.ts` (24): knowledge base shape, retrieval.
- `tests/retrieval.test.ts` (18+): buildPatientPassages, retrievePassages, evidenceAnswer, cache.
- `tests/portal-brain.test.ts` (21): assistant, dictation, SBAR, context.
- `tests/care-quick-answers.test.ts` (14): quick answer intents.
- `tests/care-assistant.test.ts` (26), `tests/care-plumbing.test.ts` (27), `tests/care-context.test.ts` (18), `tests/care-routes.test.ts` (7), `tests/care-repo.test.ts` (23), `tests/care-store.test.ts` (13), `tests/care-auth.test.ts` (17), `tests/care-client.test.ts`, `tests/care-access.test.ts`, `tests/care-ai.test.ts` (17), `tests/care-gemini.test.ts`, `tests/network-sync.test.ts` (32), `tests/frontend-audio.test.ts` (19), `tests/frontend-storage.test.ts` (32), `tests/frontend-tts.test.ts` (34), `tests/gesture-log.test.ts` (20), `tests/audit-trail.test.ts` (18), `tests/pairing.test.ts` (11), `tests/server-store.test.ts` (17), `tests/redis-client.test.ts` (7), `tests/outbox.test.ts` (7), `tests/vision-preload.test.ts` (10), `tests/i18n.test.ts` (6), `tests/platform-api.test.ts` (19), `tests/escalation-live.test.ts` (18), `tests/engines-deep.test.ts` (26), `tests/engines.test.ts` (22), `tests/enginesFix.test.ts` (5), `tests/explainability.test.ts` (8), `tests/forecast.test.ts` (4), `tests/handClassifier.test.ts` (9), `tests/vitals.test.ts` (4), `tests/analytics.test.ts` (2).
- Non-care total ≈ 374 tests across 25 files; care total ≈ 380 tests across 19 files.

Python (pytest, 534 tests): `backend/tests/` — test_bridge, test_call_manager, test_core_units, test_database, test_engine_units, test_llm_units, test_platform_api, test_providers, test_speech_units, test_telephony_routes, test_telephony_service, test_telephony_units, test_vad_units, test_telephony_platform, test_voicelink_core. Uses isolated tmp DB/secrets via `conftest.py`, deterministic mock provider, no real calls. `@async_test` (asyncio.run) because pytest-asyncio is not installed.

## 8.2 Key invariants tested
- Σ Shapley φ_i = S (risk score) — zero approximation.
- Join atomicity: two concurrent joins → one 200, one 409.
- Seeding idempotency: running twice doesn't duplicate.
- Tombstone: deleted patient not resurrected by re-seed.
- Retrieval: unrelated question → `strong == false`.
- Emergency: "chest pain now" → emergency; "warning signs of heart attack" → not emergency; negated/resolved symptoms → not emergency.
- Cache: voice turns bypass cache; namespace bump invalidates; cache bounded at 40.
- Typo tolerance: "potasium" / "furosemid" retrieve the right passage.
- Quick answers: medication-status questions do not return live readings; named series only.
- Privacy: API never returns full E.164 numbers, only masked values.

## 8.3 How to add a test
1. Place in `tests/`. Import from `@/lib/care/...`.
2. Call `resetDriverForTests()` in `beforeEach` for store isolation.
3. Use `getSeedPatient("pt_0001")` for fixtures.
4. Run `npx vitest run <file>`.

---

# PART 9 — Roadmap
| Phase | Focus | Deliverables |
|---|---|---|
| 0 (done) | Hackathon MVP | Web voice call, RAG, Twilio, VoiceLink, 14 patients, 31 docs |
| 1 (0–3 mo) | Pilot | 2–3 wards, case study, TURN config, monitoring |
| 2 (3–9 mo) | Productize | Multi-tenant, SSO, FHIR integration, billing, on-prem installer |
| 3 (9–18 mo) | Scale | Multilingual packs (10→22 langs), mobile nurse app, analytics dashboard |
| 4 (18–36 mo) | Platform | Marketplace for clinical content packs, third-party device integrations, research API |

---

# PART 10 — Appendix

## 10.1 File index (core)
| File | Role |
|---|---|
| `src/lib/care/types.ts` | Domain types |
| `src/lib/care/store.ts` | KV driver abstraction |
| `src/lib/care/repo.ts` | Repository layer |
| `src/lib/care/crypto.ts` | Tokens, hashing, ids |
| `src/lib/care/auth.ts` | Nurse/participant auth |
| `src/lib/care/sessionAccess.ts` | Session authorization |
| `src/lib/care/qr.ts` | QR generation |
| `src/lib/care/voice.ts` | Twilio helpers |
| `src/lib/care/portalBrain.ts` | askPortal, context, SBAR |
| `src/lib/care/retrieval.ts` | MiniSearch RAG |
| `src/lib/care/knowledge.ts` | Routing, small talk, emergency |
| `src/lib/care/careKnowledge.ts` | General-care passages (26) |
| `src/lib/care/gemini.ts` | Gemini client |
| `src/lib/care/browserAI.ts` | Browser TTS + ASR |
| `src/lib/care/seedPatients.ts` | 14 synthetic patients |
| `src/lib/care/seedDemo.ts` | 31 records + demo history |
| `src/hooks/useCareRtc.ts` | WebRTC |
| `src/hooks/useVoiceCall.ts` | Python WS client |
| `src/components/care/PhoneCall.tsx` | Browser call UI |
| `src/app/api/care/voice/route.ts` | Twilio webhook |
| `backend/app/main.py` | FastAPI entry |
| `backend/app/call_manager.py` | Call lifecycle |
| `backend/app/prompts.py` | LLM prompts |
| `backend/app/database.py` | SQLite + seeds |

## 10.2 Environment variable count
32+ variables (see 6.1). All optional except Redis for multi-instance production.

## 10.3 Glossary of abbreviations
AI, API, ARPU, ASR, BOM, CAS, CSP, CSV, EHR, FHIR, HLD, HTTP, ICE, ICU, LLD, LLM, LTV, M/M/1, NAT, NFR, PCM, QR, RAG, RPS, SaaS, SBAR, SDP, SSE, STUN, TAM/SAM/SOM, TTS, TTL, TURN, URL, WebRTC, WSS.

---

*End of document.*
