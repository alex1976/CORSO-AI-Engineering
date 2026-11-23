# Module 6: Evaluation and Optimization

## Learning objectives

By the end of this module you will be able to:

- build an evaluation dataset (golden dataset) that is representative of your use case;
- distinguish retrieval metrics from generation metrics and understand which problem each one measures;
- use a framework such as RAGAS to measure faithfulness to the documents, relevance and context precision;
- evaluate reliably with LLM-as-a-judge, recognizing its limits;
- detect hallucinations and optimize the system prompt in a measurable way.

## Introduction: without measurement there is no improvement

A prompt or a RAG system that "seems to work" on three questions is an experiment, not a product. LLM answers vary, hard cases are rare, and your intuition tends to test the cases you already know. Evaluation turns an impression into a number you can compare before and after every change.

Rule of thumb: no change to the prompt, the chunking or the model should be considered an improvement until you have measured it on a fixed dataset.

## 1. The evaluation dataset

The golden dataset is a set of cases with this minimal structure:

```json
{
  "id": "E18-001",
  "domanda": "What does error E18 mean?",
  "risposta_attesa": "Water drainage problem; clean the filter.",
  "fonte_attesa": {"documento": "manuale_lavatrice.pdf", "pagina": 12},
  "tipo": "diretta",
  "difficolta": "facile"
}
```

How to build it:

- **Real cases**: actual questions, anonymized, taken from logs or from customer service. They are the most valuable.
- **Synthetic cases**: generated from the document chunks, for example by asking a model to write a question that the chunk answers. Useful for covering the corpus, but they must be reviewed by hand: they can be trivial or ambiguous.
- **Negative cases**: questions the document does not cover. They are mandatory: without them you do not measure the most dangerous behavior, hallucination.
- **Adversarial cases**: synonyms, typos, ambiguous questions, prompt injection attempts.

Size: for a prototype 30-50 cases are enough, balanced across types and difficulty. For production you need hundreds of cases, updated as real errors emerge.

Fundamental rule: do not modify the dataset to make it pass. If a case fails, the question is whether the system is wrong or the case is wrong. Decide case by case and document the decision.

### Exercise

Build a dataset of 30 cases for your project: 15 direct, 5 with synonyms, 5 negative, 5 adversarial. For each case write the expected source, not just the answer. Have a colleague review the dataset before you use it.

## 2. Separating retrieval and generation

A common mistake is to evaluate only the final answer. If the answer is wrong, you do not know whether the fault lies with retrieval (the right chunk was not found) or with generation (the chunk was there but the model did not use it well). You need separate metrics.

### Retrieval metrics

- **Recall@k**: is the correct chunk among the top k results? It is the most important metric: if it is not retrieved, no prompt will save it.
- **Precision@k**: how many of the k chunks are actually relevant? It measures the noise in the context.
- **MRR (Mean Reciprocal Rank)**: how high the right chunk appears, on average. Useful for evaluating the reranker.

### Generation metrics

- **Faithfulness**: is every statement in the answer supported by the provided chunks?
- **Answer relevance**: does the answer actually address the question?
- **Correctness**: does the answer match the expected answer in the dataset?
- **Appropriate refusal**: on negative cases, does the system say it does not know?

## 3. RAGAS

RAGAS is a framework that computes metrics for RAG systems, partly with the help of an LLM. The most commonly used metrics are:

- **Faithfulness**: is the answer supported by the retrieved context?
- **Answer relevancy**: is the answer relevant to the question?
- **Context precision**: are the retrieved chunks relevant and well ordered?
- **Context recall**: does the retrieved context contain the information needed for the expected answer?

How to use it:

1. run the system on all the dataset cases, saving the question, the retrieved chunks and the answer;
2. pass this data to RAGAS together with the expected answer where needed;
3. compare the metrics between two versions of the system; do not look for an absolute value.

Warning: RAGAS metrics use an LLM as a judge, so they have their own variability and their own cost. Use the same judge model in all comparisons and, if possible, spot-check the scores by hand.

## 4. LLM-as-a-judge

To evaluate open-ended answers, a model is often used as a judge. The judge receives the question, the answer, the criteria and, if available, the expected answer, and returns a score or a label.

Good practices:

- **Explicit criteria**: "The answer is correct if it contains information X. A different level of detail is not penalized." Without criteria the judge applies its own.
- **Structured output**: score and rationale in JSON, see Module 2.
- **Pairwise comparison**: it is often more reliable to ask which of two answers is better rather than to assign an absolute score.
- **Known biases**: judges tend to prefer longer answers, more formal ones, and those written by the same model. Check for them by comparing with human evaluation.
- **Calibration**: validate the judge on 50 cases judged by a person. If agreement is low, the judge is not reliable for that task.

### Example of a judge prompt for faithfulness

```text
You are an evaluator. You must determine whether the ANSWER is fully supported by the CHUNKS.

Procedure:
1. List the factual statements contained in the ANSWER.
2. For each one, indicate whether it is supported by at least one chunk (yes/no) and which one.
3. A statement is supported only if the chunk states it explicitly or implies it directly.
   General knowledge does not count as support.

Reply only with JSON:
{"affermazioni": [{"testo": "...", "supportata": true, "frammento": 2}],
 "punteggio": <number of supported statements / total>,
 "verdetto": "faithful" | "partially faithful" | "unfaithful"}

CHUNKS:
{frammenti}

ANSWER:
{risposta}
```

Note the structure: the judge first breaks the answer down into statements, then verifies each one. It is more reliable than a generic question such as "is the answer faithful? Score from 1 to 10", and the per-statement rationale makes the score verifiable by a person.

### Exercise

Apply the judge prompt to 20 answers from your system. Evaluate the same 20 answers yourself with the scorecard in section 5. Calculate in how many cases the judge's verdict matches yours. Analyze the cases of disagreement: is the judge too strict, too lenient, or is the criterion ambiguous?

## 5. Human evaluation

Human evaluation remains the reference. It does not have to be performed on everything, but on a representative sample and on the cases where the automatic judge is uncertain.

A simple scorecard:

| Criterion | Scale | Guiding question |
|---|---|---|
| Correctness | 0 / 1 / 2 | Does the answer contain the right information? |
| Faithfulness | 0 / 1 / 2 | Is every statement in the manual? |
| Usefulness | 0 / 1 / 2 | Does the customer know what to do next? |
| Safety | yes / no | Did the system suggest dangerous operations? |

Also measure agreement between two human evaluators: if two people do not agree, the criterion is ambiguous and must be rewritten.

## 6. Hallucinations

A hallucination is a statement that is unsupported by the sources or false, presented with confidence. In RAG two forms are distinguished:

- **Context hallucination**: the answer contradicts or adds information to the retrieved chunks.
- **Hallucination from absence**: the context does not contain the answer, but the model answers anyway.

Detection techniques:

- sentence-by-sentence faithfulness check, with the judge or with RAGAS;
- citation check: does the cited page really contain the information?
- systematic negative tests: the system must refuse questions outside the corpus;
- comparison with a reference answer for numerical information (dates, quantities, codes), which is the riskiest.

Mitigations:

- improve retrieval, because better context reduces hallucinations from absence;
- relevance threshold and refusal response;
- a prompt that requires citations and forbids unsupported statements;
- low temperature for extraction and document-based answering tasks;
- deterministic verification where possible (for example, checking that a cited error code exists in the manual).

## 7. Prompt tuning and the system prompt

The system prompt is the most persistent part of the prompt and therefore the one most worth optimizing. The goal set out in the course is to limit the model's creativity and make it stick to the facts, but tuning must be done by measurement, not by trial and error based on gut feeling.

Recommended process:

1. define the main metric (for example average faithfulness and correct refusal rate on the negative cases);
2. establish a baseline with the current prompt on the fixed dataset;
3. change only one element at a time (a constraint, an example, the format);
4. rerun and compare, also checking the secondary metrics to avoid regressions;
5. version the prompt with an identifier and record the results.

Typical levers:

- **Explicit constraints** on sources: "use only the chunks", "do not use general knowledge".
- **Refusal behavior**: the exact wording to use when the information is missing.
- **Temperature**: lower for faithfulness, checking that the answer does not become mechanical or incomplete.
- **Length and format**: shorter answers reduce unverifiable statements, but may lose usefulness.
- **Few-shot examples**: useful for the citation format and for refusal cases.

Pitfall: a very restrictive constraint can increase refusals on legitimate questions. Always monitor both faithfulness and the rate of useful answers.

### Tuning exercise

Start from the prompt in Module 5. On a dataset of 30 cases, measure: faithfulness, correct refusal rate on negatives, answer rate on legitimate questions. Then try three variants and compare them. Write a one-page note on which change produced the biggest improvement and what side effect it had.

## 8. Quality, cost and latency together

A variant that improves faithfulness by 2% but doubles the cost or the latency is not necessarily better. Every run of the evaluation suite should report, alongside the quality metrics:

- average input and output tokens per request;
- average cost per request and per thousand requests;
- p50 and p95 latency, and the time to first token if the interface uses streaming.

Example of a comparison report:

| Variant | Recall@3 | Faithfulness | Correct refusal | Cost / 1000 req. | p95 latency |
|---|---|---|---|---|---|
| Baseline | 0.72 | 0.81 | 0.60 | 1.00 (reference) | 2.1 s |
| + hybrid search | 0.84 | 0.85 | 0.60 | 1.02 | 2.3 s |
| + reranking | 0.88 | 0.88 | 0.65 | 1.10 | 2.9 s |

The values are illustrative. The decision depends on the use case: for an interactive assistant 2.9 seconds may be acceptable, for a real-time suggestion while typing it is not.

## 9. Monitoring in production

Evaluation does not end with release. In production:

- log every question, the chunks used and the answer, complying with data privacy rules;
- collect explicit feedback (helpful/not helpful) and implicit feedback (the customer rephrases the question, opens a ticket);
- periodically sample answers for human evaluation;
- turn every real error into a new dataset case;
- run the evaluation suite on every change, like a regression test.

## Use case to develop

Take the prototype from Module 5 and build:

- a dataset of 30 cases with expected source;
- a script that runs the system on the whole dataset and produces a table with recall@3, faithfulness, correct refusal rate;
- three prompt or chunking parameter variants, compared on the same metrics, with cost and latency (section 8);
- a short analysis of three failures, classified as a retrieval or a generation problem.

## Further reading

- [Es et al., "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023)](https://arxiv.org/abs/2309.15217): the reference paper on RAGAS metrics.
- [RAGAS documentation](https://docs.ragas.io): implementation of the retrieval and generation metrics.
- [Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023)](https://arxiv.org/abs/2306.05685): reliability and biases of LLM-based judges.
- [Promptfoo documentation](https://www.promptfoo.dev/docs/intro/): automated testing and regression for prompts.
