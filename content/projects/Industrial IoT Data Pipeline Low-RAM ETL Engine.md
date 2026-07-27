---
title: "Industrial IoT Data Pipeline & Low-RAM ETL Engine"
date: 2026-07-27
description: "Architettura completa per l'acquisizione, la persistenza e l'elaborazione ETL di dati da macchinari industriali."
tags: ["Python", "IoT", "ETL", "Pandas", "Automation", "System Administration"]
draft: false
---

# Industrial IoT Data Pipeline &amp; Low-RAM ETL Engine

Un'architettura completa per l'acquisizione, la persistenza e l'elaborazione ETL di dati da macchinari di pesatura industriale. Il sistema collega l'infrastruttura operativa (OT) con i sistemi d'analisi IT aziendali, risolvendo le limitazioni di RAM dei dispositivi periferici (*edge*) e garantendo la gestione di file di log di grandi dimensioni con un'impronta di memoria minimale.

## Architettura di Sistema


<div _ngcontent-ng-c1436864849="" class="code-block ng-tns-c1436864849-193 ng-animate-disabled ng-trigger ng-trigger-codeBlockRevealAnimation" id="bkmrk--1" jslog="223238;track:impression,attention;BardVeMetadataKey:[["r_318ef91d8f7a4a90","c_61e9f8d1fca2527b",null,"rc_bee685d8a2fdd1ac",null,null,"it",null,1,null,null,1,0]]"><div _ngcontent-ng-c1436864849="" class="formatted-code-block-internal-container ng-tns-c1436864849-193"><div _ngcontent-ng-c1436864849="" class="animated-opacity ng-tns-c1436864849-193"></div></div></div>```
+---------------------------+
| Industrial Batch Scale    |
| (Edge Memory Limited)     |
+-------------+-------------+
              | (HTTP POST Download)
              v
+-------------+-------------+      (Append / Atomic Sync)      +-------------------------+
| 01. Ingestion Poller      | -------------------------------> | Raw Log Storage         |
| (10-min Cron/Task)        |                                  | (storico.log)           |
+-------------+-------------+                                  +------------+------------+
              |                                                             |
              | (HTTP POST Clear RAM)                                       | (Streamed Reader)
              v                                                             v
+-------------+-------------+                                  +------------+------------+
| Edge Memory Cleared       |                                  | 02. Daily Archiver     |
+---------------------------+                                  | (Nightly Batch Job)     |
                                                               +------------+------------+
                                                                            |
                                                                            v (Transformed Data)
+---------------------------+     (Interactive Low-RAM Stream) +------------+------------+
| 03. Historical Recovery   | -------------------------------->| Structured Excel Output |
| CLI Utility (On-Demand)   |                                  | (/YYYY/MM/Report.xlsx)  |
+---------------------------+                                  +-------------------------+

```

<div _ngcontent-ng-c1436864849="" class="code-block ng-tns-c1436864849-193 ng-animate-disabled ng-trigger ng-trigger-codeBlockRevealAnimation" id="bkmrk--2" jslog="223238;track:impression,attention;BardVeMetadataKey:[["r_318ef91d8f7a4a90","c_61e9f8d1fca2527b",null,"rc_bee685d8a2fdd1ac",null,null,"it",null,1,null,null,1,0]]"><div _ngcontent-ng-c1436864849="" class="formatted-code-block-internal-container ng-tns-c1436864849-193"><div _ngcontent-ng-c1436864849="" class="animated-opacity ng-tns-c1436864849-193"></div></div></div>
## Il Problema Aziendale &amp; La Soluzione

### **Contesto e Criticità**

- **Memoria Periferica Limitata:** La scheda di controllo del macchinario industriale accumula i dati di pesatura nella RAM interna. Senza una pulizia periodica, il sistema va in overflow bloccando le registrazioni.
- **Rischio di Perdita Dati:** Network glitch o failure durante lo scarico rischiano di svuotare la RAM del macchinario prima della scrittura sicura su disco aziendale.
- **Gestione Grandi Volumi (RAM Bottleneck):** Il file di log cumulativo accumula centinaia di migliaia di righe. Caricare l'intero file in memoria con tradizionali librerie d'analisi genera crash di sistema sui server di produzione.

### **Soluzione Ingegneristica**

1. <span data-path-to-node="12,0,0,0">**Poller Atomico a 4 Fasi:** Un demone schedulato esegue l'estrazione dati, la deduplica basata su timestamp, la scrittura atomica su disco con *fsync* fisico e, solo ad operazione confermata, l'azzeramento della RAM della macchina</span><span data-path-to-node="12,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="12,0,0,2">.</span>
2. <span data-path-to-node="12,1,0,0">**Streaming Parser (Low-RAM Processing):** Tutti gli script utilizzano una lettura sequenziale *line-by-line* (I/O streaming) isolando solo i record necessari prima di alimentarli al motore d'analisi Pandas, riducendo l'occupazione di memoria da gigabyte a pochi megabyte</span><span data-path-to-node="12,1,0,1"><sup class="superscript"></sup></span><span data-path-to-node="12,1,0,2">.</span>
3. <span data-path-to-node="12,2,0,0">**ETL &amp; Feature Engineering:** Conversione automatica delle serie di sensori grezzi (<span class="math-inline" data-index-in-node="81" data-math="W_{01..16}">$W\_{01..16}$</span> e <span class="math-inline" data-index-in-node="94" data-math="B_{01..16}">$B\_{01..16}$</span>) in metriche operative aggregate esportate in file Excel categorizzati per Anno/Mese</span><span data-path-to-node="12,2,0,1"><sup class="superscript"></sup></span><span data-path-to-node="12,2,0,2">.</span>

## Tech Stack &amp; Requisiti

- <span data-path-to-node="15,0,0,0">**Language:** Python 3.8+</span><span data-path-to-node="15,0,0,1"><sup class="superscript"></sup></span>
- **Libraries:** `pandas`, `openpyxl`, `requests`, `pathlib`
    
    <sup class="superscript"></sup>
- <span data-path-to-node="15,2,0,0">**OS / Environment:** Windows Server / Linux Enterprise Host</span><span data-path-to-node="15,2,0,1"><sup class="superscript"></sup></span>
- <span data-path-to-node="15,3,0,0">**Scheduling:** Windows Task Scheduler / Linux Cron</span><span data-path-to-node="15,3,0,1"><sup class="superscript"></sup></span>
- <span data-path-to-node="15,4,0,0">**Protocols:** HTTP/REST API (Communication with Edge Hardware)</span><span data-path-to-node="15,4,0,1"><sup class="superscript"></sup></span>

## Struttura dei Moduli

### 1. `poller_daemon.py` — *Ingestion Engine*

- <span data-path-to-node="19,0,0,0">**Frequenza:** Esecuzione ogni 10 minuti via Task Scheduler</span><span data-path-to-node="19,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="19,0,0,2">.</span>
- **Logica Operativa:**
    
    
    - <span data-path-to-node="19,1,1,0,0,0">Effettua una richiesta `HTTP POST` con azione `download` al macchinario</span><span data-path-to-node="19,1,1,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="19,1,1,0,0,2">.</span>
    - <span data-path-to-node="19,1,1,1,0,0">Verifica il file di stato (`last_timestamp.txt`) per filtrare ed eliminare record duplicati</span><span data-path-to-node="19,1,1,1,0,1"><sup class="superscript"></sup></span><span data-path-to-node="19,1,1,1,0,2">.</span>
    - <span data-path-to-node="19,1,1,2,0,0">Scrive le nuove righe sul log di persistenza ed esegue `os.fsync()` per forzare la scrittura fisica su disco evitando l'I/O caching</span><span data-path-to-node="19,1,1,2,0,1"><sup class="superscript"></sup></span><span data-path-to-node="19,1,1,2,0,2">.</span>
    - <span data-path-to-node="19,1,1,3,0,0">Invia il comando `POST` `azzera_log` al macchinario per liberare la memoria periferica</span><span data-path-to-node="19,1,1,3,0,1"><sup class="superscript"></sup></span><span data-path-to-node="19,1,1,3,0,2">.</span>

### 2. `daily_archiver.py` — *Nightly Batch ETL*

- <span data-path-to-node="21,0,0,0">**Frequenza:** Esecuzione notturna automatizzata (es. ore 00:30)</span><span data-path-to-node="21,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="21,0,0,2">.</span>
- **Logica Operativa:**
    
    
    - <span data-path-to-node="21,1,1,0,0,0">Calcola dinamicamente la data <span class="math-inline" data-index-in-node="30" data-math="T-1">$T-1$</span> (Giorno precedente)</span><span data-path-to-node="21,1,1,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="21,1,1,0,0,2">.</span>
    - <span data-path-to-node="21,1,1,1,0,0">Scansiona in modalità *stream* il log storico filtrando solo le righe pertinenti</span><span data-path-to-node="21,1,1,1,0,1"><sup class="superscript"></sup></span><span data-path-to-node="21,1,1,1,0,2">.</span>
    - <span data-path-to-node="21,1,1,2,0,0">Normalizza le serie numeriche e calcola i pesi totali e parziali secondo il modello</span><span data-path-to-node="21,1,1,2,0,1"><sup class="superscript"></sup></span><span data-path-to-node="21,1,1,2,0,2">: </span>
        
        <div data-path-to-node="21,1,1,2,1"><div class="math-block" data-math="\text{PESO TOT} = \frac{\sum_{i=1}^{16} W_i + \sum_{i=1}^{16} B_i}{10}">$$\text{PESO TOT} = \frac{\sum_{i=1}^{16} W_i + \sum_{i=1}^{16} B_i}{10}$$</div></div><div data-path-to-node="21,1,1,2,2"><div class="math-block" data-math="\text{PESO CANALE B} = \frac{\sum_{i=1}^{16} W_i}{10}, \quad \text{PESO CANALE A} = \frac{\sum_{i=1}^{16} B_i}{10}">$$\text{PESO CANALE B} = \frac{\sum_{i=1}^{16} W_i}{10}, \quad \text{PESO CANALE A} = \frac{\sum_{i=1}^{16} B_i}{10}$$</div></div>
    - <span data-path-to-node="21,1,1,3,0,0">Genera e organizza i file Excel nell'alberatura `/Archivio_Excel/YYYY/MM/WeightReport_YYYY-MM-DD.xlsx`</span><span data-path-to-node="21,1,1,3,0,1"><sup class="superscript"></sup></span><span data-path-to-node="21,1,1,3,0,2">.</span>

### 3. `recovery_cli.py` — *Interactive Historical Extractor*

- <span data-path-to-node="23,0,0,0">**Modalità:** Utility ad uso manuale / interattivo</span><span data-path-to-node="23,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="23,0,0,2">.</span>
- **Logica Operativa:**
    
    
    - <span data-path-to-node="23,1,1,0,0,0">Accetta in input una lista di date arbitrarie (formato `YYYY-MM-DD`)</span><span data-path-to-node="23,1,1,0,0,1"><sup class="superscript"></sup></span><span data-path-to-node="23,1,1,0,0,2">.</span>
    - <span data-path-to-node="23,1,1,1,0,0">Scansiona il log con approccio Low-RAM ed estrae i dati storici rigenerando la struttura reportistica senza caricare il file cumulativo in memoria</span><span data-path-to-node="23,1,1,1,0,1"><sup class="superscript"></sup></span><span data-path-to-node="23,1,1,1,0,2">.</span>