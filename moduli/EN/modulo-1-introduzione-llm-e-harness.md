# Module 1: Introduction - What an LLM is, how it works and what a Harness is

## Learning objectives

By the end of the module you will be able to:

- explain what an LLM does in a single sentence, without resorting to magic formulas;
- describe the path of a prompt from tokenization to the generated token;
- tell training and inference apart and say which of the two happens when you use a chatbot;
- define a harness and identify its components in a product you use every day;
- explain why the LLM alone cannot "do" anything in the world and what it takes to turn it into an agent.

## 1. What an LLM is

A Large Language Model is a very large mathematical function. It receives a sequence of tokens and returns a probability distribution over the next token. Everything else (answering questions, writing code, summarizing contracts) is a consequence of this operation repeated over and over.

### Tokens

The model sees neither words nor characters, but tokens: fragments of text produced by a tokenizer. In English a token corresponds on average to about 3-4 characters; in Italian, with longer words and inflectional endings, the ratio is lower and a text takes up more tokens.

Example exercise: open an online tokenizer (there are several for the most widespread model families) and compare how many tokens the same sentence takes in Italian and in English. Note the difference: you will need it in module 3 to estimate costs and context limits.

### Parameters

Parameters are the weights of the neural network, floating-point numbers learned during training. A model with billions of parameters does not contain a readable "knowledge base": knowledge is distributed across all the weights together.

## 2. How it works: the Transformer

The dominant architecture is the Transformer. The flow, simplified, is this:

1. **Embedding**: each token becomes a vector of hundreds or thousands of numbers.
2. **Self-attention**: in each layer, each token computes how much to "look at" the other tokens in the sequence. This is how the word "it" acquires the meaning of the noun it refers to, even if that noun is far away in the sentence.
3. **Feed-forward networks**: apply non-linear transformations at each position.
4. **Repeated layers**: the process repeats tens or hundreds of times, refining the representation.
5. **Output**: the last vector is projected onto the vocabulary, producing a score for every possible token, which is then converted into probabilities.

Concept to remember: the model does not "plan" the whole sentence before writing it. It always computes a single step, the next token. The coherence you observe is the result of this step repeated over a very rich distribution.

## 3. How it is trained

Training happens in distinct phases, each with a different objective and cost.

### Pre-training

The model reads enormous amounts of text (web pages, books, code, documentation) and, at each position, has to predict the next token. The loss function (cross-entropy) measures how far the prediction is from the real token. After each batch of examples, the weights are corrected with backpropagation and gradient descent.

The result is a base model: an excellent text completer that still cannot hold a dialogue. If you write "What is the capital of France?" it might answer with more questions, because in the training text questions are often followed by other questions.

### Supervised Fine-Tuning (SFT)

People write high-quality instruction-response pairs. The model is trained on these examples and learns the assistant format: answering, following instructions, using an appropriate tone.

### Alignment (RLHF, RLAIF, preference optimization)

For each prompt the model generates several responses, and raters (humans or other models) indicate which they prefer. A reward model learns these preferences and steers the optimization of the main model toward more helpful, honest and safe responses. This phase is what explains why a model refuses certain requests or admits it does not know.

Point not to be confused: none of these phases happens while you are using the model. When you chat, the weights are fixed.

### Reflection exercise

A colleague says: "The model learned from my conversation yesterday". Write in three lines why this is wrong and which mechanism, instead, makes it look like the model remembers (hint: think about the context and about what is sent again on every call).

## 4. How inference works

Inference is the use of the already-trained model. Every request goes through these steps.

1. **Tokenization**: the prompt becomes a sequence of token IDs.
2. **Prefill**: the model processes all the prompt tokens in parallel. During this computation it saves the attention keys and values in a **KV cache**, so as not to recompute them at every following token. The time of this phase affects perceived latency, in particular the time-to-first-token.
3. **Autoregressive decoding**: the model generates one token at a time. The chosen token is added to the sequence and the cycle starts again. Every step requires a full computation: this is why long output takes more time.
4. **Sampling**: the next token is chosen from the distribution.
   - Low *temperature*: almost always picks the most probable token, stable and repeatable responses.
   - High *temperature*: spreads the choice also over less probable tokens, more variability and more risk of incoherence.
   - *Top-p* (nucleus sampling) and *top-k*: limit the set of candidates before sampling.
5. **Stop**: generation ends with an end-of-sequence token, when the output limit is reached, or with a stop sequence defined by the caller.

### Practical consequences you must remember

- **Stateless**: the model retains nothing between one call and the next. Every request includes everything the model needs to know. The chat "memory" is the history that the application sends again at every turn.
- **Finite context window**: what does not fit in the context does not exist for the model, neither for the current response nor for later ones.
- **Hallucinations**: the model produces the most plausible text, even when it does not have the information. There is no internal fact check: verification must be designed from the outside (module 4 and module 6).
- **Non-determinism**: with a temperature greater than zero the same prompt can produce different outputs. This must be kept in mind in testing.

### Use case to develop

Take an activity you already perform (for example summarizing the team's emails). Write the prompt as you would send it to the model, then note down: how many tokens you estimate for input and output, which temperature you would choose and why, and which missing piece of information could lead to a hallucination.

## 5. What a harness is

A harness is the software that surrounds the model and orchestrates its use. The name conveys the idea: the model is the powerful horse, the harness is the gear that guides it, gives it a direction and keeps it from causing damage.

A harness, in practice:

- builds the prompt, including the system prompt and the tool definitions;
- manages the context: what to include, when to summarize the history, when to truncate it;
- sends the request to the model and interprets its response;
- executes the tools requested by the model (tool calling) and puts the results back into the context;
- repeats the cycle until the task is completed or a limit is reached;
- applies permissions, validations, logging, and error and rate-limit handling.

Concrete examples: a chat assistant with retrieval over documents, a coding agent that reads and edits files, a bot that queries a database on request. In all these cases there is a program around the model doing the work "outside the text".

### Typical components

| Component | Question it answers |
|---|---|
| System prompt | Who is the assistant and which rules must it follow? |
| Context management | Which information enters the window, and in what order? |
| Tool registry | Which actions can the model ask to perform, and with which parameters? |
| Execution loop | When to call the model again and when to stop? |
| Security and permissions | Which actions require confirmation or are forbidden? |
| Observability | What happened, how much did it cost, where did it go wrong? |

## 6. The relationship between LLM and harness

The key distinction is this: the LLM decides, the harness acts.

The model produces only text. When it "uses a tool", it actually generates a structured text that describes the call (tool name and arguments). It is the harness that recognizes that request, actually executes the function and sends the result back to the model.

Typical cycle, step by step:

1. The harness sends the model: instructions, conversation history, tool definitions.
2. The model replies: "call `cerca_ordine` with `id = 4411`".
3. The harness executes `cerca_ordine(4411)` on the real system.
4. The harness sends the result back to the model, for example `{"stato": "spedito", "data": "2026-11-12"}`.
5. The model generates the final response for the user, or asks for another tool.

Important consequences:

- **Same model, different harness**: two products with the same LLM can behave very differently, because the prompt, tools, context management and checks change.
- **The model is not solely responsible for security**: a malicious instruction in the data can lead it to request a dangerous action. It is the harness that must limit what can be executed.
- **Quality depends on the pair**: if the model is good but the harness provides wrong context, the result will be wrong. The same holds the other way around.

### Exercise

Choose an application that uses an LLM (a writing assistant, a website chatbot, a coding tool). List: what the model does, what the harness does, which tools are available and where the boundary between decision and action lies. If you cannot identify it, you are probably looking only at the model and not at the harness.

## 7. Model families and selection criteria

There is no such thing as "the" best model: there is the model suited to a task, a budget and a latency constraint. It is worth knowing the main families.

### Closed and open-weight

- **Closed models**: accessible only through the provider's API. They usually offer the highest capabilities and no infrastructure to manage, but data transits through the provider and the version can be retired.
- **Open-weight models**: the weights are downloadable. They can be run on your own servers or through third-party providers (for example Groq or Together AI). They give control over data, version and costs at volume, but require operational expertise.

### Size and cost

Large models are more capable on complex tasks; small ones are faster and cheaper and often sufficient for classifying, extracting fields or routing requests. A common architecture uses a small model for the simple steps and a large one only where needed (see the routing pattern in module 7).

### Reasoning models

Some models generate internal reasoning before the final response (often called thinking or reasoning). It is the Chain of Thought of module 2 built into the training. They improve on math, code and multi-step problems, but consume more tokens and increase latency. Many APIs let you adjust how much "reasoning budget" to grant.

### Multimodality

Many models accept images, PDFs and sometimes audio as input. This changes how you design a RAG: a manual with technical diagrams can be read in its figures too, not only in the extracted text.

### Context window

The window ranges from tens of thousands to over a million tokens. A large window does not remove the need for retrieval: it costs, slows things down and the quality of attention tends to drop when the context is full of irrelevant material.

### Selection table

| Need | Guidance |
|---|---|
| High-volume classification | Small model, low temperature, structured output |
| Answers over company documents | Medium or large model + RAG |
| Complex multi-step analysis | Reasoning model, controlled token budget |
| Data that cannot leave the company | Open-weight on your own infrastructure, or specific contractual agreements |
| Very low latency | Small model, streaming, short prompts and cache |

### Exercise

For three use cases from your work, fill in the table: task, estimated daily volume, acceptable latency, data sensitivity, chosen model family and rationale. You will come back to this table in module 3 to estimate costs and in module 8 for production choices.

## Further reading

- [Vaswani et al., "Attention Is All You Need" (2017)](https://arxiv.org/abs/1706.03762): the Transformer architecture.
- [Ouyang et al., "Training language models to follow instructions with human feedback" (2022)](https://arxiv.org/abs/2203.02155): the role of alignment.
- [Jay Alammar, "The Illustrated Transformer"](https://jalammar.github.io/illustrated-transformer/): a visual, step-by-step explanation of the Transformer.
- [Andrej Karpathy, "Intro to Large Language Models" (video, 2023)](https://www.youtube.com/watch?v=zjkBMFhNj_g): an overview of training, inference and the limits of models.
- [Hugging Face, "LLM Course"](https://huggingface.co/learn/llm-course): a free course on tokenization, models and inference.
