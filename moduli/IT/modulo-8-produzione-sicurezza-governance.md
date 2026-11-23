# Modulo 8: Produzione, Sicurezza e Governance

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- descrivere un'architettura di riferimento per un'applicazione LLM in produzione;
- versionare e osservare prompt, modelli, indici e dataset come parti del software;
- controllare i costi con budget, routing e cache;
- costruire un threat model per un'applicazione LLM usando la OWASP Top 10 per applicazioni LLM;
- individuare gli obblighi principali su dati personali e trasparenza (GDPR, AI Act) e sapere quando coinvolgere i referenti legali e privacy;
- scegliere tra prompt engineering, RAG e fine-tuning con criteri espliciti;
- preparare una checklist di rilascio.

## 1. Dal prototipo alla produzione

Il prototipo del modulo 5 risponde bene in demo. In produzione cambiano le domande:

| Prototipo | Produzione |
|---|---|
| Funziona sulle mie dieci domande? | Funziona sulla distribuzione reale delle domande, che cambia nel tempo? |
| Quanto costa una chiamata? | Quanto costa al mese con mille utenti, e chi viene avvisato se il costo esplode? |
| Il modello risponde? | Cosa succede se il provider è lento, irraggiungibile o ritira la versione del modello? |
| La risposta è corretta? | Possiamo ricostruire perché una risposta di tre settimane fa era sbagliata? |
| Chi lo usa? | Chi può vedere quali documenti? Dove finiscono i dati personali nei prompt e nei log? |

Il resto del modulo affronta queste domande una per una.

## 2. Architettura di riferimento

Componenti tipici di un'applicazione LLM in produzione:

- **Interfaccia** (web, chat, integrazione in un prodotto esistente).
- **Backend applicativo**: autentica l'utente, applica permessi e limiti, orchestra il flusso. È l'harness del modulo 1.
- **Gateway verso i modelli**: unico punto di uscita verso i provider. Gestisce chiavi, retry, fallback tra modelli, limiti di spesa, logging uniforme. Può essere una libreria interna o un componente dedicato.
- **Servizio di retrieval**: vector database, ricerca ibrida, filtri per permessi.
- **Pipeline di ingestione**: separata dal servizio, eseguita quando i documenti cambiano; produce un indice versionato.
- **Osservabilità**: tracing di ogni richiesta, metriche di costo e latenza, feedback degli utenti.
- **Valutazione continua**: la suite del modulo 6, eseguita su ogni modifica e periodicamente su un campione del traffico reale.

### Fallback e degradazione

Definisci in anticipo cosa succede quando qualcosa non funziona:

- provider principale non disponibile → modello alternativo (di un altro provider o open-weight), con la stessa suite di valutazione già superata;
- retrieval lento o in errore → messaggio esplicito, non una risposta generata senza contesto;
- budget giornaliero esaurito → modalità ridotta o messaggio all'utente, e avviso al team.

### Esercizio

Disegna l'architettura del tuo progetto finale con i componenti sopra. Per ogni freccia tra componenti indica: cosa succede se il componente di destinazione non risponde?

## 3. LLMOps: versionare e osservare

### Cosa versionare

In un'applicazione LLM il comportamento dipende da più elementi del solo codice. Ognuno va versionato e registrato nei log di ogni richiesta:

| Elemento | Esempio di versione | Perché |
|---|---|---|
| Prompt | `risposta_manuale@v7` | Una modifica di una riga cambia il comportamento |
| Modello | identificativo completo con data o versione | Il provider può aggiornare o ritirare i modelli |
| Indice | `manuale_2026-11-10` | Documenti, chunking ed embedding cambiano il retrieval |
| Dataset di valutazione | `eval@v3` | Confrontare metriche su dataset diversi non ha senso |
| Parametri | temperatura, k, soglia | Fanno parte del comportamento quanto il prompt |

### Tracing

Una traccia raccoglie tutti i passi di una richiesta: input dell'utente, riscrittura della query, frammenti recuperati con punteggi, prompt finale, risposta, token, costo, latenza di ogni passo, versioni usate. Strumenti di osservabilità per LLM (ad esempio Langfuse, o le funzioni equivalenti delle piattaforme cloud) offrono questa vista; in alternativa un log strutturato in JSON con un identificativo di richiesta è già un ottimo inizio.

### Rilascio graduale

- **Valutazione offline**: la suite del modulo 6 deve passare prima del rilascio, come i test automatici.
- **Shadow**: la nuova versione elabora le richieste reali in parallelo, ma le sue risposte non vengono mostrate; si confrontano offline.
- **Canary**: la nuova versione serve una piccola percentuale degli utenti; si monitorano feedback, rifiuti, costi.
- **Rollback**: deve essere possibile tornare alla versione precedente di prompt, modello o indice in pochi minuti.

## 4. Gestione dei costi

I costi di un'applicazione LLM crescono con l'uso in modo meno prevedibile di quelli di un servizio tradizionale. Leve principali:

- **Budget e allarmi**: limite mensile per funzionalità, limite giornaliero per utente, allarme quando il consumo supera la media.
- **Routing per difficoltà**: un modello piccolo gestisce le richieste semplici, quello grande solo quelle che lo richiedono (pattern di routing, modulo 7).
- **Prompt caching**: prefissi stabili all'inizio del prompt, per beneficiare della cache del provider (modulo 3).
- **Cache delle risposte**: domande identiche ricevono la stessa risposta senza nuove chiamate. La cache "semantica" (domande simili) va usata con cautela: due domande simili possono richiedere risposte diverse, e le risposte non devono passare tra utenti con permessi diversi.
- **Batch** per i lavori non interattivi.
- **Costo per esito, non per token**: misura quanto costa un ticket risolto o un documento analizzato. È la metrica che interessa a chi decide.

### Esercizio

Per il progetto finale, stima il costo mensile con tre scenari di traffico (basso, atteso, picco). Definisci i limiti di spesa e il comportamento dell'applicazione quando vengono superati.

## 5. Sicurezza

### OWASP Top 10 per applicazioni LLM

L'OWASP mantiene un elenco dei rischi principali per le applicazioni basate su LLM. È un buon punto di partenza per un threat model. I rischi più rilevanti per i progetti di questo corso:

- **Prompt injection**: istruzioni nell'input dell'utente (diretta) o in contenuti letti dal sistema, come documenti e pagine web (indiretta), che alterano il comportamento. Non esiste una difesa completa a livello di prompt: servono permessi limitati e conferme (modulo 7).
- **Divulgazione di informazioni sensibili**: il modello restituisce dati che l'utente non dovrebbe vedere, perché erano nel contesto. Nel RAG, il filtro per permessi va applicato nel retrieval, non chiesto al modello.
- **Gestione insicura dell'output**: l'output del modello usato senza controlli in HTML (XSS), in query SQL, in comandi di shell. L'output del modello è input non fidato.
- **Eccesso di autonomia**: strumenti con più permessi del necessario, azioni irreversibili senza conferma.
- **Fuga del system prompt**: il system prompt può essere estratto. Non deve contenere segreti, chiavi o informazioni riservate.
- **Debolezze di vettori ed embedding**: documenti avvelenati inseriti nell'indice, accesso all'indice senza controllo dei permessi.
- **Disinformazione**: allucinazioni presentate come fatti, con utenti che si fidano troppo. Mitigazione: citazioni, rifiuto esplicito, valutazione (modulo 6).
- **Consumo illimitato**: richieste molto lunghe o loop agentici che esauriscono budget e capacità. Mitigazione: limiti di input, di passi e di spesa.

L'elenco completo e le mitigazioni dettagliate sono nella documentazione OWASP, che viene aggiornata periodicamente.

### Threat model in quattro domande

Per ogni funzionalità:

1. Quali dati entrano nel contesto del modello, e da quali fonti non fidate?
2. Quali azioni può innescare l'output del modello (strumenti, scritture, invii, rendering)?
3. Chi può vedere la risposta, e ha il diritto di vedere tutti i dati del contesto?
4. Cosa succede nel caso peggiore se il modello esegue un'istruzione malevola?

### Esercizio

Compila il threat model del progetto finale con le quattro domande. Per ognuno dei rischi OWASP elencati sopra indica: si applica? Con quale mitigazione? Come la verifichi con un test?

## 6. Privacy e conformità

Questa sezione orienta, non sostituisce il parere dei referenti legali e del responsabile della protezione dei dati della tua organizzazione. Coinvolgili all'inizio del progetto, non alla fine.

### Dati personali (GDPR)

- **Minimizzazione**: invia al modello solo i dati necessari. Valuta il mascheramento o la pseudonimizzazione di nomi, codici fiscali, indirizzi prima della chiamata.
- **Base giuridica e finalità**: il trattamento tramite LLM deve rientrare in una finalità e una base giuridica valide, come qualunque altro trattamento.
- **Fornitori**: il provider del modello tratta i dati per tuo conto. Servono accordi sul trattamento, verifica della conservazione, dell'uso per addestramento e della regione di elaborazione.
- **Log e tracce**: contengono prompt e risposte, quindi spesso dati personali. Definisci chi vi accede, per quanto tempo restano e come si cancellano.
- **Diritti degli interessati**: se un utente chiede la cancellazione, devi sapere dove sono i suoi dati: log, memoria a lungo termine dell'agente, indice vettoriale.

### AI Act europeo

Il regolamento europeo sull'intelligenza artificiale (Regolamento UE 2024/1689) segue un approccio basato sul rischio e si applica in modo graduale. In sintesi:

- alcune pratiche sono vietate;
- i sistemi ad alto rischio (ad esempio in ambiti come selezione del personale, accesso al credito, servizi essenziali) hanno obblighi stringenti di gestione del rischio, qualità dei dati, documentazione, supervisione umana;
- molti sistemi hanno obblighi di trasparenza: ad esempio, le persone devono sapere che stanno interagendo con un sistema di AI, e alcuni contenuti generati devono essere riconoscibili come tali;
- i fornitori di modelli di uso generale hanno obblighi propri, distinti da quelli di chi li integra in un'applicazione.

Domanda pratica per ogni progetto: in quale categoria di rischio ricade il caso d'uso? La risposta cambia in modo sostanziale i requisiti.

### Esercizio

Per il progetto finale, compila una scheda di una pagina: dati personali trattati, provider e regione, durata di conservazione dei log, categoria di rischio ipotizzata secondo l'AI Act, misure di trasparenza verso gli utenti. Indica le domande da porre ai referenti legali e privacy.

## 7. Prompt, RAG o fine-tuning?

Una delle decisioni più frequenti, e più spesso sbagliate. Ordine consigliato: prima il prompt, poi il RAG, il fine-tuning solo se i primi due non bastano e i dati lo giustificano.

| Esigenza | Approccio adatto |
|---|---|
| Il modello deve seguire un formato o uno stile | Prompt con esempi; fine-tuning solo per volumi molto alti o stili molto specifici |
| Il modello deve conoscere documenti aziendali che cambiano | RAG |
| Servono citazioni e verificabilità | RAG |
| Il modello deve imparare un compito ristretto ad alto volume, con costo e latenza minimi | Fine-tuning di un modello piccolo, con un buon dataset etichettato |
| Il modello deve "imparare fatti" | Quasi mai fine-tuning: è costoso da aggiornare e non riduce le allucinazioni. Usa il RAG |

Il fine-tuning richiede: un dataset di centinaia o migliaia di esempi di qualità, una suite di valutazione per confrontare prima e dopo, un processo per ripeterlo quando il modello di base cambia. Se manca uno di questi elementi, non è il momento.

## 8. Checklist di rilascio

Prima di andare in produzione, verifica:

**Qualità**
- [ ] La suite di valutazione passa con le soglie concordate.
- [ ] I casi negativi e avversari sono inclusi.
- [ ] Esiste un meccanismo di feedback degli utenti.

**Affidabilità**
- [ ] Retry con backoff, timeout e fallback sono configurati.
- [ ] Il comportamento in caso di errore del provider o del retrieval è definito e testato.
- [ ] È possibile fare rollback di prompt, modello e indice.

**Costi**
- [ ] Limiti di spesa sul provider e per utente.
- [ ] Allarmi sul consumo anomalo.
- [ ] Stima del costo mensile approvata.

**Sicurezza**
- [ ] Threat model compilato e revisionato.
- [ ] Le chiavi sono in un secret manager.
- [ ] L'output del modello è trattato come input non fidato.
- [ ] Le azioni irreversibili richiedono conferma.
- [ ] I permessi sui documenti sono applicati nel retrieval.

**Dati e conformità**
- [ ] Referenti legali e privacy coinvolti, scheda compilata.
- [ ] Conservazione e accesso ai log definiti.
- [ ] Gli utenti sono informati che interagiscono con un sistema di AI.

**Osservabilità**
- [ ] Ogni richiesta ha una traccia con versioni, token, costo e latenza.
- [ ] Un cruscotto mostra qualità, costi e errori nel tempo.

## Consegna del modulo

Per il progetto finale:

- schema dell'architettura con fallback;
- tabella delle versioni (prompt, modello, indice, dataset) e formato del log di traccia;
- stima dei costi in tre scenari, con limiti e allarmi;
- threat model con mitigazioni e test;
- scheda privacy e conformità;
- checklist di rilascio compilata, con le voci non ancora soddisfatte e il piano per completarle.

## Letture di approfondimento

- [OWASP, "Top 10 for Large Language Model Applications"](https://genai.owasp.org/llm-top-10/): i rischi più comuni delle applicazioni LLM (versione più recente).
- [Regolamento (UE) 2024/1689 sull'intelligenza artificiale (AI Act)](https://eur-lex.europa.eu/eli/reg/2024/1689/oj): testo ufficiale su EUR-Lex.
- [Regolamento (UE) 2016/679 (GDPR)](https://eur-lex.europa.eu/eli/reg/2016/679/oj): testo ufficiale su EUR-Lex; vedere anche le linee guida delle autorità di protezione dei dati sull'uso dell'AI.
- [NIST, "AI Risk Management Framework"](https://www.nist.gov/itl/ai-risk-management-framework): framework per identificare e gestire i rischi dei sistemi AI.
- Documentazione dei provider scelti: politiche di conservazione e uso dei dati.
