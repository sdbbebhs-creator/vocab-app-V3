# AGENTS.md

## Architecture

- React 19 + Vite SPA (`src/`). No Express server — the old `server.ts` was removed in favor of Netlify Functions.
- `netlify/functions/*.mts` — serverless endpoints using the Web `Request`/`Response` API with `export const config: Config = { path: '/api/...' }` for routing:
  - `lookup-item.mts` → `/api/ai/lookup-item` (Gemini-backed vocab/grammar autofill, offline fallback)
  - `optimize-fsrs.mts` → `/api/ai/optimize-fsrs` (Gemini-backed FSRS parameter tuning, offline fallback)
  - `context-practice.mts` → `/api/ai/context-practice` (Luyện Output 5 ngữ cảnh: `action` = `suggestContexts` | `sampleSentence` | `validate`, works for both vocab and grammar via `kind`). All Gemini calls use `responseMimeType: 'application/json'` + a `responseSchema`, try a small model fallback list, and return HTTP 429 `{ code: 'RATE_LIMITED' }` when every model is rate-limited.
  - `sync.mts` → `/api/sync` (legacy Netlify DB storage; now only read once by `useSyncData` to import old data into Supabase when an account has no Supabase rows)
- `db/schema.ts` — Drizzle schema (`vocab_items`, `grammar_items`, each row is `{ user_id, id, data: jsonb, updated_at }`, one JSONB blob per item so the full `VocabItem`/`GrammarItem` shape doesn't need to be hand-modeled in SQL).
- `db/index.ts` — Drizzle client via `drizzle-orm/netlify-db`.
- Migrations live in `netlify/database/migrations/`, generated with `npx drizzle-kit generate --name <slug>`. Never edit or apply migrations manually — Netlify applies them automatically at deploy time.
- Data & auth: Supabase handles identity **and** stores app data. `src/hooks/useSyncData.ts` (used by `App.tsx`; `setVocabList`/`setGrammarList` wrap its `saveData`) diffs every list change and upserts/deletes only the changed items in Supabase `vocab_items` / `grammar_items` (`{ user_id, id, data, updated_at }`, `onConflict: 'user_id,id'`), with an offline outbox, Realtime cross-device updates and a `study_data` whole-list backup. Table/RLS setup SQL lives in `supabase/migrations/` and must be run in the Supabase SQL editor.
- New items get `id: crypto.randomUUID()`; `ensureFSRSCard` in `src/utils/storage.ts` backfills a UUID for any item missing an id. IndexedDB/localStorage is only a local cache.

## Conventions

- Gemini keys stay server-side. The browser only talks to `/api/ai/*`; `src/utils/geminiPractice.ts` wraps the context-practice endpoint and owns the fallbacks (default tabs `DEFAULT_CONTEXTS`, offline scoring from `contextEngine.ts`). `ContextOutputModal.tsx` never calls Gemini directly.
- Gemini 1.5/2.0 Flash are retired and not available through Netlify AI Gateway; `gemini-2.5-flash` is the primary model. Check the `netlify-ai-gateway` skill's model list before changing models.

- Do not reintroduce automatic sample-data seeding. `getSampleData()` in `src/utils/storage.ts` exists only for the manual "Nạp mẫu"/"Nạp bộ thẻ mẫu" button (`BackupModal.tsx`, `handleLoadSampleData` in `App.tsx`) — new accounts must start empty.
- Netlify Functions in this repo use `.mts` and the Web API signature (`export default async (req: Request) => ...`), not the older Lambda-style handler signature.
- The Supabase URL and publishable anon key are safe to embed client-side; they're hardcoded as fallbacks in `src/utils/supabaseClient.ts` and `netlify/functions/sync.mts`, overridable via `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY` (client) and `SUPABASE_URL`/`SUPABASE_ANON_KEY` (server) env vars.
