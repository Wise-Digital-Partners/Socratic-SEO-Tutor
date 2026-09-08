# Artifact: Evaluation Rubric — Comprehension Checks

**Derived from:** PRD-guided-learning-app.md (§3.3, AC-7, AC-8, AC-9), data-model-schema.md (§2.5, §3), api-contract.md (§3.5)
**Scope:** This rubric governs exactly one concern — **how a comprehension check is authored so it can be graded well, and how the pass/fail decision is made.** It does NOT cover exploratory-dialogue classification (`correct` / `error` / `confusion` / `ambiguous` / `reveal_request`) — that logic lives entirely in system-prompt-socratic-tutor.md and is out of scope here.
**Audience:** Claude Code (implements grading + plan-generation prompts that author check questions), content-quality reviewers.

---

## 1. Resolved Design Baseline (read this first)

Per data-model-schema.md and api-contract.md, the following is already decided and this rubric builds on top of it rather than re-litigating it:

- Each sub-topic has **one comprehension check**, which is a **single question** (not a multi-question quiz). PRD §3.3 raised "score ≥ X% on a multi-question quiz" as one possible pass-threshold model — the implemented design instead uses **single question, answered correctly within a maximum of 3 attempts**. This rubric documents guidance for authoring that one question well, not for scoring a multi-item quiz.
- Grading is **fully deterministic string matching** — no LLM judgment is involved in the pass/fail decision itself (confirmed in data-model-schema.md §3, no `pending_review` result state).
- All matching normalizes both the accepted answer(s) and the user's answer: trim whitespace, collapse internal whitespace, lowercase, strip trailing punctuation — uniformly across `contains_all`, `contains_any`, and `exact`.
- Pass/fail at the `SubTopic` level: `PASSED` on first correct attempt (however many attempts it took, up to 3); `FAILED_RETRY` after 3 incorrect attempts, followed by a mandatory re-teach and a new, non-identical question (PRD AC-9).

**What this rubric adds on top of that baseline:** guidance for *authoring* a check question (during `generateLearningPlan` / sub-topic setup) such that the deterministic grader above produces fair, expected results — i.e., how to avoid writing a technically-passing question that's actually a bad test of understanding, or a technically-correct answer key that unfairly fails a reasonable user response.

---

## 2. Pass/Fail Decision Rule (authoritative summary)

A comprehension check attempt is scored `CORRECT` if and only if the normalized user answer satisfies the question's `match_mode` against its `question_payload`, per data-model-schema.md §3:

| `question_type` | Scored `CORRECT` when... |
|---|---|
| `multiple_choice` | the submitted option exactly equals `correct_option_index`'s option (post-normalization; e.g., matching by option text or letter, whichever the client submits — see §4.1 for the letter-vs-text flag) |
| `short_free_text`, `match_mode: contains_all` | the normalized answer contains **every** normalized phrase in `accepted_phrases` |
| `short_free_text`, `match_mode: contains_any` | the normalized answer contains **at least one** normalized phrase in `accepted_phrases` |
| `short_free_text` / `short_numeric_or_word`, `match_mode: exact` | the normalized answer **equals** one of the normalized `accepted_phrases` / `accepted_answers` |

A `SubTopic` is `PASSED` the moment one attempt scores `CORRECT`. It becomes `FAILED_RETRY` only after 3 consecutive `INCORRECT` attempts on the *same* question (attempt counter resets to 0 entering `FAILED_RETRY`, per data-model-schema.md §2.3).

---

## 3. Authoring Rubric by Question Type

### 3.1 `multiple_choice`
**Goal:** test recognition/discrimination of the correct concept from plausible-but-wrong alternatives — not test reading comprehension or trick the user.

**Do:**
- Write exactly one unambiguously correct option.
- Make distractors plausible misconceptions tied to the sub-topic (e.g., a common mix-up), not random or absurd options — a distractor that's obviously wrong doesn't test anything.
- Keep option lengths roughly balanced (an answer that's noticeably longer/more detailed than the others is a well-known "tell").

**Don't:**
- Use "All of the above" / "None of the above" — they don't test the specific concept and complicate grading against a single `correct_option_index`.
- Write double negatives or multi-part conditions in the question stem.
- Reuse the *identical* question after a `FAILED_RETRY` re-teach (violates AC-9) — vary the phrasing, the framing, or switch question type entirely (e.g., MC → short answer) for the retry.

**Example — good:**
> Which two substances does a plant take in to perform photosynthesis?
> A) Oxygen and glucose  B) Water and carbon dioxide  C) Nitrogen and water  D) Glucose and carbon dioxide

**Example — bad (why):**
> Which of the following are NOT things a plant does NOT need? *(double negative, confusing, poor test of the actual concept)*

---

### 3.2 `short_free_text`
**Goal:** test whether the user can articulate the concept in their own words, without requiring them to guess the exact phrasing the author had in mind.

**Do:**
- Choose `contains_all` when the answer genuinely requires multiple distinct ideas to be present (e.g., "light energy" AND "chemical energy" for a conversion question).
- Choose `contains_any` when several different correct phrasings exist and any one of them fully answers the question (e.g., accept "carbon dioxide", "CO2", "co₂").
- Populate `accepted_phrases` generously with realistic synonyms and common phrasings a correct-thinking user might actually type — this list IS the grading logic; an incomplete list produces false negatives (a genuinely correct user marked wrong), which is the single biggest risk with deterministic grading.
- Keep required phrases short and conceptually essential — avoid requiring exact technical vocabulary the tutor hasn't explicitly introduced yet.

**Don't:**
- Use `contains_all` with more than 2–3 required phrases — every additional required phrase compounds the false-negative risk.
- Require a specific sentence structure or word order (matching is order-independent by design — don't fight that by relying on phrase adjacency).
- Use `short_free_text` for something that's really a yes/no or single-term answer — use `short_numeric_or_word` or restructure as `multiple_choice` instead.

**Example — good:**
```json
{
  "question_text": "In one sentence, what is chlorophyll's main job in the light-dependent reactions?",
  "question_type": "short_free_text",
  "question_payload": {
    "accepted_phrases": ["absorb light", "absorbs light", "capture light", "captures light"],
    "match_mode": "contains_any"
  }
}
```

**Example — bad (why):**
```json
{
  "question_payload": {
    "accepted_phrases": ["chlorophyll absorbs light energy and uses it to excite electrons which begin the electron transport chain"],
    "match_mode": "exact"
  }
}
```
*(Requires the user to reproduce a full technical sentence verbatim — nearly guaranteed to false-negative a correct-thinking user who phrases it differently.)*

---

### 3.3 `short_numeric_or_word`
**Goal:** test recall of a specific fact, count, or single term where there's a genuinely small, enumerable set of correct forms.

**Do:**
- Enumerate all reasonable equivalent forms in `accepted_answers` (e.g., digit and word form: `["6", "six"]`).
- Use the `tolerance` field (data-model-schema.md §3) for any numeric answer derived from calculation rather than pure recall, so minor rounding isn't unfairly marked wrong.
- Keep the expected answer to a single word, number, or short fixed phrase (e.g., a named term, a count, a date).

**Don't:**
- Use this type for anything requiring explanation or reasoning in the answer — that's `short_free_text`'s job.
- Leave `tolerance: null` for a calculated (not recalled) numeric answer — this will false-negative correct answers that round differently.

**Example — good:**
```json
{
  "question_text": "How many main sub-topics are in a typical photosynthesis breakdown, per this plan?",
  "question_type": "short_numeric_or_word",
  "question_payload": { "accepted_answers": ["3", "three"], "tolerance": null }
}
```

---

## 4. Cross-Cutting Guidance

### 4.1 Multiple-choice answer submission format (resolved)
The client submits the option's **text** (not a letter like "B") as `answer` to `submitComprehensionCheckAnswer`. The backend compares the normalized submitted text against the normalized option string at `correct_option_index`. This is more robust than a letter/index submission if the UI ever shuffles option order, and keeps `submitComprehensionCheckAnswer`'s `answer: String!` argument uniform across all three `question_type`s (no special-casing for multiple choice).

### 4.2 Avoiding false negatives (the primary grading risk)
Because grading is deterministic string matching, not LLM judgment, the single biggest failure mode is a **correct user being marked wrong** because their exact phrasing wasn't anticipated. Mitigations, in priority order:
1. Prefer `multiple_choice` or `short_numeric_or_word` over `short_free_text` wherever the concept allows — they have a bounded, enumerable answer space and are inherently safer under string matching.
2. When `short_free_text` is necessary, err toward `contains_any` with a generous phrase list over `contains_all` or `exact`.
3. Sub-topic authoring (during `generateLearningPlan`) should generate `accepted_phrases` by considering how a learner who just went through the preceding exploratory dialogue would plausibly phrase the answer — not just the "textbook" phrasing.

### 4.3 Relationship to the exploratory "readiness" gate
system-prompt-socratic-tutor.md §1.4 defines when the tutor *offers* a comprehension check (at least one reasoned `correct` response during exploratory dialogue). That readiness judgment is **not** scored by this rubric — it's a dialogue-flow decision, not a pass/fail decision. This rubric only governs the check itself, once offered. Keeping these separate avoids the readiness heuristic (soft, LLM-judged) contaminating the pass/fail decision (hard, deterministic) — a deliberate design boundary between the two artifacts.

---

## 5. Explicit Assumptions in This Artifact (flag any that are wrong)

1. One question per comprehension check (not a multi-question quiz) — inherited from the schema/API design, not independently re-confirmed with you here; flagging since PRD §3.3 originally floated a multi-question quiz as an option.
2. Multiple-choice submission format is confirmed as option text (§4.1), not letter/index.
3. `accepted_phrases` / `accepted_answers` lists are assumed to be authored automatically at plan-generation time (by the same LLM call that generates the sub-topic), not hand-curated by a human reviewer — consistent with the app having no content-authoring UI in v1 (PRD Non-Goals).

## 6. Open Questions for Stakeholder / User Confirmation

- Should there be any human/manual review step for auto-generated `accepted_phrases` lists before a plan goes live, or is fully automatic generation acceptable for v1 (given the false-negative risk in §4.2)?
