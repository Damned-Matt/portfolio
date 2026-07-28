---
title: "Automated Enterprise Software Deployment via GPO & PowerShell"
date: 2026-07-28
description: "Architettura Idempotente per la distribuzione automatizzata e 'zero-touch' di software ed agenti aziendali su centinaia di endpoint Windows via Active Directory."
tags: ["PowerShell", "Active Directory", "GPO", "Automation", "Windows Server", "DevOps"]
draft: false
---

# 🚀 Enterprise Zero-Touch Deployment Engine (GPO + Idempotent PowerShell)

Un framework di distribuzione software automatizzato progettato per installare agenti di gestione, software aziendali e patch di sicurezza su centinaia di endpoint Windows all'interno di un dominio Active Directory, eliminando la necessità di interventi manuali sul campo.

Il sistema adotta il principio dell'**Idempotenza**, garantendo che lo script d'installazione venga eseguito solo quando necessario, azzerando il traffico di rete e i tempi di boot dei client già configurati.

---

## 📐 Architettura & Flusso Operativo

```
+-----------------------------------------------------------------------+
| Active Directory Group Policy Object (Computer Startup Script)        |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| 01. State Verification (Registry / Flag Check)                        |
|     Controlla se il software/agente è già presente nel registro HKLM   |
+-----------------------------------------------------------------------+
                |                                      |
       [Già Installato]                          [Non Trovato]
                |                                      |
                v                                      v
+-------------------------------+      +--------------------------------+
| EXIT 0 (Zero Network Overhead)|      | 02. Repository Mount & Pull    |
| Il boot prosegue istantaneamente     |     Preleva l'installer da SYSVOL|
+-------------------------------+      +--------------------------------+
                                                       |
                                                       v
                                       +--------------------------------+
                                       | 03. Silent Background Install  |
                                       |     Esegue msiexec /qn /exe    |
                                       +--------------------------------+
                                                       |
                                                       v
                                       +--------------------------------+
                                       | 04. State Persistence & Audit  |
                                       |     Scrive Flag e Log Centrale |
                                       +--------------------------------+
```

---

## 💡 Il Problema Aziendale & La Soluzione

### **Contesto e Criticità**
* **Inefficienza dei Deploy Manuali:** Installare o aggiornare agenti (es. GLPI Agent, sistemi di sicurezza, software di teleassistenza) macchina per macchina richiede centinaia di ore/uomo di supporto tecnico.
* **Limiti dei Tool Nativi GPO:** La funzionalità nativa *Software Installation* di Active Directory supporta solo file `.msi` semplici e spesso fallisce con installer `.exe`, installazioni silenziose personalizzate o prerequisiti complessi.
* **Congestione di Rete (Boot Storms):** Senza un controllo dello stato locale, le macchine del dominio tenterebbero di scaricare e rieseguire gli installer ad ogni riavvio, paralizzando la rete aziendale.

### **Soluzione Ingegneristica**
1. **Engine PowerShell Idempotente:** Sviluppo di uno script che interroga il Registro di Sistema (`HKLM:\SOFTWARE\...`) o il file system locale (`C:\ProgramData\...`) prima di avviare il processo.
2. **Esecuzione con Privilegi di Sistema:** Deployment configurato come *Computer Startup Script* (ambiente `NT AUTHORITY\SYSTEM`), superando le limitazioni di UAC e le restrizioni degli utenti standard.
3. **Audit & Logging Centralizzato:** Ogni installazione genera un log locale e notifica l'esito (Success/Fail) su una share di rete protetta per il tracciamento dei failure rate.

---

## 🛠️ Tech Stack & Requisiti

* **Language:** PowerShell 5.1+ / Batch Scripting
* **Directory Services:** Microsoft Active Directory / Group Policy Management (GPMC)
* **Execution Context:** Windows Startup Scripts (`NT AUTHORITY\SYSTEM`)
* **Target OS:** Windows 10, Windows 11, Windows Server 2016/2019/2022

---

## 💻 Codice Sorgente Sanificato (`deploy_agent.ps1`)

```powershell
<#
.SYNOPSIS
    Framework Idempotente per il Deploy Automatizzato di Software via GPO.
.DESCRIPTION
    Verifica lo stato di installazione dell'agente target tramite Registro/Flag.
    Se assente, esegue l'installazione silenziosa, scrive i log e aggiorna lo stato.
#>

# --- CONFIGURAZIONE SANIFICATA ---
$AppName        = "EnterpriseAgent"
$AppVersion     = "2.4.1"
$RegistryPath   = "HKLM:\SOFTWARE\CompanyDeployment\$AppName"
$FlagFilePath   = "$env:ProgramData\CompanyDeployment\${AppName}_installed.flag"

$NetworkShare   = "\\DOMAIN-DC01\SYSVOL\domain.local\scripts\installers"
$InstallerPath  = "$NetworkShare\setup_agent.msi"
$LogCentralDir  = "\\DOMAIN-DC01\DeploymentLogs$"
$LocalLogPath   = "$env:Temp\install_${AppName}.log"

# --- FASE 1: VERIFICA IDEMPOTENZA (STATE CHECK) ---
Function Test-IsInstalled {
    # Controllo via Chiave di Registro
    if (Test-Path $RegistryPath) {
        $InstalledVer = (Get-ItemProperty -Path $RegistryPath -Name "Version" -ErrorAction SilentlyContinue).Version
        if ($InstalledVer -eq $AppVersion) { return $true }
    }
    # Controllo di backup via File Flag
    if (Test-Path $FlagFilePath) { return $true }
    
    return $false
}

if (Test-IsInstalled) {
    # Il software è già installato: uscita immediata (Zero Overhead)
    Exit 0
}

# --- FASE 2: PREPARAZIONE AMBIENTE & LOGGING ---
$Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
$HostName  = $env:COMPUTERNAME

$LogMessage = "[$Timestamp] Avvio installazione $AppName v$AppVersion su $HostName"
Add-Content -Path $LocalLogPath -Value $LogMessage

# --- FASE 3: ESECUZIONE INSTALLAZIONE SILENZIOSA ---
if (-not (Test-Path $InstallerPath)) {
    Add-Content -Path $LocalLogPath -Value "[$Timestamp] ERRORE: Installer non trovato su $InstallerPath"
    Exit 1
}

try {
    # Esecuzione MSIExec in modalità Silent (/qn) senza riavvio (/norestart)
    $Process = Start-Process -FilePath "msiexec.exe" `
                             -ArgumentList "/i `"$InstallerPath`" /qn /norestart /L*V `"$LocalLogPath`"" `
                             -Wait -PassThru

    if ($Process.ExitCode -eq 0 -or $Process.ExitCode -eq 3010) {
        # --- FASE 4: PERSISTENZA DELLO STATO (WRITE MARKER) ---
        New-Item -Path $RegistryPath -Force | Out-Null
        Set-ItemProperty -Path $RegistryPath -Name "Version" -Value $AppVersion
        Set-ItemProperty -Path $RegistryPath -Name "InstallDate" -Value $Timestamp

        New-Item -Path (Split-Path $FlagFilePath) -ItemType Directory -Force -ErrorAction SilentlyContinue | Out-Null
        Set-Content -Path $FlagFilePath -Value "Installed on $Timestamp"

        # Notifica Share Centrale per Audit
        if (Test-Path $LogCentralDir) {
            Copy-Item -Path $LocalLogPath -Destination "$LogCentralDir\SUCCESS_${HostName}_${AppName}.log" -Force
        }
        Exit 0
    } else {
        throw "MSIExec restituito codice di errore: $($Process.ExitCode)"
    }
}
catch {
    $ErrMsg = "[$Timestamp] ERRORE CRITICO: $_"
    Add-Content -Path $LocalLogPath -Value $ErrMsg
    if (Test-Path $LogCentralDir) {
        Copy-Item -Path $LocalLogPath -Destination "$LogCentralDir\FAIL_${HostName}_${AppName}.log" -Force
    }
    Exit 1
}
```

---

## ⚙️ Guida all'Implementazione in Active Directory

1. **Posizionamento Script:** Copiare lo script `.ps1` all'interno della cartella `SYSVOL` del Domain Controller (`\\domain.local\SYSVOL\domain.local\scripts\`).
2. **Creazione GPO:**
   * Aprire `GPMC.msc` e creare una GPO nominata `DEPLOY_Software_EnterpriseAgent`.
   * Navigare in: `Configurazione Computer` -> `Impostazioni di Windows` -> `Script (Avvio/Arresto)` -> `Avvio`.
   * Nella scheda **PowerShell Scripts**, aggiungere il percorso dello script `SYSVOL`.
3. **Targeting & WMI Filtering (Opzionale):** Applicare la GPO alle Organization Unit (OU) contenenti i Computer Target, escludendo eventuali server critici tramite WMI Filter o Security Filtering.