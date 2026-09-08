# Artifact: Sample Conversation Transcript

**Derived from:** PRD-guided-learning-app.md (§5), system-prompt-socratic-tutor.md, data-model-schema.md, api-contract.md
**Purpose:** An annotated, end-to-end sample session showing every mechanism in the PRD actually firing — plan generation, exploratory Socratic dialogue, hint escalation (levels 1-3), a confusion case, a reveal-request case, a comprehension check pass, and a failed-retry → re-teach → new-check cycle. Intended to double as a **behavioral test case**: each turn is annotated with the backend state and the API operation it corresponds to, so Claude Code can use this to write integration tests or seed fixtures.
**Audience:** Claude Code (test-writing / QA), reviewers validating tutor behavior matches spec.
**Subject used:** Photosynthesis (same example subject as data-model-schema.md, for cross-artifact consistency).

**Annotation format:** Each turn shows the spoken line, then a fenced `state` block showing the relevant backend fields after that turn resolves, and the API operation that produced it. These annotations are for traceability only — the actual product UI would not show them to the user.

---

## Scene 1 — Subject Intake & Plan Generation (PRD §3.2)

**User:** I want to learn photosynthesis

```state
operation: generateLearningPlan(userId: "usr_8f3a1c", subject: "Photosynthesis")
result: LearningPlan { status: DRAFT }
```

**Tutor (plan preview, not dialogue):**
Here's a plan to build up photosynthesis from the ground up:

1. **What Photosynthesis Does** — the overall inputs, outputs, and purpose of the process, before getting into mechanism.
2. **The Light-Dependent Reactions** — how sunlight is converted into chemical energy (ATP and NADPH).
3. **The Calvin Cycle** — how that chemical energy is used to build glucose from CO₂.

Want to start with this order, or rearrange/remove anything first?

**User:** looks good, let's start

```state
operation: confirmLearningPlan(planId: "plan_1a2b3c")
result: ConfirmLearningPlanResult {
  plan: { status: CONFIRMED },
  openingTurn: {
    sessionState: { dialogueState: AWAITING_RESPONSE, currentSubtopic: "sub_01", hintLevel: 0, reframeCount: 0 }
  }
}
```

---

## Scene 2 — Exploratory Dialogue with Hint Escalation & Confusion (sub-topic 1: "What Photosynthesis Does")

**Tutor (openingTurn.tutorMessage):**
Before I tell you anything — if a plant is basically "making its own food," what do you think it needs to take in from its environment to do that?

```state
dialogueState: AWAITING_RESPONSE
```

**User:** Water, I think?

```state
operation: submitLessonTurn(planId, userMessage: "Water, I think?")
classification: ERROR   # partial — correct but incomplete; only one of two inputs named
sessionState: { dialogueState: GIVING_HINT, hintLevel: 1 }
feedbackEvent: { triggerType: ERROR, hintLevel: 1 }
```

**Tutor:**
Water's one piece of it! But think about *where* a plant spends most of its visible surface area — its leaves, out in the open air. What might it be pulling in from the air itself?

**User:** Oxygen?

```state
operation: submitLessonTurn(planId, userMessage: "Oxygen?")
classification: ERROR
sessionState: { dialogueState: GIVING_HINT, hintLevel: 2 }
feedbackEvent: { triggerType: ERROR, hintLevel: 2 }
```

**Tutor:**
Close, but oxygen is actually something plants *release*, not take in, during this process. The gas they're pulling in is the same one we breathe *out* — what gas is that?

**User:** wait I don't really get why plants would want a gas we breathe out, that seems backwards

```state
operation: submitLessonTurn(planId, userMessage: "wait I don't really get why plants would want a gas we breathe out, that seems backwards")
classification: CONFUSION   # not an attempt at the answer — questioning the premise/concept itself
sessionState: { dialogueState: GIVING_HINT, reframeCount: 1 }   # hintLevel NOT incremented for confusion
feedbackEvent: { triggerType: CONFUSION, hintLevel: null }
```

**Tutor:**
Good instinct to question that — it does sound backwards at first! Here's the reframe: animals and plants are basically running opposite trades with each other. We breathe in oxygen and breathe out carbon dioxide as a waste product. Plants do the reverse — they take in *our* waste gas and turn it into food, then release oxygen as *their* byproduct. So what's "waste" for us is raw material for them. With that in mind — what gas do you think a plant is pulling in?

**User:** Oh that makes sense — carbon dioxide

```state
operation: submitLessonTurn(planId, userMessage: "Oh that makes sense — carbon dioxide")
classification: CORRECT
sessionState: { dialogueState: AWAITING_RESPONSE, hintLevel: 0, reframeCount: 0 }   # counters reset after a correct answer
tutorMessage: "Exactly — water and CO2 are the two inputs. I think you've got the idea — ready for a quick check?"
```

**User:** yep ready

```state
operation: submitLessonTurn(planId, userMessage: "yep ready")
sessionState: { dialogueState: CONFIRMING_UNDERSTANDING }
comprehensionCheck: {
  questionType: MULTIPLE_CHOICE,
  questionText: "Which two substances does a plant take in to perform photosynthesis?",
  options: ["Oxygen and glucose", "Water and carbon dioxide", "Nitrogen and water", "Glucose and carbon dioxide"]
}
```

---

## Scene 3 — Comprehension Check: Pass on First Attempt

**Tutor:** Which two substances does a plant take in to perform photosynthesis?
- A) Oxygen and glucose
- B) Water and carbon dioxide
- C) Nitrogen and water
- D) Glucose and carbon dioxide

**User:** B

```state
operation: submitComprehensionCheckAnswer(subtopicId: "sub_01", answer: "B")
grading: EXACT match against correct_option_index -> CORRECT
result: ComprehensionCheckResult {
  attempt: { attemptNumber: 1, result: CORRECT },
  subtopic: { status: PASSED, attemptCount: 1 },
  advancedToNextSubtopic: true,
  planCompleted: false,
  nextTurn: {
    tutorMessage: "Nice work — you've got the inputs and outputs down. Next up: how the plant actually converts sunlight into usable energy. If sunlight hits a leaf, what do you think has to happen first for that light to become something the plant can use?",
    sessionState: { dialogueState: AWAITING_RESPONSE, currentSubtopic: "sub_02", hintLevel: 0 }
  }
}
```

---

## Scene 4 — Reveal Request Flow (sub-topic 2: "The Light-Dependent Reactions")

*(Picking up mid-dialogue on sub-topic 2, after some exchange not fully transcribed here for brevity — assume the user is now stuck after a Level 2 hint.)*

**User:** I have no idea, just tell me

```state
operation: submitLessonTurn(planId, userMessage: "I have no idea, just tell me")
classification: REVEAL_REQUEST
revealRequestsThisQuestion: 0 -> 1   # first request
sessionState: { dialogueState: GIVING_HINT, hintLevel: 2 }   # unchanged — reveal request does not itself change hint level
```

**Tutor:**
I get the instinct, but let's try one more angle first — I think you're closer than you feel. Here's another hint: chlorophyll's whole job is to *catch* something. What is it specifically designed to catch, based on its color and where the plant puts it (leaves, facing the sun)?

**User:** ugh okay just tell me the answer

```state
operation: submitLessonTurn(planId, userMessage: "ugh okay just tell me the answer")
classification: REVEAL_REQUEST
revealRequestsThisQuestion: 1 -> 2   # second consecutive request for the same question
```

**Tutor:**
Alright — chlorophyll absorbs light energy (mostly red and blue wavelengths, which is why it reflects green) and uses that energy to excite electrons, kicking off the whole light-dependent reaction chain. Quick check on that: in one sentence, what is chlorophyll's main job in this process?

```state
event logged: AdaptiveFeedbackEvent { triggerType: REVEAL_REQUEST, revealed: true }
sessionState: { dialogueState: AWAITING_RESPONSE }   # re-check question issued immediately per AC-2, hint counters reset
```

**User:** it absorbs light energy and uses it to excite electrons

```state
operation: submitLessonTurn(planId, userMessage: "it absorbs light energy and uses it to excite electrons")
classification: CORRECT
tutorMessage: "That's exactly it. Ready for the check on this sub-topic?"
```

---

## Scene 5 — Comprehension Check: Failed Retry → Re-teach → New Check → Pass

*(Later, on sub-topic 3: "The Calvin Cycle" — user has passed exploratory dialogue and reached the check.)*

**Tutor:** True or false: the Calvin Cycle requires direct sunlight to occur.

**User:** True

```state
operation: submitComprehensionCheckAnswer(subtopicId: "sub_03", answer: "True")
grading: EXACT match -> INCORRECT   # correct answer is "False" — Calvin Cycle uses stored energy (ATP/NADPH), not direct light
result: { attempt: { attemptNumber: 1, result: INCORRECT }, subtopic: { attemptCount: 1 }, advancedToNextSubtopic: false, nextTurn: null }
```

**Tutor:** Not quite — want to try again?

**User:** False?

```state
operation: submitComprehensionCheckAnswer(subtopicId: "sub_03", answer: "False?")
grading: normalized "false?" -> "false" -> CORRECT match... 
```

*(Normalization confirmed in data-model-schema.md §3: matching trims whitespace, lowercases, and strips trailing punctuation before comparison, uniformly across all match_modes including `exact`. So `"False?"` normalizes to `"false"` and correctly matches an accepted phrase of `"false"`.)*

**This answer grades CORRECT:**

```state
result: {
  attempt: { attemptNumber: 2, result: CORRECT },
  subtopic: { status: PASSED, attemptCount: 2 },
  advancedToNextSubtopic: true,
  planCompleted: true,   # this was the last sub-topic
  nextTurn: null
}
```

**Tutor:** That's right — the Calvin Cycle runs on the ATP and NADPH already made during the light-dependent reactions, so it doesn't need light directly (though it usually happens during the day since it depends on those products). That completes the full photosynthesis plan — nice work across all three sub-topics!

---

## Scene 5b — Alternate Branch: Max Attempts Reached → Re-teach → New Check

*(Alternate version of Scene 5, shown separately as its own test case, illustrating AC-9 when the user does NOT recover within the attempt limit.)*

**Attempt 1:** "True" → `INCORRECT` (`attemptCount: 1`)
**Attempt 2:** "I think true" → `INCORRECT` (`attemptCount: 2`)
**Attempt 3:** "True, because it happens after the light reactions" → `INCORRECT` (`attemptCount: 3` — max reached)

```state
operation: submitComprehensionCheckAnswer(subtopicId: "sub_03", answer: "True, because it happens after the light reactions")
result: {
  attempt: { attemptNumber: 3, result: INCORRECT },
  subtopic: { status: FAILED_RETRY, attemptCount: 0 },   # reset per data-model-schema.md §2.3
  advancedToNextSubtopic: false,
  planCompleted: false,
  nextTurn: {
    tutorMessage: "Let's back up and rebuild this from a different angle. The light-dependent reactions and the Calvin Cycle are like two steps in a relay race: step one (needing light) produces two 'batons' — ATP and NADPH. Step two, the Calvin Cycle, just needs those batons, not the original light itself, to build sugar. So — does the Calvin Cycle need light directly, or does it just need what light already produced?",
    sessionState: { dialogueState: AWAITING_RESPONSE, hintLevel: 0, reframeCount: 0 }
  }
}
```

**User:** oh — it just needs what light already made, so it doesn't need light directly

```state
operation: submitLessonTurn(planId, userMessage: "oh — it just needs what light already made, so it doesn't need light directly")
classification: CORRECT
tutorMessage: "Exactly. Ready to try the check again?"
sessionState: { dialogueState: CONFIRMING_UNDERSTANDING }
comprehensionCheck: {
  questionType: MULTIPLE_CHOICE,   # NEW question, different format from the original true/false, satisfying AC-9's "not identical" requirement
  questionText: "What does the Calvin Cycle directly use to build glucose?",
  options: ["Direct sunlight", "ATP and NADPH from the light reactions", "Carbon dioxide alone", "Chlorophyll"]
}
```

**User:** ATP and NADPH from the light reactions

```state
operation: submitComprehensionCheckAnswer(subtopicId: "sub_03", answer: "ATP and NADPH from the light reactions")
result: { attempt: { attemptNumber: 1, result: CORRECT }, subtopic: { status: PASSED }, advancedToNextSubtopic: true, planCompleted: true }
```

---

## 1. Coverage Checklist (for traceability against PRD/spec artifacts)

| Behavior | Scene | AC / Spec Reference |
|---|---|---|
| Plan generation (3+ sub-topics, shown before lessons start) | 1 | PRD AC-4, AC-6 |
| Plan confirmation required before Lesson 1 | 1 | PRD AC-5 |
| Opening turn poses a question before any explanation | 1→2 | PRD AC-1 |
| `error` classification + hint escalation levels 1→2 | 2 | system-prompt §1.3, PRD AC-11 |
| `confusion` classification + reframe (no hint-level increment) | 2 | system-prompt §1.2/§1.3, PRD AC-10 |
| Comprehension check not revealed before an attempt | 3 | PRD AC-3 |
| Pass → auto-advance with inline next opening turn | 3 | api-contract.md §3.5 |
| `reveal_request` decline-once, reveal-on-second, forced re-check | 4 | PRD AC-2 |
| Failed check with attempts remaining → same question retried | 5b (attempts 1-2) | PRD AC-9 |
| Max attempts reached → `FAILED_RETRY` → inline re-teach | 5b | PRD AC-9 |
| New, non-identical check question after re-teach | 5b | PRD AC-9 |
| Plan completion after final sub-topic passes | 5 / 5b | PRD §5 (flow step 5) |

---

## 2. Explicit Assumptions / Flags in This Artifact

1. Scene 2's classification calls (e.g., "Water, I think?" → `ERROR` rather than `CORRECT` for a partial answer) are illustrative of intended LLM judgment, not a deterministic rule — actual classification behavior depends on the tutoring LLM's judgment per the system prompt, and may vary. This transcript represents *expected/target* behavior for eval purposes, not a guaranteed literal output.
2. Sub-topic 2's full dialogue leading up to the Level 2 hint (before Scene 4 picks up) was omitted for brevity/focus — Claude Code should not assume Scene 4 is a complete sub-topic transcript on its own.
