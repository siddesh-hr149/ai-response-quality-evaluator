# Research Analysis: AI Response Quality Evaluator Agent

**Document status:** Initial research synthesis and implementation proposal  
**Date:** 19 October 2026  
**Scope:** LLM-as-a-judge evaluation, RAG grounding, hallucination analysis, benchmark selection, and reliability validation.

## 1. Executive summary

The proposed system evaluates AI-generated answers across relevance, factual accuracy, source grounding/hallucination risk, and completeness. It combines structured LLM judging with retrieval of reference evidence, a deterministic aggregation layer, batch processing, and a dashboard.

The main research finding is that this is a measurement system, not simply a collection of prompts. LLM judges can produce useful scalable assessments, but their scores can be biased, inconsistent, and sensitive to rubric and prompt design. Therefore, the project must include a human-reviewed validation set, repeatability checks, evidence auditing, and clear treatment of insufficient evidence.

**Recommended MVP:** Start with a transparent, mostly deterministic pipeline. Use four specialized judge functions that return schema-validated outputs, then use a deterministic Verdict service to aggregate them. Use a source-aware claim-verification process for grounding. Add a graph-based orchestration framework only if the workflow's complexity warrants it.

## 2. Problem definition

Manual review of AI answers is slow and difficult to apply consistently across many questions, models, and versions. A useful evaluator should answer:

- Does the answer address the user's question?
- Are its factual claims supported by the supplied reference answer or source documents?
- Which claims are contradicted, unsupported, or impossible to assess from the available evidence?
- Does the response omit required aspects?
- How reliable are the evaluator's own judgments?
- How does quality vary across a dataset, model, or release?

The system should not claim to establish universal truth. It assesses responses against an explicit evidence set and evaluation rubric, and should state when evidence is absent, ambiguous, conflicting, or stale.

## 3. Research questions

1. How well do LLM judges align with human reviewers for each evaluation dimension?
2. Does source retrieval improve the precision of factual and hallucination judgments?
3. Which claim-level labels best distinguish false claims from claims that are merely unsupported by the supplied source?
4. How stable are scores across repeated runs, judge models, and small prompt variations?
5. Can a weighted score plus explicit critical-failure rules produce useful verdicts without concealing serious errors?
6. Which dataset mix provides useful coverage without overfitting to a single benchmark or domain?

## 4. Related approaches and implications

### 4.1 RAGAS

RAGAS documents metrics such as faithfulness, response relevancy, context precision, context recall, and factual correctness. Its faithfulness approach decomposes a response into claims and checks whether the retrieved context supports them. This is closely related to this project's grounding judge.

**Implication:** Use established metric definitions as reference points rather than inventing vague labels. The project can implement its own explainable workflow while comparing selected outputs with RAGAS metrics during validation. RAGAS is a useful baseline, not proof that the evaluator is correct.

Reference: https://docs.ragas.io/en/latest/concepts/metrics/available_metrics/ and https://github.com/vibrantlabsai/ragas/blob/main/docs/concepts/metrics/available_metrics/faithfulness.md

### 4.2 TruLens and the RAG triad

TruLens describes evaluations around answer relevance, context relevance, and groundedness, and provides feedback functions that can return scores and reasons. These concepts map well to response relevance and source grounding.

**Implication:** Keep the distinction between the relevance of an answer to the question and the relevance/quality of the retrieved context. The current project's core score dimensions focus on the answer, but retrieval diagnostics should be available as supporting metadata so retrieval failures are not mistaken for answer hallucinations.

Reference: https://www.trulens.org/docs/ and https://www.trulens.org/reference/trulens/feedback/

### 4.3 LLM-as-a-judge reliability and bias

Research published in ACL Anthology has examined position bias in LLM judges and found that judgments can vary with answer position and task conditions. Other work studies broader judgement biases and the alignment between human and model evaluations.

**Implication:**
- Use explicit rubrics with anchored score definitions and examples.
- Ask for compact structured judgments, not unrestricted essays.
- Randomize or neutralize answer order if pairwise comparisons are later added.
- Repeat a sample of evaluations to measure stability.
- Compare judgments with human labels and report agreement metrics.
- Keep judge model/version and rubric version in each evaluation record.

References: https://aclanthology.org/2025.ijcnlp-long.18/ and https://aclanthology.org/2024.emnlp-main.474/

### 4.4 TruthfulQA

TruthfulQA was designed to test whether models reproduce common misconceptions and false beliefs. The original benchmark includes 817 questions across 38 categories.

**Implication:** It can provide a useful challenge set for testing whether the evaluator catches confidently stated misconceptions. It should not be the only benchmark, because its coverage and answer format do not represent every production use case. Check dataset licensing and terms before redistributing benchmark data in the repository.

References: https://openai.com/index/truthfulqa/ and https://github.com/sylinrl/TruthfulQA

### 4.5 SQuAD and evidence-based QA

SQuAD is a reading-comprehension dataset built around questions and supporting passages. It can help test reference-grounded answer assessment, especially where answer spans or passage context are available.

**Implication:** Use SQuAD-style samples to test passage grounding and answer coverage. Keep benchmark provenance and licence/usage terms documented. Do not assume a dataset designed for extractive QA is a comprehensive hallucination benchmark.

Reference: https://rajpurkar.github.io/SQuAD-explorer/

## 5. Proposed evaluation dimensions

Use a common normalized scale from 0 to 100, but define separate rubrics and evidence requirements for each dimension.

| Dimension | Question | Evidence used | Output requirements |
|---|---|---|---|
| Relevance | Does the answer address the question and stay on task? | Question and response | Score, concise rationale, off-topic content if any |
| Accuracy | Are verifiable claims correct relative to the supplied reference? | Reference answer and/or retrieved passages | Score, claim-level findings, citations/evidence, uncertainty |
| Grounding / hallucination risk | Are claims supported, contradicted, or unassessable against supplied sources? | Retrieved source chunks and response claims | Supported / contradicted / unsupported / insufficient-evidence labels with source IDs and evidence spans |
| Completeness | Does the answer cover the necessary parts of the question? | Question, expected answer elements, reference where available | Score, list of missing required elements |

**Terminology recommendation:** In the UI, prefer “Grounding / unsupported-claim risk” over presenting every unsupported claim as a hallucination. An absent source is not proof that a claim is false.

### Initial score rubric

- **0–19:** Very poor / absent or fundamentally mismatched.
- **20–39:** Major issues; substantial errors or omissions.
- **40–59:** Partial; important gaps or uncertain claims.
- **60–79:** Mostly useful, with identifiable weaknesses.
- **80–100:** Strong performance against the defined rubric and evidence.

Each judge prompt must include dimension-specific anchors and examples. These ranges are initial design guidance and must be calibrated empirically.

## 6. Verdict and aggregation design

Initial proposed weights:

- Relevance: 20%
- Accuracy: 30%
- Grounding safety: 30%
- Completeness: 20%

Let each normalized dimension score be in `[0, 100]`:

`overall_score = 0.20 * relevance + 0.30 * accuracy + 0.30 * grounding_safety + 0.20 * completeness`

Initial verdict bands:

- **Pass:** 80–100
- **Needs Improvement:** 60–79.99
- **Fail:** below 60

These weights and thresholds are provisional. The verdict service should apply explicit critical-failure rules after computing the weighted score. For example, a clearly contradicted central claim can cap the verdict at `Needs Improvement` or trigger `Fail`, depending on the rubric and use case. The policy should be configurable and visible to users.

If no reference answer or source is supplied, factual accuracy and grounding should be marked `NOT_ASSESSABLE` or run under a clearly labelled general-knowledge mode. The system must not silently treat missing evidence as zero accuracy or as proof of hallucination.

## 7. Grounding and RAG method

1. Accept user-supplied documents or a supported dataset record.
2. Extract text and preserve source metadata (filename, page, URL, dataset ID, and offsets where available).
3. Chunk documents with overlap, retaining stable chunk IDs and provenance.
4. Embed and index chunks in a vector store.
5. Retrieve top-k passages for the question and, where useful, individual answer claims.
6. Split the response into atomic factual claims.
7. Compare each claim against retrieved evidence and classify it as:
   - `SUPPORTED`
   - `CONTRADICTED`
   - `UNSUPPORTED`
   - `INSUFFICIENT_EVIDENCE`
   - `CONFLICTING_SOURCES`
8. Store evidence spans, source/chunk IDs, and judge explanation for each finding.
9. Aggregate claim-level results into the grounding-safety score, with a transparent formula and configurable handling for high-impact claims.

Retrieval quality must be measured separately. If relevant evidence was not retrieved, the report should distinguish retrieval failure from the judge's inability to verify a claim.

## 8. Benchmark and validation plan

### 8.1 Evaluation set composition

Create a small curated validation set first (for example, 100–200 items) with human-reviewed labels. Expand it as time allows. Include:

- Correct, relevant, complete answers.
- Relevant but factually incorrect answers.
- Partly correct answers with one unsupported detail.
- Contradictions against source documents.
- Answers that are relevant but incomplete.
- Verbose but evasive/off-topic answers.
- Questions that are unanswerable from the provided context.
- Conflicting or incomplete source documents.
- Different domains and answer lengths.

TruthfulQA and SQuAD can supplement this set, subject to licence and format review. Avoid using the same examples to design prompts and to report final performance without a held-out split.

### 8.2 Human annotation protocol

For a subset of examples, at least two reviewers should label relevance, factual correctness, claim support, and completeness using the same rubric. Resolve disagreements or record them. Store the rubric version and annotation provenance. Human reviewers should be able to mark an item as ambiguous or not assessable.

### 8.3 Metrics

- **Score agreement:** Spearman/Pearson correlation where appropriate, plus mean absolute error against human scores.
- **Ordinal agreement:** Weighted Cohen's kappa for two annotators or a suitable multi-rater statistic where applicable.
- **Claim labels:** Precision, recall, and F1 for `CONTRADICTED` and `UNSUPPORTED` findings, reported separately.
- **Verdict quality:** Confusion matrix, macro-F1, pass/fail precision and recall.
- **Repeatability:** Score variance and verdict flip rate across repeated runs.
- **Retrieval:** Recall@k or manually assessed evidence retrieval coverage on labelled examples.
- **Operational metrics:** Latency, token/API cost per item, error rate, and batch throughput.

### 8.4 Minimum validation gates (proposed)

Before claiming that the system is reliable, set measurable targets with the supervisor/team. Initial engineering targets could include:

- Every judge response validates against a strict schema or returns a controlled failure.
- Every `SUPPORTED` or `CONTRADICTED` grounding label contains traceable evidence or an explicit explanation of why evidence is unavailable.
- A held-out human-reviewed set is used for reported results.
- Repeatability and verdict flip rates are measured.
- Unsupported and contradicted claims are evaluated separately to reduce false-positive hallucination flags.

Do not publish arbitrary accuracy claims until the evaluation has been run and results recorded.

## 9. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Judge hallucination | Evaluator invents evidence or rationale | Require source IDs and quoted spans; validate IDs against retrieved context; show insufficient-evidence states |
| False-positive hallucination flags | Correct claims labelled hallucinated because retrieval missed evidence | Separate unsupported from contradicted; measure retrieval recall; allow external evidence mode later |
| Judge bias / instability | Scores change with wording, answer length, or model | Anchored rubrics, repeated-run audits, human comparison, versioned prompts |
| Reference answer is incomplete or wrong | Good answers may be penalized | Show reference provenance; support multiple sources and reviewer overrides |
| Prompt injection in documents | Untrusted text influences judge instructions | Treat documents as data, delimit retrieved content, tell judges to ignore embedded instructions, never execute document commands |
| API cost and latency | Batch runs become expensive | Limit tokens, parallelize safely, cache retrieval, offer sampling and budget limits |
| Data privacy | Uploaded content sent to an external LLM provider | Explain provider data flow, avoid logging secrets, configurable retention, redact where appropriate |
| Misleading single score | Severe error concealed by high average | Critical-failure rules, dimension breakdowns, claim-level flags |
| Dataset licensing | Unlawful redistribution | Link to source datasets, review licences, avoid committing bulk data unless allowed |

## 10. Implementation recommendations

- Build the MVP as a modular monolith rather than microservices.
- Use FastAPI and Pydantic schemas for typed input/output contracts.
- Implement a provider abstraction so the app is not locked to one LLM vendor.
- Begin with a deterministic orchestration sequence and parallelize independent judges where appropriate.
- Consider LangGraph if stateful retries, checkpoints, conditional branches, or human-in-the-loop review become necessary.
- Use ChromaDB for local prototype retrieval and keep the vector-store interface replaceable.
- Store every evaluation's inputs, reference mode, retrieved chunk IDs, rubric/prompt version, judge model/version, scores, findings, and verdict.
- Add rate limits, request size limits, file validation, and API key management before public deployment.

## 11. Research conclusion

The project is feasible as an eight-week academic implementation if scope is controlled. The strongest differentiator should be **evidence-linked, explainable evaluation plus honest validation of the evaluator itself**, rather than merely using multiple LLM calls. The most important technical decisions are to separate unsupported claims from false claims, keep retrieval diagnostics visible, and evaluate judge reliability against human-reviewed data.

## References

1. RAGAS, *Available Metrics*: https://docs.ragas.io/en/latest/concepts/metrics/available_metrics/
2. RAGAS, *Faithfulness*: https://github.com/vibrantlabsai/ragas/blob/main/docs/concepts/metrics/available_metrics/faithfulness.md
3. TruLens documentation: https://www.trulens.org/docs/
4. TruLens feedback reference: https://www.trulens.org/reference/trulens/feedback/
5. Shi et al., *Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge*, ACL Anthology: https://aclanthology.org/2025.ijcnlp-long.18/
6. Chen et al., *Humans or LLMs as the Judge? A Study on Judgement Bias*, ACL Anthology: https://aclanthology.org/2024.emnlp-main.474/
7. Lin, Hilton, and Evans, *TruthfulQA: Measuring How Models Mimic Human Falsehoods*, OpenAI: https://openai.com/index/truthfulqa/
8. TruthfulQA dataset repository: https://github.com/sylinrl/TruthfulQA
9. SQuAD dataset explorer: https://rajpurkar.github.io/SQuAD-explorer/
10. LangGraph overview: https://langchain-ai.github.io/langgraph/
11. Chroma documentation: https://docs.trychroma.com/
12. FastAPI first steps: https://fastapi.tiangolo.com/tutorial/first-steps/
