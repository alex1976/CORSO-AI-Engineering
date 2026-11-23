# Modulo 1: Introduzione - Cos'è un LLM, come funziona e che cos'è un Harness

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- spiegare che cosa fa un LLM in una sola frase, senza ricorrere a formule magiche;
- descrivere il percorso di un prompt dalla tokenizzazione fino al token generato;
- distinguere addestramento e inferenza e dire quale dei due accade quando usi un chatbot;
- definire un harness e individuarne i componenti in un prodotto che usi ogni giorno;
- spiegare perché l'LLM da solo non può "fare" nulla nel mondo e che cosa serve per renderlo un agente.

## 1. Che cos'è un LLM

Un Large Language Model è una funzione matematica molto grande. Riceve una sequenza di token e restituisce una distribuzione di probabilità sul token successivo. Tutto il resto (rispondere a domande, scrivere codice, riassumere contratti) è una conseguenza di questa operazione ripetuta.

### Token

Il modello non vede parole né caratteri, ma token: frammenti di testo prodotti da un tokenizzatore. In inglese un token corrisponde in media a circa 3-4 caratteri; in italiano, con parole più lunghe e desinenze flessive, il rapporto è più basso e un testo occupa più token.

Esempio di esercizio: apri un tokenizzatore online (ne esistono diversi per le famiglie di modelli più diffuse) e confronta quanti token servono per la stessa frase in italiano e in inglese. Annota la differenza: ti servirà nel modulo 3 per stimare costi e limiti di contesto.

### Parametri

I parametri sono i pesi della rete neurale, numeri in virgola mobile appresi durante l'addestramento. Un modello con miliardi di parametri non contiene una "base di conoscenza" leggibile: la conoscenza è distribuita in tutti i pesi insieme.

## 2. Come funziona: il Transformer

L'architettura dominante è il Transformer. Il flusso, semplificato, è questo:

1. **Embedding**: ogni token diventa un vettore di centinaia o migliaia di numeri.
2. **Self-attention**: in ogni strato, ogni token calcola quanto "guardare" gli altri token della sequenza. Così la parola "essa" acquisisce il significato del sostantivo a cui si riferisce, anche se è lontano nella frase.
3. **Reti feed-forward**: applicano trasformazioni non lineari a ogni posizione.
4. **Strati ripetuti**: il processo si ripete decine o centinaia di volte, affinando la rappresentazione.
5. **Output**: l'ultimo vettore viene proiettato sul vocabolario, producendo un punteggio per ogni token possibile, poi convertito in probabilità.

Concetto da fissare: il modello non "pianifica" la frase intera prima di scriverla. Calcola sempre un solo passo, il token successivo. La coerenza che osservi è il risultato di questo passo ripetuto su una distribuzione molto ricca.

## 3. Come viene addestrato

L'addestramento avviene in fasi distinte, ognuna con un obiettivo e un costo diversi.

### Pre-training

Il modello legge enormi quantità di testo (pagine web, libri, codice, documentazione) e, per ogni posizione, deve prevedere il token successivo. La funzione di perdita (cross-entropy) misura quanto la previsione si discosta dal token reale. Dopo ogni lotto di esempi, i pesi vengono corretti con backpropagation e gradient descent.

Il risultato è un modello base: un ottimo completatore di testi che però non sa ancora dialogare. Se gli scrivi "Qual è la capitale della Francia?" potrebbe rispondere con altre domande, perché nel testo di addestramento le domande sono spesso seguite da altre domande.

### Supervised Fine-Tuning (SFT)

Persone scrivono coppie istruzione-risposta di alta qualità. Il modello viene addestrato su questi esempi e impara il formato dell'assistente: rispondere, seguire istruzioni, usare un tono appropriato.

### Allineamento (RLHF, RLAIF, preference optimization)

Per ogni prompt il modello genera più risposte, e valutatori (umani o altri modelli) indicano quale preferiscono. Un modello di ricompensa impara queste preferenze e guida l'ottimizzazione del modello principale verso risposte più utili, oneste e sicure. Questa fase è quella che spiega perché un modello rifiuta certe richieste o ammette di non sapere.

Punto da non confondere: nessuna di queste fasi avviene durante il tuo uso del modello. Quando chatti, i pesi sono fissi.

### Esercizio di riflessione

Un collega dice: "Il modello ha imparato dalla mia conversazione di ieri". Scrivi in tre righe perché è sbagliato e quale meccanismo, invece, fa sembrare che il modello ricordi (suggerimento: pensa al contesto e a ciò che viene rimandato a ogni chiamata).

## 4. Come avviene l'inferenza

L'inferenza è l'uso del modello già addestrato. Ogni richiesta attraversa questi passi.

1. **Tokenizzazione**: il prompt diventa una sequenza di ID di token.
2. **Prefill**: il modello elabora in parallelo tutti i token del prompt. Durante questo calcolo salva in una **KV cache** le chiavi e i valori dell'attention, così da non ricalcolarli a ogni token successivo. Il tempo di questa fase incide sulla latenza percepita, in particolare sul time-to-first-token.
3. **Decoding autoregressivo**: il modello genera un token alla volta. Il token scelto viene aggiunto alla sequenza e il ciclo riparte. Ogni passo richiede un calcolo completo: per questo l'output lungo costa più tempo.
4. **Campionamento**: dalla distribuzione si sceglie il token successivo.
   - *Temperature* bassa: sceglie quasi sempre il token più probabile, risposte stabili e ripetibili.
   - *Temperature* alta: distribuisce la scelta anche sui token meno probabili, più variabilità e più rischio di incoerenza.
   - *Top-p* (nucleus sampling) e *top-k*: limitano l'insieme dei candidati prima del campionamento.
5. **Stop**: la generazione finisce con un token di fine sequenza, con il raggiungimento del limite di output o con una sequenza di stop definita dal chiamante.

### Conseguenze pratiche che devi ricordare

- **Stateless**: il modello non conserva nulla tra una chiamata e l'altra. Ogni richiesta include tutto ciò che il modello deve sapere. La "memoria" della chat è la storia che l'applicazione rimanda ad ogni turno.
- **Finestra di contesto finita**: ciò che non entra nel contesto non esiste per il modello, né per la risposta corrente né per le successive.
- **Allucinazioni**: il modello produce il testo più plausibile, anche quando non ha l'informazione. Non esiste un controllo interno dei fatti: la verifica deve essere progettata dall'esterno (modulo 4 e modulo 6).
- **Non determinismo**: con temperatura maggiore di zero lo stesso prompt può produrre output diversi. Va tenuto presente nei test.

### Caso d'uso da sviluppare

Prendi un'attività che svolgi già (ad esempio riassumere le email del team). Scrivi il prompt come lo manderesti al modello, poi annota: quanti token stimi per input e output, quale temperatura sceglieresti e perché, e quale informazione mancante potrebbe portare a un'allucinazione.

## 5. Che cos'è un harness

Un harness è il software che circonda il modello e ne orchestra l'uso. Il nome rende l'idea: il modello è il cavallo potente, l'harness è l'imbracatura che lo guida, gli dà una direzione e gli impedisce di fare danni.

Un harness, in pratica:

- costruisce il prompt, incluso il system prompt e le definizioni degli strumenti;
- gestisce il contesto: cosa includere, quando riassumere la storia, quando troncarla;
- invia la richiesta al modello e ne interpreta la risposta;
- esegue gli strumenti richiesti dal modello (tool calling) e rimette i risultati nel contesto;
- ripete il ciclo finché il compito è completato o viene raggiunto un limite;
- applica permessi, validazioni, logging, gestione degli errori e dei limiti di frequenza.

Esempi concreti: un assistente di chat con retrieval su documenti, un agente di coding che legge e modifica file, un bot che consulta un database su richiesta. In tutti questi casi c'è un programma che sta attorno al modello e fa il lavoro "fuori dal testo".

### Componenti tipici

| Componente | Domanda a cui risponde |
|---|---|
| System prompt | Chi è l'assistente e quali regole deve rispettare? |
| Gestione del contesto | Quali informazioni entrano nella finestra, in che ordine? |
| Tool registry | Quali azioni può chiedere di compiere il modello, con quali parametri? |
| Loop di esecuzione | Quando chiamare di nuovo il modello e quando fermarsi? |
| Sicurezza e permessi | Quali azioni richiedono conferma o sono vietate? |
| Osservabilità | Cosa è successo, quanto è costato, dove ha sbagliato? |

## 6. Relazione tra LLM e harness

La distinzione chiave è questa: l'LLM decide, l'harness agisce.

Il modello produce solo testo. Quando "usa uno strumento", in realtà genera un testo strutturato che descrive la chiamata (nome dello strumento e argomenti). È l'harness a riconoscere quella richiesta, eseguire davvero la funzione e rimandare il risultato al modello.

Ciclo tipico, passo per passo:

1. L'harness invia al modello: istruzioni, storia della conversazione, definizioni degli strumenti.
2. Il modello risponde: "chiama `cerca_ordine` con `id = 4411`".
3. L'harness esegue `cerca_ordine(4411)` sul sistema reale.
4. L'harness rimanda al modello il risultato, ad esempio `{"stato": "spedito", "data": "2026-11-12"}`.
5. Il modello genera la risposta finale per l'utente, oppure chiede un altro strumento.

Conseguenze importanti:

- **Stesso modello, harness diverso**: due prodotti con lo stesso LLM possono comportarsi in modo molto diverso, perché cambiano prompt, strumenti, gestione del contesto e controlli.
- **Il modello non è responsabile da solo della sicurezza**: un'istruzione malevola nei dati può indurlo a chiedere un'azione pericolosa. È l'harness che deve limitare ciò che è possibile eseguire.
- **La qualità dipende dalla coppia**: se il modello è bravo ma l'harness fornisce contesto sbagliato, il risultato sarà sbagliato. Lo stesso vale al contrario.

### Esercizio

Scegli un'applicazione che usi un LLM (un assistente di scrittura, un chatbot di un sito, uno strumento di coding). Elenca: che cosa fa il modello, che cosa fa l'harness, quali strumenti sono disponibili e dove sta il confine tra decisione e azione. Se non riesci a identificarlo, probabilmente stai guardando solo il modello e non l'harness.

## 7. Famiglie di modelli e criteri di scelta

Non esiste "il" modello migliore: esiste il modello adatto a un compito, a un budget e a un vincolo di latenza. Conviene conoscere le famiglie principali.

### Chiusi e open-weight

- **Modelli chiusi**: accessibili solo tramite API del provider. Di solito offrono le capacità più alte e nessuna infrastruttura da gestire, ma i dati transitano dal provider e la versione può essere ritirata.
- **Modelli open-weight**: i pesi sono scaricabili. Si possono eseguire sui propri server o tramite provider terzi (ad esempio Groq o Together AI). Danno controllo su dati, versione e costi a volume, ma richiedono competenze di gestione.

### Dimensione e costo

I modelli grandi sono più capaci su compiti complessi; quelli piccoli sono più veloci ed economici e spesso sufficienti per classificare, estrarre campi o smistare richieste. Un'architettura comune usa un modello piccolo per i passi semplici e uno grande solo dove serve (vedi il pattern di routing nel modulo 7).

### Modelli di ragionamento

Alcuni modelli generano un ragionamento interno prima della risposta finale (spesso chiamato thinking o reasoning). È il Chain of Thought del modulo 2 integrato nell'addestramento. Migliorano su matematica, codice e problemi a più passi, ma consumano più token e aumentano la latenza. Molte API permettono di regolare quanto "budget di ragionamento" concedere.

### Multimodalità

Molti modelli accettano in input immagini, PDF, a volte audio. Questo cambia il modo di progettare un RAG: un manuale con schemi tecnici può essere letto anche nelle sue figure, non solo nel testo estratto.

### Finestra di contesto

La finestra varia da decine di migliaia a oltre un milione di token. Una finestra grande non elimina la necessità del retrieval: costa, rallenta e la qualità dell'attenzione tende a calare quando il contesto è pieno di materiale irrilevante.

### Tabella di scelta

| Esigenza | Orientamento |
|---|---|
| Classificazione ad alto volume | Modello piccolo, temperatura bassa, output strutturato |
| Risposte su documenti aziendali | Modello medio o grande + RAG |
| Analisi complessa a più passi | Modello di ragionamento, budget di token controllato |
| Dati che non possono uscire dall'azienda | Open-weight in infrastruttura propria, o accordi contrattuali specifici |
| Latenza molto bassa | Modello piccolo, streaming, prompt corti e cache |

### Esercizio

Per tre casi d'uso del tuo lavoro, compila la tabella: compito, volume giornaliero stimato, latenza accettabile, sensibilità dei dati, famiglia di modello scelta e motivazione. Riprenderai questa tabella nel modulo 3 per stimare i costi e nel modulo 8 per le scelte di produzione.

## Letture di approfondimento

- [Vaswani et al., "Attention Is All You Need" (2017)](https://arxiv.org/abs/1706.03762): l'architettura Transformer.
- [Ouyang et al., "Training language models to follow instructions with human feedback" (2022)](https://arxiv.org/abs/2203.02155): il ruolo dell'allineamento.
- [Jay Alammar, "The Illustrated Transformer"](https://jalammar.github.io/illustrated-transformer/): spiegazione visiva del Transformer, passo per passo.
- [Andrej Karpathy, "Intro to Large Language Models" (video, 2023)](https://www.youtube.com/watch?v=zjkBMFhNj_g): panoramica su addestramento, inferenza e limiti dei modelli.
- [Hugging Face, "LLM Course"](https://huggingface.co/learn/llm-course): corso gratuito su tokenizzazione, modelli e inferenza.
