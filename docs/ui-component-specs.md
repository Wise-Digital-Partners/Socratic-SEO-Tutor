# Artifact: UI Component Specs

**Derived from:** PRD-guided-learning-app.md (§5), api-contract.md, data-model-schema.md, system-prompt-socratic-tutor.md (§1.1 dialogue states)
**Purpose:** Component-level specs for every screen/view needed to implement the user flow in PRD §5. These are **structural and behavioral specs, not final visual design** — layout, data bindings, states, and which API operations each interaction triggers. Visual polish (spacing, color, typography) is left to Claude Code's implementation using whatever design system is chosen.
**Audience:** Claude Code (frontend implementation).
**Scope note:** Kept to one concern (UI structure/behavior) — does not redefine data shapes (see data-model-schema.md) or endpoint behavior (see api-contract.md); each component below references those artifacts rather than restating them.

---

## 1. Screen Inventory (maps to PRD §5 flow steps)

| Screen / Component | PRD Flow Step |
|---|---|
| `UserIdentificationScreen` | (new) One-time "who are you?" — precedes all other screens |
| `SubjectIntakeScreen` | 1. Subject Intake |
| `LearningPlanPreview` | 2-3. Plan Generation & Confirmation |
| `LessonDialogueView` | 4a-c. Lesson Loop (exploratory dialogue) |
| `ComprehensionCheckCard` | 4d. Comprehension check administered |
| `SubtopicProgressIndicator` | (persistent, all of step 4) |
| `PlanCompletionSummary` | 5. Plan Completion |
| `SubjectReviewView` | (new) Review a completed subject |
| `ResumeDashboard` | 6. Session Resume |

---

## 2. `UserIdentificationScreen`

**Purpose:** One-time "who are you?" step, per the resolved lightweight-identification decision (api-contract.md §6, task-breakdown-build-plan.md Phase 0/7). This is not a login screen — there is no password and no verification.

**Data needed:** none on load; optionally a list of known team members' `display_name`s to populate a dropdown/autocomplete, if such a list is maintained anywhere convenient — otherwise plain free-text entry is sufficient.

**States:**
- `entering` — name field empty or being typed, submit disabled until non-empty.
- `submitting` — brief transient state while the session is established.

**Interactions:**
- Submit → creates or looks up a `User` by `display_name`, establishes a session token/cookie for subsequent requests, then routes to `SubjectIntakeScreen` (or `ResumeDashboard`/§3.1's "welcome back" variant, if the user has existing plans).

**Edge cases:**
- Returning users should not see this screen again once a valid session exists (session persists across page refreshes, per task-breakdown-build-plan.md Task 4.0's acceptance check) — this is a one-time-per-session step, not a per-visit one.
- No explicit "log out" flow is specified — not required by any AC; flag if wanted later (e.g., a shared device needing to switch identities).

---

## 3. `SubjectIntakeScreen`

**Purpose:** Entry point where a user names a subject to learn.

**Data needed:** none on load (or, optionally, `userProgressSummary(userId)` to show a "welcome back" state — see §3.1).

**States:**
- `idle` — text input empty, submit disabled.
- `ready` — text has been entered, submit enabled.
- `generating` — `generateLearningPlan` mutation in flight; show a loading state (plan generation may take a few seconds since it's LLM-backed — do not block on a spinner with no explanation; show something like "Building your learning plan…").
- `error` — mutation returned `SUBJECT_TOO_VAGUE` (api-contract.md §3.1) or a network error; show the message inline and let the user retry without losing their typed input.

**Interactions:**
- Submit → `generateLearningPlan(userId, subject)` → on success, navigate to `LearningPlanPreview` with the returned `LearningPlan`.

**Edge cases:**
- Empty/whitespace-only input: disable submit client-side, don't round-trip to the backend for this.
- `SUBJECT_TOO_VAGUE` error: surface the backend's guidance to be more specific; keep the user's original input in the field.

### 3.1 Optional: Returning-user state
If `userProgressSummary(userId).subjectsCompleted > 0` or there are `IN_PROGRESS` plans, this screen may also surface a prompt like "Continue an existing subject?" linking to `ResumeDashboard`, alongside the option to start a new one. Not required for v1 functionality — flag as a nice-to-have, not a blocking spec.

---

## 4. `LearningPlanPreview`

**Purpose:** Show the full generated curriculum and let the user edit or confirm it before lessons start (PRD AC-5, AC-6).

**Data needed:** `LearningPlan` with nested `subtopics` (title, description, orderingRationale, position).

**States:**
- `draft_reviewing` — plan shown, editable (only valid while `LearningPlan.status = DRAFT`, per api-contract.md §3.2).
- `confirming` — `confirmLearningPlan` in flight.
- `confirmed` — transient state right before navigating to `LessonDialogueView` with the `openingTurn` payload already in hand (no extra round-trip needed, per api-contract.md §3.3).

**Layout elements:**
- Ordered list of sub-topic cards, each showing `title`, `description`, and optionally `orderingRationale` as a subtle caption ("builds on: The Light-Dependent Reactions").
- Per-card controls: drag-to-reorder (or up/down buttons as a simpler alternative) and a remove ("✕") button.
- A "Start Learning" primary action.

**Interactions:**
- Reorder → `reorderLearningPlan(planId, subtopicIdsInOrder)`. Recommend optimistic UI update (reorder locally immediately) with rollback on mutation error.
- Remove → `removeSubtopicFromPlan(planId, subtopicId)`. Disable the remove control on the 3rd-to-last remaining sub-topic to preempt hitting `PLAN_BELOW_MINIMUM` (api-contract.md §3.2), rather than only handling it as an error after the fact.
- Confirm → `confirmLearningPlan(planId)` → response includes both `plan` and `openingTurn`; navigate directly to `LessonDialogueView` pre-seeded with `openingTurn.tutorMessage`, no additional fetch needed.

**Edge cases:**
- `PLAN_NOT_EDITABLE` error (attempted edit after confirmation, e.g., from a stale tab/back-button): show a message and refresh plan state rather than silently failing.

---

## 5. `LessonDialogueView`

**Purpose:** The core Socratic dialogue interface — a chat-style exchange between tutor and user for the current sub-topic's exploratory phase.

**Data needed:** `LessonSessionState` (dialogueState, currentSubtopic, hintLevel, reframeCount — see note below on visibility), plus the running turn history for display (see §5.3 on history source).

**States (mirrors `DialogueState` enum, api-contract.md):**
- `AWAITING_RESPONSE` — tutor has spoken, text input is enabled and focused.
- (submitting) — transient state while `submitLessonTurn` is in flight; disable input, show a "tutor is thinking" indicator.
- `GIVING_HINT` — functionally the same rendering as `AWAITING_RESPONSE` (a new tutor message appended, input re-enabled) — no special visual treatment needed; see note below on hint-level visibility.
- `CONFIRMING_UNDERSTANDING` — instead of a free-text input, this view hands off to `ComprehensionCheckCard` (§6) using the `comprehensionCheck` payload from the last `submitLessonTurn` or `submitComprehensionCheckAnswer` response.
- `ADVANCING` — brief transitional state (typically resolved automatically via the `nextTurn` payload from `submitComprehensionCheckAnswer` — see api-contract.md §3.5) showing a short congratulatory beat before the next sub-topic's opening message appears.

**Important design note — hint levels and classification are backend-internal, not user-facing:** Per system-prompt-socratic-tutor.md §5 (open question, resolved default: hidden), do **not** display "Hint 2 of 3," the `classification` value, or `reframeCount` anywhere in this UI. The user should experience a natural conversation; these fields exist for state-machine logic and logging (`AdaptiveFeedbackEvent`), not display. If a future version wants to surface hint progress, that's an explicit redesign, not a default in this spec.

**Layout elements:**
- Scrollable message history (tutor messages left-aligned/distinct styling, user messages right-aligned/distinct styling — standard chat UI pattern).
- Text input + submit button, disabled while a turn is in flight.
- A persistent `SubtopicProgressIndicator` (§7), typically pinned above or alongside the message history.

**Interactions:**
- Submit user message → `submitLessonTurn(planId, userMessage)`. Append the user's message to history immediately (optimistic), then append `tutorMessage` from the response once it resolves. If the response includes a `comprehensionCheck`, transition to rendering `ComprehensionCheckCard` instead of expecting further free-text input.

**Edge cases:**
- `INVALID_STATE_TRANSITION` error (e.g., a stale client tries to submit a lesson turn while state has already moved to `CONFIRMING_UNDERSTANDING` elsewhere): refetch `LessonSessionState` and re-render the correct component rather than surfacing a raw error.
- Empty message submission: client-side validation should prevent submitting empty/whitespace-only messages, consistent with `SubjectIntakeScreen`.

### 5.1 Reveal-request affordance
No special UI button is needed for "reveal the answer" — per system-prompt-socratic-tutor.md, this is handled conversationally (the user typing something like "just tell me" triggers `REVEAL_REQUEST` classification server-side). Do not add a dedicated "Give up" button in v1; it would bypass the pedagogical design intent of PRD §3.4. Flag if a future version wants an explicit escape hatch.

### 5.2 Confusion re-framing
No special visual distinction is needed when the tutor is reframing (confusion) vs. hinting (error) — both simply render as the next tutor message in the same chat flow. This is intentional, per §5's "hidden by default" note above.

### 5.3 Message history source (resolved)
The turn-by-turn conversation history is now persisted server-side as `LessonTurnMessage` records (data-model-schema.md §2.5), queryable via `planTranscript(planId)` (api-contract.md §3.7). On loading this view — whether starting fresh after `confirmLearningPlan`/`submitComprehensionCheckAnswer`'s inline `openingTurn`/`nextTurn`, or resuming an existing session — the client should call `planTranscript(planId)` to reconstruct the full visible transcript, then continue appending new messages locally as `submitLessonTurn` responses arrive (no need to re-fetch after every turn). This also means the transcript now survives a page refresh and is available for `PlanCompletionSummary`'s review flow (§8).

---

## 6. `ComprehensionCheckCard`

**Purpose:** Renders the current comprehension check and collects the user's answer, per its `questionType`.

**Data needed:** `ComprehensionCheckQuestion` (questionType, questionText, options).

**Variants (by `questionType`):**
- `MULTIPLE_CHOICE` — render `options` as selectable buttons/radio inputs. On submit, send the **selected option's text** as `answer` (per evaluation-rubric.md §4.1 — not a letter/index).
- `SHORT_FREE_TEXT` — render a text input (single line or small textarea depending on expected answer length).
- `SHORT_NUMERIC_OR_WORD` — render a text input; may use `inputmode="numeric"` as a UX hint when the answer is expected to be a number, but the field remains plain text since `accepted_answers` can include word-form equivalents (data-model-schema.md §3).

**States:**
- `answering` — input enabled, submit enabled once a selection/entry is made.
- `submitting` — `submitComprehensionCheckAnswer` in flight; disable input.
- `result_incorrect_retry_available` — show a brief "not quite, try again" message (matching the tone in sample-conversation-transcript.md Scene 5), then return to `answering` for the **same question** (do not fetch a new one — api-contract.md §3.5 point 4).
- `result_correct` — brief affirming state before handing off; the `nextTurn` payload (if present) determines what renders next: either the next sub-topic's opening message in `LessonDialogueView`, or `PlanCompletionSummary` if `planCompleted: true`.
- `result_failed_retry` — triggered when the response's `subtopic.status = FAILED_RETRY`; hand off to `LessonDialogueView` using the `nextTurn` payload (the inline re-teach message) rather than showing this card again immediately — the user re-enters exploratory dialogue before a **new** check is offered (api-contract.md §3.5 point 5).

**Interactions:**
- Submit → `submitComprehensionCheckAnswer(subtopicId, answer)` → branch rendering per the result states above, driven by `advancedToNextSubtopic`, `subtopic.status`, and `planCompleted` in the response.

**Edge cases:**
- `NO_ACTIVE_CHECK` error (stale client state): refetch `LessonSessionState` and re-render appropriately, same pattern as §5's edge case.

---

## 7. `SubtopicProgressIndicator`

**Purpose:** Persistent, at-a-glance view of where the user is in the current `LearningPlan` — supports orientation during the (potentially long) lesson loop.

**Data needed:** `LearningPlan.subtopics` (title, status, position).

**Layout:** A horizontal or vertical stepper, one entry per sub-topic, each showing its `status`:
- `NOT_STARTED` — neutral/dim styling.
- `IN_PROGRESS` — highlighted as current.
- `PASSED` — checkmark/complete styling.
- `FAILED_RETRY` — should render the same as `IN_PROGRESS` visually (it's a transient re-teach state, not a failure the user should feel is permanent or shameful — consistent with PRD AC-9's explicit "no dead ends" requirement). Do not use error/red styling for this status.

**Interactions:** Read-only in v1 — no PRD requirement for jumping directly to a past or future sub-topic from this indicator.

---

## 8. `PlanCompletionSummary`

**Purpose:** Shown when `LearningPlan.status = COMPLETED` (PRD §5 flow step 5).

**Data needed:** The completed `LearningPlan` with its `subtopics` (for a per-topic recap).

**Layout elements:**
- Congratulatory header naming the subject.
- List of completed sub-topics (simple recap, reusing `SubtopicProgressIndicator`'s all-`PASSED` rendering is sufficient — no need for a separate visual pattern).
- Primary actions: "Start a new subject" (→ `SubjectIntakeScreen`) and **"Review this subject"** (→ `SubjectReviewView`, §8.1 below).

### 8.1 `SubjectReviewView`
**Purpose:** Read-only view of a completed subject's full dialogue, per the confirmed review-mode decision (data-model-schema.md §5).

**Data needed:** `planTranscript(planId)` (api-contract.md §3.7) for the message history; `LearningPlan.subtopics` for section headers/navigation.

**Layout elements:**
- Same chat-transcript rendering pattern as `LessonDialogueView`, but with no input box and no "tutor is thinking" states — it's a static read-through.
- Sectioned/anchored by sub-topic (using each message's `subtopicId`) so the user can jump to a specific sub-topic's dialogue rather than scrolling one long undifferentiated transcript.
- No `ComprehensionCheckCard` re-rendering — comprehension check Q&A is a separate log (`ComprehensionCheckAttempt`) and is out of scope for this view unless a future version wants to show "you answered X, the correct answer was Y" recaps; flag as optional, not required by any AC.
- A "Back" / "Done" action returning to `ResumeDashboard` or `SubjectIntakeScreen`.

**Interactions:** Read-only — no mutations triggered from this view in v1.

---

## 9. `ResumeDashboard`

**Purpose:** Lets a user see and resume any `IN_PROGRESS` plan, supporting PRD US-6 and the confirmed multiple-concurrent-plans design (data-model-schema.md §5).

**Data needed:** `learningPlans(userId, status: IN_PROGRESS)` — each with `subject`, `subtopics` (for a "3 of 5 sub-topics complete" style progress summary), and enough of `sessionState` to resume directly.

**Layout elements:**
- One card per in-progress plan: subject name, simple progress fraction (e.g., "2 of 4 complete"), "Resume" action.
- A secondary section listing `COMPLETED` plans (via `learningPlans(userId, status: COMPLETED)`), each with a "Review" action linking to `SubjectReviewView` (§8.1) for that plan.

**Interactions:**
- Resume → call `planTranscript(planId)` and `lessonSessionState(planId)` together to reconstruct visible history and current position, then route into `LessonDialogueView` or `ComprehensionCheckCard`, whichever `dialogueState` indicates (`CONFIRMING_UNDERSTANDING` → check card; anything else → dialogue view, pre-seeded with the fetched transcript).
- Review → route to `SubjectReviewView` for the selected completed plan.

**Edge cases:**
- Empty state (no in-progress plans): route to or prominently link `SubjectIntakeScreen` instead of showing an empty dashboard.

---

## 10. Explicit Assumptions in This Artifact (flag any that are wrong)

1. Hint levels, classification labels, and reframe counts are never shown to the user — treated as backend-internal state (§5). This was the recommended default from system-prompt-socratic-tutor.md's own open question; reconfirming it here since it directly shapes UI scope.
2. No "reveal/give up" button exists in the UI — reveal is conversational only (§5.1), consistent with preserving the Socratic design intent.
3. Conversation transcript is now persisted server-side and fetched via `planTranscript` (§5.3, resolved) — client-side state is only used to append new messages as they arrive within an active session, not as the source of truth.
4. `FAILED_RETRY` status is deliberately styled identically to `IN_PROGRESS` in the progress indicator (§7), to avoid the UI contradicting AC-9's "no dead ends, no shame" requirement.
5. `SubjectReviewView` (§8.1) shows dialogue transcript only, not comprehension-check Q&A recaps (which would require pulling `ComprehensionCheckAttempt` history per sub-topic) — flagged as optional scope, not included by default.
6. `UserIdentificationScreen` (§2) is a one-time, no-password identity step, not a login system — consistent with the resolved lightweight-identification decision (api-contract.md §6, task-breakdown-build-plan.md Phase 0/7). No logout flow is specified.

## 11. Open Questions for Stakeholder / User Confirmation

- Should `SubjectReviewView` also show comprehension-check recaps (question asked, answer given, correct answer) per sub-topic, or is the exploratory dialogue transcript sufficient for v1 review? This affects whether `SubjectReviewView` also queries `ComprehensionCheckAttempt` history.
- Should `planTranscript` support pagination/lazy-loading for very long sessions (flagged as a possible future need in api-contract.md §8), or is loading the full transcript in one call acceptable for the lifetime of a single-user subject (max 10 sub-topics)?
