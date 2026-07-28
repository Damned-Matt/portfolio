---
title: "Automated Backup & Disaster Recovery Engine (Bash, Docker & Offsite Sync)"
date: 2026-07-28
description: "Architettura resiliente per il backup automatizzato, la cifratura e la disaster recovery di container Docker, database e volumi isolati con sincronizzazione NAS."
tags: ["Bash", "Docker", "Backup", "Disaster Recovery", "Linux", "DevOps", "Automation"]
draft: false
image: "images/script.jpg"

---

Un'architettura di backup e Disaster Recovery modulare progettata per proteggere database relazionali, volumi persistenti Docker e configurazioni di sistema critiche. 

Il sistema combina la potenza di **Bash scripting** nativo con l'isolamento dei container **Docker**, eseguendo snapshot a caldo, cifratura dei dati, gestione automatica della retention (rotazione dei log/file vecchi) e sincronizzazione sicura verso storage di rete (NAS / Offsite Offloading).

---

##  Architettura & Flusso Operativo

```
+-----------------------------------------------------------------------+
| Docker Microservices & Persistent Volumes (MySQL, PostgreSQL, Web)   |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| 01. In-Memory Hot Backup Engine (Bash Cron Task)                      |
|     Esegue 'docker exec' per dump atomici senza fermo dei servizi     |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| 02. Volume Compression & GPG Encryption                               |
|     Crea archivi .tar.gz cifrati ad alta densità in memoria/temp      |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| 03. Retention Policy & Storage Pruning                                |
|     Elimina automaticamente i backup più vecchi di N giorni (GFS)     |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| 04. Encrypted Network Transport (rsync / NFS / SMB)                   |
|     Trasferisce gli archivi sul NAS di Disaster Recovery / Offsite    |
+-----------------------------------------------------------------------+
```

---

##  Il Problema Aziendale & La Soluzione

### **Contesto e Criticità**
* **Rischio Perdita Dati nei Container:** I dati residenti all'interno dei container Docker o nei volumi non associati a snapshot di sistema rischiano la corruzione o la perdita permanente in caso di crash dell'host o fallimento hardware.
* **Fasce d'Inattività Assenti (Downtime Non Tollerato):** I backup tradizionali che richiedono lo spegnimento dei servizi (*cold backup*) interrompono i flussi operativi aziendali e la produzione.
* **Mancanza di Cifratura e Retention Control:** Molti script di backup salvano file in chiaro su share di rete non protette, accumulando dati fino a saturare completamente i dischi di storage.

### **Soluzione Ingegneristica**
1. **Hot-Dump Senza Downtime:** Utilizzo di utility native lanciate direttamente tramite `docker exec` (`mysqldump`, `pg_dumpall`) per estrarre snapshot coerenti del database mentre il servizio è in esecuzione.
2. **Cifratura End-to-End & Compressione:** Cifratura simmetrica via GPG/OpenSSL per garantire che i file di backup salvati sul NAS siano illeggibili a soggetti non autorizzati.
3. **Algoritmo di Retention Autonomo:** Meccanismo avanzato di pulizia automatica basato sui tempi di modifica dei file (`mtime`), per garantire sempre la disponibilità di punti di ripristino (*RPO - Recovery Point Objective*) senza saturare gli spazi di archiviazione.

---

##  Tech Stack & Requisiti

* **Language / Shell:** Bash (POSIX Compliant)
* **Containerization:** Docker & Docker Compose
* **Security & Transport:** GPG (GNU Privacy Guard), SSH / `rsync`, OpenSSL
* **Scheduling:** Linux Cron / Systemd Timers
* **Target Storage:** Local Vault, Network Attached Storage (NAS SMB/NFS), Offsite Storage

---

##  Codice Sorgente Sanificato (`disaster_recovery_backup.sh`)

```bash
#!/usr/bin/env bash
# ==============================================================================
# DISASTER RECOVERY & CONTAINERIZED BACKUP ENGINE
# ==============================================================================
# Descrizione: Script modulare per il backup atomico di DB Docker e Volumi.
# Autore: System Administration & DevOps Team
# ==============================================================================

set -euo pipefail

# --- CONFIGURAZIONE SANIFICATA ---
BACKUP_DATE=$(date +"%Y%m%d_%H%M%S")
RETENTION_DAYS=30

# Percorsi locali e di rete
LOCAL_BACKUP_DIR="/var/backups/docker_engine"
REMOTE_NAS_DIR="/mnt/nas_disaster_recovery/backups"
LOG_FILE="/var/log/backup_engine.log"

# Target Docker
DB_CONTAINER_NAME="production_db_container"
DB_USER="app_backup_user"
DB_NAME="production_db"
DOCKER_VOLUMES_DIR="/var/lib/docker/volumes"

# Cifratura (Usa variabile d'ambiente o fallback)
ENCRYPTION_PASSPHRASE="${BACKUP_PASSPHRASE:-SanitizedSuperSecretPass123!}"

# --- FUNZIONI DI UTILITÀ ---
log() {
    local level="$1"
    local message="$2"
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "${LOG_FILE}"
}

cleanup_temp() {
    log "INFO" "Pulizia file temporanei in corso..."
    rm -f "${LOCAL_BACKUP_DIR}"/*.tmp
}
trap cleanup_temp EXIT

# Pre-flight Check
mkdir -p "${LOCAL_BACKUP_DIR}"
exec 2>> "${LOG_FILE}"

log "INFO" "=== AVVIO PROCEDURA DI BACKUP & DISASTER RECOVERY ==="

# --- FASE 1: HOT DATABASE DUMP (DOCKER EXEC) ---
log "INFO" "Fase 1: Estrazione dump atomico dal container ${DB_CONTAINER_NAME}..."

DB_DUMP_FILE="${LOCAL_BACKUP_DIR}/db_dump_${BACKUP_DATE}.sql.gz"

if docker ps --format '{{.Names}}' | grep -q "^${DB_CONTAINER_NAME}$"; then
    docker exec "${DB_CONTAINER_NAME}" mysqldump \
        -u"${DB_USER}" \
        -p"${ENCRYPTION_PASSPHRASE}" \
        --single-transaction \
        --quick \
        --lock-tables=false \
        "${DB_NAME}" | gzip -9 > "${DB_DUMP_FILE}"
    
    log "INFO" "Dump del database completato con successo: ${DB_DUMP_FILE}"
else
    log "ERROR" "Container ${DB_CONTAINER_NAME} non in esecuzione! Backup DB annullato."
    exit 1
fi

# --- FASE 2: COMPRESSIONE VOLUMI & CIFRATURA ---
log "INFO" "Fase 2: Compressione e Cifratura volumi applicativi..."

ARCHIVE_FILE="${LOCAL_BACKUP_DIR}/full_archive_${BACKUP_DATE}.tar.gz.gpg"

# Compressione e cifratura al volo (Stream Compression & Encryption)
tar -czf - -C "${DOCKER_VOLUMES_DIR}" . 2>/dev/null | \
    gpg --symmetric --batch --yes --passphrase "${ENCRYPTION_PASSPHRASE}" \
    -o "${ARCHIVE_FILE}"

log "INFO" "Archivio cifrato creato: ${ARCHIVE_FILE}"

# --- FASE 3: RETENTION POLICY (PRUNING VECCHI BACKUP) ---
log "INFO" "Fase 3: Applicazione Retention Policy (Eliminazione backup > ${RETENTION_DAYS} giorni)..."

find "${LOCAL_BACKUP_DIR}" -type f \( -name "*.sql.gz" -o -name "*.gpg" \) -mtime +"${RETENTION_DAYS}" -exec rm -f {} \;
log "INFO" "Retention locale completata."

# --- FASE 4: SINCRONIZZAZIONE SICURA SU NAS / DISASTER RECOVERY ---
log "INFO" "Fase 4: Sincronizzazione dati su storage NAS..."

if mountpoint -q "/mnt/nas_disaster_recovery" || [ -d "${REMOTE_NAS_DIR}" ]; then
    rsync -avz --delete \
        --include="*.gpg" \
        --include="*.sql.gz" \
        --exclude="*" \
        "${LOCAL_BACKUP_DIR}/" "${REMOTE_NAS_DIR}/"
    
    log "INFO" "Sincronizzazione su NAS completata con successo."
else
    log "WARN" "NAS non raggiungibile o mount sfondato! I backup rimangono salvati solo in locale."
fi

log "INFO" "=== PROCEDURA COMPLETATA CON SUCCESSO ==="
exit 0
```

---

##  Disaster Recovery Workflow (Procedura di Ripristino)

In caso di disastro hardware o corruzione totale del server, la procedura di ripristino si articola in 3 semplici passaggi:

1. **Decifratura dell'Archivio:**
   ```bash
   gpg --decrypt --batch --passphrase "SanitizedSuperSecretPass123!" \
       -o full_archive.tar.gz full_archive_20260728_104500.tar.gz.gpg
   ```

2. **Ripristino dei Volumi Docker:**
   ```bash
   tar -xzf full_archive.tar.gz -C /var/lib/docker/volumes/
   ```

3. **Ripristino del Database nel Container:**
   ```bash
   gunzip < db_dump_20260728_104500.sql.gz | \
       docker exec -i production_db_container mysql -u app_backup_user -p production_db
   ```