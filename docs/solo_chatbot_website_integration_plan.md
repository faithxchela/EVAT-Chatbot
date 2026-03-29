# Solo Implementation Plan: EVAT Chatbot First, Then Website Integration

## 1) Current architecture (what exists today)

### Chatbot repository (`EVAT-Chatbot`)
- **Rasa bot engine** in `rasa/`:
  - NLU + policies: `rasa/config.yml`
  - Intent/entity/slot/action registry: `rasa/domain.yml`
  - Training data: `rasa/data/nlu.yml`, `rasa/data/stories.yml`, `rasa/data/rules.yml`
  - Action server endpoints/config: `rasa/endpoints.yml`, `rasa/credentials.yml`
- **Custom action/business logic** in `rasa/actions/`:
  - Core flow actions are in `rasa/actions/actions.py`
  - Data + station lookup helpers in `rasa/actions/data_service.py`
- **Datasets** in `data/raw/`:
  - `charger_info_mel.csv`, `Co-ordinates.csv`
- **Frontend chat prototype** in `frontend/`:
  - UI and webhook wiring in `frontend/script.js`
  - Current REST webhook target is hardcoded to `http://localhost:5005/webhooks/rest/webhook`

### Website repository (`Chameleon-Website`)
- **Not accessible from this environment** (clone/network access unavailable), so integration points below are provided as practical templates and should be mapped to your actual file structure after checkout.

---

## 2) Highest-priority chatbot fixes (before integration)

1. **NLU data cleanup first**
   - Remove duplicate/conflicting examples and malformed entries in `rasa/data/nlu.yml`.
   - Keep each intent semantically tight, especially around:
     - `menu_selection`
     - `preference_selection`
     - `route_station_selection` / `emergency_station_selection` / `preference_station_selection`
     - `nlu_fallback`

2. **Fallback behavior second**
   - Current fallback handling is mostly context rules; add robust fallback messages + confidence threshold handling.
   - Ensure fallback always gives a recovery path (menu, buttons/chips, example utterances).

3. **Domain response coverage third**
   - `rasa/domain.yml` has minimal template responses currently.
   - Add dedicated `utter_...` responses for: ask location, ask menu selection, ask preference type, fallback, clarify station name.

4. **Station recommendation flow (MVP-critical)**
   - Strengthen route/emergency/preference station selection behavior in `rasa/actions/actions.py`.
   - Ensure station cards and “Get Directions” always work with valid station ids.

---

## 3) Solo-friendly phased implementation plan

## Phase 0 — Baseline and observability (0.5 day)
**Goal:** Create a reproducible baseline so every improvement is measurable.

**Files/folders:**
- `rasa/` (all config + data)
- `frontend/script.js`

**Changes:**
- Add/standardize a simple local test script (`rasa test nlu`, `rasa data validate`, `rasa train`).
- Save current intent confusion matrix results to compare later.

**How to test:**
- Run:
  - `rasa data validate`
  - `rasa train`
  - `rasa test nlu --cross-validation 5`
- Pass condition: no schema errors, model trains, and baseline metrics captured.

## Phase 1 — NLU cleanup for biggest gain (1–2 days)
**Goal:** Improve intent classification accuracy with minimal code changes.

**Files/folders:**
- `rasa/data/nlu.yml`
- `rasa/domain.yml` (intent alignment only)

**Changes:**
- Remove duplicate examples and obvious typos that overlap intents.
- Remove malformed samples (e.g., placeholder-like entries) from station-selection intents.
- Reduce overlap between `route_planning`, `route_info`, and `menu_selection`.
- Normalize connector/preference examples to avoid ambiguity.
- Keep about 20–40 high-quality examples per key intent rather than many noisy examples.

**How to test:**
- `rasa data validate`
- `rasa train`
- `rasa test nlu`
- Manual tests in shell:
  - “I need a charger near Carlton now”
  - “show fastest chargers”
  - “from Richmond to Geelong”
- Pass condition: fewer misclassifications between route/emergency/preference/menu intents.

## Phase 2 — Fallback + recovery UX (1 day)
**Goal:** Bot fails gracefully and recovers user to target flow.

**Files/folders:**
- `rasa/config.yml`
- `rasa/data/rules.yml`
- `rasa/domain.yml`
- `rasa/actions/actions.py`

**Changes:**
- Add `FallbackClassifier` to pipeline with explicit threshold tuning.
- Add clear fallback utterances in domain (with examples and quick options).
- Add/adjust rules so fallback in each conversation context routes to the right recovery action.
- In actions, if user text is close to known station names, offer top 3 suggestions instead of dead-end fallback.

**How to test:**
- Inputs intentionally ambiguous:
  - “hmm not that”
  - “asdfghj”
  - partial station names
- Pass condition: fallback always returns a useful next step and does not break context.

## Phase 3 — Station recommendation flow hardening (1–2 days)
**Goal:** Make charger-finding path reliable (your MVP core).

**Files/folders:**
- `rasa/actions/actions.py`
- `rasa/actions/data_service.py`
- `data/raw/charger_info_mel.csv`
- `data/raw/Co-ordinates.csv`

**Changes:**
- Tighten location resolution and station matching logic.
- Ensure every returned station has consistent fields required by frontend cards:
  `station_id`, `name`, `address`, `distance_km`, `cost`, `power`, `availability`.
- Add guardrails when no stations found (expand radius, suggest nearby suburb, prompt preference switch).

**How to test:**
- End-to-end scenarios:
  1. Emergency flow + connector type
  2. Preference flow + cheapest/fastest/premium
  3. Route flow + station pick + get directions
- Pass condition: all three flows return station cards and handle zero-result cases cleanly.

## Phase 4 — Frontend API URL externalization (0.5 day)
**Goal:** Make frontend deployable without code edits for each environment.

**Files/folders:**
- `frontend/script.js`
- (optional) website env file after integration

**Changes:**
- Replace hardcoded localhost webhook URL with environment-driven base URL.
- Keep default localhost for local dev.

**How to test:**
- Local: URL resolves to localhost.
- Staging: URL resolves to public Rasa endpoint.
- Pass condition: same frontend code works in local and deployed environments.

---

## 4) Integration strategy recommendation

### Compare options
1. **Direct frontend → Rasa webhook**
- Pros: simplest, fastest for solo dev.
- Cons: CORS/security/exposed endpoint/rate-limit concerns.

2. **Frontend → backend proxy → Rasa**
- Pros: better security/control/logging, can hide tokens and apply throttling.
- Cons: extra service to build and deploy.

3. **Embedded chatbot UI widget**
- Pros: easiest UX insertion on website pages.
- Cons: still needs option 1 or 2 for backend transport.

### Recommended for one person (MVP)
- **Start with direct frontend → Rasa webhook** for speed.
- Then move to **light proxy** only after MVP is stable.

---

## 5) Step-by-step website integration plan

Because `Chameleon-Website` was inaccessible in this environment, treat file names below as likely targets and map to your repo structure.

1. **Locate website integration points**
   - Search for:
     - global layout/app shell (e.g., `src/App.*`, `layouts/*`)
     - shared components folder (e.g., `components/`)
     - env handling (`.env`, `netlify.toml`)

2. **Add chatbot launcher + panel component**
   - Create component `ChatWidget` (or equivalent):
     - floating button bottom-right
     - expandable panel containing existing chat UI or iframe/embed

3. **Hook message send logic to Rasa REST webhook**
   - Endpoint: `https://<your-rasa-host>/webhooks/rest/webhook`
   - Payload:
     ```json
     {"sender":"web-user-<id>","message":"<text>","metadata":{"lat":...,"lng":...}}
     ```

4. **Add robust client behavior**
   - Loading state, retry on timeout, offline message, and simple local history.

5. **Configure Netlify env var**
   - Add `RASA_WEBHOOK_URL` (or `VITE_RASA_WEBHOOK_URL` for Vite).
   - Use env var in website code; no hardcoded backend URL.

6. **Smoke test in deploy preview**
   - Validate CORS + HTTPS + mixed-content constraints.

---

## 6) Deployment plan (Render + Netlify)

### Why Render is a good fit
- Quick Docker/Web Service setup.
- Easy public HTTPS endpoint required by Netlify frontend.
- Environment variables and scaling are straightforward.

### Recommended deployment setup
1. **Deploy Rasa server on Render Web Service**
   - Start command (example):
     - `rasa run --enable-api --cors "*" --port $PORT`
   - Ensure binding host `0.0.0.0`.

2. **Deploy action server on Render (second service)**
   - Start command:
     - `rasa run actions --port $PORT`
   - Update `rasa/endpoints.yml` action endpoint to public Render URL.

3. **Model availability**
   - Option A: commit model artifacts (quick but less ideal)
   - Option B: run `rasa train` in build stage and start with latest model (better long term)

4. **Set required env vars on Render**
   - TomTom/API keys and any secrets.

5. **CORS + credentials**
   - For MVP: allow Netlify domain + localhost.
   - Tighten later (avoid `*` once stable).

6. **Point website (Netlify) to Render URL**
   - `RASA_WEBHOOK_URL=https://<rasa-service>.onrender.com/webhooks/rest/webhook`

7. **Post-deploy validation**
   - From browser network tab: confirm 200 on webhook calls.
   - Confirm action callbacks succeed (no 5xx in Render logs).

---

## 7) MVP scope (minimum you should ship first)

Ship this first before polish:
1. User opens website chat widget.
2. User shares location or types suburb.
3. User selects **Emergency** or **Preferences**.
4. Bot returns top 3 station cards.
5. “Get Directions” works reliably.
6. Fallback always offers recovery choices.

Avoid for MVP:
- multi-region expansion
- advanced personalization
- deep analytics dashboards
- complex auth/rate-limiting proxy layer

---

## 8) Risks and blockers

1. **Website repo access / unknown structure**
   - Mitigation: once accessible, map this plan to exact files in first 1–2 hours.

2. **NLU ambiguity from overlapping intents**
   - Mitigation: strict intent boundaries + regular `rasa test nlu` checks.

3. **Fallback loops / context drift**
   - Mitigation: context-specific fallback rules and explicit slot resets.

4. **Public endpoint reliability**
   - Mitigation: Render health checks, logs, and conservative timeout handling in frontend.

5. **Third-party API rate limits (TomTom)**
   - Mitigation: cache frequent lookups; graceful degraded response when API fails.

---

## Recommended order for next 1–2 weeks

### Week 1
1. Phase 0 baseline
2. Phase 1 NLU cleanup
3. Phase 2 fallback hardening
4. Regression test core station flows

### Week 2
5. Phase 3 station-flow hardening
6. Deploy Rasa + actions to Render
7. Integrate website chat widget with env-based webhook URL
8. End-to-end MVP smoke tests on Netlify + Render

Deliverable at end of week 2: a stable **charger-finding MVP** on the live website, with reliable fallback and directions.
