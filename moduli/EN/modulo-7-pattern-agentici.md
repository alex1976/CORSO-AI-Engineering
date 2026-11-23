# Module 7: Agentic Development Patterns

## Learning objectives

By the end of the module you will be able to:

- distinguish an agent from a workflow and choose the level of autonomy that suits the problem;
- implement the ReAct loop with tool calling and execution limits;
- choose among the composition patterns (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer);
- design control mechanisms: human confirmations, permissions, guardrails, observability;
- manage an agent's memory and context for long tasks;
- assess when a multi-agent system is justified and when it is needless complexity.

## 1. Agent or workflow?

The starting point is a distinction that avoids many design mistakes:

- **Workflow**: the path of the steps is defined by the code. The LLM performs individual steps (classify, extract, summarize). The flow is predictable, testable, cheap.
- **Agent**: the LLM dynamically decides which tools to use, in which order, and when to stop. The flow emerges from the conversation with the tools.

An agent is not "smarter" than a workflow. It is more flexible and less predictable. When the path is known, the workflow almost always wins.

Example. Processing an invoice received by email:

- *Workflow*: extract the PDF → extract the fields → validate the VAT → save to the management system → notify. Every step is fixed.
- *Agent*: you receive the request "check this invoice and flag anything that doesn't add up". The path depends on what you find: if the supplier is not in the registry you look for an order, if the amount does not match you look for the contract, and so on.

Exercise: take five tasks from your work and classify them as workflow, agent, or "no LLM needed". For each agent, explain which decision the model makes that the code cannot make in advance.

## 2. The base loop: ReAct and tool use

ReAct (Reason + Act) is the cycle on which most agents are based. The model alternates three phases:

1. **Reasoning**: decides the next step.
2. **Action**: calls a tool with structured arguments.
3. **Observation**: receives the result and updates its understanding.

The cycle ends when the model produces a final answer instead of a tool call.

### Minimal implementation

```python
def esegui_agente(llm, strumenti: dict, richiesta: str, max_passi: int = 8) -> str:
    messaggi = [{"role": "user", "content": richiesta}]

    for _ in range(max_passi):
        risposta = llm.genera(
            system=SYSTEM_PROMPT,
            messaggi=messaggi,
            strumenti=descrizioni(strumenti),   # name, purpose, parameter schema
        )

        if not risposta.chiamate_strumenti:
            return risposta.testo               # final answer

        for chiamata in risposta.chiamate_strumenti:
            funzione = strumenti[chiamata.nome]
            try:
                esito = funzione(**chiamata.argomenti)
            except Exception as errore:
                esito = {"errore": str(errore)}  # the model can correct itself
            messaggi.append({"role": "tool", "nome": chiamata.nome, "contenuto": esito})

    raise RuntimeError("Step limit reached without a final answer")
```

Things to note:

- the **step limit** is mandatory: without it a loop may never terminate and burn through the budget;
- **tool errors** go back to the model as an observation, they do not crash the loop: the model often corrects the arguments;
- the **tool descriptions** are part of the prompt. A vague name or an ambiguous schema produces wrong calls.

### Designing a tool

A well-designed tool:

- has a clear verb-object name (`cerca_ordine`, not `gestisci`);
- describes when to use it and when not to use it;
- has typed parameters with explicit allowed values;
- returns compact, structured results, not entire databases;
- has no hidden side effects: an irreversible action must be declared as such.

Exercise: design three tools for a helpdesk assistant (for example `cerca_cliente`, `leggi_stato_ticket`, `apri_ticket`). For each one, write the description for the model and the parameter schema. Then ask yourself: which of these can do damage if the model misuses it?

## 3. Composition patterns

Not all problems require an agent. The following patterns are building blocks to combine.

### Prompt chaining

The task is split into sequential steps, each with a dedicated prompt. The output of one step is the input of the next.

Use case: generating a reply to a negative review in three steps: (1) classify the problem, (2) check in the policies which remedy to apply, (3) write the reply in the brand's tone. Each step can be tested separately.

### Routing

A classifier directs the request to a specialized path: a prompt, a smaller or cheaper model for simple cases, a dedicated flow for sensitive cases.

Use case: a helpdesk that sends questions about invoices to the document RAG, refund requests to a workflow with approval, and out-of-scope requests to a human operator.

### Parallelization

Several independent calls are run in parallel:

- **Sectioning**: the task is divided into independent parts (analyzing different sections of a contract) and the results are aggregated.
- **Voting**: the same task is run several times, with the results compared or voted on, to increase reliability on critical decisions.

### Orchestrator-workers

An orchestrator analyzes the request, decides which sub-tasks are needed and assigns them to workers, which can be specialized agents or simple calls. The orchestrator synthesizes the results.

Use case: analysis of a tender notice. The orchestrator delegates to the workers: technical requirements, economic requirements, deadlines, penalties. Each worker works on a subset of the document, the orchestrator produces the synthesis.

### Evaluator-optimizer

A generator produces a draft, an evaluator compares it against explicit criteria and sends back feedback. The cycle continues until an acceptance criterion or an iteration limit is reached.

Use case: generating a product description for an e-commerce site, with an evaluator that checks length, presence of the mandatory features, absence of unverified claims. The evaluator should use deterministic checks where possible (length, forbidden words) and the model only for semantic criteria.

## 4. Control and reliability patterns

An agent that acts in the world (sends emails, modifies data, runs code) requires controls. The model must not be the only party responsible for security.

### Human-in-the-loop

The agent stops and asks for confirmation before irreversible or costly actions. The approval point must be designed:

- which actions require confirmation (sending, deleting, paying, changing permissions);
- what the approver sees (proposed action, rationale, data involved);
- what happens if nobody approves within a given time (cancellation, non-execution by default).

Common mistake: asking for confirmation at every step, to the point of making the agent useless. Ask for confirmation for actions that have external or irreversible effects, not for reading.

### Permissions and least privilege

Each tool declares what it can do and the harness limits its access: read-only or also write, only certain tables, only certain folders, limited service credentials. An agent that reads emails must not also be able to send them, unless it is necessary.

Beware of prompt injection (already introduced in module 2): if the agent reads untrusted content (web pages, emails, documents), that content may contain instructions to use the tools in a harmful way. The effective defense is not in the prompt but in the permissions and the confirmations.

A useful rule for assessing risk: an agent is particularly exposed when it combines three capabilities together:

1. access to private data (emails, internal documents, databases);
2. exposure to untrusted content (web pages, attachments, third-party messages);
3. the ability to communicate externally (send emails, call URLs, write to shared systems).

With all three, an instruction hidden in a piece of content that is read can lead the agent to exfiltrate private data. If the use case requires all three, at least one must be mediated by a human confirmation or a deterministic check, for example a list of allowed recipients.

### Guardrails

Validations on input and output:

- parameter schema before the tool is executed;
- policy filters on actions (for example, no sending to external domains);
- check of the final output before showing or sending it.

### Limits and budgets

- maximum number of steps;
- token and cost budget per request;
- timeout for each tool and for the entire execution;
- explicit exit conditions (for example: after two failed attempts, hand over to a human with a summary).

### Observability

For an agent, logging is not optional. Record every step: visible reasoning, call, arguments, outcome, tokens, duration. When an agent makes a mistake, without the complete trace you cannot tell whether the problem was the model, the tool or the tool description.

Exercise: take the loop example from section 2 and add: a token budget limit, a human confirmation for a tool of your choice, and a structured log of every step. Write a test case that verifies that the agent stops when the budget is exhausted.

## 5. Memory and context

An agent working on long tasks accumulates messages, tool results and reasoning. The context grows and the cost rises. Management strategies are needed.

### Short-term memory

The session history in the context. Strategies:

- **truncation**: the oldest messages are discarded (simple, but information is lost);
- **summarization**: every N steps the model summarizes the history into a compact block that replaces the original messages;
- **compact results**: the tool returns only the useful fields, not entire objects.

### Long-term memory

Facts and preferences saved externally (files, database, vector store) and retrieved when needed. Example: a user's preferences on report format, saved and loaded at the start of the session.

Warning: a long-term memory can store wrong or outdated information. It needs an update policy and, if it contains personal data, a legal basis and a way to delete it.

### Context engineering

Choosing carefully what to put in the context at each step. What the model does not see it cannot use, but what it sees in excess distracts it. It is the skill that links module 2 (prompt), module 4 (retrieval) and this module (memory).

### Sub-agents

An isolated task is handed to an agent with its own clean context. The sub-agent explores (for example searches the code or the documents), then returns only the condensed result to the main agent. The main context stays light.

Use case: a coding agent that needs to understand how a large module works. A sub-agent reads dozens of files and returns a one-page description; the main agent does not have to load all the files into its own context.

## 6. Protocols and reuse

### MCP (Model Context Protocol)

An open standard with which a harness discovers and uses tools, resources and prompts exposed by external servers. The advantage is reuse: a server that exposes a CRM's data can be used by any compatible client, without a custom integration for every model-application pair.

Exercise: look up the protocol documentation and describe in your own words, in half a page, the difference between a tool integration built directly in the code and an integration through an MCP server. What advantages and what risks does the use of third-party servers introduce?

### Skills and reusable instructions

Procedures and domain knowledge gathered in files (a description, instructions, possibly scripts) that the agent loads only when the task requires them. This keeps the context light and makes the skills shareable across projects.

Use case: a "meeting minutes writing" skill with the company format, the list of mandatory elements and the tone rules. The agent loads it when it receives a transcript, not in every conversation.

## 7. Multi-agent: when and why

A multi-agent system has several agents with distinct roles that collaborate. The most common forms:

- **Supervisor**: a coordinator receives the request, routes to the workers and decides when the task is complete.
- **Handoff**: an agent, recognizing that another skill is needed, passes control of the conversation to another agent.
- **Role pipeline**: for example a planner, an executor and a reviewer in sequence.

Advantages: specialization of prompts and tools, separate contexts, the ability to use different models for different roles.

Costs: more calls (hence more cost and latency), more points of failure, communication between agents that can lose information, harder debugging.

Rule of thumb: start with a single agent with well-designed tools. Move to multi-agent only when you measure that the single agent cannot cope for clear reasons: context too large for a single agent, roles with incompatible permissions, parallelism needed.

### Use case to develop

An assistant for analyzing a tender notice. Compare three architectures: (a) a single agent with four tools (reads document, searches requirements, calculates deadlines, writes summary); (b) orchestrator-workers with three workers; (c) a fixed workflow with chaining. For each one define: estimated cost, latency, error risks and how you would measure them with the dataset from module 6.

## 8. Evaluating an agent

An agent is evaluated like a RAG system (module 6), with two differences: the result depends on a sequence of decisions, and the path matters as much as the final answer.

### Metrics

- **Success rate**: percentage of tasks completed correctly. The success criterion must be defined for each case, ideally verifiable by code (the ticket was created with the right fields, the file contains the expected result).
- **Efficiency**: number of steps, tokens and cost per completed task. Two agents with the same success rate can differ by an order of magnitude in cost.
- **Trajectory**: did the agent use the right tools, with correct arguments, without superfluous or dangerous actions?
- **Robustness**: does the same task run several times give consistent outcomes? With temperature above zero, an agent that succeeds 1 time out of 3 is not reliable.
- **Security**: on adversarial cases (instructions hidden in a document that is read), the agent must not take unauthorized actions.

### Structure of a test case

```json
{
  "id": "helpdesk-007",
  "richiesta": "Customer 4411 says the order has not arrived, check and open a ticket if needed",
  "stato_iniziale": {"ordine_4411": {"stato": "spedito", "giorni_dalla_spedizione": 12}},
  "atteso": {
    "strumenti_obbligatori": ["cerca_ordine", "apri_ticket"],
    "strumenti_vietati": ["rimborsa_ordine"],
    "verifica_finale": "a ticket exists with category SPEDIZIONE for customer 4411"
  },
  "max_passi": 6
}
```

In tests, the tools work on a simulated environment (test database, fake services): the agent must not touch real systems during evaluation.

### Exercise

Write ten test cases in the format above for the agent of the final project, including two adversarial cases. Run each case three times and report: success rate, average steps, average cost, and the cases in which the trajectory contains a forbidden tool.

## 9. When NOT to use an agent

- The path of the steps is known in advance: a deterministic workflow is more reliable.
- The cost of an error is high and the task is repeatable: a pipeline with validations and tests is worth more than an autonomous agent.
- An immediate, low-latency answer is needed: every reasoning and tool step adds time.
- There is not yet a way to evaluate the behavior: without a dataset and metrics you cannot know whether the agent is improving or getting worse.

Guiding principle: start from the simplest workflow that works, increase autonomy only where needed and measure the effect with the evaluation from module 6.

## Suggested final project

Build an agent for a real problem in your field (helpdesk, order management, HR support, document search). Deliverable:

- the rationale for choosing agent vs workflow, with an honest comparison;
- the list of tools with descriptions and permissions;
- the loop with step and budget limits;
- at least one human confirmation for a sensitive action;
- the log of three complete runs;
- a suite of ten test cases, including a prompt injection attempt through content read by the agent.

## Further reading

- [Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022)](https://arxiv.org/abs/2210.03629): the reasoning-action loop underlying tool use.
- [Anthropic, "Building effective agents" (2024)](https://www.anthropic.com/research/building-effective-agents): distinction between workflows and agents and an overview of the patterns discussed in this module.
- [Anthropic, "Effective context engineering for AI agents"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): management of context, memory and sub-agents.
- [Lilian Weng, "LLM Powered Autonomous Agents" (2023)](https://lilianweng.github.io/posts/2023-06-23-agent/): overview of planning, memory and tool use.
- [Model Context Protocol](https://modelcontextprotocol.io): specification and documentation of the MCP protocol.
