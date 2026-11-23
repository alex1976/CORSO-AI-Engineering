# Modulo 3: Architettura e Chiamate API

## Obiettivi di apprendimento

Al termine del modulo sarai in grado di:

- costruire una richiesta a un LLM tramite API con i ruoli corretti e i parametri essenziali;
- stimare costi e latenza di una chiamata prima di eseguirla;
- classificare gli errori delle API e decidere quali ritentare;
- implementare un retry con exponential backoff e jitter;
- consumare una risposta in streaming e mostrarla in tempo reale;
- progettare un livello di astrazione che permetta di cambiare provider senza riscrivere l'applicazione.

## Premessa: una chiamata LLM è una chiamata di rete costosa e imprevedibile

Chi arriva dal software tradizionale è abituato a chiamate HTTP con latenza di pochi millisecondi, costo fisso e risposta deterministica. Una chiamata a un LLM è diversa: può durare da alcune centinaia di millisecondi a decine di secondi, il costo dipende dal numero di token, la risposta cambia tra una chiamata e l'altra e il provider può rispondere con errori di capacità anche quando la tua richiesta è valida.

Gestire le chiamate significa quindi gestire tre cose: costi, latenza e fallimenti.

## 1. Anatomia di una richiesta

La maggior parte delle API di chat ha una struttura simile. Anche se i nomi dei campi cambiano tra provider, i concetti sono questi:

- **Modello**: identificatore della versione da usare. Va trattato come una dipendenza: versionato, documentato, sostituibile.
- **Messaggi**: una lista ordinata di turni, ciascuno con un ruolo.
  - `system`: istruzioni persistenti e regole.
  - `user`: input dell'utente o dell'applicazione.
  - `assistant`: risposte precedenti del modello, quando si simula una conversazione.
- **Parametri di generazione**: temperatura, top-p, numero massimo di token in output, sequenze di stop.
- **Strumenti (opzionali)**: definizioni di funzioni che il modello può chiedere di invocare (vedi modulo 7).
- **Formato di risposta (opzionale)**: schema JSON atteso, vedi modulo 2.

Esempio minimo in Python, con un client generico (il nome del client e dei campi varia da provider a provider; adatta alla documentazione che usi):

```python
import os

def chiedi_al_modello(client, domanda: str) -> str:
    risposta = client.chat(
        model=os.environ["LLM_MODEL"],   # il modello è configurazione, non codice
        system="Rispondi in italiano, in modo conciso.",
        messages=[{"role": "user", "content": domanda}],
        temperature=0.2,
        max_tokens=400,
    )
    return risposta.text
```

Nota: il nome del modello viene da una variabile d'ambiente. Cambiare modello (o provider) non dovrebbe richiedere una modifica al codice applicativo.

### Esercizio

Scrivi lo stesso prompt con e senza `system`. Osserva quali regole il modello segue meglio quando sono nel system prompt rispetto a quando sono nel primo messaggio utente, e perché.

## 2. Costi, latenza e token

### Costo

Il costo di una chiamata è, in forma semplificata:

```
costo = (token_input × prezzo_input) + (token_output × prezzo_output)
```

Due dettagli spesso ignorati:

- l'output costa di solito più dell'input;
- ogni chiamata reinvia la storia della conversazione: una chat di venti turni paga ogni volta anche i turni precedenti. Il costo cresce più che linearmente con la lunghezza della sessione.

Molti provider offrono sconti per il prompt caching (riutilizzo di prefissi identici) e per l'elaborazione in batch. Verifica sempre la documentazione aggiornata: prezzi e condizioni cambiano spesso.

### Latenza

Due metriche da misurare separatamente:

- **Time-to-first-token (TTFT)**: quanto passa prima che arrivi il primo pezzo di risposta. Dominata dal prefill del prompt.
- **Throughput**: token al secondo durante la generazione. Determina quanto dura una risposta lunga.

Per la UX conta soprattutto il TTFT: uno streaming che parte in fretta sembra più veloce anche se il tempo totale è uguale.

### Esercizio di stima

Per un'applicazione che riassume documenti, stima: token medi di input (documento più system prompt), token medi di output, numero di richieste al giorno. Calcola il costo mensile con i prezzi del provider che scegli. Poi chiediti: quale parte del costo potresti ridurre tagliando il contesto senza perdere qualità?

## 3. Gestione degli errori

Gli errori delle API si dividono in categorie con comportamenti diversi:

| Tipo | Esempio | Azione |
|---|---|---|
| Errore di richiesta | 400, input malformato, contesto troppo lungo | Non ritentare: correggi la richiesta |
| Autenticazione | 401, 403 | Non ritentare: verifica le credenziali |
| Limite di frequenza | 429, quota superata | Ritenta con backoff |
| Errore lato provider | 500, 502, 503, 529 (sovraccarico) | Ritenta con backoff, poi fallback |
| Timeout | La risposta non arriva in tempo | Ritenta con cautela, limita le dimensioni |

Attenzione: un retry aggressivo su un provider sovraccarico peggiora la situazione per tutti. Per questo serve il backoff.

### Exponential backoff con jitter

```python
import random
import time

def con_retry(operazione, max_tentativi=5, base=1.0, massimo=30.0):
    for tentativo in range(max_tentativi):
        try:
            return operazione()
        except ErroreRitentabile:          # 429, 5xx, timeout
            if tentativo == max_tentativi - 1:
                raise
            attesa = min(massimo, base * (2 ** tentativo))
            time.sleep(random.uniform(0, attesa))   # full jitter
```

Il jitter evita che molti client ritentino tutti nello stesso istante, creando picchi sincronizzati. Se il provider restituisce un header esplicito di attesa (ad esempio `retry-after`), quello ha la precedenza sul calcolo locale.

### Esercizio

Simula un servizio che fallisce con probabilità 70% e restituisce 429 o 503. Implementa il retry sopra e misura: quante chiamate totali servono per avere una risposta, e quanto tempo totale impiega il client. Poi confronta con un retry senza backoff.

## 4. Streaming

Lo streaming permette di ricevere la risposta mentre viene generata, di solito tramite Server-Sent Events o un protocollo analogo. Il client riceve eventi con piccoli frammenti di testo che va accumulati e visualizzati.

Punti di attenzione:

- gestire la chiusura anomala della connessione a metà risposta;
- mostrare uno stato intermedio (cursore, indicatore) se la risposta è lenta a iniziare;
- decidere cosa succede se l'utente annulla: interrompere la richiesta lato client non sempre interrompe il calcolo lato provider, quindi può avere senso anche un timeout lato server;
- non applicare la validazione di output strutturato finché la risposta non è completa.

Esempio di flusso in pseudocodice:

```text
per ogni frammento ricevuto dallo stream:
    accumula il testo
    invia il frammento alla UI
al termine:
    valida l'output completo (se strutturato)
    salva nel log: token usati, TTFT, durata totale
```

### Caso d'uso da sviluppare

Una pagina web con chat che mostra la risposta in streaming. Implementa il flusso lato server che inoltra i frammenti al browser, registra TTFT e token usati per ogni richiesta, e gestisce l'annullamento da parte dell'utente.

## 5. Astrazione sul provider

Un'applicazione che chiama direttamente le API di un provider in mille punti diventa costosa da cambiare. Una buona pratica è definire un'interfaccia interna:

```python
from typing import Protocol

class ModelloLLM(Protocol):
    def genera(self, system: str, messaggi: list[dict], **opzioni) -> "Risposta": ...
```

Ogni provider (OpenAI, Anthropic, un modello open source servito tramite Groq o Together AI) ha un'implementazione di questa interfaccia. Il resto dell'applicazione dipende solo dall'interfaccia.

Attenzione però a non nascondere troppo: le differenze reali (gestione degli strumenti, formato dei messaggi, comportamento con l'output strutturato, limiti di contesto) esistono e vanno testate. Un'astrazione che ignora queste differenze produce bug difficili da diagnosticare.

### Confronto tra provider: criteri, non verdetti

Quando confronti provider e modelli, valuta almeno:

- qualità sul tuo compito (non sui benchmark generici): misura con il tuo dataset, modulo 6;
- costo per task completato, non per token;
- latenza p50 e p95 nel tuo scenario;
- disponibilità, limiti di frequenza e regione dei dati;
- supporto a strumenti, output strutturato, streaming, dimensione del contesto;
- condizioni contrattuali sull'uso dei dati.

Esercizio: scegli tre modelli (uno chiuso, uno aperto servito da un provider terzo, uno più piccolo ed economico) e definisci la tabella di confronto con questi criteri. Non serve completarla con dati reali: il punto è decidere quali misure contano prima di guardare i risultati.

## 6. Osservabilità

Ogni chiamata dovrebbe lasciare una traccia: modello usato, versione del prompt, token input e output, TTFT, durata totale, esito, numero di retry, costo stimato. Senza questi dati non sai quanto costa una funzionalità né dove si rompe.

Consiglio pratico: assegna a ogni richiesta un identificativo e registralo insieme alla risposta. Quando un utente segnala un problema, potrai ricostruire esattamente che cosa è successo.

## 7. Ottimizzazione dei costi

Il costo di una funzionalità LLM si controlla con scelte di architettura, non solo negoziando il prezzo.

- **Prompt caching**: se molte richieste condividono un prefisso lungo e identico (system prompt, documentazione, esempi few-shot), molti provider permettono di riutilizzarne il calcolo a un costo ridotto. Regola pratica: metti le parti stabili all'inizio del prompt e quelle variabili alla fine.
- **Elaborazione batch**: per compiti non interattivi (classificare 50.000 ticket storici) le API batch costano meno, in cambio di tempi di completamento più lunghi.
- **Modello giusto per il compito**: un modello piccolo per classificare e smistare, quello grande solo per la generazione complessa.
- **Controllo dell'output**: `max_tokens` adeguato, istruzioni sulla lunghezza, output strutturato al posto di testo libero quando il destinatario è un programma.
- **Contesto essenziale**: in una chat lunga, riassumere la storia invece di rimandarla per intero.

### Esercizio

Riprendi la stima di costo della sezione 2. Calcola l'effetto di tre interventi: caching del system prompt, passaggio a un modello piccolo per il 70% delle richieste semplici, riduzione del 30% dei token di output. Quale intervento rende di più nel tuo caso?

## 8. Sicurezza di chiavi, dati e budget

- **Chiavi API**: mai nel codice né nel repository. Usa variabili d'ambiente in sviluppo e un secret manager in produzione. Una chiave per ambiente (sviluppo, test, produzione) facilita la revoca.
- **Chiavi lato client**: un'applicazione web o mobile non deve mai chiamare il provider direttamente con la chiave. Le chiamate passano dal tuo backend, che autentica l'utente e applica i limiti.
- **Budget**: imposta limiti di spesa sul provider e limiti per utente nel backend. Un bug in un loop può generare migliaia di chiamate in pochi minuti.
- **Dati inviati**: tutto ciò che è nel prompt viene inviato al provider. Verifica le condizioni contrattuali (conservazione, uso per addestramento, regione) e valuta il mascheramento dei dati personali prima dell'invio. Il tema è approfondito nel modulo 8.

### Esercizio

Scrivi un piccolo backend (ad esempio FastAPI) che espone `/chat` e inoltra la richiesta al provider. Aggiungi: lettura della chiave da variabile d'ambiente, limite di dieci richieste al minuto per utente, rifiuto delle richieste con input oltre una soglia di token.

## Letture di approfondimento

- [Documentazione dei provider scelti](https://docs.claude.com): sezioni su rate limit, errori, streaming e prezzi (verificare sempre la versione corrente).
- [AWS Architecture Blog, "Exponential Backoff And Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/): la teoria del retry con backoff e jitter.
- [MDN, "Using server-sent events"](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events): come funziona lo streaming di risposte dal server.
- [OpenTelemetry, "Semantic conventions for Generative AI"](https://opentelemetry.io/docs/specs/semconv/gen-ai/): convenzioni standard per tracciare chiamate a modelli.
