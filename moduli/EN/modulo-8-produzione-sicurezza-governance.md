# Module 8: Production, Security and Governance

## Learning objectives

By the end of the module you will be able to:

- describe a reference architecture for an LLM application in production;
- version and observe prompts, models, indexes and datasets as parts of the software;
- control costs with budgets, routing and caching;
- build a threat model for an LLM application using the OWASP Top 10 for LLM Applications;
- identify the main obligations on personal data and transparency (GDPR, EU AI Act) and know when to involve legal and privacy contacts;
- choose between prompt engineering, RAG and fine-tuning using explicit criteria;
- prepare a release checklist.

## 1. From prototype to production

The prototype from Module 5 answers well in a demo. In production the questions change:

| Prototype | Production |
|---|---|
| Does it work on my ten questions? | Does it work on the real distribution of questions, which changes over time? |
| How much does one call cost? | How much does it cost per month with a thousand users, and who is alerted if the cost explodes? |
| Does the model respond? | What happens if the provider is slow, unreachable or retires the model version? |
| Is the answer correct? | Can we reconstruct why an answer from three weeks ago was wrong? |
| Who uses it? | Who can see which documents? Where does personal data end up in prompts and logs? |

The rest of the module tackles these questions one by one.

## 2. Reference architecture

Typical components of an LLM application in production:

- **Interface** (web, chat, integration into an existing product).
- **Application backend**: authenticates the user, applies permissions and limits, orchestrates the flow. This is the harness from Module 1.
- **Model gateway**: the single exit point towards the providers. It manages keys, retries, fallback between models, spending limits, uniform logging. It can be an internal library or a dedicated component.
- **Retrieval service**: vector database, hybrid search, permission filters.
- **Ingestion pipeline**: separate from the service, run when documents change; it produces a versioned index.
- **Observability**: tracing of every request, cost and latency metrics, user feedback.
- **Continuous evaluation**: the suite from Module 6, run on every change and periodically on a sample of real traffic.

### Fallback and degradation

Define in advance what happens when something stops working:

- primary provider unavailable → alternative model (from another provider or open-weight), with the same evaluation suite already passed;
- retrieval slow or failing → explicit message, not an answer generated without context;
- daily budget exhausted → reduced mode or a message to the user, and an alert to the team.

### Exercise

Draw the architecture of your final project with the components above. For each arrow between components, state: what happens if the destination component does not respond?

## 3. LLMOps: versioning and observing

### What to version

In an LLM application the behavior depends on more elements than the code alone. Each one must be versioned and recorded in the logs of every request:

| Element | Example version | Why |
|---|---|---|
| Prompt | `risposta_manuale@v7` | A one-line change alters the behavior |
| Model | full identifier with date or version | The provider can update or retire models |
| Index | `manuale_2026-11-10` | Documents, chunking and embeddings change the retrieval |
| Evaluation dataset | `eval@v3` | Comparing metrics on different datasets makes no sense |
| Parameters | temperature, k, threshold | They are as much a part of the behavior as the prompt |

### Tracing

A trace collects all the steps of a request: user input, query rewriting, retrieved fragments with scores, final prompt, answer, tokens, cost, latency of each step, versions used. LLM observability tools (for example Langfuse, or the equivalent features of cloud platforms) offer this view; alternatively, a structured JSON log with a request identifier is already an excellent start.

### Gradual rollout

- **Offline evaluation**: the suite from Module 6 must pass before release, like automated tests.
- **Shadow**: the new version processes real requests in parallel, but its answers are not shown; they are compared offline.
- **Canary**: the new version serves a small percentage of users; you monitor feedback, refusals, costs.
- **Rollback**: it must be possible to return to the previous version of the prompt, model or index within minutes.

## 4. Cost management

The costs of an LLM application grow with usage in a less predictable way than those of a traditional service. Main levers:

- **Budgets and alerts**: monthly limit per feature, daily limit per user, alert when consumption exceeds the average.
- **Routing by difficulty**: a small model handles simple requests, the large one only those that require it (routing pattern, Module 7).
- **Prompt caching**: stable prefixes at the start of the prompt, to benefit from the provider's cache (Module 3).
- **Response cache**: identical questions receive the same answer without new calls. The "semantic" cache (similar questions) should be used with caution: two similar questions may require different answers, and answers must not pass between users with different permissions.
- **Batch** for non-interactive jobs.
- **Cost per outcome, not per token**: measure how much a resolved ticket or an analyzed document costs. It is the metric that matters to decision makers.

### Exercise

For the final project, estimate the monthly cost under three traffic scenarios (low, expected, peak). Define the spending limits and the behavior of the application when they are exceeded.

## 5. Security

### OWASP Top 10 for LLM Applications

OWASP maintains a list of the main risks for LLM-based applications. It is a good starting point for a threat model. The most relevant risks for the projects in this course:

- **Prompt injection**: instructions in the user's input (direct) or in content the system reads, such as documents and web pages (indirect), that alter the behavior. There is no complete defense at the prompt level: you need limited permissions and confirmations (Module 7).
- **Sensitive information disclosure**: the model returns data that the user should not see, because it was in the context. In RAG, the permission filter must be applied in the retrieval, not requested from the model.
- **Insecure output handling**: model output used without checks in HTML (XSS), in SQL queries, in shell commands. Model output is untrusted input.
- **Excessive agency**: tools with more permissions than necessary, irreversible actions without confirmation.
- **System prompt leakage**: the system prompt can be extracted. It must not contain secrets, keys or confidential information.
- **Vector and embedding weaknesses**: poisoned documents inserted into the index, access to the index without permission checks.
- **Misinformation**: hallucinations presented as facts, with users who trust too much. Mitigation: citations, explicit refusal, evaluation (Module 6).
- **Unbounded consumption**: very long requests or agentic loops that exhaust budget and capacity. Mitigation: limits on input, steps and spending.

The complete list and the detailed mitigations are in the OWASP documentation, which is updated periodically.

### Threat model in four questions

For each feature:

1. What data enters the model's context, and from which untrusted sources?
2. What actions can the model's output trigger (tools, writes, sends, rendering)?
3. Who can see the answer, and are they entitled to see all the data in the context?
4. What happens in the worst case if the model follows a malicious instruction?

### Exercise

Fill in the threat model of the final project with the four questions. For each of the OWASP risks listed above, state: does it apply? With which mitigation? How do you verify it with a test?

## 6. Privacy and compliance

This section provides guidance; it does not replace the advice of the legal contacts and the data protection officer of your organization. Involve them at the start of the project, not at the end.

### Personal data (GDPR)

- **Minimization**: send the model only the data that is necessary. Consider masking or pseudonymizing names, tax codes, addresses before the call.
- **Legal basis and purpose**: processing through an LLM must fall within a valid purpose and legal basis, like any other processing.
- **Vendors**: the model provider processes data on your behalf. You need processing agreements, and checks on retention, use for training and processing region.
- **Logs and traces**: they contain prompts and answers, and therefore often personal data. Define who has access, how long they are kept and how they are deleted.
- **Data subject rights**: if a user requests deletion, you must know where their data is: logs, the agent's long-term memory, the vector index.

### EU AI Act

The European regulation on artificial intelligence (Regulation (EU) 2024/1689) follows a risk-based approach and applies gradually. In summary:

- some practices are prohibited;
- high-risk systems (for example in areas such as personnel selection, access to credit, essential services) have strict obligations on risk management, data quality, documentation, human oversight;
- many systems have transparency obligations: for example, people must know they are interacting with an AI system, and some generated content must be recognizable as such;
- providers of general-purpose models have their own obligations, distinct from those of whoever integrates them into an application.

Practical question for each project: which risk category does the use case fall into? The answer substantially changes the requirements.

### Exercise

For the final project, fill in a one-page sheet: personal data processed, provider and region, log retention period, hypothesized risk category under the AI Act, transparency measures towards users. List the questions to ask the legal and privacy contacts.

## 7. Prompt, RAG or fine-tuning?

One of the most frequent decisions, and the one most often gotten wrong. Recommended order: prompt first, then RAG, fine-tuning only if the first two are not enough and the data justifies it.

| Need | Suitable approach |
|---|---|
| The model must follow a format or a style | Prompt with examples; fine-tuning only for very high volumes or very specific styles |
| The model must know company documents that change | RAG |
| Citations and verifiability are needed | RAG |
| The model must learn a narrow, high-volume task, with minimal cost and latency | Fine-tuning of a small model, with a good labeled dataset |
| The model must "learn facts" | Almost never fine-tuning: it is expensive to update and does not reduce hallucinations. Use RAG |

Fine-tuning requires: a dataset of hundreds or thousands of quality examples, an evaluation suite to compare before and after, a process to repeat it when the base model changes. If any of these elements is missing, it is not the time.

## 8. Release checklist

Before going to production, verify:

**Quality**
- [ ] The evaluation suite passes with the agreed thresholds.
- [ ] Negative and adversarial cases are included.
- [ ] A user feedback mechanism exists.

**Reliability**
- [ ] Retry with backoff, timeouts and fallback are configured.
- [ ] The behavior in case of provider or retrieval error is defined and tested.
- [ ] It is possible to roll back prompt, model and index.

**Costs**
- [ ] Spending limits on the provider and per user.
- [ ] Alerts on abnormal consumption.
- [ ] Monthly cost estimate approved.

**Security**
- [ ] Threat model completed and reviewed.
- [ ] Keys are in a secret manager.
- [ ] Model output is treated as untrusted input.
- [ ] Irreversible actions require confirmation.
- [ ] Document permissions are applied in the retrieval.

**Data and compliance**
- [ ] Legal and privacy contacts involved, sheet completed.
- [ ] Log retention and access defined.
- [ ] Users are informed that they are interacting with an AI system.

**Observability**
- [ ] Every request has a trace with versions, tokens, cost and latency.
- [ ] A dashboard shows quality, costs and errors over time.

## Module deliverable

For the final project:

- architecture diagram with fallback;
- version table (prompt, model, index, dataset) and trace log format;
- cost estimate in three scenarios, with limits and alerts;
- threat model with mitigations and tests;
- privacy and compliance sheet;
- completed release checklist, with the items not yet satisfied and the plan to complete them.

## Further reading

- [OWASP, "Top 10 for Large Language Model Applications"](https://genai.owasp.org/llm-top-10/): the most common risks of LLM applications (latest version).
- [Regulation (EU) 2024/1689 on artificial intelligence (AI Act)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj): official text on EUR-Lex.
- [Regulation (EU) 2016/679 (GDPR)](https://eur-lex.europa.eu/eli/reg/2016/679/oj): official text on EUR-Lex; see also the data protection authorities' guidelines on the use of AI.
- [NIST, "AI Risk Management Framework"](https://www.nist.gov/itl/ai-risk-management-framework): framework for identifying and managing the risks of AI systems.
- Documentation of the chosen providers: data retention and usage policies.
