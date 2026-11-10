# KenyaLugha AI

Voice-first livestock triage for Kenyan farmers. A farmer describes a sick animal in their own language, the system transcribes and translates it, structures the case, applies **fixed veterinary safety rules** to set urgency, returns simple guidance, and routes emergency and urgent cases to the nearest responder.

> Guidance to help prepare for veterinary care, not a diagnosis. The AI never diagnoses, prescribes, or sets final urgency.

## What is in the app

| Area | Where | Notes |
| --- | --- | --- |
| Voice intake | `/cases/new` | 30–90 s recording (follow-ups up to 3 min), live text when the browser supports it, transcript check with language detection, GPS |
| Offline | `src/lib/offline/store.ts`, `public/sw.js`, `/emergency` | IndexedDB drafts + outbox, idempotency keys, exponential backoff, dead-letter, service-worker shell cache |
| Triage rules | `src/lib/domain/rules.ts` | 40+ rules with IDs and fixed reasons. Authoritative for urgency. Runs on server and device |
| Extraction | `src/lib/domain/heuristic-extract.ts`, `src/lib/server/llm.ts` | Rules-based extractor always runs. An OpenAI-compatible LLM can add structure and clarifying questions |
| Safety validator | `src/lib/domain/safety-validator.ts` | Blocks doses, medicine names, diagnosis claims, cure promises, and "wait" advice in model output |
| Guidance | `src/lib/domain/retrieval.ts`, `src/lib/domain/analysis.ts` | Composed only from approved, retrieved guidance and fixed templates (English + Swahili) |
| Cases | `/cases`, `/cases/[id]` | Full CRUD, transcript correction (re-analyses), follow-ups, outcomes, audio playback (decrypted on request) |
| Vet network | `/vets` | Directory CRUD, nearest-responder routing, notification inbox at `/alerts` |
| Knowledge base | `/guidance` | Approved guidance CRUD. Entries containing doses or drug names are rejected |
| Feedback loop | `/review`, `/evaluation` | Corrections to transcripts, translations, triage (human decision), guidance ratings, outcomes; rule pack and metrics |
| Audit | `/audit` | Every model call, prompt version, safety block, and human override |
| Admin | `/users`, `/settings` | Roles: FARMER, AGENT, VET, ADMIN |

## Quick start

```bash
npm install
# .env must contain DATABASE_URL, for example postgresql://postgres:postgres@127.0.0.1:5432/app_db
npx drizzle-kit push --force   # create tables
npm run build && npm start     # or use the preview tooling
```

The first request seeds demo data (vets, farmers, cases, guidance, feedback, audit trail). Demo accounts use password `demo1234`:

| Role | Phone |
| --- | --- |
| Farmer (Meru) | 0733000001 |
| Community agent | 0722000001 |
| Veterinarian (Kisumu) | 0711000001 |
| Administrator | 0700000000 |

Phone numbers and the hotline in the seed are fictional placeholders. Replace them before any real use.

## Optional model endpoints

Every AI step works without a model. Configure these environment variables to switch a step to a model. The app records which engine handled each step in the audit trail.

| Variable | Purpose | Default when unset |
| --- | --- | --- |
| `ASR_API_URL`, `ASR_API_KEY`, `ASR_MODEL` | OpenAI-compatible `/audio/transcriptions` (Paza Whisper, MMS, or a fine-tuned Whisper) | Browser speech recognition, or typed text |
| `LLM_API_URL`, `LLM_API_KEY`, `LLM_MODEL` | OpenAI-compatible `/chat/completions` (Llama 3.1 8B / 3.3 70B, Phi-4-mini, etc.) | Rules-based extractor |
| `TRANSLATION_API_URL`, `TRANSLATION_API_KEY`, `TRANSLATION_MODEL` | NLLB-style `/translate` endpoint | Approximate glossary translation, clearly labelled |
| `SMS_API_URL`, `SMS_API_KEY`, `SMS_SENDER_ID` | Generic JSON SMS gateway | Alerts queued in the escalation inbox |
| `APP_ENCRYPTION_KEY` | Key material for AES-256-GCM audio encryption at rest | Development-only fallback. **Set this in production.** |
| `COOKIE_SECURE=true` | Mark session cookies secure (HTTPS) | Off |

## Safety design

- **Urgency is decided by rules.** The LLM can only add findings and questions. If it suggests a higher level than the rules, the case is escalated for review, but the rules' level stands.
- **Red flags are scanned twice**: in the translated English text and in the LLM's output. The model cannot hide a finding from the rules.
- **Escalation**: emergency and urgent cases create alerts for the nearest responder, preferring 24/7 cover for emergencies. Weekly review sampling flags emergency, urgent, low-confidence, and a random routine sample.
- **Guidance is retrieved, not generated.** Only approved entries from the knowledge base (and fixed templates) reach the farmer.
- **Auditability**: every AI call, prompt version, safety block, triage decision, and human override is written to `audit_logs`.

## Architecture

```text
Farmer (PWA / browser)
  ├─ IndexedDB: drafts + outbox (idempotency keys) ──► POST /api/sync (priority order)
  └─ POST /api/cases  (or /v1/* service endpoints)
         │
   Case pipeline (src/lib/server/pipeline.ts)
     ASR ─► translation ─► heuristic extraction (+ LLM) ─► merge findings
          ─► deterministic triage ─► safety validator ─► retrieval ─► guidance
          ─► farmer-language delivery ─► transaction ─► vet routing ─► audit
         │
   PostgreSQL (Drizzle)  ·  encrypted audio (bytea, AES-256-GCM)
```

The same pure modules (`src/lib/domain/*`) run on the server and on the device, so offline triage behaves the same as online triage.

## Commands

```bash
npx next typegen
npm exec tsc -- --noEmit
npm run build
```

## Known limits (pilot scope)

- Swahili templates are drafted and need native-speaker review. Other languages show Swahili and English until reviewed.
- The glossary translation is approximate. Configure an NLLB endpoint for real translation.
- Approved guidance entries are marked `PENDING_REVIEW` until a Kenyan veterinarian signs them off.
- The Android app is not in this repository. `docs/device-schema.sql` is the Room/SQLite reference for that build.
"# KenyaLugha-ai" 
