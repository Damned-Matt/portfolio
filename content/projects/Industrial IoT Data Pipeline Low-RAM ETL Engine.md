---
title: "Industrial IoT Data Pipeline & Low-RAM ETL Engine"
date: 2026-07-27
description: "Architettura per l'acquisizione, la persistenza e l'elaborazione ETL di dati da macchinari industriali con approccio Low-RAM."
tags: ["Python", "IoT", "ETL", "Pandas", "Automation", "System Administration"]
draft: false
image: "images/script.jpg"

---


Un'architettura completa per l'acquisizione, la persistenza e l'elaborazione ETL di dati da macchinari di pesatura industriale. Il sistema collega l'infrastruttura operativa (OT) con i sistemi d'analisi IT aziendali, risolvendo le limitazioni di RAM dei dispositivi periferici (*edge*) e garantendo la gestione di file di log di grandi dimensioni con un'impronta di memoria minimale.

---

## Architettura di Sistema

```
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
+---------------------------+     ---------------------------->+-------------------------+
```

---

## Il Problema Aziendale & La Soluzione

### **Contesto e Criticità**
* **Memoria Periferica Limitata:** La scheda di controllo del macchinario industriale accumula i dati di pesatura nella RAM interna. Senza una pulizia periodica, il sistema va in overflow bloccando le registrazioni.
* **Rischio di Perdita Dati:** Network glitch o failure durante lo scarico rischiano di svuotare la RAM del macchinario prima della scrittura sicura su disco aziendale.
* **Gestione Grandi Volumi (RAM Bottleneck):** Il file di log cumulativo accumula centinaia di migliaia di righe. Caricare l'intero file in memoria con tradizionali librerie d'analisi genera crash di sistema sui server di produzione.

### **Soluzione Ingegneristica**
1. **Poller Atomico a 4 Fasi:** Un demone schedulato esegue l'estrazione dati, la deduplica basata su timestamp, la scrittura atomica su disco con *fsync* fisico e, solo ad operazione confermata, l'azzeramento della RAM della macchina.
2. **Streaming Parser (Low-RAM Processing):** Tutti gli script utilizzano una lettura sequenziale *line-by-line* (I/O streaming) isolando solo i record necessari prima di alimentarli al motore d'analisi Pandas, riducendo l'occupazione di memoria da gigabyte a pochi megabyte.
3. **ETL & Feature Engineering:** Conversione automatica delle serie di sensori grezzi ($W_{01..16}$ e $B_{01..16}$) in metriche operative aggregate esportate in file Excel categorizzati per Anno/Mese.

---

## Tech Stack & Requisiti

* **Language:** Python 3.8+
* **Libraries:** `pandas`, `openpyxl`, `requests`, `pathlib`
* **OS / Environment:** Windows Server / Linux Enterprise Host
* **Scheduling:** Windows Task Scheduler / Linux Cron
* **Protocols:** HTTP/REST API (Communication with Edge Hardware)

---

## Struttura dei Moduli

### 1. `poller_daemon.py` — *Ingestion Engine*
* **Frequenza:** Esecuzione ogni 10 minuti via Task Scheduler.
* **Logica Operativa:**
  * Effettua una richiesta `HTTP POST` con azione `download` al macchinario.
  * Verifica il file di stato (`last_timestamp.txt`) per filtrare ed eliminare record duplicati.
  * Scrive le nuove righe sul log di persistenza ed esegue `os.fsync()` per forzare la scrittura fisica su disco evitando l'I/O caching.
  * Invia il comando `POST` `azzera_log` al macchinario per liberare la memoria periferica.

### 2. `daily_archiver.py` — *Nightly Batch ETL*
* **Frequenza:** Esecuzione notturna automatizzata (es. ore 00:30).
* **Logica Operativa:**
  * Calcola dinamicamente la data $T-1$ (giorno precedente).
  * Scansiona in modalità *stream* il log storico filtrando solo le righe pertinenti.
  * Normalizza le serie numeriche e calcola i pesi totali e parziali secondo i modelli:

PESO TOT = ( ∑ᵢ₌₁¹⁶ Wᵢ + ∑ᵢ₌₁¹⁶ Bᵢ ) / 10

PESO CANALE B = ( ∑ᵢ₌₁¹⁶ Wᵢ ) / 10,   PESO CANALE A = ( ∑ᵢ₌₁¹⁶ Bᵢ ) / 10

  * Genera e organizza i file Excel nell'alberatura `/Archivio_Excel/YYYY/MM/WeightReport_YYYY-MM-DD.xlsx`.

### 3. `recovery_cli.py` — *Interactive Historical Extractor*
* **Modalità:** Utility ad uso manuale / interattivo.
* **Logica Operativa:**
  * Accetta in input una lista di date arbitrarie (formato `YYYY-MM-DD`).
  * Scansiona il log con approccio Low-RAM ed estrae i dati storici rigenerando la struttura reportistica senza caricare il file cumulativo in memoria.

---

##  Configurazione & Variabili d'Ambiente

Per l'installazione in ambienti di produzione o di test, configurare i parametri principali all'interno delle variabili di sistema o dei file `.env`:

```python
# Network Endpoint Macchinario Industriale
MACHINE_URL = "http://<INDUSTRIAL_MACHINE_IP>/api/v1/data"
TIMEOUT_SEC = 15

# Percorsi di Sistema & Archiviazione
BASE_DIR = Path(r"C:\IoT_WeightLogger")
STORICO_FILE = BASE_DIR / "storico.log"
STATE_FILE = BASE_DIR / "last_timestamp.txt"
ARCHIVIO_DIR = BASE_DIR / "Archivio_Excel"
MONITOR_FILE = BASE_DIR / "pipeline_execution.log"
```

---

##  Codice Sorgente Sanificato

### Modulo 1: Ingestion Engine (`poller_daemon.py`)

```python
#!/usr/bin/env python3
"""
poller_daemon.py
================
Demone di Ingestion per scarico dati da macchinari industriali.
Esegue persistenza atomica su disco e svuotamento memoria dell'Edge Device.
"""

import os
import sys
import logging
import requests
from pathlib import Path
from typing import Optional, Tuple, List

# --- CONFIGURAZIONE SANIFICATA ---
MACHINE_URL = os.getenv("MACHINE_ENDPOINT", "[http://192.168.1.100/api/index.php](http://192.168.1.100/api/index.php)")
TIMEOUT_SEC = 15

BASE_DIR = Path(os.getenv("DATA_DIR", r"C:\IoT_WeightLogger"))
STORICO_FILE = BASE_DIR / "storico.log"
STATE_FILE = BASE_DIR / "last_timestamp.txt"
MONITOR_FILE = BASE_DIR / "monitor_script.log"

HEADER_LINE = (
    "date\ttipoalarm\terr\tcombiSize\thopperIdx"
    "\terrA\terrB\terrC\terrD"
    + "".join(f"\tW{i:02d}" for i in range(1, 17))
    + "".join(f"\tB{i:02d}" for i in range(1, 17))
)

def setup_logging() -> None:
    BASE_DIR.mkdir(parents=True, exist_ok=True)
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    if logger.hasHandlers():
        logger.handlers.clear()

    file_handler = logging.FileHandler(str(MONITOR_FILE), encoding="utf-8")
    formatter = logging.Formatter("%(asctime)s  %(levelname)-8s  %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)

def load_last_timestamp() -> Optional[str]:
    if STATE_FILE.exists():
        ts = STATE_FILE.read_text(encoding="utf-8").strip()
        return ts if ts else None
    return None

def save_last_timestamp(ts: str) -> None:
    STATE_FILE.write_text(ts, encoding="utf-8")

def download_log() -> str:
    response = requests.post(MACHINE_URL, data={"action": "download"}, timeout=TIMEOUT_SEC)
    response.raise_for_status()
    response.encoding = "utf-8"
    return response.text

def parse_new_lines(raw_text: str, last_ts: Optional[str]) -> Tuple[List[str], Optional[str]]:
    new_lines: List[str] = []
    latest_ts: Optional[str] = last_ts

    for raw_line in raw_text.splitlines():
        line = raw_line.rstrip()
        if not line:
            continue
        parts = line.split("\t")
        if len(parts) < 5:
            continue
        ts = parts[0].strip()
        if not ts or not ts[0].isdigit():
            continue

        if last_ts is None or ts > last_ts:
            new_lines.append(line)
            if latest_ts is None or ts > latest_ts:
                latest_ts = ts

    return new_lines, latest_ts

def append_to_storico(lines: List[str]) -> None:
    BASE_DIR.mkdir(parents=True, exist_ok=True)
    write_header = not STORICO_FILE.exists()

    with open(STORICO_FILE, "a", encoding="utf-8", newline="\n") as f:
        if write_header:
            f.write(HEADER_LINE + "\n")
        for line in lines:
            f.write(line + "\n")
        f.flush()
        os.fsync(f.fileno())  # Scrittura atomica/sincronizzata su disco fisico

def clear_machine_log() -> None:
    response = requests.post(MACHINE_URL, data={"action": "azzera_log"}, timeout=TIMEOUT_SEC)
    response.raise_for_status()

def main() -> None:
    setup_logging()

    try:
        raw_text = download_log()
    except Exception as exc:
        logging.error("FASE 1 FALLITA | Errore download: %s", exc)
        sys.exit(1)

    last_ts = load_last_timestamp()
    new_lines, latest_ts = parse_new_lines(raw_text, last_ts)

    if not new_lines:
        logging.info("Nessun nuovo dato trovato.")
        sys.exit(0)

    try:
        append_to_storico(new_lines)
    except Exception as exc:
        logging.error("FASE 3 FALLITA | Errore scrittura disco: %s", exc)
        sys.exit(1)

    save_last_timestamp(latest_ts)

    cleared = False
    try:
        clear_machine_log()
        cleared = True
    except Exception as exc:
        logging.warning("FASE 4 | Azzeramento RAM periferica fallito: %s", exc)

    logging.info("OK | %d righe salvate | ultimo ts: %s | RAM azzerata: %s", len(new_lines), latest_ts, cleared)

if __name__ == "__main__":
    main()
```

---

### Modulo 2: Daily Archiver (`daily_archiver.py`)

```python
#!/usr/bin/env python3
"""
daily_archiver.py
=================
Modulo ETL notturno: Estrazione Low-RAM dati T-1, calcolo aggregati
e generazione reportistica Excel strutturata per data.
"""

import sys
import io
import logging
import pandas as pd
from datetime import datetime, timedelta
from pathlib import Path

BASE_DIR = Path(r"C:\IoT_WeightLogger")
STORICO_FILE = BASE_DIR / "storico.log"
MONITOR_FILE = BASE_DIR / "monitor_archiviazione.log"
ARCHIVIO_DIR = BASE_DIR / "Archivio_Excel"

def setup_logging() -> logging.Logger:
    logger = logging.getLogger("Archiviazione")
    logger.setLevel(logging.INFO)
    if logger.hasHandlers():
        logger.handlers.clear()
    
    file_handler = logging.FileHandler(str(MONITOR_FILE), encoding="utf-8")
    formatter = logging.Formatter("%(asctime)s  %(levelname)-8s  %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)
    return logger

def main():
    log = setup_logging()
    ieri = datetime.now() - timedelta(days=1)
    data_target_str = ieri.strftime("%Y-%m-%d")
    
    log.info(f"Avvio ETL Low-RAM per la data: {data_target_str}")
    
    if not STORICO_FILE.exists():
        log.error("File di log storico non trovato. Interruzione.")
        sys.exit(1)

    try:
        matching_lines = []
        # Lettura in streaming riga per riga (Low-RAM Footprint)
        with open(STORICO_FILE, "r", encoding="utf-8") as f:
            header_line = f.readline()
            for line in f:
                if line.startswith(data_target_str):
                    matching_lines.append(line)
        
        if not matching_lines:
            log.info(f"Nessun dato registrato per il {data_target_str}.")
            sys.exit(0)

        dati_filtrati = header_line + "".join(matching_lines)
        df_ieri = pd.read_csv(io.StringIO(dati_filtrati), sep='\t')
        df_ieri['date'] = pd.to_datetime(df_ieri['date'], errors='coerce')

        # Feature Engineering: Calcoli su canali sensori W e B
        colonne_w = [f'W{i:02d}' for i in range(1, 17)]
        colonne_b = [f'B{i:02d}' for i in range(1, 17)]

        df_ieri[colonne_w] = df_ieri[colonne_w].apply(pd.to_numeric, errors='coerce').fillna(0)
        df_ieri[colonne_b] = df_ieri[colonne_b].apply(pd.to_numeric, errors='coerce').fillna(0)

        # Calcolo dei pesi aggregati
        df_ieri['PESO SCARICO TOT'] = df_ieri[colonne_w + colonne_b].sum(axis=1) / 10
        df_ieri['PESO SCARICO B'] = df_ieri[colonne_w].sum(axis=1) / 10
        df_ieri['PESO SCARICO A'] = df_ieri[colonne_b].sum(axis=1) / 10

        df_ieri['date'] = df_ieri['date'].dt.strftime("%Y-%m-%d %H:%M:%S")

        # Organizzazione Output /YYYY/MM/
        percorso_out = ARCHIVIO_DIR / str(ieri.year) / f"{ieri.month:02d}"
        percorso_out.mkdir(parents=True, exist_ok=True)
        
        file_excel = percorso_out / f"WeightReport_{data_target_str}.xlsx"
        df_ieri.to_excel(file_excel, index=False, engine='openpyxl')
        
        log.info(f"OK | Generato report Excel: {file_excel.name} ({len(df_ieri)} righe elaborate).")

    except Exception as exc:
        log.error(f"Errore critico durante l'elaborazione ETL: {exc}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

### Modulo 3: Recovery CLI (`recovery_cli.py`)

```python
#!/usr/bin/env python3
"""
recovery_cli.py
===============
Utility CLI interattiva per estrazione e recupero storico dati Low-RAM
su range di date arbitrarie.
"""

import sys
import io
import pandas as pd
from datetime import datetime
from pathlib import Path

STORICO_FILE = Path(r"C:\IoT_WeightLogger\storico.log")
ARCHIVIO_DIR = Path(r"C:\IoT_WeightLogger\Archivio_Excel")

if not STORICO_FILE.exists():
    print(f"Errore: Il file {STORICO_FILE} non esiste!")
    sys.exit(1)

print("--- RECUPERO PESATE STORICHE (Low-RAM Engine) ---")
input_utente = input("Inserisci le date da recuperare (YYYY-MM-DD) separate da virgola: ")

giorni_da_recuperare = [data.strip() for data in input_utente.split(",") if data.strip()]

if not giorni_da_recuperare:
    print("Nessuna data valida inserita. Interruzione.")
    sys.exit(0)

for data_target_str in giorni_da_recuperare:
    print(f"Elaborazione: {data_target_str}...")
    matching_lines = []
    
    try:
        # Stream lettura file
        with open(STORICO_FILE, "r", encoding="utf-8") as f:
            header_line = f.readline()
            for line in f:
                if line.startswith(data_target_str):
                    matching_lines.append(line)
    except Exception as e:
         print(f"-> ERRORE lettura file: {e}")
         continue
                
    if not matching_lines:
        print(f"-> Nessun record trovato per il {data_target_str}. Salto.")
        continue
        
    try:
        dati_filtrati = header_line + "".join(matching_lines)
        df_data = pd.read_csv(io.StringIO(dati_filtrati), sep='\t')
        df_data['date'] = pd.to_datetime(df_data['date'], errors='coerce')
        
        colonne_w = [f'W{i:02d}' for i in range(1, 17)]
        colonne_b = [f'B{i:02d}' for i in range(1, 17)]
        
        df_data[colonne_w] = df_data[colonne_w].apply(pd.to_numeric, errors='coerce').fillna(0)
        df_data[colonne_b] = df_data[colonne_b].apply(pd.to_numeric, errors='coerce').fillna(0)
        
        df_data['PESO SCARICO TOT'] = df_data[colonne_w + colonne_b].sum(axis=1) / 10
        df_data['PESO SCARICO B'] = df_data[colonne_w].sum(axis=1) / 10
        df_data['PESO SCARICO A'] = df_data[colonne_b].sum(axis=1) / 10
        
        df_data['date'] = df_data['date'].dt.strftime("%Y-%m-%d %H:%M:%S")
        
        dt_oggetto = datetime.strptime(data_target_str, "%Y-%m-%d")
        percorso_out = ARCHIVIO_DIR / str(dt_oggetto.year) / f"{dt_oggetto.month:02d}"
        percorso_out.mkdir(parents=True, exist_ok=True)
        
        file_excel = percorso_out / f"WeightReport_{data_target_str}.xlsx"
        df_data.to_excel(file_excel, index=False, engine='openpyxl')
        print(f"-> COMPLETATO: Generato {file_excel.name} ({len(df_data)} righe).")
        
    except Exception as e:
        print(f"-> ERRORE durante l'elaborazione del {data_target_str}: {e}")

print("\n--- PROCEDURA COMPLETATA ---")
```