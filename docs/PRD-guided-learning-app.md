# Product Requirements Document: Guided Learning App (Socratic Tutor)

**Status:** Draft v1.0
**Audience:** Claude Code (implementation agent), engineers, reviewers
**Purpose:** Define what the product must do so downstream specs (data models, API contracts, system prompts, UI specs) can be built against a single source of truth.

---

## 1. Product Overview

The Guided Learning App is an interactive tutor that teaches complex subjects through **Socratic dialogue** rather than direct answers. Instead of handing a user a finished explanation, it breaks a subject into a sequenced curriculum, walks the user through each sub-topic using probing questions, checks comprehension before advancing, and adapts its explanations when the user struggles.

The product is **not** a chatbot that answers questions on demand. It is a **guided, stateful, multi-session learning flow** with a defined structure: plan → lesson → check → adapt → advance.

---

## 2. Goals and Non-Goals

### 2.1 Goals
- Teach a user-specified subject through structured, sequenced sub-topics.
- Use questioning (not lecturing) as the default teaching mode.
- Verify understanding before allowing progression.
- Detect and respond differently to "confusion" vs. "a wrong answer."
- Persist progress so a user can resume a learning plan across sessions.

### 2.2 Non-Goals (v1)
- Not a general-purpose Q&A assistant or search tool.
- Not a content authoring platform (users don't upload/author curricula manually in v1).
- Not multi-user/classroom/teacher-dashboard functionality in v1.
- Not grading against external standards (e.g., school grading systems) in v1.
- No monetization, pricing, or account-tier logic in v1 (assumption — flag if incorrect).

---

## 3. Core Mechanisms (Functional Pillars)

### 3.1 Socratic Prompting & Scaffolding
The tutor must default to guiding reasoning via probing questions rather than stating answers outright.

- The tutor asks a question, waits for a user response, and evaluates that response before proceeding.
- The tutor may only give a direct, complete answer when explicitly permitted by the escalation rules in §3.4 or if the user explicitly and repeatedly asks to be told the answer (see Acceptance Criteria).
- Every "turn" in a lesson should map to one of a small set of dialogue states (see `system-prompt` spec, to be produced separately): `posing_question`, `awaiting_response`, `evaluating_response`, `giving_hint`, `confirming_understanding`, `advancing`.

**Acceptance Criteria**
- AC-1: Given an active lesson, when the tutor introduces a new concept, it must pose at least one probing question before offering any explanation of that concept.
- AC-2: If a user explicitly asks "just tell me the answer" twice in a row for the same question, the tutor is permitted to give a direct answer, but must log this as a "revealed" event (see data model) and briefly re-check understanding afterward with a simpler follow-up question.
- AC-3: The tutor must never reveal the final answer to a comprehension-check question before the user has submitted at least one attempt (see §3.3).

---

### 3.2 Custom Learning Plans
When a user names a subject, the tool must generate a sequenced curriculum of sub-topics before the first lesson begins.

- Input: a free-text subject (e.g., "photosynthesis," "supply and demand," "big-O notation").
- Output: an ordered list of sub-topics (a "Learning Plan") presented to the user for confirmation before lessons start.
- The user must be able to see the full plan up front, not just the first topic.
- The user should be able to reorder, skip, or remove sub-topics from the plan before starting (assumption — flag if editing should be post-v1).

**Acceptance Criteria**
- AC-4: Given a subject input, the system generates a Learning Plan of 3–10 sub-topics (exact bounds configurable; default min 3, max 10 — assumption, adjust in data model if wrong).
- AC-5: The Learning Plan must be shown to the user and require explicit confirmation (or edit-then-confirm) before Lesson 1 begins.
- AC-6: Each sub-topic in the plan must have: a title, a one-sentence description of what will be covered, and an estimated relative difficulty or ordering rationale (e.g., "builds on sub-topic 2").

---

### 3.3 Comprehension Checks & Quizzes
At regular intervals, the tool pauses to give practice problems, multiple-choice questions, or short quizzes, and requires a passing result before advancing.

- Checks occur at minimum: once per sub-topic, before moving to the next sub-topic.
- Question types supported in v1: multiple choice, short free-text answer, and short numeric/word answer. (Assumption: no file upload, drawing, or code-execution-based questions in v1.)
- A "pass" threshold must be defined per check (e.g., answer correct within N attempts, or score ≥ X% on a multi-question quiz).
- Failing a check should not simply block the user — it should route into the Adaptive Feedback flow (§3.4), not a dead end.

**Acceptance Criteria**
- AC-7: The user cannot advance to sub-topic N+1 until they have passed the comprehension check for sub-topic N, as defined by the passing rule in the Evaluation Rubric (separate artifact).
- AC-8: Every comprehension check attempt (correct or incorrect) must be recorded with a timestamp, the question asked, the user's answer, and the evaluation result.
- AC-9: If a user fails a comprehension check after the maximum allowed attempts (default: 3 — assumption), the system must offer a simplified re-teach of the sub-topic followed by a new (not identical) check, rather than permanently blocking progress.

---

### 3.4 Adaptive Feedback
On mistakes or confusion, the tutor adjusts explanation depth (hints, analogies, simplified breakdowns) instead of immediately revealing the answer.

- The system must distinguish between **user error** (attempted the reasoning, got it wrong) and **user confusion** (expressed not understanding the question/concept itself, e.g., "I don't get what this is asking").
- Response to error vs. confusion should differ:
  - **Error** → a hint that narrows the reasoning gap (Level 1), then a more specific hint (Level 2), then a worked partial example (Level 3), before any full answer reveal.
  - **Confusion** → a re-explanation using a different framing or analogy, not a harder hint.
- Hint escalation must be bounded and explicit (exact levels/count belong in the system-prompt spec, but the PRD requires that escalation be finite and observable/loggable).

**Acceptance Criteria**
- AC-10: The system must classify each non-passing user response as either `error` or `confusion` (or `ambiguous`, routed to a clarifying question) before choosing a feedback strategy.
- AC-11: Hint escalation must never exceed a fixed maximum number of levels (default: 3 — assumption) before either the user passes or the system offers a direct answer with re-check (per AC-2/AC-9).
- AC-12: Each adaptive feedback event must be logged with: trigger type (error/confusion), hint level given, and the content of the hint, for later analysis and tutor-quality evaluation.

---

## 4. User Stories

| ID | As a... | I want to... | So that... |
|----|---------|---------------|------------|
| US-1 | learner | enter a subject I want to learn | the app builds me a structured plan instead of a wall of text |
| US-2 | learner | see the full learning plan before starting | I know what I'm committing to and can adjust it |
| US-3 | learner | be asked questions instead of told answers | I actually build understanding, not just read passively |
| US-4 | learner | get a hint when I'm stuck instead of the answer | I stay in the productive struggle zone |
| US-5 | learner | take a short quiz before moving to the next topic | I know I actually understood the material |
| US-6 | learner | resume where I left off in a later session | I don't lose progress across sessions |
| US-7 | learner | ask to just be told the answer if I'm truly stuck | I'm not permanently blocked from progressing |
| returning learner | learner | see my past performance/progress summary | I can tell how I'm doing over time |

---

## 5. High-Level User Flow

1. **Subject Intake** — user enters a subject.
2. **Plan Generation** — system generates and presents a Learning Plan (ordered sub-topics).
3. **Plan Confirmation** — user confirms or edits the plan.
4. **Lesson Loop** (repeats per sub-topic):
   a. Tutor introduces the sub-topic with a probing question (not a lecture).
   b. User responds.
   c. Tutor evaluates: correct → proceed deeper; error → hint escalation; confusion → re-explain.
   d. Comprehension check administered.
   e. Pass → advance to next sub-topic. Fail (after max attempts) → simplified re-teach → new check.
5. **Plan Completion** — summary of performance across all sub-topics; option to review weak areas or start a new subject.
6. **Session Resume** — if a user leaves mid-plan, they can return and resume at their exact last state.

*(A full sample conversation transcript demonstrating this flow end-to-end will be produced as a separate artifact.)*

---

## 6. Data & State Requirements (Summary Only)

The following entities must exist in some form (full schema to be a separate artifact):
- **Learning Plan** — subject, ordered sub-topics, status, created/confirmed timestamps.
- **Sub-topic** — title, description, ordering, status (not started / in progress / passed / failed-retry).
- **Lesson Session State** — current sub-topic, current dialogue state, attempt count, hint level.
- **Comprehension Check Attempt** — question, user answer, result, timestamp, attempt number.
- **Adaptive Feedback Event** — trigger type (error/confusion/ambiguous), hint level, content.
- **User Progress Summary** — aggregate pass rate, time spent, sub-topics completed, subjects completed historically.

---

## 7. Explicit Assumptions (flag any that are wrong)

1. Single-user experience in v1 — no classroom/multi-learner features.
2. Text-based interaction only in v1 — no voice, image, or drawing input.
3. Learning Plans default to 3–10 sub-topics unless told otherwise.
4. Max 3 hint-escalation levels and max 3 comprehension-check attempts before re-teach, both configurable.
5. Users can edit a generated Learning Plan before confirming it (reorder/skip/remove), but cannot hand-author one from scratch in v1.
6. No third-party LMS integration (Canvas, Google Classroom, etc.) in v1.
7. Backend/frontend technology stack is undetermined — to be confirmed before API contract and UI component specs are finalized.

---

## 8. Out of Scope (v1)

- Final production code (this PRD feeds Claude Code; it does not contain implementation).
- Monetization, billing, or account tiers.
- Multi-learner/classroom management.
- Content authoring tools for instructors.
- Non-text input modalities (voice, image, handwriting recognition).

---

## 9. Open Questions for Stakeholder / User Confirmation

- Should the Learning Plan be editable by the user pre-confirmation, or system-generated only (Assumption 5 above)?
- What subject domains are in scope — is this general-purpose (any subject) or constrained to specific domains (e.g., STEM only)?
- Should there be a difficulty/level setting (beginner/intermediate/advanced) as part of Subject Intake, or is difficulty inferred adaptively during the lesson?
- Is session resume required to work across devices (i.e., server-persisted state) or is local/browser persistence acceptable for v1?

---

## 10. Downstream Artifacts Derived From This PRD

- Socratic Tutor System Prompt (dialogue states, hint escalation logic, confusion-vs-error detection rules)
- Data Model / Schema (Learning Plan, Lesson State, Progress, Quiz Attempts)
- API Contract (endpoints for plan generation, lesson turns, comprehension check submission, progress retrieval)
- UI Component Specs (Plan view, Lesson/Dialogue view, Quiz view, Progress summary view)
- Sample Conversation Transcripts (end-to-end behavioral test cases)
- Evaluation Rubric (defining "comprehension demonstrated" per question type)
- Task Breakdown / Build Plan for Claude Code
