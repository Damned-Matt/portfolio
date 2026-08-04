---
title: "Automated Local Backup Rotation & Defensive Pruning Engine"
date: 2026-07-28
description: "Script Bash difensivo per la rotazione automatizzata dei backup locali con Sanity Check sulla nomenclatura delle directory e prevenzione della perdita dati."
tags: ["Bash", "Linux", "Backup", "Automation", "SysAdmin", "DevOps"]
draft: false
image: "images/script.jpg"
---

Script Bash professionale progettato per l'amministrazione di sistemi Linux e la gestione automatizzata della rotazione dei backup locali generati da procedure schedulate o pannelli gestionali[cite: 2]. 

A differenza degli script di pruning tradizionali che eliminano i file basandosi solo sull'anzianità di modifica, questo motore implementa una logica di **Scripting Difensivo (Sanity Check)**[cite: 2]: verifica la reale presenza di un backup recente (oggi o ieri) *prima* di procedere con l'eliminazione delle directory obsolete, azzerando il rischio di cancellazione totale in caso di fallimento del job notturno[cite: 2].

---

## 🏗️ Architettura & Flusso Operativo

```
+-----------------------------------------------------------------------+
| Automated Nightly Backup Job (04:00 AM)                               |
| Genera directory target: /backup/local/MM.DD.YYYY_HH-MM-SS            |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| STEP 0: Absolute Safety Checks                                        |
| Verifica validità BACKUP_DIR (No Root, No Empty Path, Directory Exist)|
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| STEP 1: Naming Sanity Check (FAIL-SAFE)                               |
| Cerca la presenza di backup recenti ($TODAY_ / $YESTERDAY_)           |
| [!] Se NON trova backup recenti -> BLOCCO IMMEDIATO (Exit Code 1)    |
+-----------------------------------------------------------------------+
                                   |
                                   v (Superato)
+-----------------------------------------------------------------------+
| STEP 2: Dynamic Retention Regex Generation                            |
| Calcola la regex delle date valide per gli ultimi N giorni (e.g. 4d)  |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| STEP 3: Pattern Validation & Pruning                                  |
| Scansiona /backup/local/*:                                            |
| - Ignora cartelle con nomenclatura non conforme                       |
| - Mantiene le directory incluse nella finestra di ritenzione           |
| - Esegue 'rm -rf' esclusivamente sulle cartelle scadute               |
+-----------------------------------------------------------------------+
```

---

## 🛡️ Il Problema Aziendale & La Soluzione

### **Contesto e Criticità**
* **Rischio di Cancellazione a Cieca:** I classici comandi di pulizia (es. `find -mtime +N -delete`) eliminano i file vecchi a prescindere. Se il processo di backup notturno si blocca per mancanza di spazio o errori di sistema, uno script tradizionale continuerà a eliminare i vecchi backup fino a lasciare il server privo di punti di ripristino (*Zero-Backup State*).
* **Nomenclatura Non Standard:** In ambienti condivisi possono essere presenti directory temporanee o file che non devono essere toccati dal processo di rotazione.

### **Soluzione Ingegneristica**
1. **Sanity Check Preventivo:** Lo script controlla che esista almeno una directory di backup creata nella data odierna (`TODAY`) o in quella di ieri (`YESTERDAY`)[cite: 2]. Se non la trova, interrompe l'esecuzione e invia un errore fatale nei log[cite: 2].
2. **Regex Matching Dinamico:** La finestra di retention (es. 4 giorni) viene tradotta dinamicamente in una regola di espressione regolare generata al volo per convalidare le date del formato `MM.DD.YYYY`[cite: 2].
3. **Validazione del Formato Directory:** Tutte le cartelle che non rispettano la convenzione `MM.DD.YYYY_HH-MM-SS` vengono marcate come `IGNORATO` e saltate, proteggendo file di configurazione o cartelle di sistema presenti nello stesso percorso[cite: 2].

---

## ⚙️ Tech Stack & Requisiti

* **Language / Shell:** Bash (POSIX Compliant, `set -euo pipefail`)[cite: 2]
* **Core Utilities:** `date`, `find`, `rm`, `basename`[cite: 2]
* **Target Directory:** `/backup/local`[cite: 2]
* **Retention Default:** 4 Giorni (`RETENTION_DAYS=4`)[cite: 2]
* **Logging System:** Log dettagliato su `/var/log/backup_rotation.log`[cite: 2]
* **Scheduling:** Cron (`0 4 * * *`)[cite: 2]

---

## 📜 Codice Sorgente Completo (`backup_rotation.sh`)

```bash
#!/usr/bin/env bash
# ==============================================================================
# AUTOMATED LOCAL BACKUP ROTATION & SANITY CHECK ENGINE
# ==============================================================================
# Descrizione: Script difensivo per la rotazione sicura dei backup locali.
# Autore: Matteo Pipola (System Administrator)
# ==============================================================================

set -euo pipefail

# --- CONFIGURAZIONE TARGET E RITENZIONE ---
BACKUP_DIR="/backup/local"
RETENTION_DAYS=4

log() {
    echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')] $*"
}

# ------------------------------------------------------------------------------
# 0. CONTROLLI DI SICUREZZA ASSOLUTA
# ------------------------------------------------------------------------------
if [[ -z "$BACKUP_DIR" || "$BACKUP_DIR" == "/" ]]; then
    log "ERRORE FATALE: BACKUP_DIR vuoto o impostato su root (/). Mi rifiuto di continuare." >&2
    exit 1
fi

if [[ ! -d "$BACKUP_DIR" ]]; then
    log "ERRORE FATALE: La directory $BACKUP_DIR non esiste." >&2
    exit 1
fi

# ------------------------------------------------------------------------------
# 1. SANITY CHECK SUL NOME (VERIFICA BACKUP RECENTE)
# ------------------------------------------------------------------------------
TODAY=$(date +%m.%d.%Y)
YESTERDAY=$(date -d "yesterday" +%m.%d.%Y)
RECENT_EXISTS=false

# Controllo presenza di cartelle recenti tramite globbing
for dir in "$BACKUP_DIR"/${TODAY}_* "$BACKUP_DIR"/${YESTERDAY}_*; do
    if [[ -d "$dir" ]]; then
        RECENT_EXISTS=true
        break
    fi
done

if [[ "$RECENT_EXISTS" == "false" ]]; then
    log "CRITICO: Nessuna cartella di backup trovata per oggi ($TODAY) o ieri ($YESTERDAY). Rotazione bloccata." >&2
    exit 1
fi

log "Sanity check superato (trovata cartella recente)."

# ------------------------------------------------------------------------------
# 2. GENERAZIONE REGEX DI RETENZIONE DAI NOMI
# ------------------------------------------------------------------------------
VALID_DATES_REGEX="^("
for i in $(seq 0 $((RETENTION_DAYS - 1))); do
    DATE_STR=$(date -d "$i days ago" +%m\.%d\.%Y)
    if [[ $i -eq 0 ]]; then
        VALID_DATES_REGEX+="$DATE_STR"
    else
        VALID_DATES_REGEX+="|$DATE_STR"
    fi
done
VALID_DATES_REGEX+=")"

# ------------------------------------------------------------------------------
# 3. PRUNING MANUALE BASATO SUI NOMI DELLE DIRECTORY
# ------------------------------------------------------------------------------
for DIR_PATH in "$BACKUP_DIR"/*; do
    [[ -e "$DIR_PATH" ]] || continue
    if [[ ! -d "$DIR_PATH" ]]; then
        continue
    fi

    DIR_NAME=$(basename "$DIR_PATH")

    # Validazione formato: Ignora cartelle che non iniziano con MM.DD.YYYY
    if [[ ! "$DIR_NAME" =~ ^[0-9]{2}\.[0-9]{2}\.[0-9]{4} ]]; then
        log "IGNORO: '$DIR_NAME' non rispetta la convenzione di nome backup."
        continue
    fi

    # Verifica ritenzione tramite Regex
    if [[ "$DIR_NAME" =~ $VALID_DATES_REGEX ]]; then
        log "MANTENGO: $DIR_NAME"
    else
        log "ELIMINO: $DIR_NAME"
        rm -rf "$DIR_PATH"
    fi
done

log "Processo di rotazione terminato."
```

---

## ⏱️ Schedulazione Crontab e Consultazione Log

### Schedulazione Automatica (Root Cron)
Lo script è configurato per andare in esecuzione ogni notte alle **04:00 AM**, subito dopo la conclusione del job di backup di sistema[cite: 2]:

```bash
0 4 * * * /backup/local/backup_rotation.sh >> /var/log/backup_rotation.log 2>&1
```

### Consultazione e Audit dei Log
Tutte le operazioni (Sanity Check, directory mantenute, cartelle eliminate o ignorate) vengono tracciate su `/var/log/backup_rotation.log`[cite: 2].

Per verificare l'esito dell'ultima rotazione da terminale[cite: 2]:
```bash
tail -n 50 /var/log/backup_rotation.log
```