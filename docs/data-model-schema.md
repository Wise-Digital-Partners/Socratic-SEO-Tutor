# Artifact: Data Model / Schema

**Derived from:** PRD-guided-learning-app.md (§6), system-prompt-socratic-tutor.md (§1, §2 context variables)
**Purpose:** Defines the persisted entities, their fields, relationships, and enums needed to implement the app's state. This is the single source of truth for schema — the API Contract and UI Component Spec artifacts should reference these entities rather than redefine fields.
**Audience:** Claude Code (implementation), backend/database design.
**Format note:** Field types are given in a storage-agnostic form (works for a relational DB with JSON columns, a document DB, or an in-memory store). IDs are assumed to be strings (UUIDs) unless noted.

---

## 1. Entity-Relationship Overview

```
User (1) ──< (many) LearningPlan
LearningPlan (1) ──< (many, ordered) SubTopic
LearningPlan (1) ──< (many, append-only) LessonTurnMessage
SubTopic (1) ──< (many) LessonTurnMessage
SubTopic (1) ──< (many) ComprehensionCheckAttempt
SubTopic (1) ──< (many) AdaptiveFeedbackEvent
LearningPlan (1) ──1 LessonSessionState   // current position/state within this plan
User (1) ──1 UserProgressSummary          // aggregate, recomputed or incrementally updated
```

- A `LearningPlan` belongs to one `User` and contains an ordered list of `SubTopic` entries.
- `LessonSessionState` tracks *where* the user currently is within one `LearningPlan` (one active session state per plan; a user can have multiple plans, each with its own paused state, enabling multi-subject resume).
- `LessonTurnMessage` is the append-only, persisted transcript of every tutor/user message exchanged during exploratory dialogue — this backs both mid-session refresh recovery and post-completion review (see §2.4 and PRD/UI updates).
- `ComprehensionCheckAttempt` and `AdaptiveFeedbackEvent` are append-only logs scoped to a `SubTopic`, used for both gating progression and later analysis/tutor-quality evaluation.
- `UserProgressSummary` is a derived/aggregate entity — implementation may compute it on read or maintain it incrementally; schema is defined either way for API/UI consistency.

---

## 2. Entities

### 2.1 `User`

Minimal in v1 — no auth/profile detail is specified in the PRD beyond identity needed to scope plans.

```json
{
  "user_id": "usr_8f3a1c",
  "display_name": "Jordan",
  "created_at": "2026-08-01T14:22:00Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `user_id` | string (UUID) | yes | Primary key |
| `display_name` | string | no | Optional; assumption per PRD Open Questions — auth model undetermined |
| `created_at` | ISO 8601 datetime | yes | |

---

### 2.2 `LearningPlan`

Represents one subject's full sequenced curriculum, per PRD §3.2 (AC-4, AC-5, AC-6).

```json
{
  "plan_id": "plan_1a2b3c",
  "user_id": "usr_8f3a1c",
  "subject": "Photosynthesis",
  "status": "in_progress",
  "subtopic_order": ["sub_01", "sub_02", "sub_03"],
  "created_at": "2026-08-20T09:00:00Z",
  "confirmed_at": "2026-08-20T09:03:12Z",
  "completed_at": null
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `plan_id` | string (UUID) | yes | Primary key |
| `user_id` | string (UUID) | yes | Foreign key → User |
| `subject` | string | yes | Raw user-entered subject text |
| `status` | enum | yes | `draft` \| `confirmed` \| `in_progress` \| `completed` \| `abandoned` |
| `subtopic_order` | array of string (SubTopic IDs) | yes | Defines sequence; source of truth for ordering (AC-6 allows reordering pre-confirmation — reordering this array is how that's implemented) |
| `created_at` | ISO 8601 datetime | yes | When the plan was generated |
| `confirmed_at` | ISO 8601 datetime \| null | no | Set when user confirms plan (AC-5); null while `status = draft` |
| `completed_at` | ISO 8601 datetime \| null | no | Set when all sub-topics reach `passed` |

**Status enum notes:**
- `draft` — generated, shown to user, awaiting confirmation/edits (AC-5).
- `confirmed` — user approved; lessons have not yet started.
- `in_progress` — at least one sub-topic lesson has begun.
- `completed` — all sub-topics `passed`.
- `abandoned` — optional/future use if a user explicitly discards a plan; not required for v1 but reserved to avoid a later breaking migration.

---

### 2.3 `SubTopic`

One sequenced unit within a `LearningPlan`, per PRD §3.2 (AC-6) and §3.3.

```json
{
  "subtopic_id": "sub_02",
  "plan_id": "plan_1a2b3c",
  "title": "The Light-Dependent Reactions",
  "description": "How sunlight is converted into chemical energy (ATP and NADPH).",
  "ordering_rationale": "Builds on sub_01's overview of inputs/outputs before detailing the mechanism.",
  "position": 2,
  "status": "not_started",
  "attempt_count": 0,
  "started_at": null,
  "passed_at": null
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `subtopic_id` | string (UUID) | yes | Primary key |
| `plan_id` | string (UUID) | yes | Foreign key → LearningPlan |
| `title` | string | yes | Per AC-6 |
| `description` | string | yes | One-sentence description, per AC-6 |
| `ordering_rationale` | string | no | Per AC-6 ("builds on sub-topic 2", etc.) — useful for UI tooltip and for plan-edit validation |
| `position` | integer | yes | Redundant with `LearningPlan.subtopic_order` index but kept for query convenience; must stay in sync |
| `status` | enum | yes | `not_started` \| `in_progress` \| `passed` \| `failed_retry` |
| `attempt_count` | integer | yes | Number of comprehension-check attempts made for this sub-topic's current check cycle; resets to 0 on `failed_retry` → re-teach → new check (AC-9) |
| `started_at` | ISO 8601 datetime \| null | no | |
| `passed_at` | ISO 8601 datetime \| null | no | |

**Status enum notes:**
- `failed_retry` — exceeded max attempts (default 3, PRD Assumption 4) on the current check; system has issued a simplified re-teach and a new check is pending. This is a transient state that returns to `in_progress` once the re-teach begins, not a terminal failure state (PRD explicitly requires no dead ends — AC-9).

---

### 2.4 `LessonSessionState`

Tracks exactly where a user is within one `LearningPlan`, enabling resume (PRD §5, US-6) and providing the live context variables the system prompt needs each turn (see system-prompt-socratic-tutor.md §2).

```json
{
  "session_state_id": "lss_5d6e7f",
  "plan_id": "plan_1a2b3c",
  "current_subtopic_id": "sub_02",
  "dialogue_state": "giving_hint",
  "hint_level": 2,
  "reframe_count": 0,
  "reveal_requests_this_question": 0,
  "last_updated_at": "2026-08-21T11:47:03Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `session_state_id` | string (UUID) | yes | Primary key |
| `plan_id` | string (UUID) | yes | Foreign key → LearningPlan (1:1) |
| `current_subtopic_id` | string (UUID) | yes | Foreign key → SubTopic |
| `dialogue_state` | enum | yes | `posing_question` \| `awaiting_response` \| `evaluating_response` \| `giving_hint` \| `confirming_understanding` \| `advancing` — matches system-prompt-socratic-tutor.md §1.1 |
| `hint_level` | integer (0-3) | yes | Per system-prompt §1.3; resets to 0 at the start of each new comprehension-check question (see system-prompt Assumption 2 — confirm before finalizing) |
| `reframe_count` | integer | yes | Confusion-triggered re-explanations given for the current concept; tracked separately from `hint_level` |
| `reveal_requests_this_question` | integer | yes | Resets to 0 each time a new question (exploratory or check) is posed; drives reveal logic (AC-2) |
| `last_updated_at` | ISO 8601 datetime | yes | For staleness checks / abandoned-session detection |

---

### 2.5 `LessonTurnMessage`

Append-only, persisted transcript of every message exchanged during exploratory dialogue — tutor and user alike. Added to support (a) resuming a session with full visible history intact after a refresh/new device, and (b) post-completion review of a finished subject.

```json
{
  "message_id": "ltm_4f5e6d",
  "plan_id": "plan_1a2b3c",
  "subtopic_id": "sub_02",
  "role": "tutor",
  "content": "Think about which part of the chlorophyll molecule interacts with photons — what happens to its electrons?",
  "classification": null,
  "occurred_at": "2026-08-21T11:47:03Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `message_id` | string (UUID) | yes | Primary key |
| `plan_id` | string (UUID) | yes | Foreign key → LearningPlan; primary scope for fetching a full-session transcript across sub-topics |
| `subtopic_id` | string (UUID) | yes | Foreign key → SubTopic; allows fetching a transcript scoped to just one sub-topic (used by review mode, see UI spec) |
| `role` | enum | yes | `tutor` \| `user` |
| `content` | string | yes | The literal message text (tutor's question/hint/reframe, or the user's raw reply) |
| `classification` | enum \| null | no | For `role = user` messages only: `correct` \| `error` \| `confusion` \| `ambiguous` \| `reveal_request`, per system-prompt-socratic-tutor.md §1.2. Always `null` for `role = tutor` messages. Stored for analysis/tutor-quality evaluation; **not intended for direct display** in the UI transcript (per ui-component-specs.md §5's "hidden by default" design) |
| `occurred_at` | ISO 8601 datetime | yes | Ordering key for transcript reconstruction |

**Note:** comprehension-check Q&A is intentionally **not** stored here — it already has its own dedicated log (`ComprehensionCheckAttempt`, §2.6 below), which is the correct source for check questions/answers in both a resumed session and a review view. `LessonTurnMessage` covers only the exploratory dialogue portion.

---

### 2.6 `ComprehensionCheckAttempt`

Append-only log of every comprehension-check submission, per PRD §3.3 (AC-7, AC-8, AC-9).

```json
{
  "attempt_id": "cca_9f8e7d",
  "subtopic_id": "sub_02",
  "attempt_number": 1,
  "question_type": "multiple_choice",
  "question_text": "Which molecule directly captures light energy in the light-dependent reactions?",
  "question_payload": {
    "options": ["Chlorophyll", "Glucose", "ATP", "CO2"],
    "correct_option_index": 0
  },
  "user_answer": "Glucose",
  "result": "incorrect",
  "submitted_at": "2026-08-21T11:44:10Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `attempt_id` | string (UUID) | yes | Primary key |
| `subtopic_id` | string (UUID) | yes | Foreign key → SubTopic |
| `attempt_number` | integer | yes | 1-indexed within the current check cycle (resets alongside `SubTopic.attempt_count`) |
| `question_type` | enum | yes | `multiple_choice` \| `short_free_text` \| `short_numeric_or_word` — per PRD §3.3 |
| `question_text` | string | yes | The literal question shown |
| `question_payload` | object | depends | Shape depends on `question_type` (see §3 below); holds options/correct-answer info needed for scoring |
| `user_answer` | string | yes | Raw user input, regardless of type, stored as text |
| `result` | enum | yes | `correct` \| `incorrect` — all question types, including `short_free_text`, are graded via deterministic string matching (confirmed, no LLM grading in v1); see §3 for match rules |
| `submitted_at` | ISO 8601 datetime | yes | |

---

### 2.7 `AdaptiveFeedbackEvent`

Append-only log of hint/reframe/reveal events, per PRD §3.4 (AC-10, AC-11, AC-12).

```json
{
  "event_id": "afe_2c3d4e",
  "subtopic_id": "sub_02",
  "trigger_type": "error",
  "hint_level": 2,
  "content": "Think about which part of the chlorophyll molecule interacts with photons — what happens to its electrons?",
  "occurred_at": "2026-08-21T11:47:03Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `event_id` | string (UUID) | yes | Primary key |
| `subtopic_id` | string (UUID) | yes | Foreign key → SubTopic |
| `trigger_type` | enum | yes | `error` \| `confusion` \| `ambiguous` \| `reveal_request` — matches system-prompt §1.2 classification labels |
| `hint_level` | integer (0-3) \| null | no | Populated only when `trigger_type = error`; null for `confusion`/`ambiguous`/`reveal_request` |
| `content` | string | yes | The literal hint/reframe/reveal text delivered |
| `occurred_at` | ISO 8601 datetime | yes | |

Note: a `reveal_request` event with a second consecutive occurrence for the same question corresponds to the `revealed` event referenced in PRD AC-2 — implementation can either add a boolean `revealed: true` field here or treat "second `reveal_request` event for the same question" as the marker. **Recommendation:** add an explicit `revealed: boolean` field (default `false`) to avoid ambiguity — reflected in the schema above as a reserved addition; confirm before finalizing.

---

### 2.8 `UserProgressSummary`

Aggregate view, per PRD §6 and the returning-learner user story.

```json
{
  "user_id": "usr_8f3a1c",
  "subjects_completed": 2,
  "subjects_in_progress": 1,
  "overall_pass_rate": 0.78,
  "last_active_at": "2026-08-21T11:47:03Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `user_id` | string (UUID) | yes | Foreign key → User (1:1) |
| `subjects_completed` | integer | yes | Count of `LearningPlan.status = completed` |
| `subjects_in_progress` | integer | yes | Count of `LearningPlan.status = in_progress` (multiple plans may be `in_progress` simultaneously — confirmed, see §4) |
| `overall_pass_rate` | float (0-1) | yes | `correct` attempts / total attempts across all `ComprehensionCheckAttempt` records |
| `last_active_at` | ISO 8601 datetime | yes | |

Time-spent tracking (`total_time_spent_minutes`) is **out of scope for v1** — confirmed. Not included in this schema; can be added in a later version without breaking this shape.

---

## 3. `question_payload` Shapes by `question_type`

Referenced from §2.6. Kept separate here since the shape is conditional.

**`multiple_choice`**
```json
{ "options": ["A", "B", "C", "D"], "correct_option_index": 0 }
```

**`short_free_text`**
```json
{ "accepted_phrases": ["light energy", "chemical energy"], "match_mode": "contains_all", "case_sensitive": false }
```
Graded via **string matching** (confirmed — no LLM grading step for v1). All matching (`contains_all`, `contains_any`, and `exact`) is performed after normalizing both the user's answer and the accepted phrase(s): trim leading/trailing whitespace, collapse internal whitespace to single spaces, lowercase, and strip trailing sentence punctuation (`.`, `?`, `!`, `,`). This normalization is **not optional per match_mode** — it applies uniformly so that `exact` still behaves reasonably for a conversational text interface (e.g., `"False?"` matches an accepted phrase of `"false"`). `match_mode` supported values:
- `contains_all` — normalized user answer must contain every normalized phrase in `accepted_phrases` (order-independent).
- `contains_any` — normalized user answer must contain at least one normalized phrase in `accepted_phrases`.
- `exact` — normalized user answer must equal one of the normalized `accepted_phrases`.

Sub-topic authoring (plan generation) must choose a `match_mode` per free-text question. Default recommendation: `contains_all` for concept-recall questions, `exact` for single-term or true/false recall.

**`short_numeric_or_word`**
```json
{ "accepted_answers": ["6", "six"], "tolerance": null }
```

`tolerance` is reserved for numeric ranges (e.g., `"tolerance": 0.5`) if a check involves a calculated value rather than an exact word/number — not required for v1 unless a quantitative subject (e.g., math) is in scope; ties to PRD Open Question on subject domain.

---

## 4. Explicit Assumptions in This Artifact (flag any that are wrong)

1. IDs are UUIDs as strings; no assumption made about relational vs. document database — both are supported by this shape.
2. `LessonSessionState` is 1:1 with `LearningPlan` (a user can pause multiple plans simultaneously, each retaining its own state) rather than 1:1 with `User` — this is required for AC-6/US-6 multi-subject resume to work correctly. Flag if only one active plan per user should be allowed at a time.
3. `hint_level` and `reveal_requests_this_question` reset per-question rather than per-sub-topic (see system-prompt Assumption 2) — confirm this is the intended reset boundary.
4. Free-text comprehension-check grading uses deterministic **string matching** (`contains_all` / `contains_any` / `exact` per §3), with mandatory normalization (trim, collapse whitespace, lowercase, strip trailing punctuation) applied uniformly across all match modes — confirmed, no LLM grading step in v1. The Evaluation Rubric artifact should document match_mode selection guidance per question style, not scoring logic.
5. Time-spent tracking is **out of scope for v1** — confirmed; `UserProgressSummary` has no time-based field.
6. No explicit `revealed` boolean was in the PRD's data summary — added here as a recommended field on `AdaptiveFeedbackEvent` to make AC-2 unambiguous to query; flag if you'd rather derive it instead of storing it.
7. Multiple `LearningPlan`s may be `in_progress` simultaneously per user — confirmed. `LessonSessionState` remaining 1:1 with `LearningPlan` (not `User`) is therefore the correct design, no change needed.
8. `LessonTurnMessage` (§2.5) scopes each message by both `plan_id` and `subtopic_id` rather than `subtopic_id` alone — this lets a full-session transcript be fetched in one query (by `plan_id`) while still supporting a single-sub-topic view (by `subtopic_id`) for review mode, without needing to join through `SubTopic.plan_id` for the common case.
9. `LessonTurnMessage.classification` is stored per user message for analysis purposes but is explicitly **not** meant to be rendered in any transcript UI (exploratory or review) — consistent with ui-component-specs.md's "hidden by default" design for hint levels/classification.

---

## 5. Resolved Decisions (previously open questions)

- ✅ Users may run multiple `LearningPlan`s `in_progress` at once.
- ✅ `short_free_text` comprehension checks are graded via string matching, not LLM rubric grading.
- ✅ Time-spent tracking is not needed for v1.
- ✅ A persisted `LessonTurnMessage` entity (§2.5) has been added to support both session-refresh recovery and post-completion review of a subject's full dialogue — this also means review mode (viewing a `COMPLETED` plan's transcript) is now in scope, superseding the PRD's earlier "no review mode" framing in its Non-Goals; flag if the PRD itself should be updated to reflect this.
