# Corso: AI Engineering & LLM Integration

**Lingua / Language:** italiano · [English version](#course-ai-engineering--llm-integration)

Obiettivo: Trasformare un software engineer in un AI Engineer capace di orchestrare modelli linguistici per risolvere problemi reali.

## Struttura del corso

### Prerequisiti

- Programmazione in Python a livello intermedio (funzioni, classi, gestione delle eccezioni, ambienti virtuali).
- Familiarità con API HTTP e JSON.
- Uso di base di Git e della riga di comando.
- Non sono richieste conoscenze di machine learning: il Modulo 1 fornisce le basi necessarie.

### Percorso

| Modulo | Tema | Durata indicativa | Consegna |
|---|---|---|---|
| 1 | Introduzione: LLM, inferenza, harness | 4 h | Analisi di un prodotto AI esistente (modello vs harness) |
| 2 | Fondamenti e Prompt Engineering | 6 h | Classificatore few-shot ed estrattore JSON validato |
| 3 | Architettura e chiamate API | 6 h | Client con retry, streaming e log di costi e latenza |
| 4 | Sistema RAG | 10 h | Prototipo RAG su un corpus a scelta |
| 5 | Esempio pratico passo-passo | 8 h | Assistente per il manuale tecnico, con interfaccia Streamlit |
| 6 | Valutazione e ottimizzazione | 8 h | Dataset di 30 casi e report di confronto tra varianti |
| 7 | Pattern di sviluppo agentico | 10 h | Agente con strumenti, limiti e conferma umana |
| 8 | Produzione, sicurezza e governance | 6 h | Checklist di rilascio e threat model del progetto |

I moduli sono progressivi: ogni consegna riusa il lavoro dei moduli precedenti. Il materiale di approfondimento di ogni modulo è in italiano nella cartella [moduli/IT/](moduli/IT/) (Markdown e HTML) e in inglese nella cartella [moduli/EN/](moduli/EN/) (HTML).

### Progetto finale

Un assistente AI su un dominio reale (documentazione interna, supporto clienti, normativa) che integri:

- retrieval su documenti con citazione delle fonti (moduli 4 e 5);
- almeno uno strumento esterno invocato dal modello, con conferma umana per le azioni sensibili (modulo 7);
- una suite di valutazione con metriche di retrieval e di generazione, eseguita prima di ogni modifica (modulo 6);
- una checklist di produzione: costi, osservabilità, sicurezza e trattamento dei dati (modulo 8).

## Modulo 1: Introduzione - Cos'è un LLM, come funziona e che cos'è un Harness

Prima di imparare a usare un modello linguistico bisogna capire che cosa è, come nasce e come lavora. Questo modulo è concettuale (nessun codice): costruisce il modello mentale su cui si basano tutti i moduli successivi.

### Che cos'è un LLM

Un Large Language Model (LLM) è una rete neurale di grandi dimensioni addestrata su enormi quantità di testo per un compito apparentemente semplice: data una sequenza di token, prevedere il token successivo. Non consulta un database di fatti: genera testo statisticamente plausibile dato il contesto. Le capacità "intelligenti" emergono dalla scala (miliardi di parametri, migliaia di miliardi di token di addestramento).

Token: il testo viene spezzato in unità chiamate token (in media circa 3-4 caratteri in inglese, di più in italiano). Il modello lavora sui token, non sulle parole. Costi, limiti di lunghezza e finestra di contesto si misurano in token.

Parametri (o pesi): i numeri della rete neurale che vengono appresi durante l'addestramento. Un modello è, in sostanza, l'insieme dei suoi pesi più il codice che li esegue.

### Come funziona: l'architettura Transformer

- Embedding: ogni token viene convertito in un vettore di numeri che ne rappresenta il significato.
- Self-attention: ogni token "guarda" gli altri token della sequenza e pesa quanto sono rilevanti per interpretarlo. È il meccanismo che permette di collegare un pronome al suo soggetto o una domanda al contesto che la precede.
- Strati impilati: decine o centinaia di strati di attention e reti feed-forward trasformano progressivamente la rappresentazione.
- Output: l'ultimo vettore viene convertito in una distribuzione di probabilità su tutto il vocabolario (decine di migliaia di token possibili).

### Come viene addestrato

1. Pre-training: il modello legge enormi corpora di testo e, a ogni posizione, cerca di prevedere il token successivo. Dopo ogni errore, i pesi vengono aggiornati con backpropagation e gradient descent. È la fase più costosa (migliaia di GPU per settimane). Il risultato è un modello base, bravo a completare testi ma non a seguire istruzioni.
2. Supervised Fine-Tuning (SFT): il modello viene addestrato su coppie istruzione-risposta scritte da persone, per imparare il formato di un assistente.
3. Allineamento (RLHF, RLAIF o preference optimization): un modello di ricompensa, addestrato su confronti di preferenza tra risposte, orienta il modello verso risposte utili, oneste e sicure.

Punto chiave: l'addestramento avviene una volta sola, non a ogni conversazione. Durante l'uso i pesi sono fissi: il modello non impara dalle tue chat in tempo reale.

### Come avviene l'inferenza

L'inferenza è l'uso del modello già addestrato per produrre un output.

1. Tokenizzazione: il prompt viene convertito in una sequenza di token.
2. Prefill: il modello elabora in parallelo tutti i token del prompt e salva le chiavi e i valori dell'attention nella KV cache, così da non ricalcolare il passato a ogni passo.
3. Decoding autoregressivo: il modello genera un token alla volta. Ogni token generato viene riaggiunto alla sequenza e il ciclo ricomincia. Per questo le risposte arrivano "a flusso" (streaming) e la latenza cresce con la lunghezza dell'output.
4. Campionamento: dalla distribuzione di probabilità si sceglie il token successivo. I parametri principali sono temperature (più alta = più variabilità), top-p e top-k (restringono i candidati) e max tokens (limite di output).
5. Stop: la generazione termina con un token di fine sequenza o al raggiungimento del limite.

Implicazioni pratiche:

- Il modello è stateless: a ogni chiamata riceve l'intera conversazione. La "memoria" esiste solo perché la rimandiamo nel contesto.
- La finestra di contesto è finita: ciò che non entra nel contesto per il modello non esiste.
- Il modello può allucinare: produce testo plausibile anche quando non ha l'informazione, senza una verifica interna dei fatti.
- Con temperatura alta, lo stesso prompt può produrre risposte diverse.

### Famiglie di modelli

- Modelli chiusi (accessibili solo via API) e modelli open-weight (pesi scaricabili, eseguibili in proprio o tramite provider terzi).
- Modelli grandi e capaci, modelli piccoli ed economici per compiti semplici come classificazione e routing.
- Modelli di ragionamento, che dedicano token di "pensiero" prima della risposta: più accurati su compiti complessi, più lenti e costosi.
- Modelli multimodali, che accettano anche immagini, documenti o audio.

### Che cos'è un Harness

Un harness (letteralmente "imbracatura", le briglie che guidano il cavallo) è il software che circonda l'LLM e ne orchestra l'uso. Costruisce il prompt e il contesto, invia le richieste al modello, interpreta la risposta, esegue gli strumenti (tool) che il modello ha richiesto, reinserisce i risultati nel contesto e ripete il ciclo finché il compito non è completato.

Esempi: Claude Code, un agente costruito con LangChain o LlamaIndex, un chatbot con retrieval (RAG).

Componenti tipici di un harness:

- System prompt e istruzioni
- Gestione del contesto (cosa includere, riassunti, compattazione della storia)
- Definizione ed esecuzione dei tool (function calling)
- Loop agentico: chiama il modello, esegue i tool richiesti, rimanda i risultati
- Gestione degli errori, retry con backoff, rate limiting, streaming
- Permessi, sicurezza, sandbox, logging e valutazione
- Memoria esterna (file, database vettoriali, storico delle sessioni)

### Relazione tra LLM e Harness

- L'LLM è il "cervello": ragiona e decide. Produce solo testo, anche quando il testo descrive l'uso di uno strumento. Da solo non ha accesso a filesystem, rete o orologio.
- L'harness è il "corpo" e l'infrastruttura: esegue davvero le azioni, fornisce al modello le informazioni e i mezzi, applica le regole.
- Ciclo tipico: l'harness invia contesto e definizioni dei tool → il modello risponde "chiama il tool X con argomenti Y" → l'harness esegue X → invia il risultato → il modello continua fino alla risposta finale.
- Lo stesso modello, con harness diversi, produce agenti con capacità e affidabilità molto diverse: la qualità di un sistema AI dipende da entrambi.
- Per un AI Engineer il modello si sceglie, l'harness si progetta. Gran parte di questo corso (prompt, output strutturato, RAG, valutazione) è lavoro sull'harness.

## Modulo 2: Fondamenti e Prompt Engineering

Prima di scrivere codice, bisogna imparare a "parlare" con il modello. Non è solo questione di parole, ma di struttura.

L'Anatomia di un Prompt: Istruzione, Contesto, Dati di input e Indicatori di output.

Tecniche Avanzate:

- Few-Shot Prompting: Fornire esempi nel prompt per guidare il comportamento.
- Chain of Thought (CoT): Chiedere al modello di "pensare ad alta voce" per compiti logici complessi.
- Output Strutturato: Costringere l'LLM a rispondere esclusivamente in JSON per l'integrazione software.
- Prompt come codice: template parametrizzati, versionamento e test di regressione dei prompt.
- Prompt injection: riconoscere quando un testo in input prova a comportarsi da istruzione.

## Modulo 3: Architettura e Chiamate API

Gestire le chiamate non significa solo fare un POST. Bisogna gestire costi, latenza e fallimenti.

- Provider a confronto: OpenAI, Anthropic e modelli Open Source tramite Groq o Together AI.
- Gestione degli Errori: Implementare logiche di Exponential Backoff per i limiti di velocità (Rate Limiting).
- Streaming delle risposte: Come implementare la UX "tipo ChatGPT" che scrive i token in tempo reale.
- Ottimizzazione dei costi: prompt caching, elaborazione batch, scelta del modello per compito.
- Sicurezza di base: gestione delle chiavi API, budget di spesa, dati inviati al provider.

## Modulo 4: Il Cuore del Progetto - Creare un Sistema RAG

Il Retrieval-Augmented Generation (RAG) permette all'LLM di rispondere basandosi su documenti privati che non ha mai visto durante il training.

**Passo 1**: Ingestione e Chunking

Non puoi dare 500 pagine di PDF in un colpo solo. Devi spezzettarli.

Strategia: Dividere il testo in blocchi (chunk) di 1000 caratteri con un overlap (sovrapposizione) di 200 per non perdere il contesto.

**Passo 2**: Embeddings e Vector Database

Trasformiamo il testo in numeri (vettori).

Tool: Utilizzo di database vettoriali come Pinecone, ChromaDB o Qdrant.

Logica: Due frasi con significato simile avranno vettori vicini nello "spazio semantico".

**Passo 3**: Il Ciclo di Recupero (Retrieval)

1. L'utente fa una domanda.
2. Trasformi la domanda in un vettore.
3. Cerchi nel Vector DB i 3-5 pezzi di documento più simili.
4. Invii all'LLM la domanda + i pezzi trovati come contesto.

**Passo 4**: Tecniche avanzate

Ricerca ibrida (vettoriale + parole chiave), riscrittura della query, reranking e chunking gerarchico, da introdurre solo quando la valutazione mostra che servono.

## Modulo 5: Esempio Pratico Passo-Passo

Immaginiamo di voler creare un assistente per i manuali tecnici di una lavatrice.

1. Caricamento: Carichiamo il file manuale_lavatrice.pdf.
2. Frammentazione: Il sistema divide il manuale in paragrafi.
3. Indexing: Ogni paragrafo diventa un vettore nel nostro DB.
4. Query: L'utente chiede: "Cosa significa l'errore E18?"
5. Retrieval: Il sistema trova il paragrafo: "Errore E18: Problema di scarico acqua, pulire il filtro".
6. Generation: L'LLM risponde: "L'errore E18 indica un problema allo scarico. Ti consiglio di controllare il filtro situato in basso a destra."
7. Interfaccia: una pagina Streamlit con chat, fonti citate e feedback dell'utente.

## Modulo 6: Valutazione e Ottimizzazione

Come capire se l'AI sta mentendo (allucinazioni)?

- Dataset di valutazione: casi reali, sintetici, negativi e avversari, con la fonte attesa.
- RAGAS: Un framework per misurare la fedeltà della risposta ai documenti forniti.
- LLM-as-a-judge: usare un modello come valutatore, con criteri espliciti e calibrazione sul giudizio umano.
- Prompt Tuning: Affinare il "System Prompt" per limitare la creatività del modello e attenersi ai fatti.

## Modulo 7: Pattern di Sviluppo Agentico

Un agente è un sistema in cui l'LLM decide dinamicamente come procedere: sceglie quali tool usare, in che ordine e quando fermarsi. Un workflow, invece, è un percorso di passaggi definito in anticipo dal codice, in cui il modello risolve solo i singoli passaggi. Prima di costruire un agente conviene chiedersi se un workflow più semplice e prevedibile basta: è più economico, più veloce e più facile da testare.

I pattern che seguono sono mattoni riutilizzabili, da combinare in base al problema.

### Pattern fondamentali

- Prompt chaining: il compito è scomposto in una sequenza di chiamate al modello, dove l'output di una diventa l'input della successiva. Esempio: estrarre i dati da una fattura, poi validarli, poi generare la risposta.
- Routing: un passo di classificazione smista la richiesta verso un prompt, un modello o un flusso specializzato. Esempio: domande tecniche al RAG, richieste di rimborso a un flusso dedicato.
- Parallelizzazione: più chiamate indipendenti vengono eseguite in parallelo, poi i risultati vengono aggregati (sectioning) oppure più risposte vengono prodotte per lo stesso input e confrontate o votate (voting).

### Pattern agentici

- ReAct (Reason + Act): il modello alterna un passo di ragionamento, un'azione (chiamata a un tool) e l'osservazione del risultato, fino alla risposta finale. È il loop di base della maggior parte degli agenti.
- Tool use: il modello riceve la descrizione dei tool disponibili (nome, scopo, schema dei parametri) e produce chiamate strutturate che l'harness esegue. Qualità delle descrizioni e degli schemi incide direttamente sull'affidabilità.
- Plan-and-Execute: prima un passo produce un piano esplicito (lista di sotto-task), poi un esecutore lo completa, eventualmente ricalcolando il piano quando un passo fallisce. Utile per compiti lunghi con più fasi.
- Reflection (self-critique): il modello valuta la propria bozza rispetto a criteri espliciti e la migliora. Variante: evaluator-optimizer, dove un secondo prompt (o modello) giudica l'output e rimanda indietro il feedback.
- Orchestrator-Workers: un agente orchestratore scompone il compito e delega sotto-task a worker specializzati (in parallelo o in sequenza), poi sintetizza i risultati.
- Multi-agent: più agenti con ruoli distinti collaborano, tipicamente con uno schema supervisor (un coordinatore smista il lavoro) oppure handoff (un agente passa il controllo a un altro). Aggiunge potenza ma anche complessità e costi: da usare solo se un singolo agente non basta.

### Pattern di controllo e affidabilità

- Human-in-the-loop: l'agente si ferma e chiede conferma prima di azioni irreversibili o costose (invio di email, modifica di dati, acquisti). Il punto di approvazione va progettato, non lasciato al caso.
- Limiti espliciti: massimo numero di iterazioni, budget di token, timeout e condizioni di uscita. Senza questi, un loop agentico può non terminare mai.
- Guardrail: validazione dell'input e dell'output (schemi JSON, filtri, controlli di policy) prima che un risultato venga usato o mostrato.
- Permessi e sandbox: ogni tool dichiara cosa può fare e l'harness ne limita l'accesso (filesystem, rete, credenziali). Principio del minimo privilegio.
- Gestione degli errori: retry con backoff per errori transitori, fallback su modelli o strumenti alternativi, messaggi d'errore restituiti al modello perché possa correggersi.

### Pattern di memoria e contesto

- Memoria a breve termine: la storia della sessione nel contesto, da gestire con troncamento o riassunti quando cresce.
- Memoria a lungo termine: fatti e preferenze salvati all'esterno (file, database, vector store) e recuperati quando servono.
- Context engineering: scegliere con cura cosa mettere nel contesto a ogni passo, perché ciò che il modello non vede non può usarlo. È la competenza che lega i moduli 2, 4 e 6.
- Sub-agenti: un compito isolato viene affidato a un agente con un contesto proprio e pulito, che restituisce solo il risultato sintetico. Protegge il contesto principale da dettagli inutili.

### Protocolli e integrazione

- MCP (Model Context Protocol): standard aperto con cui un harness scopre e usa tool, risorse e prompt esposti da server esterni, senza integrazioni custom per ogni coppia modello-applicazione.
- Skill e istruzioni riusabili: procedure e conoscenza di dominio raccolte in file richiamabili dall'agente solo quando servono, per tenere il contesto leggero.

### Valutare un agente

- Tasso di successo sul compito, numero di passi, costo e latenza per compito completato.
- Valutazione della traiettoria: gli strumenti giusti, con gli argomenti giusti, nell'ordine giusto.
- Test di sicurezza: contenuti malevoli letti dall'agente non devono portare ad azioni non autorizzate.

### Quando NON usare un agente

- Se il percorso dei passaggi è noto in anticipo, un workflow deterministico è preferibile.
- Se il costo di un errore è alto e il compito è ripetibile, una pipeline con validazioni vale più di un agente autonomo.
- Ogni livello di autonomia aggiunge latenza, costi e superfici di errore: si parte dal workflow più semplice che funziona e si aumenta l'autonomia solo dove serve, misurandone l'effetto con la valutazione (modulo 6).

## Modulo 8: Produzione, Sicurezza e Governance

Un prototipo che funziona in demo non è un sistema in produzione. Questo modulo copre ciò che serve per rilasciare e mantenere un'applicazione LLM in modo sostenibile.

- Architettura di produzione: gateway verso i modelli, cache, code, fallback tra modelli, separazione tra ingestione e servizio.
- LLMOps: versionamento di prompt, modelli e indici; tracing di ogni richiesta; valutazione continua come test di regressione.
- Costi: budget per funzionalità, routing verso modelli più economici, prompt caching, batch, limiti per utente.
- Sicurezza: i rischi della OWASP Top 10 per applicazioni LLM (prompt injection, divulgazione di dati sensibili, eccesso di autonomia, gestione insicura dell'output), threat model e difese a livello di harness.
- Privacy e conformità: dati personali nei prompt e nei log, accordi con i provider sull'uso dei dati, GDPR e obblighi dell'AI Act europeo in base alla classe di rischio.
- Scelta dell'approccio: prompt, RAG o fine-tuning, con criteri espliciti.

## Toolset consigliato per il corso

| Ambito | Strumenti |
|---|---|
| Linguaggio | Python (standard de facto) |
| Orchestrazione | LangChain o LlamaIndex, oppure SDK nativi dei provider per capire cosa succede sotto |
| Validazione | Pydantic per schemi e output strutturato |
| Vector database | ChromaDB in locale; Qdrant o Pinecone per scenari di produzione |
| Valutazione | RAGAS, dataset di test versionati nel repository |
| Osservabilità | Tracing delle chiamate (ad esempio Langfuse o strumenti equivalenti) |
| Interfaccia | Streamlit (per prototipi veloci) |
| Modelli | Un modello di frontiera di un provider commerciale (famiglie Claude di Anthropic o GPT di OpenAI), un modello open-weight servito tramite Groq o Together AI, un modello piccolo ed economico per classificazione e routing. Verificare le versioni correnti all'avvio di ogni edizione del corso. |

---

# Course: AI Engineering & LLM Integration

**Language:** [Versione italiana](#corso-ai-engineering--llm-integration) · English

Goal: Turn a software engineer into an AI Engineer able to orchestrate language models to solve real-world problems.

## Course structure

### Prerequisites

- Intermediate-level Python programming (functions, classes, exception handling, virtual environments).
- Familiarity with HTTP APIs and JSON.
- Basic use of Git and the command line.
- No machine learning knowledge is required: Module 1 provides the basics you need.

### Path

| Module | Topic | Estimated duration | Deliverable |
|---|---|---|---|
| 1 | Introduction: LLMs, inference, harness | 4 h | Analysis of an existing AI product (model vs harness) |
| 2 | Fundamentals and Prompt Engineering | 6 h | Few-shot classifier and validated JSON extractor |
| 3 | Architecture and API calls | 6 h | Client with retry, streaming, and cost and latency logs |
| 4 | RAG system | 10 h | RAG prototype on a corpus of your choice |
| 5 | Step-by-step practical example | 8 h | Assistant for a technical manual, with a Streamlit interface |
| 6 | Evaluation and optimization | 8 h | Dataset of 30 cases and a comparison report across variants |
| 7 | Agentic development patterns | 10 h | Agent with tools, limits and human confirmation |
| 8 | Production, security and governance | 6 h | Release checklist and threat model for the project |

The modules are progressive: each deliverable reuses the work of the previous modules. The supporting material for each module is in Italian in the [moduli/IT/](moduli/IT/) folder (Markdown and HTML) and in English in the [moduli/EN/](moduli/EN/) folder (HTML).

### Final project

An AI assistant on a real domain (internal documentation, customer support, regulations) that integrates:

- retrieval over documents with source citations (modules 4 and 5);
- at least one external tool invoked by the model, with human confirmation for sensitive actions (module 7);
- an evaluation suite with retrieval and generation metrics, run before every change (module 6);
- a production checklist: costs, observability, security and data handling (module 8).

## Module 1: Introduction - What an LLM Is, How It Works and What a Harness Is

Before learning to use a language model, you need to understand what it is, how it came about and how it works. This module is conceptual (no code): it builds the mental model on which all the following modules rely.

### What an LLM is

A Large Language Model (LLM) is a large neural network trained on huge amounts of text for an apparently simple task: given a sequence of tokens, predict the next token. It does not consult a database of facts: it generates statistically plausible text given the context. "Intelligent" capabilities emerge from scale (billions of parameters, trillions of training tokens).

Token: text is split into units called tokens (on average about 3-4 characters in English, more in Italian). The model works on tokens, not words. Costs, length limits and the context window are measured in tokens.

Parameters (or weights): the numbers in the neural network that are learned during training. A model is, in essence, its set of weights plus the code that runs them.

### How it works: the Transformer architecture

- Embedding: each token is converted into a vector of numbers that represents its meaning.
- Self-attention: each token "looks at" the other tokens in the sequence and weighs how relevant they are for interpreting it. It is the mechanism that lets a pronoun be linked to its subject, or a question to the context that precedes it.
- Stacked layers: dozens or hundreds of attention layers and feed-forward networks progressively transform the representation.
- Output: the last vector is converted into a probability distribution over the whole vocabulary (tens of thousands of possible tokens).

### How it is trained

1. Pre-training: the model reads enormous text corpora and, at each position, tries to predict the next token. After each error, the weights are updated with backpropagation and gradient descent. This is the most expensive phase (thousands of GPUs for weeks). The result is a base model: good at completing text, but not at following instructions.
2. Supervised Fine-Tuning (SFT): the model is trained on instruction-response pairs written by people, to learn the format of an assistant.
3. Alignment (RLHF, RLAIF or preference optimization): a reward model, trained on preference comparisons between responses, steers the model toward answers that are useful, honest and safe.

Key point: training happens once, not in every conversation. During use the weights are fixed: the model does not learn from your chats in real time.

### How inference works

Inference is the use of the already trained model to produce an output.

1. Tokenization: the prompt is converted into a sequence of tokens.
2. Prefill: the model processes all the prompt tokens in parallel and stores the attention keys and values in the KV cache, so the past does not have to be recomputed at every step.
3. Autoregressive decoding: the model generates one token at a time. Each generated token is added back to the sequence and the cycle starts again. This is why responses arrive "as a stream" (streaming), and why latency grows with the length of the output.
4. Sampling: the next token is chosen from the probability distribution. The main parameters are temperature (higher = more variability), top-p and top-k (restrict the candidates) and max tokens (output limit).
5. Stop: generation ends with an end-of-sequence token or when the limit is reached.

Practical implications:

- The model is stateless: at each call it receives the entire conversation. The "memory" exists only because we send it back in the context.
- The context window is finite: what does not fit into the context does not exist for the model.
- The model can hallucinate: it produces plausible text even when it lacks the information, with no internal fact-checking.
- With high temperature, the same prompt can produce different answers.

### Model families

- Closed models (accessible only via API) and open-weight models (weights can be downloaded, run yourself or through third-party providers).
- Large, capable models and small, inexpensive models for simple tasks such as classification and routing.
- Reasoning models, which spend "thinking" tokens before the answer: more accurate on complex tasks, but slower and more expensive.
- Multimodal models, which also accept images, documents or audio.

### What a Harness is

A harness (literally "the straps that guide a horse") is the software that surrounds the LLM and orchestrates its use. It builds the prompt and the context, sends requests to the model, interprets the response, executes the tools the model has requested, feeds the results back into the context and repeats the cycle until the task is complete.

Examples: Claude Code, an agent built with LangChain or LlamaIndex, a chatbot with retrieval (RAG).

Typical components of a harness:

- System prompt and instructions
- Context management (what to include, summaries, compaction of the history)
- Tool definition and execution (function calling)
- Agent loop: call the model, execute the requested tools, send the results back
- Error handling, retry with backoff, rate limiting, streaming
- Permissions, security, sandbox, logging and evaluation
- External memory (files, vector databases, session history)

### Relationship between LLM and Harness

- The LLM is the "brain": it reasons and decides. It produces only text, even when the text describes the use of a tool. On its own, it has no access to the filesystem, the network or a clock.
- The harness is the "body" and the infrastructure: it actually performs the actions, provides the model with information and means, and applies the rules.
- Typical cycle: the harness sends the context and the tool definitions → the model replies "call tool X with arguments Y" → the harness executes X → sends the result back → the model continues until the final answer.
- The same model, with different harnesses, produces agents with very different capabilities and reliability: the quality of an AI system depends on both.
- For an AI Engineer, the model is chosen and the harness is designed. Much of this course (prompts, structured output, RAG, evaluation) is work on the harness.

## Module 2: Fundamentals and Prompt Engineering

Before writing code, you need to learn how to "talk" to the model. It is not just a matter of words, but of structure.

Anatomy of a Prompt: Instruction, Context, Input data and Output indicators.

Advanced techniques:

- Few-Shot Prompting: providing examples in the prompt to guide the behavior.
- Chain of Thought (CoT): asking the model to "think out loud" for complex logical tasks.
- Structured Output: forcing the LLM to respond exclusively in JSON for software integration.
- Prompts as code: parameterized templates, versioning and regression tests for prompts.
- Prompt injection: recognizing when a text in the input tries to behave like an instruction.

## Module 3: Architecture and API Calls

Handling calls is not just a matter of making a POST request. You need to manage costs, latency and failures.

- Provider comparison: OpenAI, Anthropic and open-source models via Groq or Together AI.
- Error handling: implementing exponential backoff for rate limits.
- Response streaming: how to implement the ChatGPT-style UX that writes tokens in real time.
- Cost optimization: prompt caching, batch processing, choosing the model for each task.
- Basic security: API key management, spending budgets, data sent to the provider.

## Module 4: The Core of the Project - Building a RAG System

Retrieval-Augmented Generation (RAG) lets the LLM answer based on private documents it has never seen during training.

**Step 1**: Ingestion and Chunking

You cannot feed 500 pages of PDF in one go. You have to split them up.

Strategy: split the text into chunks of 1000 characters with an overlap of 200, so as not to lose context.

**Step 2**: Embeddings and Vector Database

We turn text into numbers (vectors).

Tools: vector databases such as Pinecone, ChromaDB or Qdrant.

Logic: two sentences with similar meanings have nearby vectors in the "semantic space".

**Step 3**: The Retrieval Loop

1. The user asks a question.
2. The question is turned into a vector.
3. The vector DB is searched for the 3-5 most similar pieces of document.
4. The question and the retrieved pieces are sent to the LLM as context.

**Step 4**: Advanced techniques

Hybrid search (vector + keywords), query rewriting, reranking and hierarchical chunking: to be introduced only when the evaluation shows they are needed.

## Module 5: Practical Step-by-Step Example

Let's imagine we want to build an assistant for the technical manuals of a washing machine.

1. Upload: we load the file manuale_lavatrice.pdf.
2. Splitting: the system divides the manual into paragraphs.
3. Indexing: each paragraph becomes a vector in our DB.
4. Query: the user asks: "What does error E18 mean?"
5. Retrieval: the system finds the paragraph: "Error E18: Drain problem, clean the filter."
6. Generation: the LLM answers: "Error E18 indicates a drainage problem. I recommend checking the filter located at the bottom right."
7. Interface: a Streamlit page with chat, cited sources and user feedback.

## Module 6: Evaluation and Optimization

How can you tell whether the AI is making things up (hallucinating)?

- Evaluation dataset: real, synthetic, negative and adversarial cases, with the expected source.
- RAGAS: a framework for measuring how faithful an answer is to the documents provided.
- LLM-as-a-judge: using a model as the evaluator, with explicit criteria and calibration against human judgment.
- Prompt tuning: refining the "system prompt" to limit the model's creativity and keep it anchored to the facts.

## Module 7: Agentic Development Patterns

An agent is a system in which the LLM decides dynamically how to proceed: it chooses which tools to use, in what order, and when to stop. A workflow, instead, is a path of steps defined in advance by the code, in which the model only solves the individual steps. Before building an agent, it is worth asking whether a simpler and more predictable workflow is enough: it is cheaper, faster and easier to test.

The patterns below are reusable building blocks, to be combined depending on the problem.

### Fundamental patterns

- Prompt chaining: the task is broken down into a sequence of model calls, where the output of one becomes the input of the next. Example: extract the data from an invoice, then validate it, then generate the reply.
- Routing: a classification step sends the request to a specialized prompt, model or flow. Example: technical questions to the RAG, refund requests to a dedicated flow.
- Parallelization: multiple independent calls run in parallel, then the results are aggregated (sectioning), or multiple responses are produced for the same input and compared or voted on (voting).

### Agentic patterns

- ReAct (Reason + Act): the model alternates a reasoning step, an action (a tool call) and the observation of the result, until the final answer. It is the basic loop of most agents.
- Tool use: the model receives the description of the available tools (name, purpose, parameter schema) and produces structured calls that the harness executes. The quality of the descriptions and schemas directly affects reliability.
- Plan-and-Execute: first a step produces an explicit plan (a list of subtasks), then an executor completes it, possibly recalculating the plan when a step fails. Useful for long, multi-phase tasks.
- Reflection (self-critique): the model evaluates its own draft against explicit criteria and improves it. Variant: evaluator-optimizer, where a second prompt (or model) judges the output and sends feedback back.
- Orchestrator-Workers: an orchestrator agent breaks the task down and delegates subtasks to specialized workers (in parallel or in sequence), then synthesizes the results.
- Multi-agent: several agents with distinct roles collaborate, usually with a supervisor scheme (a coordinator distributes the work) or handoff (an agent passes control to another). It adds power, but also complexity and cost: use it only if a single agent is not enough.

### Control and reliability patterns

- Human-in-the-loop: the agent stops and asks for confirmation before irreversible or costly actions (sending emails, modifying data, purchases). The approval point must be designed, not left to chance.
- Explicit limits: maximum number of iterations, token budget, timeouts and exit conditions. Without these, an agentic loop may never terminate.
- Guardrails: validation of input and output (JSON schemas, filters, policy checks) before a result is used or shown.
- Permissions and sandbox: each tool declares what it can do, and the harness limits its access (filesystem, network, credentials). Principle of least privilege.
- Error handling: retry with backoff for transient errors, fallback to alternative models or tools, and error messages returned to the model so it can correct itself.

### Memory and context patterns

- Short-term memory: the session history in the context, managed with truncation or summaries when it grows.
- Long-term memory: facts and preferences saved externally (files, databases, vector stores) and retrieved when needed.
- Context engineering: carefully choosing what goes into the context at each step, because what the model does not see, it cannot use. It is the skill that ties together modules 2, 4 and 6.
- Sub-agents: an isolated task is delegated to an agent with its own clean context, which returns only the synthesized result. It protects the main context from unnecessary detail.

### Protocols and integration

- MCP (Model Context Protocol): an open standard through which a harness discovers and uses tools, resources and prompts exposed by external servers, without custom integrations for every model-application pair.
- Skills and reusable instructions: procedures and domain knowledge collected in files that the agent calls on only when needed, to keep the context lightweight.

### Evaluating an agent

- Task success rate, number of steps, cost and latency per completed task.
- Trajectory evaluation: the right tools, with the right arguments, in the right order.
- Security tests: malicious content read by the agent must not lead to unauthorized actions.

### When NOT to use an agent

- If the path of steps is known in advance, a deterministic workflow is preferable.
- If the cost of an error is high and the task is repeatable, a pipeline with validations is worth more than an autonomous agent.
- Each level of autonomy adds latency, cost and surfaces for errors: start from the simplest workflow that works, and increase autonomy only where it is needed, measuring its effect with the evaluation (module 6).

## Module 8: Production, Security and Governance

A prototype that works in a demo is not a production system. This module covers what is needed to release and maintain an LLM application in a sustainable way.

- Production architecture: gateway to the models, cache, queues, fallback between models, separation between ingestion and serving.
- LLMOps: versioning of prompts, models and indexes; tracing of every request; continuous evaluation as regression tests.
- Costs: budget per feature, routing to cheaper models, prompt caching, batching, per-user limits.
- Security: the OWASP Top 10 risks for LLM applications (prompt injection, sensitive data disclosure, excessive agency, insecure output handling), threat model and defenses at the harness level.
- Privacy and compliance: personal data in prompts and logs, agreements with providers on data use, GDPR and the obligations of the European AI Act based on risk class.
- Choosing the approach: prompt, RAG or fine-tuning, with explicit criteria.

## Recommended course toolset

| Area | Tools |
|---|---|
| Language | Python (de facto standard) |
| Orchestration | LangChain or LlamaIndex, or the providers' native SDKs to understand what happens under the hood |
| Validation | Pydantic for schemas and structured output |
| Vector database | ChromaDB locally; Qdrant or Pinecone for production scenarios |
| Evaluation | RAGAS, versioned test datasets in the repository |
| Observability | Tracing of calls (for example Langfuse or equivalent tools) |
| Interface | Streamlit (for quick prototypes) |
| Models | A frontier model from a commercial provider (Anthropic's Claude family or OpenAI's GPT family), an open-weight model served via Groq or Together AI, a small, inexpensive model for classification and routing. Check the current versions at the start of each course edition. |
