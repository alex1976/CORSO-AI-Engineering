# Modulo 4: Il Cuore del Progetto - Creare un Sistema RAG

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- spiegare perché un LLM da solo non può rispondere su documenti privati e quali alternative esistono;
- progettare la pipeline di ingestione: caricamento, pulizia, chunking, embedding, indicizzazione;
- scegliere una strategia di chunking motivata dal tipo di documento;
- implementare il ciclo di retrieval: domanda, vettore, ricerca, costruzione del contesto, generazione;
- riconoscere i punti in cui un RAG fallisce e sapere dove intervenire.

## 1. Perché RAG

Un LLM conosce ciò che ha visto durante l'addestramento, fino a una certa data, e non conosce i documenti interni della tua azienda. Ci sono tre strade per colmare questo divario:

1. **Fine-tuning**: riaddestrare il modello sui documenti. Costoso, lento da aggiornare, difficile da verificare, e non elimina le allucinazioni.
2. **Contesto lungo**: incollare tutto il documento nel prompt. Funziona per documenti piccoli, ma costa molto, rallenta e peggiora spesso la precisione quando il contesto è pieno di informazioni irrilevanti.
3. **RAG (Retrieval-Augmented Generation)**: cercare nei documenti solo i frammenti rilevanti per la domanda e passarli al modello come contesto.

Il RAG è la scelta più comune quando i dati cambiano spesso, quando serve citare le fonti e quando il volume è troppo grande per entrare nel contesto. Ha però una qualità di risposta limitata dalla qualità del retrieval: se il frammento giusto non viene trovato, il modello risponderà male anche se è bravissimo.

## 2. La pipeline in due fasi

Un sistema RAG ha due percorsi.

**Offline (ingestione)**, eseguito quando i documenti cambiano:
caricamento → pulizia → chunking → embedding → indicizzazione nel vector database.

**Online (interrogazione)**, eseguito a ogni domanda:
domanda → embedding della domanda → ricerca dei chunk più simili → costruzione del prompt → generazione → risposta con fonti.

Molti problemi nascono dal primo percorso e si manifestano nel secondo. Quando una risposta è sbagliata, la prima domanda da porsi è: il chunk giusto esisteva nell'indice?

## 3. Ingestione

### Caricamento e pulizia

I documenti reali sono sporchi: PDF con intestazioni e piè di pagina ripetuti, tabelle spezzate, scansioni, testo su più colonne, numeri di pagina, sommari. Prima del chunking va deciso cosa tenere.

Esercizio: prendi un PDF reale (un manuale, un contratto, una policy aziendale) ed estrai il testo con due strumenti diversi. Confronta il risultato su tabelle e intestazioni. Annota che cosa andrebbe rimosso e che cosa andrebbe conservato come metadato.

### Metadati

Ogni chunk dovrebbe portare con sé informazioni che servono al filtraggio e alla citazione:

- documento di origine e versione;
- pagina o sezione;
- data di pubblicazione o di aggiornamento;
- tipo di documento o reparto;
- identificatore del chunk.

Senza metadati non puoi filtrare (ad esempio: "solo le policy in vigore") né mostrare all'utente da dove viene la risposta.

### Chunking

Il chunking divide il testo in blocchi. La scelta della dimensione e della strategia influisce più di quasi ogni altro parametro.

Strategie comuni:

- **Dimensione fissa con overlap**: blocchi di lunghezza fissa (ad esempio 1000 caratteri) con una sovrapposizione (ad esempio 200 caratteri) per non spezzare un concetto tra due blocchi senza un punto di contatto. È la strategia di partenza indicata nel corso.
- **Ricorsiva per separatori**: prova a spezzare su paragrafi, poi su frasi, poi su parole, mantenendo unità di senso quando possibile.
- **Per struttura**: usa titoli, sezioni e articoli come confini naturali. Ideale per documenti legali o manuali con numerazione.
- **Semantica**: spezza dove il tema cambia, misurato con gli embedding. Più costosa, non sempre migliore.

Trade-off:

- chunk piccoli: retrieval più preciso, ma il singolo frammento può non contenere il contesto necessario;
- chunk grandi: più contesto per frammento, ma più rumore e più token nel prompt;
- overlap alto: meno perdita di informazioni al confine, ma più duplicazione e più costo di indicizzazione.

Il valore 1000/200 è un punto di partenza, non una regola. Va validato sul tuo corpus.

### Esercizio di chunking

Prendi un documento di almeno dieci pagine con sezioni numerate. Crea tre versioni dell'indice: dimensione fissa 500, 1000 e 2000 caratteri, con overlap 200. Per dieci domande che conosci già la risposta, verifica se il chunk corretto compare tra i primi tre risultati. Annota quale configurazione funziona meglio e perché.

## 4. Embedding e vector database

### Embedding

Un embedding è un vettore di numeri (da alcune centinaia a qualche migliaio di dimensioni) che rappresenta il significato di un testo. Testi con significato simile producono vettori vicini, misurati di solito con la similarità del coseno.

Due frasi come "Come si richiede il rimborso?" e "Procedura per ottenere il reso del denaro" hanno poche parole in comune ma vettori vicini: è questo che rende la ricerca semantica più utile della ricerca per parole chiave.

Attenzione: il modello di embedding è una scelta separata dal modello di generazione. Va scelto per la lingua dei documenti (italiano, multilingua) e va usato in modo identico in ingestione e in interrogazione. Cambiare modello di embedding richiede di reindicizzare tutto.

### Vector database

Un vector database memorizza i vettori con i loro metadati e permette di cercare i vettori più vicini a un vettore dato. Strumenti citati nel corso:

- **ChromaDB**: semplice, adatto a prototipi e ambienti locali;
- **Qdrant**: prestazioni e filtri sui metadati, adatto a produzione;
- **Pinecone**: servizio gestito, senza infrastruttura da amministrare.

Per il corso è sufficiente un database locale. La scelta conta meno dell'attenzione a metadati, filtri e reindicizzazione.

### Ricerca per somiglianza e ibrida

La ricerca vettoriale pura non sempre trova codici, sigle o nomi propri esatti (ad esempio "errore E18" o "articolo 17-bis"). Per questo nei sistemi reali si combina spesso la ricerca vettoriale con la ricerca per parole chiave (BM25 o simili), fondendo i risultati. È una delle prime migliorie da considerare.

## 5. Ciclo di recupero

Il percorso online, passo per passo:

1. L'utente pone una domanda.
2. La domanda viene trasformata in un vettore con lo stesso modello di embedding.
3. Si cercano nel vector database i k frammenti più simili (tipicamente 3-5, eventualmente filtrati per metadati).
4. Opzionale: un reranker riordina i frammenti con un modello più accurato, scartando quelli poco pertinenti.
5. Si costruisce il prompt: istruzione, frammenti numerati con la loro fonte, domanda.
6. Il modello genera la risposta, citando i frammenti usati.
7. Se nessun frammento supera una soglia di pertinenza, il sistema risponde che non ha l'informazione invece di inventarla.

Esempio di prompt di generazione:

```text
Rispondi alla domanda usando solo i frammenti forniti.
Se la risposta non è presente nei frammenti, scrivi: "Non ho trovato questa informazione nei documenti disponibili."
Cita la fonte di ogni affermazione con [numero_frammento].

Frammenti:
[1] (manuale_lavatrice.pdf, p. 12) ...
[2] (manuale_lavatrice.pdf, p. 13) ...

Domanda: {{domanda}}
```

### Il valore della soglia

Senza soglia, il retrieval restituisce sempre k frammenti, anche quando nessuno è pertinente. Il modello, a quel punto, cerca di rispondere comunque. Una soglia di similarità, o un controllo di pertinenza, è spesso la differenza tra un sistema affidabile e uno che allucina con sicurezza.

## 6. Tecniche avanzate di retrieval

Il RAG di base (embedding della domanda, top-k, generazione) è un buon punto di partenza. Quando la valutazione del modulo 6 mostra un recall insufficiente, si introducono tecniche più sofisticate. Ognuna aggiunge costo o latenza: vanno introdotte una alla volta e misurate.

### Ricerca ibrida

Combina la ricerca vettoriale (significato) con quella per parole chiave, ad esempio BM25 (termini esatti). I due elenchi di risultati vengono fusi, spesso con la Reciprocal Rank Fusion: ogni documento riceve un punteggio in base alla sua posizione in ciascuna lista.

Quando serve: codici di errore, nomi di prodotto, articoli di legge, sigle. "E18" ha poco significato semantico, ma è una corrispondenza esatta preziosa.

### Riscrittura della query

La domanda dell'utente è spesso incompleta o colloquiale. Prima del retrieval, un modello la riformula.

- **Riscrittura con la storia**: "E se il filtro è già pulito?" diventa "Errore E18 persistente dopo la pulizia del filtro di scarico".
- **Multi-query**: dalla domanda si generano più varianti e si uniscono i risultati.
- **HyDE (Hypothetical Document Embeddings)**: il modello scrive una risposta ipotetica e si cerca con l'embedding di quella, che spesso somiglia ai documenti più della domanda.

### Reranking

Il retrieval vettoriale è veloce ma approssimativo. Un reranker (di solito un modello cross-encoder) valuta ogni coppia domanda-frammento con più attenzione e riordina i risultati. Schema tipico: recupera 20-50 candidati, riordina, passa al modello i migliori 3-5.

### Chunking gerarchico (small-to-big)

Si indicizzano frammenti piccoli, precisi nel retrieval, ma al modello si passa il blocco più grande che li contiene (il paragrafo o la sezione intera). Unisce la precisione dei chunk piccoli con il contesto dei chunk grandi.

### Arricchimento contestuale dei chunk

Un frammento isolato come "Pulire il filtro ogni tre mesi" non dice di quale prodotto o sezione parla. Prima dell'embedding, si antepone a ogni chunk una breve descrizione del contesto (documento, capitolo, argomento), generata una sola volta in ingestione. Migliora il retrieval quando i documenti hanno molti frammenti simili tra loro.

### Filtri sui metadati

Restringere la ricerca prima di calcolare la similarità: solo il modello di lavatrice indicato, solo le policy in vigore, solo i documenti del reparto dell'utente. È anche un meccanismo di sicurezza: un utente non deve recuperare documenti a cui non ha accesso.

### Quale tecnica, per quale sintomo

| Sintomo misurato | Tecnica da provare per prima |
|---|---|
| Codici e sigle non trovati | Ricerca ibrida |
| Domande colloquiali o di follow-up falliscono | Riscrittura della query |
| Il frammento giusto è recuperato ma in posizione bassa | Reranking |
| Il frammento giusto è trovato ma manca il contesto | Chunking gerarchico |
| Molti frammenti quasi identici tra documenti diversi | Arricchimento contestuale, filtri sui metadati |

### Esercizio

Sul tuo prototipo, misura recall@3 con il dataset del modulo 6. Aggiungi la ricerca ibrida, misura di nuovo; poi aggiungi il reranking e misura ancora. Riporta i tre valori e il tempo medio per richiesta in ciascuna configurazione.

## 7. Dove fallisce un RAG

Quando la risposta è sbagliata, classifica il problema prima di correggerlo:

| Sintomo | Probabile causa | Dove intervenire |
|---|---|---|
| Il frammento giusto non è tra i risultati | Chunking, embedding, query troppo diversa dal testo | Chunking, ricerca ibrida, riformulazione della query |
| Il frammento giusto è trovato ma in posizione bassa | Ranking debole | Reranker, numero di risultati |
| Il frammento giusto è nel contesto ma la risposta è errata | Il modello ignora il contesto o lo interpreta male | Prompt, istruzioni, formato dei frammenti |
| Risposta corretta ma senza fonte | Prompt senza requisito di citazione | Prompt di generazione |
| Domanda su più documenti, risposta parziale | Retrieval con k troppo basso o senza aggregazione | Query multiple, k più alto, sintesi |
| Informazione vecchia | Indice non aggiornato, metadati di versione assenti | Pipeline di aggiornamento, filtri per data |

Il modulo 6 mostra come misurare queste cause separatamente.

## 8. Casi d'uso da sviluppare

Scegli uno dei seguenti e costruiscilo come piccolo prototipo con interfaccia Streamlit.

- **Assistente per le policy HR**: ferie, rimborsi spese, smart working. Documenti con versioni diverse; la risposta deve riportare la versione in vigore alla data della domanda.
- **Chatbot sulla documentazione di un'API open source**: domande su endpoint e parametri. Verifica la capacità di gestire sigle e nomi di funzione (ricerca ibrida).
- **Assistente per verbali di riunione**: domande come "Quali decisioni abbiamo preso su X?" su un archivio di verbali con data e partecipanti come metadati.
- **Consulente per un regolamento comunale o un contratto**: risposte che citano articolo e comma. Qui la struttura del documento va usata come criterio di chunking.

Per ognuno, consegna: schema dei metadati, strategia di chunking motivata, dieci domande di test con la risposta attesa e la fonte attesa, e un'analisi dei fallimenti usando la tabella della sezione 7.

## Letture di approfondimento

- [Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020)](https://arxiv.org/abs/2005.11401): l'articolo che ha definito il RAG.
- [Gao et al., "Retrieval-Augmented Generation for Large Language Models: A Survey" (2023)](https://arxiv.org/abs/2312.10997): rassegna delle tecniche RAG, dal base alle varianti avanzate.
- [Anthropic, "Introducing Contextual Retrieval" (2024)](https://www.anthropic.com/news/contextual-retrieval): arricchimento contestuale dei chunk, come discusso nel modulo.
- [Documentazione di ChromaDB](https://docs.trychroma.com): sezioni su collezioni, metadati e filtri.
- [Documentazione di Qdrant](https://qdrant.tech/documentation/): sezioni su ricerca ibrida e filtri sui payload.
