# AI Response Quality Evaluator Agent

An explainable evaluation platform for assessing AI-generated responses across relevance, factual accuracy, source grounding, and completeness. The project is designed for single-response and batch evaluation, with evidence-backed reports and quality analytics.

> **Status:** Research and architecture phase. Scores, thresholds, and agent behavior are initial design proposals and must be validated against human-reviewed examples before being treated as reliable measurements.

## Project goals

- Evaluate AI responses using clearly defined dimensions and rubrics.
- Ground factual checks in optional reference answers and retrieved source passages.
- Identify unsupported or contradicted claims with evidence.
- Process individual submissions and CSV batches.
- Provide explainable verdicts, comparison analytics, and exportable reports.
- Measure evaluator quality against human annotations, not just assume that an LLM judge is correct.

## Initial technology proposal

- **Frontend:** React, Vite, TypeScript, Tailwind CSS
- **API:** Python, FastAPI, Pydantic
- **Evaluation orchestration:** A deterministic Python pipeline initially; LangGraph can be introduced if branching, retries, persistence, or human-in-the-loop review justify it
- **LLM provider:** Provider adapter supporting one selected model first; keep provider-specific code behind an interface
- **Retrieval:** ChromaDB + configurable embedding model
- **Persistence:** SQLite for local MVP, with PostgreSQL as a deployment option
- **Batch processing:** Pandas initially; background jobs can be added for larger workloads
- **Testing:** Pytest for backend, plus frontend tests as UI stabilizes
- **Reports and charts:** ReportLab and Recharts

## Repository map

- [`docs/RESEARCH_ANALYSIS.md`](docs/RESEARCH_ANALYSIS.md) — problem framing, prior approaches, benchmark strategy, risks, and validation plan.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — system components, pipeline, data contracts, storage model, APIs, and security considerations.

## Planned development stages

1. Research and architecture sign-off.
2. API contracts, schemas, and deterministic baseline tests.
3. Single-response evaluation with structured judge outputs.
4. Source ingestion and claim-level grounding checks.
5. Batch evaluation and persistence.
6. Dashboard, exports, and model-comparison views.
7. Human-labelled validation, calibration, documentation, and demo.

## Important evaluation principles

1. **Unsupported does not automatically mean false.** If a claim is not found in the retrieved sources, label it `UNSUPPORTED` or `INSUFFICIENT_EVIDENCE`; use `CONTRADICTED` only when the evidence conflicts with the claim.
2. **Separate correctness from grounding.** A response can be factually correct yet not supported by the supplied context; conversely, a source may be incomplete or outdated.
3. **Treat scores as estimates.** LLM-as-a-judge can exhibit bias and inconsistency. Validate against human annotations and report uncertainty or low-evidence states.
4. **Do not let an average hide critical failures.** Severe contradictions can trigger a verdict cap or fail rule.
5. **Keep evidence auditable.** Persist the source/chunk IDs, quoted evidence spans, rubric version, model identifier, and prompt/version metadata where possible.

## Status

This repository currently contains planning documents only. Implementation status should be updated as features are built and tested.
