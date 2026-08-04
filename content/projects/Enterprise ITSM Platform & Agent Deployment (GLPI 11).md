---
title: "Enterprise ITSM Platform & Agent Deployment (GLPI 11)"
date: 2026-07-27
description: "Implementazione end-to-end di GLPI 11 su stack LAMP: hardening di sicurezza web, integrazione Mail-to-Ticket e predisposizione al deployment via GPO."
tags: ["SysAdmin", "GLPI", "ITSM", "Linux", "Ubuntu", "Apache", "Active Directory", "Automation"]
draft: false
image: "/images/glpi.jpg"
---

**Descrizione:** Implementazione *end-to-end* di una piattaforma di IT Service Management (GLPI 11) su architettura LAMP in ambiente virtualizzato VMware vCenter. Il progetto include il setup dell'infrastruttura, l'hardening di sicurezza delle directory web, l'integrazione del sistema Mail-to-Ticket per l'helpdesk e la predisposizione al deployment automatizzato dell'Agent sui client aziendali.

**Tech Stack:** `Ubuntu Server`, `Apache2`, `MariaDB`, `PHP 8`, `GLPI 11`, `Active Directory`, `Windows Server GPO`.

---

## 1. Infrastruttura e LAMP Stack
Provisioning della VM su vCenter (2 vCPU, 4GB RAM) e configurazione dell'ambiente LAMP base.

### Configurazione Database e Servizi Core
Installazione e hardening di MariaDB tramite `mysql_secure_installation` con creazione del database dedicato e gestione dei privilegi:

```sql
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'StrongPasswordHere!';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
```

Popolamento delle tabelle per il supporto Timezone in MariaDB:
```bash
mysql -u root -p mysql < /usr/share/mysql/mysql_test_data_timezone.sql
```

Installazione dei moduli PHP richiesti dal motore GLPI e per le integrazioni di rete:
```bash
sudo apt install php php-cli php-mysqli php-curl php-gd php-intl php-mbstring php-xml php-zip php-bz2 php-apcu php-ldap php-imap php-common libapache2-mod-php -y
```

---

## 2. Web Hardening & Security Best Practices
Per prevenire l'esposizione diretta sul web di file sensibili, configurazioni e allegati dei ticket, l'albero delle directory è stato separato spostando i componenti critici all'esterno della *Document Root* di Apache.

```bash
# Isolamento directory riservate fuori dal web root
sudo mkdir -p /etc/glpi /var/lib/glpi /var/log/glpi
sudo mv /var/www/glpi/config /etc/glpi/
sudo mv /var/www/glpi/files /var/lib/glpi/
```

Configurazione dei path personalizzati tramite file PHP d'ambiente:

**`/var/www/glpi/inc/downstream.php`**
```php
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi/');
if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
    require_once GLPI_CONFIG_DIR . '/local_define.php';
}
```

**`/etc/glpi/local_define.php`**
```php
<?php
define('GLPI_VAR_DIR', '/var/lib/glpi');
define('GLPI_LOG_DIR', '/var/log/glpi');
```

**VirtualHost Apache (`glpi.conf`)**
```apache
<VirtualHost *:80>
    ServerName glpi.tuodominio.local
    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        Require all granted
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/glpi_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_access.log combined
</VirtualHost>
```

Schedulazione dell'azione di sistema per le cronjob di background (invio notifiche, pulizia cache, mailgate):
```bash
*/2 * * * * /usr/bin/php /var/www/glpi/front/cron.php &>/dev/null
```

---

## 3. Automazione Helpdesk: Mail-to-Ticket Integration
Configurazione del ricevitore email per l'apertura automatica dei ticket da parte degli utenti finali via posta elettronica, eliminando la necessità di accesso manuale al portale per le richieste standard.

* **Protocollo & Integrazione:** Sfrutta il modulo `php-imap` per interfacciarsi con il mail server aziendale tramite connessione protetta IMAP su porta 993.
* **Orchestrazione Automatic Fetching:** Il recupero dei messaggi è affidato al motore di cronjob `mailgate` eseguito ogni 2 minuti dal server Linux.
* **Regole di Ingestion:** Mappatura automatica dell'indirizzo del mittente con l'anagrafica utenti GLPI, con categorizzazione automatica del ticket in base all'oggetto dell'email.

---

## 4. Automated Client Deployment (GPO & Scripting)
Per la distribuzione silente e massiva del client d'inventario (GLPI Agent) sulle workstation aziendali, è stato sviluppato un framework dedicato basato su Active Directory GPO, Scheduled Tasks e script di controllo con meccanismo a flag per evitare esecuzioni ridondanti.

👉 **[Consulta il Framework di Deployment GPO per GLPI Agent](https://damned-matt.github.io/portfolio/projects/automated-sodtware-deployment-via-gpo--powershell/)**