# Schema Authoring Standard

**Canonical, normative.** This is the single source of truth for how JSON Schemas are
authored so that an AI assistant can collect their data conversationally and a form
surface can project them to the right widgets. It lives in the Forminator repo;
TCOV, phoenix, and the work-surface fleet **reference it, never copy it**.

Three runtimes implement against this standard — Forminator's chat
(`backend/lib/fieldPlan.js`), the tc-surface-forms renderer (`fieldPlan.ts`), and
phoenix's collector (`computeChoices`). They are independent implementations of the
rules in §5. Divergence from this document is a verification failure, not a design
choice. Lint (`POST /api/schema/lint`) enforces §6 mechanically.

---

## 1. The trio convention

Every leaf schema directory contains exactly three companions:

| File | Role |
|---|---|
| `schema.json` | The JSON Schema definition. |
| `sample.json` | A sample record that **must AJV-validate** against `schema.json`. For `documentType: "collection"`, either an array of documents (each element validates) or a wrapper object holding the array. |
| `description.md` | Human-readable explanation: what the document collects, where it is administered, where it is stored. |

Authoring rules carried over from TCOV:

- **Breadcrumb `title`.** The root `title` reads as a breadcrumb where one exists
  (`"Settings > Basics"`, `"Security > SSO > Providers"`). Plain nouns are fine for
  activity documents (`"Assignment"`).
- **Dense per-field `description`.** Every property carries a description written for
  two readers: the AI phrasing a question, and the engineer tracing provenance.
  Prefix provenance inline: `DB:` (table.column), `UI:` (control name),
  `Maps to` (XML key or settings path).
- **Root `description`.** Its first sentence states what the document collects.
  Everything after is scope, storage, and caveats.
- **Provenance-only keywords.** `documentType`, `scope`, `rootBranchOnly`,
  `adminUrl`, and `x-thinkingcap` describe where a settings document lives and who
  may edit it. Runtimes **must ignore them** — they never influence inclusion,
  ordering, or widget choice. They are authoring metadata, not runtime vocabulary.
- **Runtime exception — the `settings/*` dir convention.** A schema whose blob
  dir sits under `settings/` IS runtime-relevant: it is *domain-scoped*.
  Runtimes gate the form on a domain (LMS branch) pick before rendering, load
  the picked branch's current values, and stamp submitted records with the
  domain. Schemas carry no marker for this — the convention is the dir.

## 2. Schema draft

Declare **JSON Schema 2020-12** on every new or updated schema:

```json
"$schema": "https://json-schema.org/draft/2020-12/schema"
```

Draft-07 files exist in the fleet and remain valid inputs: runtimes load them via an
`ajv-draft-07` fallback. Lint warns on any declaration other than 2020-12 so the
fleet converges. One file, one draft — never mix meta-schemas across a trio.

## 3. Built-in properties that carry runtime meaning

`title`, `description`, `examples`, `default`, `enum`, `const`, `format`,
`minimum`/`maximum`, `minLength`/`maxLength`, `pattern`, `required`,
`dependentRequired`, `if`/`then`/`else`, `readOnly` — used per the JSON Schema
spec, with the runtime interpretations in §5. `const` and `format: "uuid"` mark
auto-assigned fields; `readOnly` marks display-only fields.

## 4. The `x-` vocabulary (normative)

Extension properties are how an author talks to the runtime. All are optional;
defaults are chosen so an un-annotated schema still collects sensibly.

| Property | Values | Default | Rule |
|---|---|---|---|
| `x-prompt` | string | — | The exact question the chat asks, verbatim. Set it whenever the key name alone would be ambiguous (`type`, `value`, `status`…). |
| `x-source` | `user` \| `app` \| `db` \| `system` | `user` | Where the value originates. `app` = injected from session/auth context; `db` = pre-existing persisted value; `system` = auto-generated. |
| `x-options-source` | `static` \| `db` \| `app` | `static` | Where a choice field's options come from. `db`/`app` fields **omit `enum`** and carry `x-options-preview` instead. |
| `x-options-preview` | array of values | — | Representative options for `db`/`app` sources, for dev/test rendering only. Never used for validation. Must be an array. |
| `x-order` | number | — | Collection order, ascending. **All-or-nothing** among sibling properties. |
| `x-group` | string | — | Section label. Siblings sharing a value render under one group header. |
| `x-hint` | string | — | Extra guidance woven naturally into the question. |
| `x-depends-on` | object (see below) | — | Simple conditional visibility, evaluated per turn. |
| `x-widget` | `radio` \| `dropdown` \| `checkbox` \| `yesno` | derived | Explicit widget override. See the defaults table and validity matrix below. |

### `x-depends-on`

Canonical form is an object with a `field` (a sibling key) and exactly one operator:

```json
{ "field": "tripType", "equals": "round-trip" }
{ "field": "tags", "in": "pro" }
{ "field": "hasPet", "truthy": true }
```

- `equals` — the field's value equals the given value.
- `in` — the field's array value contains the given value.
- `truthy` — the field has any truthy value.

Legacy string shorthand `"x-depends-on": "tripType=round-trip"` is accepted and
means `equals`; a bare field name means `truthy`. Lint suggests migrating to the
object form. Cycles in `x-depends-on` graphs are a lint **error**.

### `x-widget`

| Field shape | Default widget |
|---|---|
| `enum` with ≤ 5 values | `radio` |
| `enum` with > 5 values | `dropdown` |
| `array` + `items.enum` | `checkbox` |
| `boolean` | `yesno` |

Explicit overrides must match the field shape — invalid combos are lint **errors**:

| `x-widget` | Valid on |
|---|---|
| `radio` | `enum` field (or `x-options-source: db\|app`), not an array |
| `dropdown` | same as `radio` |
| `checkbox` | `array` with `items.enum` |
| `yesno` | `boolean` only |

`radio` with more than 5 options lints a warning — allowed, but you probably want
`dropdown`.

## 5. Runtime precedence rules (the contract)

Runtimes implement these exactly, in this order of evaluation per turn.

### 5.1 Inclusion — which fields are asked at all

1. `const`, `format: "uuid"`, `readOnly: true`, or `x-source: app | system` →
   **never asked** (auto-assigned; the form may show them disabled).
2. `x-source: db` → **never asked in chat**; shown **read-only** in the form.
3. No `x-source` → `user`: asked.

### 5.2 Ordering — which field is next

1. Fields with `x-order`, ascending.
2. Remaining **required** fields, in declaration order.
3. Remaining optional fields, in declaration order.

When no property anywhere declares `x-order`, this reduces exactly to the legacy
required-first behaviour.

### 5.3 Question text — what the chat says

1. `x-prompt` verbatim, if present.
2. Otherwise the first sentence of `description`.
3. `x-hint` is woven in naturally alongside either.

### 5.4 Conditionals — what gets skipped this turn

- `x-depends-on` is evaluated against currently collected data **every turn**;
  unsatisfied → the field is skipped (and any collected value is ignored at
  validate time).
- `if`/`then`/`else` is honoured for **requiredness**: a field made required by an
  active `then` is asked; one excluded by an active `else` is skipped.
- Evaluation is pure and per-turn — it never loops. Author-time cycles are caught
  by lint.

### 5.5 Choices — what the chips show

- `x-options-source: static` or absent → chips are built from `enum` /
  `items.enum` (or Yes/No for booleans).
- `x-options-source: db | app` → **no chips**. The chat says "pick it in the form";
  the form renders a placeholder picker in v1, populated from `x-options-preview`
  in dev/test only.
- Enum chips are rendered by the client from a structured payload — the chat model
  must not re-list options in prose.

### 5.6 Answered state — what counts as filled

A field counts as answered when its key is in the explicit **answered set**
(maintained from chip clicks / confirmed field updates), or when it holds a
non-empty value (`null`, `undefined`, `''` are empty). An explicit `false` **is** an
answer — boolean fields must never be re-asked because their value is `false`.
Auto-assigned fields (§5.1 rule 1) are always treated as answered.

## 6. Lint checklist

`POST /api/schema/lint` checks all of the following deterministically (no LLM).
Severity: **error** = will misbehave at runtime; **warning** = standard violation
that still runs; **info** = suggestion.

| # | Check | Severity |
|---|---|---|
| 1 | Every property (all nesting levels) has a `description` | warning |
| 2 | String `enum` (≤ 10 values): every value's semantics appear in the description | warning |
| 3 | Root `description` present; first sentence states what is collected (≥ 60 chars) | warning |
| 4 | `x-source` has a valid value | error |
| 5 | Auto-assigned shapes (`const`, `format: uuid`, `readOnly`) declare `x-source` | warning |
| 6 | Required field carries `x-source: app \| db \| system` (un-askable requirement) | warning |
| 7 | `x-order` is all-or-nothing per sibling level; numeric | warning / error |
| 8 | `x-prompt` present on generic keys (`type`, `name`, `value`, `data`, `mode`, `kind`, `status`, `settings`, `options`, `config`, `text`, `other`) | info |
| 9 | `x-depends-on` targets exist among siblings; graph is acyclic | error |
| 10 | `x-widget` value valid and matches the field shape (§4 matrix); `radio` > 5 options | error / warning |
| 11 | `x-options-source: db \| app` omits `enum`/`items.enum` and carries an array `x-options-preview` | error / warning |
| 12 | Trio complete: `sample.json` + `description.md` supplied alongside the schema | info |
| 13 | `sample.json` AJV-validates against the schema (draft resolved from `$schema`). Collection arrays validate element-wise; findings carry the element index (`/1/…`) | error |
| 14 | `$schema` present and declares 2020-12 (draft-07 tolerated, flagged) | warning |

**Score:** `100 − 15×errors − 5×warnings − 1×infos`, clamped to `[0, 100]`.
Deterministic: same trio in, same score out.

## 7. Worked example

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Flight Booking Request",
  "description": "Collects the information needed to search for a flight. One-way or round-trip, airports, dates, and passengers.",
  "type": "object",
  "required": ["tripType", "origin", "destination", "departureDate", "passengers"],
  "properties": {
    "bookingId": {
      "type": "string",
      "format": "uuid",
      "x-source": "system",
      "description": "Server-assigned booking reference. Never asked."
    },
    "tripType": {
      "type": "string",
      "description": "Whether the user is booking a one-way or round-trip flight. 'one-way' = no return leg; 'round-trip' = return leg required.",
      "enum": ["one-way", "round-trip"],
      "x-prompt": "Is this a one-way or round-trip flight?",
      "x-order": 1
    },
    "origin": {
      "type": "string",
      "description": "IATA code or city name for the departure location.",
      "examples": ["YYZ", "Toronto", "JFK"],
      "x-prompt": "Where are you flying from?",
      "x-hint": "You can use a city name or airport code (e.g. Toronto or YYZ)",
      "x-order": 2
    },
    "seatClass": {
      "type": "string",
      "description": "Cabin class. 'economy' = standard; 'premium' = extra legroom; 'business' = lie-flat.",
      "enum": ["economy", "premium", "business"],
      "x-widget": "radio",
      "x-order": 3
    },
    "returnDate": {
      "type": "string",
      "format": "date",
      "description": "The date the user returns. Only applies to round-trip bookings.",
      "x-prompt": "What date will you be returning?",
      "x-depends-on": { "field": "tripType", "equals": "round-trip" },
      "x-order": 4
    },
    "homeAirport": {
      "type": "string",
      "description": "The user's preferred departure airport, pre-filled from their profile.",
      "x-source": "db"
    }
  }
}
```

Note what is *not* here: `x-source: "db"` on free-text questions the user must
answer (a classic misuse — it would silence the chat), and `x-options-source`
holding an array of values (that is what `x-options-preview` is for).

## 8. Legacy compatibility

- Un-annotated TCOV trios collect fine: defaults cover inclusion (uuid/const
  skipped), ordering (required first), and question text (first sentence of the
  description). Lint emits gentle warnings nudging them toward the vocabulary.
- String-form `x-depends-on` keeps working; migrate to object form when touching
  the file.
- Draft-07 files validate via the runtime fallback; redeclare 2020-12 when
  touching the file.
