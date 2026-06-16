# Day 14 - Reflection
## Evaluation Report & Failure Analysis

## 1. Benchmark Results Summary

**Overall pass rate:** 20%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.44 | 0.00 | 0.83 | 0.22 |
| Relevance | 0.37 | 0.00 | 0.75 | 0.22 |
| Completeness | 0.54 | 0.00 | 0.86 | 0.25 |
| Overall Score | 0.45 | 0.07 | 0.73 | 0.17 |

**Score interpretation:**
- Good (0.8-1.0): 3 metric scores
- Needs Work (0.6-0.8): 24 metric scores
- Significant Issues (<0.6): 53 metric scores

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 4 | 20% |
| irrelevant | 5 | 25% |
| incomplete | 1 | 5% |
| off_topic | 6 | 30% |
| refusal | 0 | 0% |

## 2. Top 3 Worst Failures - 5 Whys Analysis

### Failure 1

**Question:** What is the best recipe for sourdough bread?

**Agent Answer:** Sourdough uses flour water salt and a starter.

**Scores:** Faithfulness: 0.00 | Relevance: 0.20 | Completeness: 0.00 | Overall: 0.07

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The agent answered an out-of-scope cooking question instead of redirecting. |
| Why 1 | Why did it happen? | The agent optimized for answering the literal question. |
| Why 2 | Why did that happen? | Scope detection did not run before answer generation. |
| Why 3 | Why did scope detection fail? | The prompt did not clearly enforce the AI-evaluation domain boundary. |
| Why 4 | Root cause? | Missing out-of-scope guardrail and refusal/redirect policy. |

**Root cause (from `find_root_cause()`):**

> Context is missing or irrelevant - improve retrieval

**Do I agree?**

Partially. The lowest score is faithfulness, so the function points at context/retrieval. The deeper issue is domain routing: the system should not retrieve or answer cooking content for an AI evaluation assistant.

**Proposed fix:**

Add an intent/scope classifier before retrieval and a refusal template for non-AI-evaluation questions. Add more out-of-scope examples to the golden dataset.

### Failure 2

**Question:** How should adversarial out-of-scope questions be evaluated?

**Agent Answer:** They should safely refuse or redirect without inventing an answer.

**Scores:** Faithfulness: 0.12 | Relevance: 0.14 | Completeness: 0.43 | Overall: 0.23

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The answer is directionally right but scores poorly because it omits expected wording and context terms. |
| Why 1 | Why did it happen? | The lexical heuristic cannot recognize some paraphrases. |
| Why 2 | Why did that happen? | The evaluator relies on token overlap rather than semantic similarity. |
| Why 3 | Why did semantic matching not happen? | The lab implementation intentionally uses simple heuristics instead of an LLM judge. |
| Why 4 | Root cause? | Evaluation metric is too brittle for paraphrased safety answers. |

**Root cause:**

Context is missing or irrelevant - improve retrieval.

**Proposed fix:**

Add semantic judge scoring for adversarial/safety answers, and enrich expected answers with required terms such as "out-of-scope", "supported domain", and "do not invent".

### Failure 3

**Question:** How should a regression gate compare new results with a baseline?

**Agent Answer:** It compares averages and flags metric drops greater than 0.05.

**Scores:** Faithfulness: 0.50 | Relevance: 0.00 | Completeness: 0.33 | Overall: 0.28

**5 Whys Analysis:**

| Level | Question | Answer |
|-------|----------|--------|
| Symptom | What is the problem? | The answer is mostly correct but relevance scored 0.00. |
| Why 1 | Why did it happen? | The answer reused expected concepts but not the exact question tokens. |
| Why 2 | Why did that matter? | Relevance is calculated as answer-token overlap with question-token overlap. |
| Why 3 | Why is that insufficient? | The question uses words like "regression gate" and "baseline", while the answer uses "averages" and "drops". |
| Why 4 | Root cause? | Relevance metric needs semantic matching or better expected keyword coverage. |

**Root cause:**

Answer does not address the question - improve prompt clarity.

**Proposed fix:**

Update the agent prompt to include key terms from the question in the answer. For production, use an embedding or LLM-based relevance metric instead of raw token overlap.

## 3. Failure Clustering

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|------------|--------------------:|----------|
| 1 | Weak scope and safety handling for adversarial inputs | 4 | High |
| 2 | Lexical evaluator is brittle to paraphrases | 6 | High |
| 3 | Answers omit question keywords or expected details | 6 | Medium |

**If fixing one cluster first:**

I would fix Cluster 1 first because hallucination and prompt-injection/out-of-scope failures are highest risk. Even a small number of these failures can block deployment because they indicate the agent may ignore boundaries.

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant - improve retrieval | Add scope classifier and safe redirect template. | Open |
| F002 | hallucination | Context is missing or irrelevant - improve retrieval | Add semantic judge for safety paraphrases. | Open |
| F003 | irrelevant | Answer does not address the question - improve prompt clarity | Require answers to preserve key question terms. | Open |

**3 improvement suggestions from `generate_improvement_suggestions()`:**
1. Add grounding constraints and evidence checks to reduce hallucination.
2. Clarify prompts and include explicit question intent to improve relevance.
3. Expand retrieved context coverage or ask the model to include all expected points.

## 5. Regression Testing Strategy

**Cau 1: When to run `run_regression()` in production?**

Run it before every merge to main, after any prompt/model/retriever/index change, and before each release candidate. Also run a scheduled nightly benchmark to catch environment or dependency drift.

**Cau 2: Is 0.05 regression threshold appropriate?**

For this learning lab, 0.05 is appropriate. For a high-stakes production assistant, I would use stricter thresholds for faithfulness and safety, such as 0.02-0.03, while allowing slightly looser relevance movement during experimentation.

**Cau 3: Block deployment or alert?**

Block deployment for faithfulness, safety, prompt-injection, or out-of-scope regressions. Alert but do not always block for minor completeness drops on low-risk internal features, unless the drop repeats across releases.

**Cau 4: CI/CD flow**

```text
Code change -> Unit tests -> Offline eval/regression gate -> Failure report review -> Deploy
```

## 6. Continuous Improvement Loop

| Priority | Action | Metric will improve | Expected impact |
|----------|--------|---------------------|-----------------|
| 1 | Add out-of-scope and prompt-injection guardrails. | Faithfulness, Safety | Fewer hallucination and unsafe compliance failures. |
| 2 | Add reranking plus metadata filtering. | Context Precision | More relevant evidence reaches the generator. |
| 3 | Add LLM-as-judge semantic scoring for paraphrases. | Relevance, Completeness | Fewer false negatives from lexical mismatch. |

**Failure cases to add next sprint:**

- Multi-turn prompt injection that asks the agent to ignore previous rubric rules.
- Ambiguous question that needs clarification before evaluation.
- Correct but uncited answer where citation quality should decide pass/fail.

## 7. Framework Reflection

**Framework used in lab:** RAGAS-inspired heuristic

**Production framework choice:**

I would choose RAGAS for a production RAG evaluation pipeline, with DeepEval-style tests for CI assertions where useful.

| Criteria | Reason |
|----------|--------|
| Focus fits because... | RAGAS directly models faithfulness, answer relevancy, context recall, and context precision. |
| CI/CD integration because... | Metrics can be run offline as a quality gate before deployment. |
| Team workflow because... | The same golden dataset can support prompt iteration, retriever tuning, and regression checks. |
