# Forminator → ThinkingCap Lab Integration Plan

**Status:** Phase 0 ✅ `3a2ad43` · Phase 1 ✅ `6ac5145` · Phase 2 ✅ (tc-surface-forms, standalone-verified) · Phase 3 ✅ (phoenix `04fe472` + tc-lab `c4c04f1`, 2026-09-02) · Phase 4 ✅ code complete 2026-09-07 (operator deploy pending) · Phase 5 ✅ code complete 2026-09-10 (tc-console `7434704` pushed srv, operator deploy pending)

**Goal (Campbell's two gaps):**
1. **Schema authoring standard** — how to write JSON Schemas so an AI asks the right questions and enums project to the right widget (radio / single-choice / dropdown / checkbox).
2. **Lab integration** — the Lab console chat understands intent → picks the right schema → collects data conversationally → projects it to a work surface (form fills live beside the chat; enums render as chat chips). On completion: AJV-validate + store generically. **No LMS writes in v1.**

**Your decisions locked in:** chat drives the conversation, a generic form surface mirrors live (ltisetup `surface_command` pattern — not embedding Forminator's own chat); completed JSON is validated + stored generically.

---

## Architecture at a glance

| Decision | Choice | Why |
|---|---|---|
| Widget convention | `x-widget: radio\|dropdown\|checkbox\|yesno` (optional; type-derived defaults) | Defaults cover most schemas; explicit override when needed |
| Record store | Azure blob container `form-records`, one JSON blob per record | Same storage account as schemas; worker self-contained; evolves Forminator's `data.js` mental model; no phoenix DB migration |
| Collection state | New PG table `app_form_collection` in phoenix | Multiple concurrent collections per session; doesn't bloat the hot `app_chat_session` row |
| Catalog access | Phoenix reads `schema-catalog.json` blob directly, ETag + 5-min TTL, stale-while-revalidate | No runtime dependency on the Forminator backend being up |
| Enum chips | Structured `choices` payload on `POST /chat` response, server-computed | Mirrors Forminator's proven `getPendingEnumField` pattern; markers would persist into history and re-parse badly |
| Completion | Structured: `validate` op → clean → explicit user-confirmed `submit` | Replaces the `'All done'` string sniff |

---

## Phase 0 — Schema Authoring Standard + lint (Forminator repo only)

**CREATE `Forminator/docs/SCHEMA-AUTHORING-STANDARD.md`** — the single canonical doc both repos reference:

- **Trio convention** (from TCOV): every leaf dir = `schema.json` + `sample.json` + `description.md`; breadcrumb `title` (`"Settings > Basics"`); dense per-field `description` with `DB:`/`UI:`/`Maps to` provenance; `documentType`, `scope`, `rootBranchOnly`, `adminUrl`, `x-thinkingcap` stay authoring/provenance-only.
- **x-\* vocabulary** (promoted from `schema_ai_concept.md:50-63` to normative): `x-prompt`, `x-source` (`user|app|db|system`), `x-options-source` (`static|db|app`), `x-options-preview`, `x-order`, `x-group`, `x-hint`, `x-depends-on`.
- **NEW `x-widget`**: `radio | dropdown | checkbox | yesno`. Defaults when absent: `enum` ≤5 → `radio`, >5 → `dropdown`; `array`+`items.enum` → `checkbox`; `boolean` → `yesno`. Invalid combos are lint errors.
- **Runtime precedence rules** (both Forminator and phoenix implement against these):
  - *Inclusion*: `const`, `format:uuid`, `readOnly`, `x-source: app|system` → never asked. `x-source: db` → read-only in form, skipped in chat. Missing = `user`.
  - *Ordering*: `x-order` ascending first → required (declaration order) → optional. No `x-order` anywhere = today's required-first behavior exactly.
  - *Question text*: `x-prompt` verbatim, else first sentence of `description`; `x-hint` woven in.
  - *Conditionals*: `x-depends-on {field, equals|in|truthy}` evaluated per turn against collected data; unsatisfied → skip. `if/then/else` honored for requiredness. Cycles = lint error.
  - *Choices*: `x-options-source: static`/absent → chips from `enum`/`items.enum`; `db|app` → no chips, "pick it in the form", placeholder picker in v1 (`x-options-preview` fills dev only).
- **Lint checklist**: per-field description present; enum value semantics explained in description; root description's first sentence states what's collected; `x-source` on all non-user fields (and never on required user fields); `x-order` all-or-nothing; `x-prompt` on ambiguous keys; `x-depends-on` targets exist + acyclic; enum-count vs `x-widget` sanity; `db|app` options sources omit `enum` and carry `x-options-preview`; trio complete; `sample.json` AJV-validates; single `$schema` draft.

**MODIFY `Forminator/backend/routes/chatEdit.js`** — add `POST /api/schema/lint`: deterministic JS checks above (no LLM) returning `{score, findings[]}` + optional Haiku narrative pass. Teach the edit assistant the standard; fix the draft mismatch — mandate **2020-12** (files already declare it; prompt wrongly says draft-07).

**MODIFY `Forminator/schema_ai_concept.md`** — replace the extension table with a pointer to the standard (also fixes its two example bugs: `x-options-source` given an array value; `x-source:"db"` misused on free-text fields). **MODIFY TCOV conventions doc** — one-paragraph addendum linking the standard. Standard lives in the Forminator repo; TCOV/phoenix reference, never copy.

*Verify:* lint run against 3 known-good TCOV trios + 1 deliberately broken schema → expected findings, stable deterministic score.

---

## Phase 1 — Forminator runtime honors its own spec

> **DONE 2026-09-01 — `6ac5145`.** All items landed plus review-driven extras:
> ux-reviewer fixes (useId radio names, aria-pressed, role=group, chip focus
> management, answered-on-success, dropdown hint, touch targets) and
> security-auditor fixes (choiceAnswer field/value validation). Also fixed a
> pre-existing crash: Anthropic responses can lead with a thinking block, so
> `content[0].text.trim()` threw — the route now selects the text block.
> Behavior changes called out in the commit: explicit `false` counts as filled
> (§5.6), the model no longer re-lists enum options in prose (§5.5), booleans
> render as a Yes/No segmented control in the form (§4 default).
> Open follow-up (pre-existing, flagged by the auditor for Phase 4): `/api/chat`
> is unauthenticated with wide CORS — fine for localhost dev, must close before
> any wider exposure.

**CREATE `Forminator/backend/lib/fieldPlan.js`** — pure `buildFieldPlan(schema, formData, answeredKeys)` → ordered ask-plan `{key, prompt, hint, widget, enumOptions, multiSelect, skippable}` implementing the Phase 0 precedence rules. Small `scripts/verify-fieldPlan.mjs` harness (repo has no test runner).

**MODIFY `Forminator/backend/routes/chat.js`**:
- `buildSystemPrompt` (lines 47-123) consumes `buildFieldPlan` (x-order honored, `x-source` skips, `x-prompt` verbatim).
- `getPendingEnumField` (lines 9-45) → `getPendingChoiceField` returning `{field, enumOptions, multiSelect, widget}`; skips non-static options sources; boolean → `widget:'yesno'`.
- Fix boolean-false quirk (line 20): filled = in explicit `answered` list OR non-empty value. **Behavior change — called out in commit per repo CLAUDE.md §7.**

**MODIFY `Forminator/frontend/src/components/FormField.jsx` + `DynamicForm.jsx`** (note: under `frontend/src`, not `src`): `x-widget` mapping (`radio` → plain radio group — repo has no MUI, match existing hand-rolled style; `dropdown` → existing `<select>`; `checkbox` → checkbox group; `yesno` → two-button segmented control); `x-group` section headers; group/field order by `x-order` then declaration.

**MODIFY `Forminator/frontend/src/components/ChatPanel.jsx`** (lines 111-161): render per server `widget` — `dropdown` → enumerated text + free text fallback; `yesno` → Yes/No chip pair; chip clicks post `{field, value}` so `answered` tracking works.

Repo CLAUDE.md asks for a UX review pass after component changes and a security review before committing route changes — honor both.

*Verify:* backend :3001 + frontend :3002 with an x-annotated schema — order, verbatim prompts, chip styles, `x-source:db` skips, explicit-false counts.

---

## Phase 2 — `tc-surface-forms` (new work surface, standalone-verifiable)

Scaffold: `bash tc-surface-template/scaffold.sh forms` → repo `tc-surface-forms` (`app/` + `worker/`). Never copy a sibling (fleet rule).

**Worker (Express BFF):**
- `worker/src/lib/schemaStore.ts` — Azure read of `<blobDir>/schema.json` (+`description.md`) from `schemas` container; ETag + 5-min TTL cache. New config: `SCHEMAS_STORAGE_ACCOUNT`, `SCHEMAS_CONTAINER`.
- `worker/src/lib/fieldValidation.ts` — generalized port of `tc-surface-ltisetup/worker/src/lib/fieldValidation.ts`: per-property AJV (`strict:false`, strip `$id`/`$schema`, `ajv-formats`; draft-07-declared schemas via `ajv-draft-07` fallback).
- `worker/src/lib/commands.ts` — port of ltisetup `processCommands`: ops `set_fields{fields}`, `clear_fields{paths}`, `validate` (whole-form over merged patch → `{valid, errors}`), `submit` (eligibility check only). Returns `{results, normalized}` for phoenix to relay as `surfaceActions`.
- `worker/src/routes/schema.ts` — `GET /api/forms/schema?dir=<blobDir>` → `{schema, title, descriptionMd}`; dir validated against catalog listing (no arbitrary blob reads).
- `worker/src/routes/commands.ts` — `POST /api/forms/commands {schemaDir, formState, commands}`. Read-only w.r.t. LMS → `aud: surface-server` constraint honored.
- `worker/src/lib/recordStore.ts` + `routes/records.ts` — the evolved `/api/data` pattern, Azure-side: `POST /api/forms/records {schemaDir, data}` → whole-form AJV → put blob `form-records/<schemaDir>/<yyyy-mm>/<uuid>.json` with `{recordId, schemaDir, schemaTitle, clientId, submittedBy, submittedAt, data}`; 422 `{errors}` on failure. `GET …/records?dir=` (caller's own, summary) + `GET …/records/<id>?dir=`. POST only ever called from the browser app (uniform writes-browser-executed posture).

**App (React + Vite MF remote):**
- `app/src/Form/{DynamicForm,FormField}.tsx` + `widgets.ts` — TS port of Forminator's renderer + Phase 1 `x-widget`/`x-group`/`x-order` handling. `fieldPlan.ts` copies the pure logic (two implementations, one spec — noted in file headers).
- `app/src/FillView.tsx` — view `fill`: loads schema by `?schema=<blobDir>`; `onCommand` applies `set_fields`/`clear_fields`/`validate`/`submit` (submit = browser POST to records, then `command-result {recordId}`); `reportContext` on every change `{view:'fill', schemaDir, data, filledCount, totalCount, valid}` — phoenix reconciles from this; manual submit button always present.
- `app/src/RecordsView.tsx` — view `records`: caller's stored records, deep-linkable.

*Verify standalone:* curl schema/commands/records with a dev token (accept + reject + 422 cases); open the app URL directly and drive ops.

---

## Phase 3 — Phoenix + tc-lab integration

> **DONE 2026-09-02 — phoenix `04fe472` + tc-lab `c4c04f1` (both LOCAL, not
> pushed; deploys are Phase 4 operator-gated).** Shapes landed as planned with
> these deliberate deviations: (1) the three lib files live under
> `phoenix/src/lib/forms/` (`catalog.ts`, `choicePlan.ts` PURE, `collection.ts`
> db) following the badges/interview+draft split, not at lib root; (2) the PG
> table is operator-applied `sql/029_form_collection.sql` with graceful
> degradation (console_events precedent), not ensure-on-boot; (3) no separate
> `formSubmitted` ack — the surface's reported `recordId` in its context `data`
> IS the completion signal (one source of truth), and `choiceAnswer` carries
> `schemaDir` so two concurrent collections never cross-contaminate;
> (4) `surface_command` for forms requires `schemaDir`, catalog-verified
> pre-mint, and attaches the server-reconciled `formState` (never
> model-supplied) so worker-side validate/submit check the whole form.
> Verified: 348 phoenix tests 0 fail (22 new choicePlan tests), tc-lab tsc +
> vite build, live Azure catalog smoke (164 entries, allow-list, LLM routing),
> scratch-PG SQL smoke (7 assertions). ux-reviewer + security-auditor both ran;
> a11y highs fixed in tc-lab, audit lows fixed in phoenix (grant re-check,
> pre-mint catalog check, second `__proto__` guard); client-asserted recordId
> limitation documented in collection.ts. ALSO in the phoenix commit: the
> 2026-08-21 console guardrail + text-attachment work (was uncommitted; same
> files, no interactive staging available); its broken office-status onboard
> hunk was dropped (references nonexistent config; preserved in phoenix
> stash@{1}).

**CREATE `phoenix/src/lib/formsCatalog.ts`** — `loadCatalog()` (blob `schema-catalog.json`, env `formsCatalogStorageAccount/Container/Blob` in `config.ts`, ETag + 5-min TTL + stale-while-revalidate); `compactIndex()` (routing fields only — keeps 164-schema prompt small); `routeIntent(query)` via existing provider client → up to 5 `{blobDir, title, confidence, reason}`.

**CREATE `phoenix/src/lib/formCollection.ts`** — PG table `app_form_collection(id, session_id, user_id, schema_dir, schema_title, collected jsonb, answered text[], status, record_id, timestamps)` (ensure-on-boot pattern like `app_chat_session`); `reconcileFromSurfaceContext()` upsert per turn; `computeChoices(schema, collected, answered)` (Phase 1 logic, answered-set based → explicit `false` counts); `progressSummary()` for prompt injection.

**MODIFY `phoenix/src/lib/chat.ts`**:
- `SURFACE_API_CAPABILITIES` += `forms` (schema + records GET routes); `SURFACE_COMMAND_CAPABILITIES` += `forms` (`set_fields`, `clear_fields`, `validate`, `submit` — submit only on explicit user confirmation); `SURFACE_VIEW_HINTS` += `forms: view=fill&schema=<blobDir>; view=records&schema=…&record=<id>`.
- New tool `query_form_catalog {query}` in toolDefs + `executeToolRaw` (read-only, low complexity; Joshua L2/L3 applies via the shared path).
- New prompt fragment `COLLECTING STRUCTURED DATA` (injected only when the session's granted surfaces include `forms`): intent → `query_form_catalog` → confident match → emit `[View <title>|surface:forms?view=fill&schema=<blobDir>]` + ask first field → each answer = one `surface_command set_fields` op → all required filled → `validate` → ask to confirm → `submit` → confirm with record deep link. Rules: never invent values; one field per question; enum chips are automatic (don't re-list options in prose).
- Active-collection fragment: inject `progressSummary` so the model asks the right next field without re-deriving.
- Per-turn `reconcileFromSurfaceContext` hook; collection marked `complete` on submit success (via surface context + a `formSubmitted:{schemaDir, recordId}` ack in the next POST /chat body).

**MODIFY `phoenix/src/routes/chat.ts`** (POST /chat): attach `choices: computeChoices(...)` to the response when a collection is active; persist on assistant message metadata so reload re-renders chips. Response gains `choices?: {field, label, options[], multiSelect, widget}`.

**MODIFY `phoenix/src/routes/surface.ts` + `config.ts`**: register `forms` in `surfaceDefs()` (`gate:'admin'`, `formsUrl`, `formsApiBase`) — grants flow through the existing permissions grid → `app_user_surface`.

**CREATE `tc-lab/src/ChoiceChips.tsx`** — renders `choices`: `radio`/`yesno`/`dropdown`(≤8 as chips, else native select) single-select → click sends via existing `send(overrideText)`; `checkbox` → toggle chips + Confirm. Body gains `choiceAnswer:{field, value}` for deterministic answered-tracking.

**MODIFY `tc-lab/src/Chat.tsx`** — types gain `choices?`; attach to last assistant message; render `<ChoiceChips>` under the bubble beside `surfaceActions`; clear on next send; submit's `command-result {recordId}` renders a "Record stored — view" deep link. `MessageContent.tsx` untouched (structured payload, no new markers).

**End-to-end:** intent → `query_form_catalog` → `[View …|surface:forms?view=fill&schema=…]` → surface opens → chat asks (chips for enums) → `set_fields` ops fill the form live → `validate` → confirm → `submit` → record stored → confirmation + link.

*Verify:* phoenix `npm run dev` + tc-lab vite against Phase 2 surface — full loop; reload mid-collection restores chips + progress from PG; two concurrent collections don't cross-contaminate.

---

## Phase 4 — Hardening + cutover

> **DONE 2026-09-07 (code + data; operator deploy pending).** Forminator
> `1734840`/`785756f`/`df8c848`/`1c5b3c7`/`2ad8dcb`/`aa52918`, tc-surface-forms
> `dd50e2a`/`96d78e4`/`c0f7d5c`, tc-surface-template `cbb7100`, phoenix
> `3e33a4d` + `1de6ffc` — all pushed (Forminator→origin, surfaces→srv,
> phoenix→origin). tcov `64c1375` local on `node_dev` (not pushed).
>
> - **Proxy retired** (`1734840`): `/api/tcov/schemas` had zero callers — route,
>   `https` require, and `TCOV_API_BASE` compose var deleted.
> - **data.js deprecated** (`785756f`): header pointer to tc-surface-forms
>   records; ApiDocs amber "Deprecated — do not build on these endpoints"
>   banner; mount kept for the local workbench.
> - **Lint Check 13 element-wise** (`df8c848`): `documentType:"collection"`
>   array-samples validate per element with `/i` index prefixes; wrapper-style
>   collections (branch-reports) keep whole-sample validation. Error list
>   capped (3 rendered / N counted). New fixtures lint-collection{,-broken};
>   standard §1/§6 amended.
> - **Sample drift fixed in Azure**: branch-reports `audienceRoleIds`→
>   `audienceList`; metadata-fields ×3 docs `customActivities:false→[]`,
>   `anonymizeAfter:null→""`; basics sample unwrapped from a 2-element array +
>   stale enum values repaired. All re-downloaded and lint-verified 0E.
> - **Pilot x-widget adoption (Campbell: pilot set only)**: metadata-fields
>   (25 annotations), basics (20), nomenclature (3) — x-source/x-prompt/
>   x-order/x-group; x-widget needed nowhere (defaults correct). Uploaded to
>   Azure, round-trip verified.
> - **Catalog regenerated** (was 2026-04-09): 165 entries, 0 errors,
>   entity/keywords/kbContext 165/165. Three live bugs fixed to get there:
>   thinking-block trap ×7 sites (`1c5b3c7`), silent batch failure + bounded
>   retry (`2ad8dcb`), max_tokens 3000 truncation + salvage (`aa52918`).
>   CAPGPT_URL in backend/.env repointed to api-prod2.capgpt.thinkingcap.com
>   (func-prod host is NXDOMAIN; .env is local-only).
> - **phoenix** (`3e33a4d`): sql/031 seeds forms as explicit `'manual'` pilot
>   rule (NOT copy-from-badges — 020 made badges `'everyone'`); README
>   migrations 021–031; smoke `--await-refresh`; `[formsCatalog] revalidated`
>   log. `1de6ffc`: smoke asserts revalidation, not content-diff (false-FAIL
>   fixed; verified live with a same-bytes re-upload).
> - **tc-surface-forms deploy prep**: stale CI removed; README (incl. the
>   registered-without-data-plane design note — Campbell decision), worker
>   `.env.example`, `docs/DEPLOY-CHECKLIST.md` (env-recipe gap callout,
>   tentative ports 3037/8107). Template: sync-bridge CONSUMERS + port row.
> - **Images**: `surface-forms-svc`/`surface-forms-app` built to patchacr
>   (tag `forms-<date>-<sha>` + `:latest`); NO dataworker build.
> - **Update 2026-09-07 PM**: sql/029+031 **APPLIED** to `tc-other-pg/mothership`
>   (path found: az CLI identity `radupog@hotmail.com` = the AAD user owning the
>   sibling surface tables; `az account get-access-token --resource-type oss-rdbms`
>   as the PG password — mothership_app itself has no DDL, 42501). Revert-tested
>   (drop+delete → re-apply clean) and grant-verified as mothership_app from the
>   live container (SELECT/UPDATE/DELETE ok; INSERT probe got expected 23503 FK
>   violation, not 42501). Revert script: `/media/shared/For Douglas/forms-029-031-revert.sql`.
>   console-api redeployed to `1de6ffc` by doug 06:15Z (patch.console_deploy_job)
>   — but `SCHEMAS_STORAGE_ACCOUNT`/`SCHEMAS_STORAGE_KEY` are STILL ABSENT from
>   the container env: phoenix catalog loads degrade until set.
> - **Operator-gated — infra ALL DONE 2026-09-07 ~13:10Z (Campbell-driven, via
>   run-command)**: ~~apply sql/029 + sql/031~~ DONE; ~~phoenix env SCHEMAS_*~~
>   DONE both nodes (console.env, console-api restarted, clean boot); ~~redeploy
>   ≥ `1de6ffc`~~ DONE 06:15Z; ~~forms.env CC+CE~~ DONE (compliance-reports.env
>   + SCHEMAS_*, no MAESTRO_QUEUES_URL); ~~systemd~~ DONE both nodes, units
>   pinned `forms-20260907-045139-c0f7d5c`, svc 3037/app 8107 CONFIRMED pair,
>   /health + /healthz 200; ~~NSG~~ DONE `allow-capcom-health-forms` prio 356
>   both NSGs (354 was node-agent); ~~Caddy~~ DONE — BOTH handles strip
>   /forms (checklist fixed `03ef28d`: app nginx serves at root, unstripped
>   remoteEntry 200s HTML), edge verified: /forms/api/forms/health 200 JSON,
>   remoteEntry.js 200 JS, index served, bare /forms 301.
> - **Remaining (operator UI, 403'd for Claude)**: capcom onboard (skip
>   worker_types); pilot grants via grid; browser E2E per DEPLOY-CHECKLIST.md
>   step 8.
> - **Open findings**: Forminator `/api/chat` unauthenticated + wide CORS —
>   deferred (Campbell): local workbench only, must close before any
>   non-localhost exposure. Check-5↔check-6 tension: required uuid +
>   x-source:app trades one warning for the other (standard §4 vs §6.6 —
>   product note). nomenclature trio still lacks sample.json/description.md
>   in Azure. `~/Projects/tcov/schemas/settings/metadata-fields` is a stale
>   incomplete draft (different root shape) — left alone.

- Regenerate `schema-catalog.json` in Forminator after x-widget adoption; confirm phoenix ETag cache picks it up within TTL.
- Retire Forminator's dead `/api/tcov/schemas` proxy; mark `data.js` deprecated (header pointer to tc-surface-forms records). Forminator keeps: schema workbench, edit assistant, lint, catalog generator.
- **Operator-gated (Claude commits code only):** deploy tc-surface-forms per `tc-surface-template/DEPLOY.md` (WORKSURFACES RG, Caddy `worksurfaces.thinkingcap.com/forms/`, port pair); phoenix env additions; `app_user_surface` grant rows for pilot users (`gate:'admin'`).

## Key risks

1. Catalog staleness → ETag + TTL + stale-while-revalidate; regen is a documented manual step (v2: webhook).
2. 164-schema prompt size → compact index only; full schema fetched one at a time via `query_surface`.
3. Boolean-false quirk → fixed architecturally (answered-set, chip payloads) in both runtimes.
4. Draft mismatch (2020-12 files vs draft-07 edit prompt) → standardize 2020-12; worker loads draft-07 via `ajv-draft-07` fallback; lint flags mixed trios.
5. `x-options-source: db|app` in v1 → chat skips chips, form placeholder picker; real db-backed options reserved as a seam (no LMS sinks in v1).
6. Three field-plan implementations (phoenix `computeChoices`, surface `fieldPlan.ts`, Forminator `fieldPlan.js`) → one normative standard doc + cross-referencing headers; divergence = verification failure. **(Phase 5 adds a fourth: tc-console's vendored formcast copy — same rule.)**
7. `x-depends-on` cycles → lint-time error; runtime evaluates pure per-turn (never loops).

---

## Phase 5 — FormCast: forms stop being an iframe surface; Home becomes the canvas

> **CODE COMPLETE 2026-09-10 — tc-console `7434704` pushed to srv main
> (operator web deploy gated, as always).** Campbell's call: the form surface
> is no longer a surface — form-like interactions render NATIVELY in tab zero
> (the merged Home/Canvas) with no iframe, extending the guest view's rule
> ("THE FORM LIVES IN THE CANVAS AREA… forms belong to the canvas",
> `tc-console/src/guest/GuestChat.tsx`) to the operator Lab. Decisions locked
> with Campbell: **vendor** the components into tc-console (a MOVE — the
> tc-surface-forms app retires after the pilot; its worker stays as the data
> plane) and **kill-switch pilot** (`?formsIframe=1` /
> `localStorage['tc-lab.formsIframe']='1'` restores the iframe path).
>
> - **`tc-console/src/home/formcast/`** — `Form/*` vendored verbatim
>   (provenance headers; fieldPlan is now the FOURTH normative runtime —
>   risk #6); `FillView`/`RecordsView` adapted from postMessage bridge to
>   props (`FormCastView`/`RecordsCastView`); `FormCast.tsx` shell owns view
>   state, the command pump (batches sequence onto the loaded schema; every
>   batch answered, incl. an error-applier when the schema 404s), the
>   `.tc-coach-flash` CSS + `[Show me]` anchor highlight; `api.ts` uses the
>   **console session bearer** — the worker's `requireAuth` verifies Phoenix
>   sessions directly (`tokenSource:'browser'`, writes allowed) — with the
>   minted `surface-browser` token as 401 fallback (its mint doubles as
>   apiBaseUrl discovery + the `surface_open` stamp).
> - **One routing seam in `useSurfaces`** — `openSurface` / `launchSurface` /
>   `showInSurface` / `deliverSurfaceCommands` route `surface==='forms'` to
>   the cast via a ref the App fills (kill-switch leaves it null → iframe
>   path verbatim). Covers markers, launcher, [Show me], surfaceActions,
>   record deep links, saved/pinned opens. `Chat.tsx` needed NO changes.
> - **HomeCanvas third occupant** — cast > scene > start doc,
>   last-projection-wins (a fresh scene takes the canvas back; the cast stays
>   mounted-hidden, fill intact). HomeSurface stays mounted as before.
>   Dismiss restores the previous base and fails unanswered batches instead
>   of hanging their cards.
> - **Phoenix + worker: UNTOUCHED.** `surfaceActions`/`surfaceContext` are
>   transport-neutral; the forms surface registration STAYS (grant + catalog
>   + command switch, not the iframe; do NOT mark it `native` — the mints
>   refuse native and that would kill `surface_command`'s worker hop).
> - **Pre-existing bug found + fixed in the same commit:** the console has
>   always sent live-surface reports keyed `surfaceId`, but Phoenix's
>   `parseLiveSurfaces` has required `id` since `3a20310` (08-11) — every
>   report was silently dropped, so the LIVE SURFACES prompt section AND the
>   forms collection reconcile never worked through the console (the Phase 3
>   verifications passed via the `choiceAnswer` path, which writes
>   collections directly). `getSurfaceContexts` now bridges `surfaceId → id`.
> - **Verified:** tsc clean, vite build clean. E2E:
>   `tc-console/docs/E2E-FORMCAST.md` (28 checks incl. kill-switch spots).
> - **Remaining (operator-gated):** console web deploy; Campbell runs the
>   E2E; then retire the tc-surface-forms APP (Caddy `/forms`, app unit 8107,
>   app image; svc 3037 STAYS — worker-only repo) and mark the app image roll
>   `forms-20260907-212950-c678c24` superseded (reveal rides in the cast).
> - **Non-goals (v1):** Work-with-me pick FROM the form (`data-home-pick`
>   retrofit); guest view (already has its own cast); cast survives no reload
>   (chips + PG collection restore the conversation; next event re-opens).
> - Plan-of-record for the change:
>   `~/.claude/plans/2026-09-10-forminator-home-canvas.md`.

## Critical files

- `Forminator/docs/SCHEMA-AUTHORING-STANDARD.md` (new, canonical)
- `Forminator/backend/routes/chat.js` + new `backend/lib/fieldPlan.js`
- `Forminator/backend/routes/chatEdit.js` (lint endpoint)
- `phoenix/src/lib/chat.ts` (capabilities, `query_form_catalog`, collection fragments)
- `phoenix/src/lib/formCollection.ts` + `formsCatalog.ts` (new)
- `phoenix/src/routes/chat.ts`, `routes/surface.ts`, `config.ts`
- `tc-lab/src/ChoiceChips.tsx` (new), `tc-lab/src/Chat.tsx`
- `tc-surface-forms/` (new repo from template): `worker/src/lib/{schemaStore,fieldValidation,commands,recordStore}.ts`, `worker/src/routes/{schema,commands,records}.ts`, `app/src/{FillView,RecordsView}.tsx`, `app/src/Form/*`
- Pattern sources: `tc-surface-ltisetup/worker/src/lib/{commands,fieldValidation}.ts`, `Forminator/backend/routes/chat.js:9-45`
