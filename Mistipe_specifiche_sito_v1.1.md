# Mistipè — Specifiche per lo sviluppo del sito

**Versione 1.1 — documento di handoff per gli sviluppatori**
Progetto di dottorato in Heritage Science, Sapienza Università di Roma. Referente: Enrico [Cognome].

> **Novità rispetto alla v1.0**: formato date (EDTF + ISO 8601 + calendario), chiarimento tecnico sulle relazioni Person–Event e Theme–Event/Annotation (tabelle di giunzione, non campi FK semplici), nuova sezione sull'interoperabilità semantica verso CIDOC-CRM/Linked Art. Tutto il resto del documento è invariato rispetto alla v1.0.

---

## 0. Cos'è Mistipè, in breve

Mistipè è un archivio narrativo digitale stratificato sulla storia e la cultura rom. Non è né una biblioteca digitale (catalogo + ricerca) né un sito divulgativo: è una piattaforma in cui fonti storiche, analisi critica computazionale, testimonianze orali e materiali visivi contemporanei convivono attraverso un **modello dati relazionale comune**, esplorabile in due modalità diverse a seconda di chi la usa.

Il vincolo più importante da capire prima di scrivere codice: **il controllo di accesso non è un dettaglio da aggiungere dopo**. Una parte dei contenuti (le interviste) ha permessi granulari decisi persona per persona, revocabili in ogni momento. Se il modello dati non lo prevede fin dallo schema iniziale, va rifatto.

---

## 1. Le due modalità di accesso agli stessi dati

Non sono due sezioni del sito: sono due modi di navigare **lo stesso database**.

| | Modalità STORIA | Modalità ARCHIVIO |
|---|---|---|
| Pubblico | visitatori generici | ricercatori, studiosi |
| Interfaccia | narrazione guidata, mappa/timeline cliccabili, storytelling | tabella filtrabile, tipo database viewer |
| Filtri | tematici, narrativi | per campo: data, luogo, tipo di fonte, livello di certezza, tag |
| Obiettivo | far scoprire e coinvolgere | permettere ricerca sistematica e verifica delle fonti |

Ogni record del database deve poter essere raggiunto da entrambe le modalità.

---

## 2. I quattro ingressi al sito (+ un quinto condizionale)

1. **Mappa / Timeline** — esplorazione per luogo e tempo, prime attestazioni, nodi documentari
2. **Temi** — mestieri, lingua romani, integrazione, samudaripen, pregiudizio
3. **Voci** — interviste e persone (layer testimoniale)
4. **Metodo** — fonti, pipeline, criteri, limiti dichiarati del progetto

**5. Mappatura partecipativa (condizionale)** — un quinto ingresso, legato al layer di counter-surveying, va previsto solo come possibile estensione futura. Non bloccare l'architettura su questo, ma non escluderlo nello schema (vedi §4.6).

---

## 3. Modello dati — sei entità

Questo è il cuore del sistema. Le relazioni sono più importanti degli attributi: un record `Person` acquisisce senso storico dal suo nodo di relazioni (luogo, evento, fonte), non da un singolo campo "etnia".

### 3.1 Place
| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `place_historical` | testo | nome storico del luogo |
| `place_current` | testo | nome attuale |
| `country_current` | testo | |
| `historical_region` | testo | es. "area veneziana", "Mediterraneo orientale" |
| `latitude` / `longitude` | float | **da aggiungere al dataset esistente, ancora mancante** |
| `presence_type` | enum | vedi §3.4 (nodo documentario forte / nodo interpretativo) |

### 3.2 Source
| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `source_type` | enum | paper, documento d'archivio, intervista, foto, video |
| `citation_short` | testo | citazione breve |
| `doi_or_link` | testo | se disponibile |
| `certainty_level` | enum | alto / medio / basso |
| `rights_status` | enum | pubblico dominio / uso critico con citazione / **consenso revocabile (interviste, foto, video)** |

`rights_status = consenso revocabile` è il campo che collega una fonte al sistema di permessi descritto al §5. Nessuna fonte con questo stato può essere mostrata pubblicamente senza controllare lo stato del consenso al momento della richiesta, non solo al momento del caricamento.

### 3.3 Person
| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `person_type` | enum | storica / testimone contemporaneo / autore di fonte |
| `display_name` | testo | **dipende dalla scelta di visibilità, vedi §5.2** |
| `name_variants` | array di testo | grafie storiche alternative |

Per le persone testimoni contemporanee, `display_name` non è un campo statico: è calcolato in base all'opzione di visibilità scelta nel consenso (nome e cognome / solo nome / pseudonimo / anonimo). Non salvare il nome reale nello stesso record pubblico se l'opzione è anonimato o pseudonimo — vedi §5.3 sulla separazione dei dati.

**Relazione con Event — chiarimento tecnico.** `Person` non ha un campo `event_id`: una persona storica può comparire in più attestazioni, e un'attestazione può coinvolgere più persone (es. una famiglia). È una relazione **molti-a-molti**, da implementare con una tabella di giunzione, non con una chiave esterna semplice:

```
event_person (event_id FK → Event, person_id FK → Person, role testo opzionale)
```

`role` è facoltativo ma utile ("maniscalco", "canonico", "capofamiglia") per distinguere il ruolo della persona in quello specifico evento.

### 3.4 Event
| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `event_type` | enum | attestazione, intervista, memoria orale, episodio storico |
| `date_label` | testo | etichetta leggibile per l'interfaccia, es. "metà XV secolo", "1521–1572" |
| `date_edtf` | stringa EDTF | **nuovo** — valore autorevole della data, vedi §3.4.1 |
| `date_start` / `date_end` | data ISO 8601 | limiti calcolati dall'`date_edtf`, per ordinamento/filtri/timeline — non inserirli a mano indipendentemente dall'EDTF |
| `calendar` | enum | **nuovo** — `gregoriano` / `giuliano` / `non specificato nella fonte`, vedi §3.4.1 |
| `presence_type` | enum | **"prima attestazione documentaria nota"**, non "anno di arrivo" — terminologia vincolante, non un dettaglio stilistico |
| `certainty_level` | enum | nodo documentario forte / nodo interpretativo |
| `place_id` | FK → Place | |
| `source_id` | FK → Source | |

#### 3.4.1 Formato delle date — EDTF, non solo ISO 8601

Il dataset storico è pieno di date non esatte ("1316, 1319", "1521–1572", "metà XV secolo"): ISO 8601 da solo non basta a rappresentarle. Lo standard da usare per `date_edtf` è l'**EDTF (Extended Date/Time Format)** della Library of Congress, lo stesso che CIDOC-CRM/Linked Art citano come riferimento per i time-span (vedi §10).

| Caso | Sintassi EDTF | Esempio dal dataset |
|---|---|---|
| Data certa | `YYYY` | `1486` (Modone) |
| Intervallo noto | `YYYY/YYYY` | `1521/1572` (Belgrado) |
| Data incerta | `YYYY?` | `1588?` |
| Data approssimativa | `YYXX~` | `14XX~` (metà '400, Nauplio) |
| Doppia attestazione separata | `YYYY,YYYY` | `1316,1319` (Creta) — attenzione: non è un range |

`date_start` e `date_end` restano colonne ISO 8601 pure (`YYYY-MM-DD` o solo `YYYY`), calcolate come limite inferiore/superiore dell'`date_edtf`: sono per il computer (ordinamento, filtri, disegno della timeline), mentre `date_label` resta per il lettore.

**Calendario.** Buona parte delle attestazioni (area ottomana, Mediterraneo orientale: Creta, Corfù, Filibe/Plovdiv) cade prima o a cavallo della riforma gregoriana del 1582, in territori che hanno mantenuto il calendario giuliano a lungo anche dopo. Registrare una data giuliana come fosse gregoriana introduce un errore sistematico di alcuni giorni che non emerge finché qualcuno non confronta due date vicine tra fonti diverse. Il campo `calendar` rende esplicita questa informazione invece di darla per scontata.

**Relazioni con Theme.** Un `Event` può essere taggato con uno o più `Theme` per la navigazione tematica (pagina Temi), indipendentemente dal fatto che esista o meno un'`Annotation` associata. Anche questa è una relazione molti-a-molti:

```
event_theme (event_id FK → Event, theme_id FK → Theme)
```

### 3.5 Theme
| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `label` | testo | lingua romani, samudaripen, mestiere, pregiudizio, cittadinanza... |
| `description` | testo | |

**Relazioni — chiarimento tecnico.** `Theme` non ha campi FK propri: è un'entità "tag", referenziata da altre entità tramite tabelle di giunzione, non il contrario.
- `event_theme` (vedi §3.4) — per la navigazione tematica diretta
- `annotation_theme` (vedi §3.6) — per collegare un tema alle coordinate del pregiudizio o alle categorie del "positivo"

### 3.6 Annotation
Entità più delicata del modello: collega le due pipeline metodologiche (positivo / coordinate del pregiudizio) e le note editoriali ai passaggi di testo o di intervista.

| Campo | Tipo | Note |
|---|---|---|
| `id` | stringa/UUID | |
| `annotation_type` | enum | "positivo" (agency/integrazione) / coordinata pregiudizio (P1–P10) / nota editoriale |
| `code` | testo | es. "P3" per criminalizzazione; vuoto se non applicabile |
| `level` | enum | lessicale / strutturale |
| `text_window` | testo | il passaggio annotato (±50 parole per il "positivo") |
| `source_id` | FK → Source | |
| `annotated_by` | testo | chi ha fatto l'annotazione (utile per revisione, non da mostrare pubblicamente) |

**Relazione con Theme**: tabella di giunzione `annotation_theme (annotation_id FK → Annotation, theme_id FK → Theme)`, come indicato al §3.5.

---

## 4. I layer (contenuto per tipo, non per tabella separata)

I layer **non sono tabelle a sé stanti**: sono viste diverse sullo stesso modello dati, filtrate per tipo di `Source`/`Event`.

| Layer | Contenuto | Fonte nel modello dati |
|---|---|---|
| 1. Storico-documentario | attestazioni, luoghi, ruoli, mestieri | `Place`, `Event`, `Source` (tipo documento) |
| 2. Interpretativo-computazionale | output delle due pipeline | `Annotation` |
| 3. Testimoniale | interviste audio/video | `Source` (tipo intervista) + `Person` (testimone) |
| 4. Visuale-contemporaneo | foto/video del presente | `Source` (tipo foto/video) |
| 5. Spaziale partecipativo (**condizionale**, counter-surveying) | mappatura partecipativa di luoghi scomparsi | estensione futura di `Event`/`Place`, non ancora specificata nel dettaglio — **non sviluppare ora**, ma non usare nomi di campo che la escludano strutturalmente in futuro |

---

## 5. Sistema di permessi e consenso — la parte critica

Questa sezione traduce in requisiti tecnici il protocollo di consenso informato già scritto (documento separato, allegato). **Va letta per intero prima di progettare lo schema di `Source` e `Person`.**

### 5.1 Livelli di accesso per intervista (non per l'intero sito)

Ogni intervista (`Source` di tipo intervista) ha un proprio livello di accesso, scelto dalla persona intervistata, indipendente dagli altri contenuti:

- **Pubblico** — visibile a chiunque
- **Ristretto** — visibile solo a un gruppo definito (es. "solo ricercatori registrati", "solo la mia comunità"). Richiede un sistema di autenticazione/autorizzazione, anche minimale, non solo un link "nascosto".
- **Non online** — esiste nel database per finalità di ricerca/tesi ma non è mai servita al front-end pubblico.

Questo significa: **il controllo di accesso va implementato a livello di record, non di pagina.** Una pagina "Voci" che elenca le interviste deve filtrare lato server in base al livello di accesso di ciascun record e al ruolo di chi sta guardando, non nascondere elementi lato client.

### 5.2 Restrizioni granulari dentro una singola intervista

Una persona può acconsentire alla pubblicazione dell'intervista ma segnalare che alcuni minuti/argomenti restano riservati (es. riguardano persone defunte, la famiglia, saperi non per tutti). Il sistema deve poter:
- marcare intervalli temporali o segmenti di trascrizione come riservati, con motivazione tipizzata
- mostrare un **avviso di contenuto** prima della riproduzione, se richiesto
- mostrare un'**etichetta di attribuzione richiesta** (nome esatto o dicitura da usare sempre)

Non serve costruire un'interfaccia di editing video sofisticata al primo rilascio: basta poter marcare `[start, end]` come "non pubblicato" e tagliare il segmento nel file servito, o mostrare un placeholder.

### 5.3 Separazione dei dati identificativi

Il recapito della persona (email, telefono) **non va mai nello stesso record pubblico** del materiale. Tenerli in tabelle separate, con accesso più ristretto, è un requisito del protocollo GDPR già firmato, non un'opzione architetturale.

### 5.4 Consenso non definitivo (rolling consent)

Le scelte di accesso, restrizione ed etichettatura **non sono fissate una volta per tutte**: la persona può cambiarle in qualsiasi momento (vedi protocollo etico). Serve quindi:
- un campo `consent_last_updated` per intervista
- una procedura (anche manuale, per ora) per aggiornare il record quando arriva una richiesta di modifica o ritiro
- **ritiro del consenso → rimozione effettiva entro il termine dichiarato nel modulo** (attualmente 15 giorni), non solo "flag nascosto"

### 5.5 Etichette di contesto (in stile Traditional Knowledge Labels)

Da mostrare accanto al materiale, non da nascondere nei metadati: attribuzione, non-commerciale, riuso educativo, riuso solo su richiesta. Vocabolario preliminare, da confermare con la comunità — non hardcodare un set chiuso, prevedere che le etichette possano cambiare.

---

## 6. Dataset pilota — schema CSV già definito

Campi stabiliti per il dataset delle 20 città (da usare come seed iniziale del database, non da trattare come formato finale):

```
id, place_historical, place_current, country_current, historical_region,
date_label, date_edtf, date_start, date_end, calendar,
event_type, presence_type, certainty_level,
description_short, source_short, tags,
latitude, longitude   ← da aggiungere, attualmente mancanti
```

`date_edtf` e `calendar` sono le due colonne nuove rispetto alla versione precedente del dataset (vedi §3.4.1).

**Attenzione terminologica vincolante**: `date_label` rappresenta la prima attestazione documentaria nota, mai "anno di arrivo". Se l'interfaccia mostra questa data, deve essere sempre accompagnata da questa qualifica, non presentata come una data di fatto.

`certainty_level` distingue nodi documentari forti (attestazione localizzata) da nodi interpretativi (ricostruzione storica/linguistica): questa distinzione deve essere visibile nell'interfaccia (es. stile visivo diverso sulla mappa), non solo salvata nel database.

---

## 7. Struttura di massima delle pagine

- **Home** — mappa interattiva + timeline, accesso ai quattro ingressi
- **Mappa** — città, cronologia, prime attestazioni, filtri per certezza/tipo
- **Temi** — presenze attive (output pipeline "positivo"), coordinate del pregiudizio (P1–P10), lingua, samudaripen, cittadinanza, mestieri
- **Voci** — interviste, clip, trascrizioni, tag (soggetta ai permessi del §5)
- **Atlante visivo** — foto/video contestualizzati del presente
- **Metodo** — fonti, pipeline, note critiche, limiti dichiarati del progetto (questa pagina è parte integrante del progetto, non una nota a piè di pagina: deve spiegare esplicitamente cosa il dataset non copre)

---

## 8. Vincoli e requisiti non funzionali

- **Localizzazione**: contenuti in italiano e inglese come minimo; le interviste possono essere in altre lingue (spagnolo, croato, romani) con trascrizione/sottotitoli.
- **Formati media**: video 1080p H.264, audio WAV 48kHz per gli originali (i file serviti al pubblico possono essere compressi).
- **Hosting/conservazione dati**: server nell'Unione Europea (vincolo GDPR dichiarato nel protocollo di consenso — non è negoziabile in fase di scelta hosting).
- **Nessun uso del materiale per addestrare sistemi di IA** — impegno scritto nel modulo di consenso: se si integrano servizi terzi (trascrizione automatica, ecc.), verificare le condizioni sul riutilizzo dei dati.
- **Accessibilità**: la mappa/timeline e i filtri di ricerca devono restare utilizzabili anche senza JavaScript pesante lato client per la modalità ARCHIVIO, pensata per un uso da ricercatori.

---

## 9. Cosa NON è richiesto in questa fase

Per evitare di sovradimensionare il primo rilascio:
- il layer 5 (counter-surveying/mappatura partecipativa) — condizionato, non ancora confermato
- un sistema di co-governance con voto/decisioni della comunità — è un processo organizzativo, non una feature software, almeno per ora
- traduzioni oltre italiano/inglese
- un'app mobile dedicata
- **l'implementazione dell'export verso CIDOC-CRM/Linked Art** — ma lo schema va progettato in modo da non precluderlo in futuro (vedi §10)

---

## 10. Interoperabilità semantica (CIDOC-CRM / Linked Art) — da tenere presente, non da implementare ora

Il dottorato a cui appartiene questo progetto è ancorato alle infrastrutture europee DARIAH ed E-RIHS, che si aspettano dati del patrimonio culturale mappabili su standard semantici riconosciuti. Non è un requisito del primo rilascio, ma **lo schema del database deve restare compatibile con questa mappatura**, per non dover essere riscritto in seguito. Le tabelle di giunzione già indicate al §3 (`event_person`, `event_theme`, `annotation_theme`) servono esattamente a questo: CIDOC-CRM rappresenta le relazioni in modo esplicito, non tramite campi impliciti.

Mappatura di riferimento (documentata per esteso in un foglio di calcolo a parte, allegato):

| Entità Mistipè | Classe CIDOC-CRM | Nota |
|---|---|---|
| `Place` | E53 Place | diretta |
| `Event` | E5 Event | collegato a Place (`P7 took place at`) e a Source |
| `Person` | E21 Person | |
| `Source` | E73 Information Object | differenziata per `source_type` |
| `Theme` | SKOS Concept (non una classe CRM) | |
| `Annotation` | E13 Attribute Assignment | pensata per registrare valori attribuiti da un ricercatore — adatta sia al codebook P1–P10 sia al livello di certezza |

**Implicazione pratica per ora**: non serve scrivere codice per questo, ma evitare scelte di schema che lo renderebbero impossibile in futuro — in particolare, mantenere le relazioni come tabelle esplicite (non come stringhe concatenate o campi liberi) e mantenere `date_edtf` come valore autorevole delle date (§3.4.1), perché è il formato che CIDOC-CRM/Linked Art si aspettano per i time-span incerti.

---

## 11. Domande aperte per la scelta dello stack

Da decidere insieme, non assunte in questo documento:
1. Sito statico con dati versionati (es. JSON/CSV in repo + generatore) vs CMS/database dinamico — dipende dalla frequenza di aggiornamento e da chi dovrà inserire dati dopo la tesi.
2. Come implementare l'autenticazione per il livello di accesso "ristretto" (§5.1) — servizio esistente o costruito ad hoc.
3. Chi ospita i file media originali (non compressi) e chi ha accesso al backend per applicare le richieste di modifica/ritiro del consenso.
4. Chi si occupa dell'inserimento dati dopo la fine dello sviluppo — questo condiziona quanto deve essere "amichevole" l'interfaccia di back-office.
