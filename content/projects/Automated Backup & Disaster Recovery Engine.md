---
title: "Automated Backup Infrastructure: Defensive Local Pruning & Dockerized NAS Sync"
date: 2026-07-28
description: "Infrastruttura completa per la gestione e l'automazione dei backup: include uno script Bash difensivo per la rotazione locale con Sanity Check e un container Docker per la sincronizzazione remota automatizzata su NAS via SFTP."
tags: ["Bash", "Docker", "Linux", "Backup", "Automation", "SysAdmin", "DevOps", "NAS", "SFTP"]
draft: false
image: "images/script.jpg"
---
**Panoramica del Progetto**
Questo progetto illustra un'infrastruttura completa e automatizzata per la gestione, la rotazione e l'archiviazione sicura dei backup di server web. L'architettura è suddivisa in due moduli principali:
1. **Script di Rotazione Locale (Server Sorgente):** Uno script bash che si assicura di mantenere solo i backup recenti sul disco locale del server web, prevenendo la saturazione dello spazio tramite un rigoroso "Sanity Check" per evitare perdite di dati.
2. **Container di Sincronizzazione (Infrastruttura di Rete):** Un container Docker isolato che effettua il pulling automatizzato dei backup dal server web e li archivia direttamente su uno storage di rete locale (NAS) aziendale.

---

## MODULO 1: Rotazione e Sicurezza dei Backup Locali

Lo script bash esegue una rotazione automatizzata dei backup locali presenti sul server. Implementa un controllo di sicurezza (Sanity Check) basato sulla data stampata nel nome delle cartelle per verificare la presenza di un backup recente prima di procedere con l'eliminazione delle directory obsolete, prevenendo la perdita accidentale in caso di fallimento dei job di backup notturni.

### Logica di Funzionamento
*   **Step 0 - Controlli di sicurezza base:** Verifica che la variabile della directory non sia vuota o punti alla radice di sistema (`/`).
*   **Step 1 - Sanity Check sulla nomenclatura:** Calcola le date di oggi e ieri e verifica tramite globbing se esiste almeno una cartella recente. In caso negativo, blocca la cancellazione.
*   **Step 2 - Pruning e Validazione:** Scansiona le directory, applica un controllo regex sul nome (es. `MM.DD.YYYY_HH-MM-SS`) ed elimina tramite `rm -rf` solo i backup più vecchi dei giorni di ritenzione impostati (es. 4 giorni).

### Codice Bash (Rotazione Locale)

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/var/backups/web"
RETENTION_DAYS=4

log() {
    echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')] $*"
}

# 0. CONTROLLI DI SICUREZZA ASSOLUTA
if [[ -z "$BACKUP_DIR" || "$BACKUP_DIR" == "/" ]]; then
    log "ERRORE FATALE: BACKUP_DIR vuoto o impostato su root (/). Mi rifiuto di continuare." >&2
    exit 1
fi

if [[ ! -d "$BACKUP_DIR" ]]; then
    log "ERRORE FATALE: La directory $BACKUP_DIR non esiste." >&2
    exit 1
fi

# 1. SANITY CHECK SUL NOME
TODAY=$(date +%m.%d.%Y)
YESTERDAY=$(date -d "yesterday" +%m.%d.%Y)
RECENT_EXISTS=false

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

# 2. GENERAZIONE REGEX DI RETENZIONE DAI NOMI
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

# 3. PRUNING MANUALE BASATO SUI NOMI DELLE DIRECTORY
for DIR_PATH in "$BACKUP_DIR"/*; do
    [[ -e "$DIR_PATH" ]] || continue
    if [[ ! -d "$DIR_PATH" ]]; then
        continue
    fi

    DIR_NAME=$(basename "$DIR_PATH")
    if [[ ! "$DIR_NAME" =~ ^[0-9]{2}\.[0-9]{2}\.[0-9]{4} ]]; then
        log "IGNORO: '$DIR_NAME' non rispetta la convenzione di nome backup."
        continue
    fi

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

## MODULO 2: Sincronizzazione Automatizzata su NAS via Docker

Questo componente automatizza lo scaricamento dei backup dal server sorgente e la successiva archiviazione diretta su un NAS locale, utilizzando un container Docker isolato per garantire sicurezza e portabilità.

### Flusso dell'Infrastruttura
1.  **Autenticazione Asimmetrica:** La VM Docker utilizza una chiave privata SSH priva di passphrase per connettersi in modo sicuro e non presidiato al server web.
2.  **Montaggio del Volume (CIFS):** Docker monta una cartella condivisa del NAS tramite protocollo SMB, mappandola all'interno del container (`/mnt/nas_storage`).
3.  **Sincronizzazione Dinamica (LFTP):** Tramite uno script bash in cronjob, `lftp` interroga la directory remota, identifica automaticamente la cartella di backup più recente ed esegue un `mirror` ricorsivo dei file direttamente sul volume del NAS.

### Script di Trasferimento (`backup_sync.sh`)

```bash
#!/bin/sh
echo "[$(date)] Avvio ricerca della cartella di backup più recente via SFTP..."

LFTP_CONN="set sftp:connect-program 'ssh -a -x -i /root/ssh_key -o StrictHostKeyChecking=no'; open sftp://<IP_SERVER_WEB>; user sftp_user"

# Individua la cartella più recente escludendo percorsi non conformi
RECENT_DIR=$(lftp -c "$LFTP_CONN; cls -1 -t /var/backups/web/" | grep -E '^[0-9]{2}\.[0-9]{2}\.[0-9]{4}' | head -n 1)

if [ -z "$RECENT_DIR" ]; then
    echo "[$(date)] ERRORE: Nessuna cartella di backup valida trovata in /var/backups/web/"
    exit 1
fi

RECENT_DIR=$(basename "$RECENT_DIR")
echo "[$(date)] La cartella di backup individuata è: $RECENT_DIR"
echo "[$(date)] Avvio il download ricorsivo della cartella..."

# Sincronizzazione diretta verso il volume montato sul NAS
lftp -c "$LFTP_CONN; mirror /var/backups/web/$RECENT_DIR /mnt/nas_storage/$RECENT_DIR; quit"

echo "[$(date)] Sincronizzazione completata con successo."
```

### Configurazione Docker (`docker-compose.yml`)

```yaml
services:
  sftp-cron:
    image: alpine:latest
    container_name: sync_web_nas
    volumes:
      - ./backup_sync.sh:/backup_sync.sh:ro
      - ./crontab:/etc/crontabs/root:ro
      - ./ssh_key:/root/ssh_key:ro
      - nas_volume:/mnt/nas_storage
    command: >
      sh -c "apk update && apk add --no-cache lftp openssh-client tzdata ca-certificates &&
      cp /usr/share/zoneinfo/Europe/Rome /etc/localtime &&
      crond -f -l 2"
    restart: unless-stopped

volumes:
  nas_volume:
    driver: local
    driver_opts:
      type: cifs
      o: "username=nas_user,password=${PASSWORD_NAS},vers=3.0"
      device: "//<IP_NAS_STORAGE>/Shares/Backup_Web"
```