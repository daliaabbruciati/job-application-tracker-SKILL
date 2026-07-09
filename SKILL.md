---
name: candidature-tracker
description: Sincronizza automaticamente Gmail e Google Calendar con un database Notion per tenere traccia dello stato delle candidature di lavoro inviate dall'utente. Usa questa skill ogni volta che l'utente chiede di "fare il sync delle candidature", "aggiornare il tracker lavoro", "controllare a che punto sono i colloqui", o qualcosa di simile — anche se non nomina esplicitamente Notion, Gmail o Calendar, e anche se non ha ancora un database Notion pronto per questo scopo. Richiede (o guida alla creazione di) i connettori MCP Notion, Gmail e Google Calendar.
---

# Candidature Tracker

Skill per mantenere aggiornato un database Notion di candidature di lavoro, incrociando le email di Gmail e gli eventi di Google Calendar degli ultimi giorni. Pensata per essere usata da chiunque, anche da chi parte da zero (nessun database Notion ancora creato, connettori non ancora collegati).

## Step 0 — Verifica prerequisiti (fallo sempre per primo)

### 0.1 Connettori
Controlla se hai a disposizione i tool di **Notion**, **Gmail** e **Google Calendar**. Se uno o più mancano:
- Usa `search_mcp_registry` con le keyword pertinenti (es. `["notion"]`, `["gmail", "email"]`, `["calendar"]`).
- Usa `suggest_connectors` per farli collegare all'utente.
- Non proseguire con lo step successivo finché l'utente non ha collegato almeno Notion. Gmail e Calendar servono entrambi per una scansione completa, ma se l'utente vuole partire con uno solo dei due va bene: fallo presente e procedi con quello disponibile.

### 0.2 Database Notion
Chiedi all'utente (o cerca tu) se esiste già una pagina/database dedicato alle candidature. Prova con `notion-search` usando parole chiave come "candidature", "job tracker", "colloqui", "applications".

**Se non esiste alcun database:**
Proponi di crearlo tu stesso, spiegando cosa farai, e chiedi conferma prima di creare pagine nel workspace dell'utente. Crea una pagina con un database inline con questo schema minimo (puoi arricchirlo se l'utente lo desidera):

| Proprietà | Tipo | Note |
|---|---|---|
| `Azienda` | title | nome dell'azienda o dell'agenzia |
| `Ruolo` | text | posizione per cui ci si è candidati |
| `Stato candidatura` | status | opzioni: `Nessun contatto`, `Inviata`, `Contatto HR`, `Call conoscitiva`, `Test tecnico`, `Rifiutata`, `Accettata` |
| `Ultimo aggiornamento` | date | data dell'ultimo cambiamento rilevato |
| `Note` | text | sintesi libera di cosa è successo / prossimi step |

Usa `notion-create-pages` (o il tool di creazione database disponibile) per generarla, poi condividi il link con l'utente e chiedi conferma che vada bene prima di iniziare a popolarla.

**Se esiste già un database ma con nomi di colonne diversi** (es. "Company" invece di "Azienda", "Stage" invece di "Stato candidatura", o senza una colonna Note):
- Adatta il matching ai nomi reali, non assumere che siano quelli sopra. Leggi sempre lo schema reale con `notion-fetch` sul data source prima di scrivere.
- Se manca una colonna che ti serve per registrare informazioni utili (tipicamente `Note` o una data di ultimo aggiornamento), proponi di aggiungerla con `notion-update-data-source` (`ADD COLUMN ...`) **chiedendo conferma prima di modificare lo schema di un database esistente dell'utente**.
- Se lo stato `Stato candidatura` ha opzioni diverse o incomplete rispetto alla tabella sopra, non forzare le opzioni esistenti: mappa la classificazione dello step 5 sulle opzioni più vicine già presenti, e se davvero mancano fasi importanti proponi (senza imporre) di aggiungerle.

### 0.3 Prima volta vs sync successivi
Se il database è vuoto o appena creato, tutte le candidature trovate saranno nuove righe. Se invece contiene già righe, esegui il matching descritto allo step 4 prima di decidere se creare o aggiornare.

## Flusso di esecuzione (dopo lo step 0)

### 1. Leggi lo stato attuale del database Notion
- `notion-fetch` sull'URL/ID della pagina per trovare il database inline e il suo `data-source-url` (`collection://...`).
- `notion-fetch` sul data source per leggere lo schema esatto (nomi proprietà, opzioni di stato disponibili) — non dare per scontati i nomi, leggili sempre.
- Prova a leggere le righe esistenti con `notion-query-database-view` o `notion-query-data-sources` (SQL). **Nota**: questi due tool richiedono un piano Notion Business+ con Notion AI: se falliscono con `validation_error`, usa `notion-search` con `data_source_url` impostato sul data source per recuperare le pagine esistenti, oppure procedi a vista.

### 2. Scansiona Gmail (se collegato)
Cerca con `Gmail:search_threads`, query tipo:
```
newer_than:7d (candidatura OR colloquio OR interview OR screening OR "call conoscitiva" OR "prova tecnica" OR recruiter OR posizione OR assunzione OR offerta)
```
Adatta le parole chiave alla lingua in cui l'utente riceve le email (es. aggiungi termini inglesi se lavora con aziende internazionali: "application received", "interview invitation", "technical assessment", "we regret to inform you"...).

Intervallo di default "7d", oppure dalla data dell'ultimo sync se nota. Per i thread ambigui, usa `Gmail:get_thread` o `Gmail:get_message` per leggere il corpo completo prima di classificare — non fidarti solo dello snippet.

Scarta le email chiaramente promozionali/non pertinenti (newsletter, alert generici di piattaforme senza una candidatura specifica dietro).

### 3. Scansiona Google Calendar (se collegato)
`Google Calendar:list_events` sul mese corrente (`startTime`/`endTime` primo-ultimo giorno del mese), includendo eventi passati e futuri. Cerca colloqui, call, interview nel titolo/descrizione/partecipanti.

### 4. Algoritmo di matching Azienda ↔ Email/Evento
Per ogni email o evento trovato:
1. Estrai il dominio del mittente (`nome@azienda.com` → `azienda`), rimuovi TLD (`.com`, `.it`, `.io`...) e suffissi societari (`srl`, `spa`, `inc`, `group`, `ltd`...).
2. Confronta il nome ripulito con la colonna "azienda" del database (qualunque sia il suo nome reale): match se uno contiene l'altro (case-insensitive).
3. **Piattaforme/agenzie di recruiting** (LinkedIn, Teamtailor, Factorial, Recruitee, Indeed, agenzie interinali come Randstad/Adecco/Gi Group/Akkodis/Grafton...): il dominio del mittente NON è l'azienda. Estrai il nome dell'azienda reale dal soggetto o dal corpo (es. "la tua candidatura per [Ruolo] presso [Azienda]", o il nome del cliente citato nel testo). Se l'agenzia non cita un cliente specifico (candidatura spontanea all'agenzia stessa), usa il nome dell'agenzia come Azienda e specificalo nelle Note.
4. Se non trovi nessuna riga corrispondente → è una nuova candidatura da creare.
5. Se trovi una riga corrispondente → valuta se lo stato o le note vanno aggiornati (vedi step 5).

### 5. Classificazione dello stato (regole)
Leggi oggetto + corpo completo dell'email (o descrizione dell'evento calendar) e classifica secondo queste regole, dalla più recente/avanzata. I nomi qui sotto sono quelli "standard" proposti allo step 0.2: se il database dell'utente usa altre etichette, mappa sul concetto più vicino.

| Stato "standard" | Segnali tipici |
|---|---|
| Rifiutata | "abbiamo deciso di non procedere", "non sussistono le condizioni ideali", esito negativo esplicito |
| Accettata | proposta di assunzione firmata/accettata, contratto confermato |
| Test tecnico | prova tecnica, live coding, HackerRank/Codility, "test completato", colloquio con team engineering |
| Call conoscitiva | invito/conferma a una call o colloquio (Teams/Meet/Zoom) con recruiter o hiring manager, "incontro conoscitivo", "primo colloquio" |
| Contatto HR | un recruiter ha risposto/scritto ma senza ancora fissare una call precisa, richiesta di informazioni aggiuntive, discussione su modalità/sede |
| Inviata | conferma automatica di ricezione candidatura ("grazie per esserti candidato", "CV ricevuto"), nessun contatto umano ancora |
| Nessun contatto | candidatura pianificata ma non ancora inviata (uso raro in questo flusso) |

Regole di priorità:
- Se un thread contiene più fasi (es. candidatura → poi test → poi call), usa **sempre l'ultimo stato raggiunto**, non il primo.
- In caso di ambiguità reale tra due stati adiacenti, preferisci quello supportato dal segnale più recente e concreto (es. un invito a calendario batte una menzione generica nel testo).
- Se lo schema Notion non ha uno stato che rappresenti fedelmente la situazione, usa lo stato disponibile più vicino e **specifica il dettaglio nelle Note** invece di forzare un nuovo stato nello schema, a meno che l'utente non chieda esplicitamente di ampliare le opzioni.

### 6. Scrivi su Notion
- **Nuova candidatura** → `notion-create-pages` con parent `data_source_id`, valorizzando le proprietà reali del database (azienda, ruolo, stato, note, data ultimo aggiornamento = oggi).
- **Candidatura esistente con stato cambiato** → `notion-update-page` (`command: update_properties`) sulla pagina corrispondente, aggiornando stato, note e data.
- Non toccare candidature esistenti se non c'è nessun segnale nuovo da email/calendar.

### 7. Report finale
Presenta all'utente un riepilogo testuale (una tabella markdown va bene) con:
- Candidature aggiunte (azienda, ruolo, stato, breve motivo)
- Candidature aggiornate (stato precedente → nuovo, motivo)
- Eventuali email escluse perché non pertinenti (menzione breve, non serve elencarle tutte)
- Eventuali problemi (es. tool Notion non disponibili sul piano attuale, connettore mancante, database creato ex novo)

## Note per chi usa questa skill per la prima volta
- Non serve avere già tutto pronto: se manca il database, il connettore, o una colonna, la skill ti guida a crearli passo passo, chiedendoti conferma prima di ogni modifica al tuo workspace Notion.
- Se non sei sicuro dei nomi delle colonne o degli stati che vuoi usare, va bene lo schema standard proposto allo step 0.2: puoi sempre rinominarlo o ampliarlo dopo, direttamente in Notion.
- Se ricevi candidature/colloqui anche in altre lingue oltre l'italiano, segnalalo: le parole chiave di ricerca email vanno adattate di conseguenza (vedi step 2).

## Limiti noti
- Questa skill non gira da sola: in una chat normale va rilanciata manualmente ("fai il sync delle candidature"). Per farla girare in automatico ogni N ore, serve **Claude Cowork** con un task pianificato (`/schedule`), incollando queste istruzioni.
- `notion-query-database-view` e `notion-query-data-sources` richiedono Notion Business+ con Notion AI: se il workspace non lo supporta, il matching con le righe esistenti va fatto via `notion-search` o a vista, con più margine di errore.
- Le email di alert generici (job alert, newsletter di piattaforme formative) non vanno mai trattate come candidature.
