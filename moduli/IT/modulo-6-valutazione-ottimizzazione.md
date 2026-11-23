# Modulo 6: Valutazione e Ottimizzazione

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- costruire un dataset di valutazione (golden dataset) rappresentativo del caso d'uso;
- distinguere le metriche di retrieval da quelle di generazione e capire quale problema misura ciascuna;
- usare un framework come RAGAS per misurare fedeltà ai documenti, pertinenza e precisione del contesto;
- valutare in modo affidabile con LLM-as-a-judge, riconoscendone i limiti;
- rilevare le allucinazioni e ottimizzare il system prompt in modo misurabile.

## Premessa: senza misura non c'è miglioramento

Un prompt o un sistema RAG che "sembra funzionare" su tre domande è un esperimento, non un prodotto. Le risposte di un LLM variano, i casi difficili sono rari e il tuo intuito tende a testare i casi che già conosci. La valutazione trasforma l'impressione in un numero che puoi confrontare prima e dopo ogni modifica.

Regola pratica: nessuna modifica al prompt, al chunking o al modello va considerata un miglioramento finché non la misuri su un dataset fisso.

## 1. Il dataset di valutazione

Il golden dataset è un insieme di casi con questa struttura minima:

```json
{
  "id": "E18-001",
  "domanda": "Cosa significa l'errore E18?",
  "risposta_attesa": "Problema di scarico dell'acqua; pulire il filtro.",
  "fonte_attesa": {"documento": "manuale_lavatrice.pdf", "pagina": 12},
  "tipo": "diretta",
  "difficolta": "facile"
}
```

Come costruirlo:

- **Casi reali**: domande effettive, anonimizzate, dai log o dal servizio clienti. Sono i più preziosi.
- **Casi sintetici**: generati a partire dai frammenti del documento, ad esempio chiedendo a un modello di scrivere una domanda a cui il frammento risponde. Utili per coprire il corpus, ma da rivedere a mano: possono essere banali o ambigui.
- **Casi negativi**: domande che il documento non copre. Sono obbligatori: senza di essi non misuri il comportamento più pericoloso, l'allucinazione.
- **Casi avversari**: sinonimi, errori di battitura, domande ambigue, tentativi di prompt injection.

Dimensione: per un prototipo bastano 30-50 casi, bilanciati tra tipi e difficoltà. Per la produzione servono centinaia di casi, aggiornati man mano che emergono errori reali.

Regola fondamentale: non modificare il dataset per farlo passare. Se un caso fallisce, la domanda è se il sistema è sbagliato o se il caso è sbagliato. Decidi caso per caso e documenta la decisione.

### Esercizio

Costruisci un dataset di 30 casi per il tuo progetto: 15 diretti, 5 con sinonimi, 5 negativi, 5 avversari. Per ogni caso scrivi la fonte attesa, non solo la risposta. Fai revisionare il dataset da un collega prima di usarlo.

## 2. Separare retrieval e generazione

Un errore frequente è valutare solo la risposta finale. Se la risposta è sbagliata, non sai se la colpa è del retrieval (il frammento giusto non è stato trovato) o della generazione (il frammento c'era ma il modello non lo ha usato bene). Servono metriche separate.

### Metriche di retrieval

- **Recall@k**: il frammento corretto è tra i primi k risultati? È la metrica più importante: se non viene recuperato, nessun prompt lo salverà.
- **Precision@k**: quanti dei k frammenti sono davvero pertinenti? Misura il rumore nel contesto.
- **MRR (Mean Reciprocal Rank)**: quanto in alto compare il frammento giusto, in media. Utile per valutare il reranker.

### Metriche di generazione

- **Fedeltà (faithfulness)**: ogni affermazione della risposta è supportata dai frammenti forniti?
- **Pertinenza della risposta (answer relevance)**: la risposta risponde davvero alla domanda?
- **Correttezza**: la risposta coincide con la risposta attesa nel dataset?
- **Rifiuto appropriato**: sui casi negativi, il sistema dichiara di non sapere?

## 3. RAGAS

RAGAS è un framework che calcola metriche per sistemi RAG, in parte con l'aiuto di un LLM. Le metriche più usate sono:

- **Faithfulness**: la risposta è sostenuta dal contesto recuperato?
- **Answer relevancy**: la risposta è pertinente alla domanda?
- **Context precision**: i frammenti recuperati sono pertinenti e ben ordinati?
- **Context recall**: il contesto recuperato contiene le informazioni necessarie per la risposta attesa?

Come usarlo:

1. esegui il sistema su tutti i casi del dataset, salvando domanda, frammenti recuperati e risposta;
2. passa questi dati a RAGAS insieme alla risposta attesa dove necessaria;
3. confronta le metriche tra due versioni del sistema, non cercare un valore assoluto.

Attenzione: le metriche di RAGAS usano un LLM come giudice, quindi hanno una propria variabilità e un proprio costo. Usa lo stesso modello giudice in tutti i confronti e, se possibile, verifica a campione i punteggi a mano.

## 4. LLM-as-a-judge

Per valutare risposte aperte, spesso si usa un modello come giudice. Il giudice riceve domanda, risposta, criteri e, se disponibili, la risposta attesa, e restituisce un punteggio o un'etichetta.

Buone pratiche:

- **Criteri espliciti**: "La risposta è corretta se contiene l'informazione X. Non è penalizzante un diverso livello di dettaglio." Senza criteri il giudice applica i propri.
- **Output strutturato**: punteggio e motivazione in JSON, vedi modulo 2.
- **Confronto a coppie**: spesso è più affidabile chiedere quale di due risposte è migliore piuttosto che assegnare un voto assoluto.
- **Bias noti**: i giudici tendono a preferire risposte più lunghe, più formali e quelle scritte dallo stesso modello. Controllali confrontando con valutazione umana.
- **Calibrazione**: valida il giudice su 50 casi giudicati da una persona. Se l'accordo è basso, il giudice non è affidabile per quel compito.

### Esempio di prompt giudice per la fedeltà

```text
Sei un valutatore. Devi stabilire se la RISPOSTA è interamente supportata dai FRAMMENTI.

Procedura:
1. Elenca le affermazioni fattuali contenute nella RISPOSTA.
2. Per ognuna, indica se è supportata da almeno un frammento (sì/no) e quale.
3. Un'affermazione è supportata solo se il frammento la dice esplicitamente o la implica in modo diretto.
   Le conoscenze generali non contano come supporto.

Rispondi solo con JSON:
{"affermazioni": [{"testo": "...", "supportata": true, "frammento": 2}],
 "punteggio": <numero di affermazioni supportate / totale>,
 "verdetto": "fedele" | "parzialmente fedele" | "non fedele"}

FRAMMENTI:
{frammenti}

RISPOSTA:
{risposta}
```

Nota la struttura: il giudice scompone prima la risposta in affermazioni, poi verifica ciascuna. È più affidabile di una domanda generica come "la risposta è fedele? Voto da 1 a 10", e la motivazione per affermazione rende il punteggio verificabile da una persona.

### Esercizio

Applica il prompt giudice a 20 risposte del tuo sistema. Valuta tu stesso le stesse 20 risposte con la scheda della sezione 5. Calcola in quanti casi il verdetto del giudice coincide con il tuo. Analizza i casi di disaccordo: è il giudice troppo severo, troppo indulgente, o il criterio è ambiguo?

## 5. Valutazione umana

La valutazione umana resta il riferimento. Non deve essere eseguita su tutto, ma su un campione rappresentativo e sui casi in cui il giudice automatico è incerto.

Una scheda di valutazione semplice:

| Criterio | Scala | Domanda guida |
|---|---|---|
| Correttezza | 0 / 1 / 2 | La risposta contiene l'informazione giusta? |
| Fedeltà | 0 / 1 / 2 | Ogni affermazione è nel manuale? |
| Utilità | 0 / 1 / 2 | Il cliente sa che cosa fare dopo? |
| Sicurezza | sì / no | Il sistema ha proposto operazioni pericolose? |

Misura anche l'accordo tra due valutatori umani: se due persone non concordano, il criterio è ambiguo e va riscritto.

## 6. Allucinazioni

Un'allucinazione è un'affermazione non supportata dalle fonti o falsa, presentata con sicurezza. Nel RAG si distinguono due forme:

- **Allucinazione di contesto**: la risposta contraddice o aggiunge informazioni ai frammenti recuperati.
- **Allucinazione da assenza**: il contesto non contiene la risposta, ma il modello risponde comunque.

Tecniche di rilevamento:

- verifica di fedeltà frase per frase, con il giudice o con RAGAS;
- controllo delle citazioni: la pagina citata contiene davvero l'informazione?
- test negativi sistematici: il sistema deve rifiutare le domande fuori dal corpus;
- confronto con una risposta di riferimento per le informazioni numeriche (date, quantità, codici), che sono le più rischiose.

Mitigazioni:

- migliorare il retrieval, perché un contesto migliore riduce le allucinazioni da assenza;
- soglia di pertinenza e risposta di rifiuto;
- prompt che richiede citazioni e vieta affermazioni non supportate;
- temperatura bassa per compiti di estrazione e risposta documentale;
- verifica deterministica dove possibile (ad esempio controllare che un codice d'errore citato esista nel manuale).

## 7. Prompt tuning e system prompt

Il system prompt è la parte del prompt più persistente e quindi quella che più conviene ottimizzare. L'obiettivo indicato nel corso è limitare la creatività del modello e farlo attenere ai fatti, ma il tuning va fatto per misura, non per tentativi a sensazione.

Processo consigliato:

1. definisci la metrica principale (ad esempio fedeltà media e tasso di rifiuto corretto sui casi negativi);
2. stabilisci una baseline con il prompt attuale sul dataset fisso;
3. modifica un solo elemento alla volta (un vincolo, un esempio, il formato);
4. riesegui e confronta, controllando anche le metriche secondarie per evitare regressioni;
5. versiona il prompt con un identificativo e registra i risultati.

Leve tipiche:

- **Vincoli espliciti** sulle fonti: "usa solo i frammenti", "non usare conoscenze generali".
- **Comportamento di rifiuto**: formula esatta da usare quando l'informazione manca.
- **Temperatura**: più bassa per la fedeltà, con verifica che la risposta non diventi meccanica o incompleta.
- **Lunghezza e formato**: risposte più brevi riducono le affermazioni non verificabili, ma possono perdere utilità.
- **Esempi few-shot**: utili per il formato delle citazioni e per i casi di rifiuto.

Trappola: un vincolo molto restrittivo può aumentare i rifiuti su domande legittime. Monitora sempre sia la fedeltà sia il tasso di risposte utili.

### Esercizio di tuning

Parti dal prompt del modulo 5. Misura su un dataset di 30 casi: fedeltà, tasso di rifiuto corretto sui negativi, tasso di risposta su domande legittime. Poi prova tre varianti e confrontale. Scrivi una nota di una pagina su quale modifica ha prodotto il miglioramento più grande e quale effetto collaterale ha avuto.

## 8. Qualità, costo e latenza insieme

Una variante che migliora la fedeltà del 2% ma raddoppia il costo o la latenza non è necessariamente migliore. Ogni esecuzione della suite di valutazione dovrebbe riportare, accanto alle metriche di qualità:

- token medi di input e output per richiesta;
- costo medio per richiesta e per mille richieste;
- latenza p50 e p95, e il tempo al primo token se l'interfaccia usa lo streaming.

Esempio di report di confronto:

| Variante | Recall@3 | Fedeltà | Rifiuto corretto | Costo / 1000 rich. | Latenza p95 |
|---|---|---|---|---|---|
| Baseline | 0,72 | 0,81 | 0,60 | 1,00 (riferimento) | 2,1 s |
| + ricerca ibrida | 0,84 | 0,85 | 0,60 | 1,02 | 2,3 s |
| + reranking | 0,88 | 0,88 | 0,65 | 1,10 | 2,9 s |

I valori sono illustrativi. La decisione dipende dal caso d'uso: per un assistente interattivo 2,9 secondi possono essere accettabili, per un suggerimento in tempo reale durante la digitazione no.

## 9. Monitoraggio in produzione

La valutazione non finisce con il rilascio. In produzione:

- registra ogni domanda, i frammenti usati e la risposta, rispettando le regole sulla privacy dei dati;
- raccogli feedback esplicito (utile/non utile) e implicito (il cliente riformula la domanda, apre un ticket);
- campiona periodicamente le risposte per la valutazione umana;
- trasforma ogni errore reale in un nuovo caso del dataset;
- esegui la suite di valutazione ad ogni modifica, come un test di regressione.

## Caso d'uso da sviluppare

Prendi il prototipo del modulo 5 e costruisci:

- un dataset di 30 casi con fonte attesa;
- uno script che esegue il sistema su tutto il dataset e produce una tabella con recall@3, fedeltà, tasso di rifiuto corretto;
- tre varianti di prompt o di parametri di chunking, confrontate sulle stesse metriche, con costo e latenza (sezione 8);
- una breve analisi di tre fallimenti, classificati come problema di retrieval o di generazione.

## Letture di approfondimento

- [Es et al., "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023)](https://arxiv.org/abs/2309.15217): il paper di riferimento sulle metriche RAGAS.
- [Documentazione di RAGAS](https://docs.ragas.io): implementazione delle metriche di retrieval e generazione.
- [Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023)](https://arxiv.org/abs/2306.05685): affidabilità e bias dei giudici basati su LLM.
- [Documentazione di Promptfoo](https://www.promptfoo.dev/docs/intro/): test automatici e regressione dei prompt.
