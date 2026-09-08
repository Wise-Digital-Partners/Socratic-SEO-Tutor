# Artifact: Task Breakdown / Build Plan for Claude Code

**Derived from:** ALL prior artifacts — PRD-guided-learning-app.md, system-prompt-socratic-tutor.md, data-model-schema.md, api-contract.md, sample-conversation-transcript.md, evaluation-rubric.md, ui-component-specs.md
**Purpose:** A sequenced, scoped set of build tasks for Claude Code to execute against. Each task names its spec source(s), a concrete deliverable, and an acceptance check tied back to a PRD acceptance criterion or artifact section wherever possible — so "done" is verifiable, not subjective.
**Audience:** Claude Code.
**How to use this doc:** Work top to bottom — phases are ordered by dependency (data layer before API layer before frontend, etc.), not by user-facing priority. Within a phase, tasks can generally be parallelized unless a dependency is noted. Do not skip Phase 0 (Outstanding Decisions) — several later tasks are blocked on it.

---

## Phase 0: Outstanding Decisions (resolve before proceeding past Phase 2)

These are gaps flagged across the spec stack that were never fully closed. None block Phase 1 (data layer), but several block later phases as noted.

| # | Decision Needed | Status | Source | Blocks |
|---|---|---|---|---|
| 1 | Auth/session layer | ✅ **Resolved** — see below | api-contract.md §6 | — (no longer blocking; implemented in Phase 2 Task 2.10 and Phase 4 Task 4.0) |
| 2 | Should `SubjectReviewView` show comprehension-check recaps (question/answer/correct answer) in addition to dialogue transcript? | Open | ui-component-specs.md §11 | Phase 5, Task 5.7 only — does not block anything else |
| 3 | Does `planTranscript` need pagination for very long sessions? | Open | api-contract.md §8, ui-component-specs.md §11 | Non-blocking for v1 (bounded session sizes per PRD); revisit if usage patterns warrant it later |

**Decision #1 (resolved):** This is an internal tool with no personal information and minimal data (per-topic learning progress only) — a full auth system is disproportionate. Approach: **lightweight named identification, no password.**

- On first use, the client prompts "who are you?" (free-text name entry or a dropdown of known team members) and creates/looks up a `User` record by `display_name`.
- The backend issues a plain session token/cookie tying subsequent requests to that `user_id` — enough to keep `ResumeDashboard` and progress correctly scoped per person, with **no password, no identity verification.**
- All entity IDs (`planId`, `subtopicId`, etc.) remain UUIDs, not sequential/guessable integers (already true per data-model-schema.md) — the one piece of cheap insurance worth keeping even without real auth, so a coworker can't stumble into someone else's session by editing a URL.
- **Explicit non-goal:** this does not prevent a user from claiming someone else's name, or from guessing another user's UUID and querying their data directly. That's an accepted tradeoff for an internal, no-PII tool. If this tool ever becomes external-facing, gets more users, or starts holding anything more sensitive than topic-progress, revisit with a real auth provider (e.g., Supabase Auth, given the Supabase connector already available) — flagged for future revisit, not part of this build.
- **Impact on Phase 2:** a small backend task (2.10) implements create-or-look-up-by-name and session issuance — no password/token-verification complexity.
- **Impact on Phase 4:** `UserIdentificationScreen` (Task 4.0) is the one-time frontend "who are you?" step, built alongside the other core screens rather than deferred to a later hardening phase.
- **Impact on Phase 7:** no longer contains identification work — scoped down to rate-limiting and error-handling hardening only.

**Action:** Decisions #2 and #3 remain open but non-blocking — proceed with the full build; revisit them opportunistically (see Appendix A).

---

## Phase 1: Data Layer

**Goal:** Implement the seven entities from data-model-schema.md in the chosen database, with correct relationships and enums.

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 1.1 | Choose and provision a database (relational or document — data-model-schema.md is storage-agnostic, so this is an open implementation choice, not a spec gap) | data-model-schema.md §1 | Schema migrations/collections exist for all 7 entities |
| 1.2 | Implement `User`, `LearningPlan`, `SubTopic` with the relationships in §1's ER diagram | data-model-schema.md §2.1-§2.3 | A `LearningPlan` can be created with an ordered set of `SubTopic`s; `subtopic_order` stays in sync with each `SubTopic.position` |
| 1.3 | Implement `LessonSessionState` (1:1 with `LearningPlan`) | data-model-schema.md §2.4 | Creating a `LessonSessionState` for a plan that already has one is rejected or upserts correctly (product decision — pick one, document it) |
| 1.4 | Implement `LessonTurnMessage` (append-only) | data-model-schema.md §2.5 | Querying by `plan_id` returns all messages ordered by `occurred_at`; querying by `subtopic_id` returns only that sub-topic's messages |
| 1.5 | Implement `ComprehensionCheckAttempt` and `AdaptiveFeedbackEvent` (both append-only, scoped to `SubTopic`) | data-model-schema.md §2.6-§2.7 | Both support insert + query-by-`subtopic_id`; no update/delete operations are exposed anywhere (append-only by construction, not just convention) |
| 1.6 | Implement `UserProgressSummary` as either a materialized/incrementally-updated table or a computed-on-read aggregate (implementation's choice, per data-model-schema.md §2.8) | data-model-schema.md §2.8 | `overall_pass_rate` correctly reflects `correct` / total across all of a user's `ComprehensionCheckAttempt` records |

**Definition of done for Phase 1:** All 7 entities exist, relationships hold, and a script/test can create a full `LearningPlan` → `SubTopic` → `LessonSessionState` chain end to end without manual intervention.

---

## Phase 2: GraphQL API Layer

**Goal:** Implement every query/mutation in api-contract.md §2's SDL as working resolvers.

**Depends on:** Phase 1 complete. No longer blocked on auth (Phase 0, Decision #1 resolved as lightweight identification, implemented within this phase as Task 2.10) — resolver logic for the core dialogue/plan operations can proceed in parallel with Task 2.10.

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 2.1 | Stand up the GraphQL server (single endpoint) and load the SDL from api-contract.md §2 | api-contract.md §2 | Schema introspection succeeds; all types/enums/queries/mutations are present exactly as specified |
| 2.2 | Implement `generateLearningPlan` — this is the first LLM-backed resolver; requires prompting an LLM to decompose a subject into 3-10 ordered sub-topics with title/description/orderingRationale | api-contract.md §3.1, PRD AC-4/AC-6 | Given "Photosynthesis," returns a plan structurally matching data-model-schema.md's sample (3+ sub-topics, each with required fields); given a deliberately vague subject (e.g., a single ambiguous word), returns `SUBJECT_TOO_VAGUE` rather than a low-quality guess |
| 2.3 | Implement `reorderLearningPlan`, `removeSubtopicFromPlan`, `abandonLearningPlan` | api-contract.md §3.2 | Editing after `DRAFT` returns `PLAN_NOT_EDITABLE`; removing below 3 sub-topics returns `PLAN_BELOW_MINIMUM` |
| 2.4 | Implement `confirmLearningPlan`, including the inline `openingTurn` generation | api-contract.md §3.3 | Response includes both `plan` (status `CONFIRMED`) and a populated `openingTurn` with a real tutor question — not a placeholder |
| 2.5 | Implement `submitLessonTurn`, wiring in the system prompt from system-prompt-socratic-tutor.md §2 | api-contract.md §3.4, system-prompt-socratic-tutor.md §2 | Given the exact context shapes in the prompt, the LLM call is made with all required `{{ }}` variables populated; classification is parsed out of the `[CLASSIFICATION: ...]` tag reliably; both messages are persisted as `LessonTurnMessage` records (§1.4 dependency) |
| 2.6 | Implement `submitComprehensionCheckAnswer`, including string-matching grading | api-contract.md §3.5, evaluation-rubric.md §2 | Grading applies normalization (trim/lowercase/strip punctuation) uniformly across `contains_all`/`contains_any`/`exact`, per data-model-schema.md §3; multiple-choice grades against submitted option **text**, not letter/index (evaluation-rubric.md §4.1) |
| 2.7 | Implement `FAILED_RETRY` re-teach path within 2.6 — inline `nextTurn` re-teach message + eventual new, non-identical check | api-contract.md §3.5 point 5, PRD AC-9 | After 3 incorrect attempts, `subtopic.status = FAILED_RETRY`, `attemptCount` resets to 0, and `nextTurn` contains a real re-teach message (not the same explanation verbatim as the original exploratory dialogue) |
| 2.8 | Implement all Query resolvers, including `planTranscript` | api-contract.md §3.6-§3.7 | `planTranscript(planId)` returns messages in correct chronological order across sub-topic boundaries |
| 2.9 | Implement `AdaptiveFeedbackEvent` writes triggered from within `submitLessonTurn` (hints, reframes, reveals) | api-contract.md §3.4 step 6, PRD AC-12 | Every hint/reframe/reveal in a test conversation produces exactly one corresponding `AdaptiveFeedbackEvent` row with correct `triggerType`/`hintLevel` |
| 2.10 | Implement the lightweight identification backend: a mutation/endpoint to create-or-look-up a `User` by `display_name` and issue a session token/cookie; middleware to resolve the session on every request (per Phase 0, Decision #1) | Phase 0, Decision #1 (resolved) | A returning session (valid cookie/token) resolves to the same `user_id` without re-prompting; this is intentionally **not** a security boundary — see Phase 0 for the accepted tradeoff |

**Note:** Task 2.10 was pulled forward from what would otherwise be a "Phase 7" concern, because `UserIdentificationScreen` (Phase 4, Task 4.0) needs it to exist before the frontend can be built end-to-end. Phase 7 handles hardening beyond this baseline.

**Definition of done for Phase 2:** A full plan-generation → confirmation → multi-turn dialogue → comprehension-check-pass → plan-completion sequence can be driven entirely through GraphQL operations (e.g., via a script or Postman/Insomnia collection), matching the flow table in api-contract.md §4.

---

## Phase 3: Comprehension Check Authoring Quality

**Goal:** Ensure `generateLearningPlan`'s sub-topic generation also produces well-formed comprehension-check questions per the authoring rubric — this is a prompt-engineering task on top of Phase 2's `generateLearningPlan` implementation, not a new endpoint.

**Depends on:** Phase 2, Task 2.2.

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 3.1 | Extend the sub-topic/plan-generation prompt to also produce a `question_payload` per sub-topic (deferred from Task 2.2 to isolate this concern) | evaluation-rubric.md §3 | Generated multiple-choice questions have exactly one correct option, no "all/none of the above"; generated free-text questions default to `contains_any` with a generous `accepted_phrases` list per §4.2's false-negative guidance |
| 3.2 | Spot-check generated questions against the good/bad examples in evaluation-rubric.md §3.1-§3.3 | evaluation-rubric.md §3 | Manual or scripted review of a sample of generated plans shows no "bad" patterns (double negatives, single-verbatim-sentence `exact` free-text answers, etc.) |

**Definition of done for Phase 3:** Generated comprehension checks pass a manual quality spot-check against the rubric's do/don't lists for at least 3 different test subjects.

---

## Phase 4: Frontend — Core Flow

**Goal:** Implement the screens/components in ui-component-specs.md covering the primary path: intake → plan → lesson loop → check → completion.

**Depends on:** Phase 2 complete (frontend needs a working API to build against, though can be stubbed/mocked earlier if parallelizing with backend work).

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 4.0 | `UserIdentificationScreen` | ui-component-specs.md §2 | Name entry creates/looks up a `User`, establishes a session that persists across refresh; screen is skipped on return visits with a valid session |
| 4.1 | `SubjectIntakeScreen` | ui-component-specs.md §3 | All states (idle/ready/generating/error) implemented; `SUBJECT_TOO_VAGUE` error preserves user input |
| 4.2 | `LearningPlanPreview` with reorder/remove | ui-component-specs.md §4 | Optimistic reorder with rollback on error; remove control disabled at 3-sub-topic floor |
| 4.3 | `LessonDialogueView` | ui-component-specs.md §5 | Hint levels/classification never rendered (§5's core design constraint); transitions correctly to `ComprehensionCheckCard` when a response includes `comprehensionCheck` |
| 4.4 | `ComprehensionCheckCard` (all 3 question-type variants) | ui-component-specs.md §6 | Multiple-choice submits option text; incorrect-with-attempts-remaining re-presents the same question; `FAILED_RETRY` hands off to `LessonDialogueView` via `nextTurn`, not back to this card |
| 4.5 | `SubtopicProgressIndicator` | ui-component-specs.md §7 | `FAILED_RETRY` renders identically to `IN_PROGRESS` (no error/red styling) |
| 4.6 | `PlanCompletionSummary` | ui-component-specs.md §8 | Reachable when `planCompleted: true`; links to `SubjectReviewView` |

**Definition of done for Phase 4:** A user can complete an entire subject end-to-end through the UI alone, hitting at least one hint, one comprehension check pass, and plan completion — matching the primary path of sample-conversation-transcript.md.

---

## Phase 5: Frontend — Resume & Review

**Goal:** Implement multi-session support and post-completion review.

**Depends on:** Phase 2 (specifically `planTranscript`), Phase 4.

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 5.1 | `ResumeDashboard` — in-progress plans list | ui-component-specs.md §9 | Lists all `IN_PROGRESS` plans for a user; supports multiple concurrent plans (confirmed design, data-model-schema.md §5) |
| 5.2 | Resume interaction — fetch `planTranscript` + `lessonSessionState` together, route to correct component by `dialogueState` | ui-component-specs.md §9 | Resuming mid-hint-escalation correctly re-renders `LessonDialogueView` with prior history visible; resuming mid-check correctly re-renders `ComprehensionCheckCard` |
| 5.3 | `ResumeDashboard` — completed plans section | ui-component-specs.md §9 | Lists `COMPLETED` plans with a "Review" action |
| 5.4 | `SubjectReviewView` | ui-component-specs.md §8.1 | Read-only transcript render, sectioned/navigable by sub-topic, no input box or live states |
| 5.5 | *(Deferred pending Phase 0, Decision #2)* Comprehension-check recap in `SubjectReviewView` | ui-component-specs.md §11 | Only build if Decision #2 resolves "yes" — otherwise skip |

**Definition of done for Phase 5:** A user can abandon a session mid-lesson, return later, resume with full visible history intact, and — after completing a different subject — review its full transcript read-only.

---

## Phase 6: Integration Testing Against the Sample Transcript

**Goal:** Validate the built system against the behavioral test cases in sample-conversation-transcript.md.

**Depends on:** Phases 1-4 minimum (Phase 5 not required for this phase).

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 6.1 | Write an integration test walking Scene 1-3 (plan generation → confirmation → hint escalation → confusion/reframe → correct → check pass) | sample-conversation-transcript.md Scenes 1-3 | Test asserts on `dialogueState`, `hintLevel`, `reframeCount`, and `classification` at each step, matching the transcript's `state` annotations |
| 6.2 | Write an integration test for the reveal-request flow (Scene 4) | sample-conversation-transcript.md Scene 4 | Asserts decline-on-first-request, reveal-on-second, `revealed: true` logged, immediate re-check issued |
| 6.3 | Write an integration test for the failed-retry branch (Scene 5b) | sample-conversation-transcript.md Scene 5b | Asserts `FAILED_RETRY` after 3 incorrect attempts, `attemptCount` resets to 0, inline re-teach delivered, new non-identical check offered afterward |
| 6.4 | Write a grading-normalization test using Scene 5's `"False?"` → matches `"false"` case | sample-conversation-transcript.md Scene 5, evaluation-rubric.md §2 | Confirms trim/lowercase/punctuation-strip normalization is actually applied in the grading implementation, not just documented |

**Note for Claude Code:** Because `submitLessonTurn`'s classification step depends on LLM judgment (system-prompt-socratic-tutor.md §5, Assumption 2), exact wording matches to the transcript are not expected — assert on **state transitions and classification labels**, not on literal tutor message text.

**Definition of done for Phase 6:** All four test scenarios pass, and the coverage checklist in sample-conversation-transcript.md §1 is fully green.

---

## Phase 7: Hardening

**Goal:** General production hardening beyond the baseline identification/session handling already built in Phase 2 (Task 2.10) and Phase 4 (Task 4.0).

| Task | Description | Spec Reference | Acceptance Check |
|---|---|---|---|
| 7.1 | Rate-limit / cost-guard the LLM-backed mutations (`generateLearningPlan`, `submitLessonTurn`) | (not previously specified — flag if a rate-limit policy is wanted; not in PRD scope explicitly) | Basic guard exists (implementation's choice of mechanism) to prevent runaway LLM cost from a single user/session |
| 7.2 | Error-handling pass across all mutations for the error codes named throughout api-contract.md (`SESSION_NOT_FOUND`, `INVALID_STATE_TRANSITION`, `NO_ACTIVE_CHECK`, `PLAN_NOT_EDITABLE`, `PLAN_BELOW_MINIMUM`, `SUBJECT_TOO_VAGUE`) | api-contract.md §3 (per-operation error lists) | Each named error code is actually returned under the condition described, verified by test |

---

## Appendix A: Consolidated Outstanding Open Questions

Repeated here from Phase 0 plus any others still unresolved across the full artifact stack, for one-place visibility:

1. ~~Auth/session layer mechanism~~ — **Resolved:** lightweight named identification, no password (Phase 0, Decision #1).
2. Whether `SubjectReviewView` includes comprehension-check recaps (Phase 5, Task 5.5 — optional).
3. Whether `planTranscript` needs pagination eventually (non-blocking for v1).
4. Rate-limiting policy for LLM-backed mutations was never specified anywhere upstream — surfaced fresh in Phase 7, Task 7.1; needs a decision before that task can be scoped precisely.

## Appendix B: Artifact Cross-Reference Map

For quick navigation while building:

| Artifact | Governs |
|---|---|
| PRD-guided-learning-app.md | What the product must do; acceptance criteria (AC-1 through AC-12) |
| system-prompt-socratic-tutor.md | Tutor LLM behavior, dialogue states, hint/classification logic |
| data-model-schema.md | Persisted entities and their fields/relationships |
| api-contract.md | GraphQL schema and operation-level behavior |
| sample-conversation-transcript.md | Behavioral test cases / expected end-to-end flow |
| evaluation-rubric.md | How comprehension checks are authored and graded |
| ui-component-specs.md | Frontend screens, states, and interactions |
| This document | Build sequencing and acceptance checks tying it all together |
