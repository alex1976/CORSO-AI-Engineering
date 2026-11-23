# Modulo 7: Pattern di Sviluppo Agentico

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- distinguere un agente da un workflow e scegliere il livello di autonomia adeguato al problema;
- implementare il loop ReAct con tool calling e limiti di esecuzione;
- scegliere tra i pattern di composizione (chaining, routing, parallelizzazione, orchestrator-workers, evaluator-optimizer);
- progettare i meccanismi di controllo: conferme umane, permessi, guardrail, osservabilità;
- gestire memoria e contesto di un agente per compiti lunghi;
- valutare quando un sistema multi-agente è giustificato e quando è complessità inutile.

## 1. Agente o workflow?

Il punto di partenza è una distinzione che evita molti errori di progettazione:

- **Workflow**: il percorso dei passaggi è definito dal codice. L'LLM esegue singoli passi (classifica, estrae, riassume). Il flusso è prevedibile, testabile, economico.
- **Agente**: l'LLM decide dinamicamente quali strumenti usare, in quale ordine, quando fermarsi. Il flusso emerge dalla conversazione con gli strumenti.

Un agente non è "più intelligente" di un workflow. È più flessibile e meno prevedibile. Quando il percorso è noto, il workflow vince quasi sempre.

Esempio. Per l'elaborazione di una fattura ricevuta via email:

- *Workflow*: estrai il PDF → estrai i campi → valida l'IVA → salva nel gestionale → notifica. Ogni passo è fisso.
- *Agente*: ricevi la richiesta "controlla questa fattura e segnala se qualcosa non torna". Il percorso dipende da ciò che trovi: se il fornitore non è in anagrafica cerchi un ordine, se l'importo non corrisponde cerchi il contratto, eccetera.

Esercizio: prendi cinque attività del tuo lavoro e classificale come workflow, agente o "non serve un LLM". Per ogni agente, spiega quale decisione il modello prende che il codice non può prendere in anticipo.

## 2. Il loop base: ReAct e tool use

ReAct (Reason + Act) è il ciclo su cui si basa la maggior parte degli agenti. Il modello alterna tre fasi:

1. **Ragionamento**: decide il prossimo passo.
2. **Azione**: chiama uno strumento con argomenti strutturati.
3. **Osservazione**: riceve il risultato e aggiorna la sua comprensione.

Il ciclo termina quando il modello produce una risposta finale invece di una chiamata a uno strumento.

### Implementazione minima

```python
def esegui_agente(llm, strumenti: dict, richiesta: str, max_passi: int = 8) -> str:
    messaggi = [{"role": "user", "content": richiesta}]

    for _ in range(max_passi):
        risposta = llm.genera(
            system=SYSTEM_PROMPT,
            messaggi=messaggi,
            strumenti=descrizioni(strumenti),   # nome, scopo, schema dei parametri
        )

        if not risposta.chiamate_strumenti:
            return risposta.testo               # risposta finale

        for chiamata in risposta.chiamate_strumenti:
            funzione = strumenti[chiamata.nome]
            try:
                esito = funzione(**chiamata.argomenti)
            except Exception as errore:
                esito = {"errore": str(errore)}  # il modello può correggersi
            messaggi.append({"role": "tool", "nome": chiamata.nome, "contenuto": esito})

    raise RuntimeError("Limite di passi raggiunto senza risposta finale")
```

Elementi da notare:

- il **limite di passi** è obbligatorio: senza di esso un loop può non terminare e consumare budget;
- gli **errori degli strumenti** tornano al modello come osservazione, non fanno crollare il loop: spesso il modello corregge gli argomenti;
- la **descrizione degli strumenti** è parte del prompt. Un nome vago o uno schema ambiguo producono chiamate sbagliate.

### Progettare uno strumento

Uno strumento ben progettato:

- ha un nome verbo-oggetto chiaro (`cerca_ordine`, non `gestisci`);
- descrive quando usarlo e quando non usarlo;
- ha parametri tipizzati con valori ammessi espliciti;
- restituisce risultati compatti e strutturati, non interi database;
- non ha effetti collaterali nascosti: un'azione irreversibile deve essere dichiarata come tale.

Esercizio: progetta tre strumenti per un assistente di helpdesk (ad esempio `cerca_cliente`, `leggi_stato_ticket`, `apri_ticket`). Scrivi per ciascuno la descrizione per il modello e lo schema dei parametri. Poi chiediti: quale di questi può fare danni se il modello lo usa male?

## 3. Pattern di composizione

Non tutti i problemi richiedono un agente. I pattern seguenti sono mattoni da combinare.

### Prompt chaining

Il compito è spezzato in passi sequenziali, ciascuno con un prompt dedicato. L'output di un passo è l'input del successivo.

Caso d'uso: generare una risposta a una recensione negativa in tre passi: (1) classifica il problema, (2) verifica nelle policy quale rimedio applicare, (3) scrive la risposta nel tono del brand. Ogni passo è testabile separatamente.

### Routing

Un classificatore smista la richiesta verso un percorso specializzato: un prompt, un modello più piccolo o più economico per i casi semplici, un flusso dedicato per i casi sensibili.

Caso d'uso: un helpdesk che manda le domande sulle fatture al RAG documentale, le richieste di rimborso a un workflow con approvazione, e le richieste fuori ambito a un operatore umano.

### Parallelizzazione

Più chiamate indipendenti vengono eseguite in parallelo:

- **Sectioning**: il compito è diviso in parti indipendenti (analizzare sezioni diverse di un contratto) e i risultati sono aggregati.
- **Voting**: lo stesso compito è eseguito più volte, con risultati confrontati o votati, per aumentare l'affidabilità su decisioni critiche.

### Orchestrator-workers

Un orchestratore analizza la richiesta, decide quali sotto-task servono e li assegna a worker, che possono essere agenti specializzati o chiamate semplici. L'orchestratore sintetizza i risultati.

Caso d'uso: analisi di un bando di gara. L'orchestratore delega ai worker: requisiti tecnici, requisiti economici, scadenze, penali. Ogni worker lavora su un sottoinsieme del documento, l'orchestratore produce la sintesi.

### Evaluator-optimizer

Un generatore produce una bozza, un valutatore la confronta con criteri espliciti e rimanda un feedback. Il ciclo continua fino a un criterio di accettazione o a un limite di iterazioni.

Caso d'uso: generare una descrizione prodotto per un e-commerce, con valutatore che controlla lunghezza, presenza delle caratteristiche obbligatorie, assenza di claim non verificati. Il valutatore dovrebbe usare controlli deterministici dove possibile (lunghezza, parole vietate) e il modello solo per i criteri semantici.

## 4. Pattern di controllo e affidabilità

Un agente che agisce nel mondo (invia email, modifica dati, esegue codice) richiede controlli. Il modello non deve essere l'unico responsabile della sicurezza.

### Human-in-the-loop

L'agente si ferma e chiede conferma prima di azioni irreversibili o costose. Il punto di approvazione va progettato:

- quali azioni richiedono conferma (invio, cancellazione, pagamento, modifica di permessi);
- che cosa vede l'approvatore (azione proposta, motivazione, dati coinvolti);
- che cosa succede se nessuno approva entro un tempo (annullamento, non esecuzione di default).

Errore comune: chiedere conferma per ogni passo, fino a rendere l'agente inutile. Chiedi conferma per le azioni che hanno effetti esterni o irreversibili, non per la lettura.

### Permessi e minimo privilegio

Ogni strumento dichiara cosa può fare e l'harness ne limita l'accesso: solo lettura o anche scrittura, solo certe tabelle, solo certe cartelle, credenziali di servizio limitate. Un agente che legge email non deve poter anche inviarle, a meno che sia necessario.

Attenzione alla prompt injection (già introdotta nel modulo 2): se l'agente legge contenuti non fidati (pagine web, email, documenti), questi possono contenere istruzioni per usare gli strumenti in modo dannoso. La difesa efficace non è nel prompt ma nei permessi e nelle conferme.

Una regola utile per valutare il rischio: un agente è particolarmente esposto quando combina tre capacità insieme:

1. accesso a dati privati (email, documenti interni, database);
2. esposizione a contenuti non fidati (pagine web, allegati, messaggi di terzi);
3. possibilità di comunicare verso l'esterno (inviare email, chiamare URL, scrivere su sistemi condivisi).

Con tutte e tre, un'istruzione nascosta in un contenuto letto può portare l'agente a esfiltrare dati privati. Se il caso d'uso le richiede tutte, almeno una va mediata da una conferma umana o da un controllo deterministico, ad esempio una lista di destinatari ammessi.

### Guardrail

Validazioni sull'input e sull'output:

- schema dei parametri prima dell'esecuzione dello strumento;
- filtri di policy sulle azioni (ad esempio, nessun invio a domini esterni);
- controllo dell'output finale prima di mostrarlo o inviarlo.

### Limiti e budget

- numero massimo di passi;
- budget di token e di costo per richiesta;
- timeout per ogni strumento e per l'intera esecuzione;
- condizioni di uscita esplicite (ad esempio: dopo due tentativi falliti, consegna all'umano con il riepilogo).

### Osservabilità

Per un agente, il log non è opzionale. Registra ogni passo: ragionamento visibile, chiamata, argomenti, esito, token, durata. Quando un agente sbaglia, senza la traccia completa non puoi capire se il problema era il modello, lo strumento o la descrizione dello strumento.

Esercizio: prendi l'esempio di loop della sezione 2 e aggiungi: un limite di budget in token, una conferma umana per uno strumento a scelta, e un log strutturato di ogni passo. Scrivi un caso di test che verifica che l'agente si fermi quando il budget è esaurito.

## 5. Memoria e contesto

Un agente che lavora su compiti lunghi accumula messaggi, risultati di strumenti e ragionamenti. Il contesto cresce e il costo sale. Servono strategie di gestione.

### Memoria a breve termine

La storia della sessione nel contesto. Strategie:

- **troncamento**: si scartano i messaggi più vecchi (semplice, ma si perde informazione);
- **riassunto**: ogni N passi il modello riassume la storia in un blocco compatto che sostituisce i messaggi originali;
- **risultati compatti**: lo strumento restituisce solo i campi utili, non interi oggetti.

### Memoria a lungo termine

Fatti e preferenze salvati all'esterno (file, database, vector store) e recuperati quando servono. Esempio: le preferenze di un utente sul formato dei report, salvate e caricate all'inizio della sessione.

Attenzione: una memoria a lungo termine può memorizzare informazioni errate o obsolete. Serve una politica di aggiornamento e, se contiene dati personali, una base giuridica e una possibilità di cancellazione.

### Context engineering

Scegliere con cura cosa mettere nel contesto a ogni passo. Ciò che il modello non vede non può usarlo, ma ciò che vede in eccesso lo distrae. È la competenza che collega il modulo 2 (prompt), il modulo 4 (retrieval) e questo modulo (memoria).

### Sub-agenti

Un compito isolato viene affidato a un agente con un contesto proprio e pulito. Il sub-agente esplora (ad esempio cerca nel codice o nei documenti), poi restituisce all'agente principale solo il risultato sintetico. Il contesto principale resta leggero.

Caso d'uso: un agente di coding che deve capire come funziona un modulo grande. Un sub-agente legge decine di file e restituisce una descrizione di una pagina; l'agente principale non deve caricare tutti i file nel proprio contesto.

## 6. Protocolli e riuso

### MCP (Model Context Protocol)

Standard aperto con cui un harness scopre e usa tool, risorse e prompt esposti da server esterni. Il vantaggio è il riuso: un server che espone i dati di un CRM può essere usato da qualunque client compatibile, senza integrazione custom per ogni coppia modello-applicazione.

Esercizio: cerca la documentazione del protocollo e descrivi con parole tue, in mezza pagina, la differenza tra l'integrazione di uno strumento costruita direttamente nel codice e l'integrazione tramite un server MCP. Quali vantaggi e quali rischi introduce l'uso di server di terze parti?

### Skill e istruzioni riusabili

Procedure e conoscenza di dominio raccolte in file (una descrizione, istruzioni, eventuali script) che l'agente carica solo quando il compito le richiede. Questo tiene il contesto leggero e rende le competenze condivisibili tra progetti.

Caso d'uso: una skill "redazione verbale di riunione" con il formato aziendale, la lista degli elementi obbligatori e le regole di tono. L'agente la carica quando riceve una trascrizione, non in ogni conversazione.

## 7. Multi-agent: quando e perché

Un sistema multi-agente ha più agenti con ruoli distinti che collaborano. Le forme più comuni:

- **Supervisor**: un coordinatore riceve la richiesta, smista ai worker e decide quando il compito è completo.
- **Handoff**: un agente, riconoscendo che un'altra competenza serve, passa il controllo della conversazione a un altro agente.
- **Pipeline di ruoli**: ad esempio un pianificatore, un esecutore e un revisore in sequenza.

Vantaggi: specializzazione dei prompt e degli strumenti, contesti separati, possibilità di usare modelli diversi per ruoli diversi.

Costi: più chiamate (quindi più costo e latenza), più punti di fallimento, comunicazione tra agenti che può perdere informazione, debugging più difficile.

Regola pratica: parti da un singolo agente con strumenti ben progettati. Passa al multi-agente solo quando misuri che il singolo agente non regge per ragioni chiare: contesto troppo grande per un solo agente, ruoli con permessi incompatibili, parallelismo necessario.

### Caso d'uso da sviluppare

Un assistente per l'analisi di un bando di gara. Confronta tre architetture: (a) un singolo agente con quattro strumenti (legge documento, cerca requisiti, calcola scadenze, scrive sintesi); (b) orchestrator-workers con tre worker; (c) workflow fisso con chaining. Per ciascuna definisci: costo stimato, latenza, rischi di errore e come li misureresti con il dataset del modulo 6.

## 8. Valutare un agente

Un agente si valuta come un sistema RAG (modulo 6), con due differenze: il risultato dipende da una sequenza di decisioni, e il percorso conta quanto la risposta finale.

### Metriche

- **Tasso di successo**: percentuale di compiti completati correttamente. Il criterio di successo va definito per ogni caso, possibilmente verificabile dal codice (il ticket è stato creato con i campi giusti, il file contiene il risultato atteso).
- **Efficienza**: numero di passi, token e costo per compito completato. Due agenti con lo stesso tasso di successo possono differire di un ordine di grandezza nel costo.
- **Traiettoria**: l'agente ha usato gli strumenti giusti, con argomenti corretti, senza azioni superflue o pericolose?
- **Robustezza**: lo stesso compito eseguito più volte dà esiti coerenti? Con temperatura maggiore di zero, un agente che riesce 1 volta su 3 non è affidabile.
- **Sicurezza**: sui casi avversari (istruzioni nascoste in un documento letto), l'agente non deve compiere azioni non autorizzate.

### Struttura di un caso di test

```json
{
  "id": "helpdesk-007",
  "richiesta": "Il cliente 4411 dice che l'ordine non è arrivato, verifica e apri un ticket se serve",
  "stato_iniziale": {"ordine_4411": {"stato": "spedito", "giorni_dalla_spedizione": 12}},
  "atteso": {
    "strumenti_obbligatori": ["cerca_ordine", "apri_ticket"],
    "strumenti_vietati": ["rimborsa_ordine"],
    "verifica_finale": "esiste un ticket con categoria SPEDIZIONE per il cliente 4411"
  },
  "max_passi": 6
}
```

Gli strumenti, nei test, lavorano su un ambiente simulato (database di prova, servizi finti): l'agente non deve toccare sistemi reali durante la valutazione.

### Esercizio

Scrivi dieci casi di test nel formato sopra per l'agente del progetto finale, inclusi due casi avversari. Esegui ciascun caso tre volte e riporta: tasso di successo, passi medi, costo medio, e i casi in cui la traiettoria contiene uno strumento vietato.

## 9. Quando NON usare un agente

- Il percorso dei passaggi è noto in anticipo: un workflow deterministico è più affidabile.
- Il costo di un errore è alto e il compito è ripetibile: una pipeline con validazioni e test vale più di un agente autonomo.
- Serve una risposta immediata e a bassa latenza: ogni passo di ragionamento e strumento aggiunge tempo.
- Non esiste ancora un modo di valutare il comportamento: senza dataset e metriche non puoi sapere se l'agente migliora o peggiora.

Principio guida: parti dal workflow più semplice che funziona, aumenta l'autonomia solo dove serve e misura l'effetto con la valutazione del modulo 6.

## Progetto finale suggerito

Costruisci un agente per un problema reale del tuo ambito (helpdesk, gestione ordini, supporto HR, ricerca documentale). Consegna:

- la motivazione della scelta agente vs workflow, con un confronto onesto;
- la lista degli strumenti con descrizioni e permessi;
- il loop con limiti di passi e budget;
- almeno una conferma umana per un'azione sensibile;
- il log di tre esecuzioni complete;
- una suite di dieci casi di test, incluso un tentativo di prompt injection tramite un contenuto letto dall'agente.

## Letture di approfondimento

- [Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022)](https://arxiv.org/abs/2210.03629): il loop ragionamento-azione alla base del tool use.
- [Anthropic, "Building effective agents" (2024)](https://www.anthropic.com/research/building-effective-agents): distinzione tra workflow e agenti e rassegna dei pattern discussi in questo modulo.
- [Anthropic, "Effective context engineering for AI agents"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): gestione del contesto, memoria e sub-agenti.
- [Lilian Weng, "LLM Powered Autonomous Agents" (2023)](https://lilianweng.github.io/posts/2023-06-23-agent/): panoramica su pianificazione, memoria e uso di strumenti.
- [Model Context Protocol](https://modelcontextprotocol.io): specifica e documentazione del protocollo MCP.
