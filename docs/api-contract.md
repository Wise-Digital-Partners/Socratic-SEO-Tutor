# Artifact: API Contract (GraphQL)

**Derived from:** PRD-guided-learning-app.md, system-prompt-socratic-tutor.md, data-model-schema.md
**Stack assumption (confirmed by user):** GraphQL. Single endpoint (`POST /graphql`), standard GraphQL error format for failures. Transport/auth layer (JWT, session cookie, etc.) is undetermined — flagged in §5.
**Purpose:** Defines every query/mutation the frontend (built by Claude Code) needs to drive the full user flow in PRD §5, backed by the entities in data-model-schema.md.
**Audience:** Claude Code (implementation — both resolver/backend and frontend client code).

---

## 1. Design Notes Before the Schema

- **One mutation drives one dialogue turn.** `submitLessonTurn` is the single entry point for exploratory Socratic dialogue (posing_question → evaluating_response → giving_hint/confirming_understanding). It wraps the system-prompt LLM call described in system-prompt-socratic-tutor.md §2 and returns the updated `LessonSessionState` plus the tutor's message.
- **Comprehension checks are a distinct mutation** (`submitComprehensionCheckAnswer`), separate from `submitLessonTurn`, because they're graded deterministically via string matching (data-model-schema.md §3), not by the tutoring LLM. When `submitLessonTurn` transitions `dialogue_state` to `confirming_understanding`, its response includes the check question; the client then calls `submitComprehensionCheckAnswer` to submit the user's answer to that specific question.
- **Plan generation is also LLM-backed** but is a one-shot structured generation (subject → ordered sub-topics), not a dialogue turn — kept as its own mutation, `generateLearningPlan`.
- All mutations return the affected entity plus enough nested state for the client to re-render without an extra round-trip (see each mutation's return type).
- Multiple concurrent `in_progress` plans per user are supported (confirmed in data-model-schema.md §5) — no endpoint enforces a single-active-plan constraint.

---

## 2. GraphQL Schema (SDL)

```graphql
# ---------- Enums (mirror data-model-schema.md) ----------

enum PlanStatus {
  DRAFT
  CONFIRMED
  IN_PROGRESS
  COMPLETED
  ABANDONED
}

enum SubTopicStatus {
  NOT_STARTED
  IN_PROGRESS
  PASSED
  FAILED_RETRY
}

enum DialogueState {
  POSING_QUESTION
  AWAITING_RESPONSE
  EVALUATING_RESPONSE
  GIVING_HINT
  CONFIRMING_UNDERSTANDING
  ADVANCING
}

enum ResponseClassification {
  CORRECT
  ERROR
  CONFUSION
  AMBIGUOUS
  REVEAL_REQUEST
}

enum QuestionType {
  MULTIPLE_CHOICE
  SHORT_FREE_TEXT
  SHORT_NUMERIC_OR_WORD
}

enum CheckResult {
  CORRECT
  INCORRECT
}

enum MatchMode {
  CONTAINS_ALL
  CONTAINS_ANY
  EXACT
}

enum MessageRole {
  TUTOR
  USER
}

# ---------- Core Types ----------

type User {
  userId: ID!
  displayName: String
  createdAt: String!
  progressSummary: UserProgressSummary!
  learningPlans(status: PlanStatus): [LearningPlan!]!
}

type LearningPlan {
  planId: ID!
  userId: ID!
  subject: String!
  status: PlanStatus!
  subtopics: [SubTopic!]!          # resolved in subtopicOrder sequence
  sessionState: LessonSessionState # null until plan is confirmed
  createdAt: String!
  confirmedAt: String
  completedAt: String
}

type LessonTurnMessage {
  messageId: ID!
  planId: ID!
  subtopicId: ID!
  role: MessageRole!
  content: String!
  occurredAt: String!
  # NOTE: classification is intentionally NOT exposed here — it's stored server-side for
  # analysis (data-model-schema.md §2.5) but is not meant to be surfaced in any transcript UI.
}

type SubTopic {
  subtopicId: ID!
  planId: ID!
  title: String!
  description: String!
  orderingRationale: String
  position: Int!
  status: SubTopicStatus!
  attemptCount: Int!
  startedAt: String
  passedAt: String
  transcript: [LessonTurnMessage!]!   # this sub-topic's exploratory dialogue messages, in order
  checkAttempts: [ComprehensionCheckAttempt!]!
  feedbackEvents: [AdaptiveFeedbackEvent!]!
}

type LessonSessionState {
  sessionStateId: ID!
  planId: ID!
  currentSubtopic: SubTopic!
  dialogueState: DialogueState!
  hintLevel: Int!
  reframeCount: Int!
  revealRequestsThisQuestion: Int!
  lastUpdatedAt: String!
}

type ComprehensionCheckQuestion {
  questionType: QuestionType!
  questionText: String!
  options: [String!]              # populated only for MULTIPLE_CHOICE
}

type ComprehensionCheckAttempt {
  attemptId: ID!
  subtopicId: ID!
  attemptNumber: Int!
  questionType: QuestionType!
  questionText: String!
  userAnswer: String!
  result: CheckResult!
  submittedAt: String!
}

type AdaptiveFeedbackEvent {
  eventId: ID!
  subtopicId: ID!
  triggerType: ResponseClassification!
  hintLevel: Int
  content: String!
  revealed: Boolean!
  occurredAt: String!
}

type UserProgressSummary {
  userId: ID!
  subjectsCompleted: Int!
  subjectsInProgress: Int!
  overallPassRate: Float!
  lastActiveAt: String!
}

# ---------- Turn Result (submitLessonTurn payload) ----------

type LessonTurnResult {
  sessionState: LessonSessionState!
  tutorMessage: String!
  classification: ResponseClassification   # null when dialogueState was POSING_QUESTION (no user input classified yet)
  comprehensionCheck: ComprehensionCheckQuestion  # populated only when sessionState.dialogueState transitions to CONFIRMING_UNDERSTANDING
  feedbackEvent: AdaptiveFeedbackEvent      # populated only when a hint/reframe/reveal was generated this turn
}

type ComprehensionCheckResult {
  attempt: ComprehensionCheckAttempt!
  subtopic: SubTopic!                # reflects updated status/attemptCount post-grading
  advancedToNextSubtopic: Boolean!
  nextTurn: LessonTurnResult         # ALWAYS populated except when planCompleted = true — see §3.5. Covers both "next sub-topic's opening question" (on pass) and "re-teach opening message" (on FAILED_RETRY)
  planCompleted: Boolean!
}

type ConfirmLearningPlanResult {
  plan: LearningPlan!
  openingTurn: LessonTurnResult!   # the first tutor message (opening probing question) for sub-topic #1, generated inline so the client never needs a sentinel call
}

# ---------- Root Types ----------

type Query {
  user(userId: ID!): User
  learningPlan(planId: ID!): LearningPlan
  learningPlans(userId: ID!, status: PlanStatus): [LearningPlan!]!
  lessonSessionState(planId: ID!): LessonSessionState
  userProgressSummary(userId: ID!): UserProgressSummary
  planTranscript(planId: ID!): [LessonTurnMessage!]!   # full session transcript across all sub-topics, in order — backs resume-with-history and review mode
}

type Mutation {
  # --- Plan lifecycle (PRD §3.2) ---
  generateLearningPlan(userId: ID!, subject: String!): LearningPlan!
  reorderLearningPlan(planId: ID!, subtopicIdsInOrder: [ID!]!): LearningPlan!
  removeSubtopicFromPlan(planId: ID!, subtopicId: ID!): LearningPlan!
  confirmLearningPlan(planId: ID!): ConfirmLearningPlanResult!
  abandonLearningPlan(planId: ID!): LearningPlan!

  # --- Lesson dialogue loop (PRD §3.1, §3.4) ---
  submitLessonTurn(planId: ID!, userMessage: String!): LessonTurnResult!

  # --- Comprehension checks (PRD §3.3) ---
  # For MULTIPLE_CHOICE questions, `answer` is the option's TEXT (not a letter/index) — see evaluation-rubric.md §4.1.
  submitComprehensionCheckAnswer(
    subtopicId: ID!
    answer: String!
  ): ComprehensionCheckResult!
}
```

---

## 3. Operation-by-Operation Contract

### 3.1 `generateLearningPlan`
**Maps to:** PRD AC-4, AC-6.
**Behavior:** Takes a free-text subject, generates 3–10 ordered `SubTopic` records, creates a `LearningPlan` with `status = DRAFT`. Does **not** create a `LessonSessionState` yet (that happens on confirm).

```graphql
mutation {
  generateLearningPlan(userId: "usr_8f3a1c", subject: "Photosynthesis") {
    planId
    status
    subtopics {
      subtopicId
      title
      description
      orderingRationale
      position
    }
  }
}
```

**Errors:**
- `SUBJECT_TOO_VAGUE` — if the subject cannot be decomposed into sub-topics with reasonable confidence (e.g., single ambiguous word); response should suggest the user be more specific rather than silently guessing.

---

### 3.2 `reorderLearningPlan` / `removeSubtopicFromPlan`
**Maps to:** PRD AC-6 (editable plan pre-confirmation).
**Behavior:** Only valid while `LearningPlan.status = DRAFT`. Reordering updates `subtopicOrder`/`position` fields; removal deletes the `SubTopic` record and re-sequences remaining positions.

**Errors:**
- `PLAN_NOT_EDITABLE` — if called on a plan that is no longer `DRAFT`.
- `PLAN_BELOW_MINIMUM` — if removal would drop sub-topic count below the 3-topic minimum (PRD AC-4).

---

### 3.3 `confirmLearningPlan`
**Maps to:** PRD AC-5.
**Behavior:** Transitions `status: DRAFT → CONFIRMED`, sets `confirmedAt`, and **creates the initial `LessonSessionState`** for this plan (`currentSubtopic` = first in order, `dialogueState = POSING_QUESTION`, all counters at 0). This is the hand-off point into the lesson loop.

**Decision (resolved):** the mutation also immediately runs the tutor's opening turn for sub-topic #1 and returns it inline as `openingTurn`, so the client can render the first probing question directly from this call's response — no follow-up sentinel call to `submitLessonTurn` is needed. `openingTurn.sessionState.dialogueState` will be `AWAITING_RESPONSE` (the tutor has posed its question and is now waiting on the user).

```graphql
mutation {
  confirmLearningPlan(planId: "plan_1a2b3c") {
    plan {
      planId
      status
      confirmedAt
    }
    openingTurn {
      tutorMessage
      sessionState {
        dialogueState
        currentSubtopic { title }
      }
    }
  }
}
```

---

### 3.4 `submitLessonTurn`
**Maps to:** PRD §3.1, §3.4 (AC-1, AC-2, AC-3, AC-10, AC-11, AC-12); implements system-prompt-socratic-tutor.md §1–§2.
**Behavior:** The core dialogue endpoint. Backend:
1. Loads current `LessonSessionState` for `planId`.
2. Injects state + `userMessage` into the system prompt context (per system-prompt-socratic-tutor.md §2).
3. Receives classification + tutor response from the LLM.
4. Persists both the incoming `userMessage` and the outgoing tutor response as `LessonTurnMessage` records (`role: USER` and `role: TUTOR` respectively), scoped to the current `planId`/`subtopicId` — this is what backs `planTranscript` (§3.7) for resume-with-history and review mode. The user message's `classification` is stored on its `LessonTurnMessage` record for analysis, per data-model-schema.md §2.5.
5. Updates `LessonSessionState` (`dialogueState`, `hintLevel`, `reframeCount`, `revealRequestsThisQuestion`) per the state-transition rules in the system prompt spec.
6. If a hint/reframe/reveal was generated, writes an `AdaptiveFeedbackEvent`.
7. If the tutor judged the user ready (per system-prompt §1.4), transitions `dialogueState → CONFIRMING_UNDERSTANDING` and returns a `comprehensionCheck` payload instead of continuing exploratory dialogue.

```graphql
mutation {
  submitLessonTurn(planId: "plan_1a2b3c", userMessage: "Because sunlight excites electrons in chlorophyll") {
    tutorMessage
    classification
    sessionState {
      dialogueState
      hintLevel
      reframeCount
    }
    comprehensionCheck {
      questionType
      questionText
      options
    }
  }
}
```

**Note on opening messages:** `submitLessonTurn` is only ever called with a real user message. Opening tutor messages for a sub-topic (whether sub-topic #1 after plan confirmation, the next sub-topic after a pass, or a re-teach after `FAILED_RETRY`) are always generated and returned inline by the triggering mutation (`confirmLearningPlan.openingTurn` or `submitComprehensionCheckAnswer.nextTurn` — see §3.3 and §3.5). No sentinel/empty-string call is needed anywhere in this contract.

**Errors:**
- `SESSION_NOT_FOUND` — plan not confirmed yet, or session state missing.
- `INVALID_STATE_TRANSITION` — e.g., calling this while `dialogueState = CONFIRMING_UNDERSTANDING` (client should be calling `submitComprehensionCheckAnswer` instead).

---

### 3.5 `submitComprehensionCheckAnswer`
**Maps to:** PRD §3.3 (AC-7, AC-8, AC-9); grading rules in data-model-schema.md §3.
**Behavior:**
1. Grades `answer` against the current check's `question_payload` using string matching (`CONTAINS_ALL` / `CONTAINS_ANY` / `EXACT` per `matchMode`), per data-model-schema.md §3 and evaluation-rubric.md §2. For `MULTIPLE_CHOICE` questions, `answer` is the submitted option's text, compared (post-normalization) against the option text at `correct_option_index` — not a letter or index.
2. Writes a `ComprehensionCheckAttempt` record, increments `SubTopic.attemptCount`.
3. **On correct:** sets `SubTopic.status = PASSED`, `passedAt` timestamp. If more sub-topics remain, advances `LessonSessionState.currentSubtopic` to the next one, resets all counters, sets `dialogueState = POSING_QUESTION`, runs the tutor's opening turn for the new sub-topic, and returns it inline via `nextTurn` (`advancedToNextSubtopic: true`). If no sub-topics remain, sets `LearningPlan.status = COMPLETED`, returns `planCompleted: true`, and `nextTurn: null` (nothing left to advance to).
4. **On incorrect, attempts remaining:** returns `advancedToNextSubtopic: false`, `nextTurn: null`; client re-presents the same check question, per AC-9 ("attempts against the same check up to the max").
5. **On incorrect, max attempts (3) reached:** sets `SubTopic.status = FAILED_RETRY`, resets `attemptCount` to 0. **Decision (resolved):** the backend immediately runs the simplified re-teach turn (per AC-9) and returns it inline via `nextTurn`, exactly like the pass-and-advance case — the client always gets a renderable next step from this one call, never needing to branch into a separate `submitLessonTurn` call to "wake up" the dialogue. `nextTurn.sessionState.dialogueState` will be `AWAITING_RESPONSE` after the re-teach message, and normal `submitLessonTurn` calls resume from there; the backend will offer a **new, non-identical** check (per AC-9) once the tutor judges the user ready again, delivered the same way any check is delivered — via the `comprehensionCheck` field on a subsequent `submitLessonTurn` response.

```graphql
mutation {
  submitComprehensionCheckAnswer(subtopicId: "sub_02", answer: "Chlorophyll") {
    attempt { result attemptNumber }
    subtopic { status attemptCount }
    advancedToNextSubtopic
    planCompleted
    nextTurn {
      tutorMessage
      sessionState { dialogueState currentSubtopic { title } }
    }
  }
}
```

**Errors:**
- `NO_ACTIVE_CHECK` — if called when the sub-topic isn't currently in `CONFIRMING_UNDERSTANDING` state.

---

### 3.6 Queries
Most reads are straightforward, backed directly by data-model-schema.md entities — no special business logic beyond standard filtering (e.g., `learningPlans(userId, status: IN_PROGRESS)` for a "resume" screen listing all paused subjects, supporting concurrent in-progress plans per PRD confirmation).

### 3.7 `planTranscript`
**Maps to:** the persisted-transcript decision in data-model-schema.md §2.5/§5 — supports both mid-session resume-with-history and post-completion review of a subject.
**Behavior:** Returns every `LessonTurnMessage` for the given `planId`, ordered by `occurredAt`, spanning all sub-topics. The client is expected to render this directly as the chat transcript (excluding `classification`, which isn't exposed on the type at all — see the type definition note in §2).

```graphql
query {
  planTranscript(planId: "plan_1a2b3c") {
    role
    content
    subtopicId
    occurredAt
  }
}
```

**Usage patterns:**
- **Resume:** on loading `ResumeDashboard` → "Resume", call this alongside `lessonSessionState(planId)` to reconstruct the visible chat history before the user's next turn.
- **Review (COMPLETED plans):** call this directly for a `COMPLETED` plan to render a read-only transcript view — no `LessonSessionState` needed since there's no active turn to take.

---

## 4. Full Flow Walkthrough (ties operations to PRD §5)

| PRD Flow Step | Operation(s) |
|---|---|
| 1. Subject Intake | `generateLearningPlan` |
| 2. Plan Generation (shown to user) | (response of above) |
| 3. Plan Confirmation (with optional edits) | `reorderLearningPlan` / `removeSubtopicFromPlan`, then `confirmLearningPlan` (returns opening question inline via `openingTurn`) |
| 4a-c. Lesson Loop (question/answer/hint/confusion) | repeated `submitLessonTurn` |
| 4d. Comprehension check administered | `comprehensionCheck` field on a `submitLessonTurn` response |
| 4d-e. Check submitted, pass/fail routing | `submitComprehensionCheckAnswer` |
| 5. Plan Completion | signaled via `planCompleted: true` / `LearningPlan.status = COMPLETED` |
| 6. Session Resume | `learningPlans(userId, status: IN_PROGRESS)` query, then `planTranscript(planId)` + `sessionState` off the selected plan to resume exactly where they left off, with history intact |
| (new) Review a completed subject | `learningPlans(userId, status: COMPLETED)` query, then `planTranscript(planId)` for a read-only transcript view |

---

## 5. Resolved Decisions (previously open questions)

- ✅ **Opening-message hand-off:** the first tutor message for a sub-topic is always returned inline by the triggering mutation — `confirmLearningPlan.openingTurn` for sub-topic #1, `submitComprehensionCheckAnswer.nextTurn` for both "advance to next sub-topic" and "re-teach after FAILED_RETRY." No sentinel calls anywhere in the contract.
- ✅ **Failed-retry re-teach delivery:** inlined via `nextTurn`, same mechanism as advancing on a pass. The **new, non-identical** check question required by AC-9 is not force-inlined a second time — it surfaces naturally via the `comprehensionCheck` field once a subsequent `submitLessonTurn` call in the re-teach dialogue judges the user ready again, keeping one consistent mechanism for "how a check gets offered" throughout the contract.
- ✅ **Transcript persistence & review mode:** `submitLessonTurn` now persists every message as a `LessonTurnMessage`, and `planTranscript(planId)` returns the full ordered history — used for both resume-with-history and reviewing a `COMPLETED` plan's dialogue after the fact.

## 6. Resolved Decisions (continued)

- ✅ **Auth/session layer:** this is an internal tool with no personal information and minimal data (per-topic learning progress only), so a full auth system was judged disproportionate. Resolved approach: **lightweight named identification, no password.** A one-time "who are you?" prompt creates/looks up a `User` by `display_name`; the backend issues a plain session token/cookie mapping subsequent requests to that `user_id`. `userId`/`planId` remain explicit in the schema as shown above — the session token determines *which* `userId` a request is allowed to act as, rather than replacing the argument. There is no password and no identity verification: a user could claim someone else's name, or query another user's data directly via a guessed UUID. This is an accepted tradeoff for a no-PII internal tool — see task-breakdown-build-plan.md Phase 0 for the full rationale and the revisit trigger (external-facing use, more sensitive data, or more users) if this ever needs to become real auth.

## 7. Open Questions for Stakeholder / User Confirmation

None outstanding at the API-contract level — see task-breakdown-build-plan.md Appendix A for the remaining cross-artifact open items (none of which affect this schema).

---

## 8. Explicit Assumptions in This Artifact (flag any that are wrong)

1. Single GraphQL endpoint, no REST fallback endpoints — per confirmed stack choice.
2. `submitLessonTurn` and `submitComprehensionCheckAnswer` are separate mutations rather than one unified "submit turn" mutation that branches internally — chosen for clarity of contract (deterministic grading vs. LLM dialogue are different concerns) and to match the data model's separation of `ComprehensionCheckAttempt` from `AdaptiveFeedbackEvent`.
3. No subscription/real-time type defined — tutor responses are assumed request/response (synchronous mutation), not streamed. Flag if streaming token-by-token tutor responses is desired for UX (would require a different transport, e.g., GraphQL subscriptions or SSE alongside GraphQL).
4. No pagination arguments on list fields (`subtopics`, `checkAttempts`, `feedbackEvents`, `learningPlans`, `transcript`, `planTranscript`) — acceptable given PRD's bounded sizes (max 10 sub-topics, max 3 attempts/events per cycle), but flag if a user could accumulate enough plans/history over time to warrant `first`/`after` cursor pagination. `planTranscript` is the most likely candidate to eventually need this, since a long multi-sub-topic session could accumulate many messages.
