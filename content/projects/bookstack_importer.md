---
title: "BookStack Folder Importer"
date: 2026-06-25
draft: false
tags: ["PowerShell", "REST API", ".NET GUI"]
image: "images/script.jpg"
summary: "Applicativo Windows Form sviluppato in PowerShell con interfaccia grafica .NET. Esegue lo scanning ricorsivo di share di rete ed effettua l'importazione massiva di documentazione aziendale (.docx, .pdf, .xlsx) in BookStack via API REST."
---

## Panoramica

Script PowerShell con interfaccia grafica (GUI) che importa automaticamente documenti da cartelle di rete in **BookStack**, creando la struttura gerarchica Shelf → Book → Chapter → Page.

 

## Requisiti

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-requisito-note-power"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Requisito</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Note</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">PowerShell</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">5.1+ (consigliato 7+)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Microsoft Word</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Per leggere `.docx` e convertire `.pdf` (Word 2013+)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Modulo `ImportExcel`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Per leggere `.xlsx` senza Excel installato</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Connettività</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Accesso all'URL di BookStack e alla cartella di rete</td></tr></tbody></table>

</div>> Il pulsante **"Installa dipendenze"** nella GUI installa automaticamente: NuGet provider, modulo ImportExcel e Poppler (per PDF).

 

## Configurazione iniziale

Prima di eseguire lo script, modificare le variabili in cima al file:

- **`$BOOKSTACK_URL`** — URL dell'istanza BookStack
- **`$API_TOKEN_ID`** — Token ID API
- **`$API_TOKEN_SECRET`** — Token Secret API
- **`$CARTELLA_ROOT`** — Percorso UNC della cartella sorgente

**Opzioni aggiuntive:**

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-variabile-default-de"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Variabile</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Default</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$DRY_RUN`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$true`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Se `$true`, analizza senza importare</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$DELAY_TRA_FILE`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`300`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Millisecondi di attesa tra un file e l'altro</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$MAX_CONTENT_KB`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`10240`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Dimensione massima contenuto pagina (KB)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$ESTENSIONI`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`.docx .pdf .xlsx .xls`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Formati importati come pagine</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`$ESTENSIONI_IMMAGINI`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`.jpg .png .gif` ecc.</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Formati riconosciuti come immagini</td></tr></tbody></table>

</div> 

## Struttura gerarchica generata

Lo script mappa la struttura delle cartelle in BookStack nel seguente modo:

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-cartella-oggetto-boo"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Cartella</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Oggetto BookStack</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Cartella root</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">**Shelf**</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Sottocartella di primo livello</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">**Book**</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Sottocartella di secondo livello</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">**Chapter**</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">File (`.docx`, `.pdf`, `.xlsx`)</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">**Page**</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Cartella con sole immagini</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">**Page galleria**</td></tr></tbody></table>

</div> 

## Come si usa

### 1. Avvio

Eseguire lo script in PowerShell:

<div aria-label="Codice powershell" class="relative group/copy bg-bg-000/50 border-0.5 border-border-400 rounded-lg focus:outline-none focus-visible:ring-2 focus-visible:ring-accent-100" id="bkmrk-powershell" role="group" tabindex="0"><div class="sticky opacity-0 group-hover/copy:opacity-100 group-focus-within/copy:opacity-100 top-2 py-2 h-12 w-0 float-right"><div class="absolute right-0 h-8 px-2 items-center inline-flex z-10"><div class="relative"><div class="transition-all opacity-100 scale-100"><svg aria-hidden="true" class="transition-all opacity-100 scale-100" fill="currentColor" height="20" viewbox="0 0 20 20" width="20" xmlns="http://www.w3.org/2000/svg"><path d="M12.5 3A1.5 1.5 0 0 1 14 4.5V6h1.5A1.5 1.5 0 0 1 17 7.5v8a1.5 1.5 0 0 1-1.5 1.5h-8A1.5 1.5 0 0 1 6 15.5V14H4.5A1.5 1.5 0 0 1 3 12.5v-8A1.5 1.5 0 0 1 4.5 3zm1.5 9.5a1.5 1.5 0 0 1-1.5 1.5H7v1.5a.5.5 0 0 0 .5.5h8a.5.5 0 0 0 .5-.5v-8a.5.5 0 0 0-.5-.5H14zM4.5 4a.5.5 0 0 0-.5.5v8a.5.5 0 0 0 .5.5h8a.5.5 0 0 0 .5-.5v-8a.5.5 0 0 0-.5-.5z"></path></svg></div><div class="absolute inset-0 flex items-center justify-center"><div class="transition-all opacity-0 scale-50"><svg aria-hidden="true" class="transition-all opacity-0 scale-50" fill="currentColor" height="20" viewbox="0 0 20 20" width="20" xmlns="http://www.w3.org/2000/svg"><path d="M15.188 5.11a.5.5 0 0 1 .752.626l-.056.084-7.5 9a.5.5 0 0 1-.738.033l-3.5-3.5-.064-.078a.501.501 0 0 1 .693-.693l.078.064 3.113 3.113 7.15-8.58z"></path></svg></div></div></div></div></div><div class="text-text-500 font-small p-3.5 pb-0">powershell</div><div class="overflow-x-auto">  
</div></div>

.\BookstackImporter.ps1

Si aprirà la GUI principale.

### 2. GUI principale — campi da compilare

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-campo-descrizione-ur"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Campo</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">URL BookStack</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">URL dell'istanza BookStack</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Token ID</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Token API (da BookStack → Profilo → API Tokens)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Token Secret</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Segreto del token API</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Cartella da importare</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Percorso UNC della cartella sorgente</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Nome Shelf</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Nome con cui apparirà in BookStack</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">DRY RUN</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Anteprima struttura senza importare nulla</td></tr></tbody></table>

</div>

### 3. Anteprima struttura

Cliccando **"Avanti: Anteprima struttura"** si apre una finestra ad albero che mostra esattamente come verrà importata la struttura:

- 🔵 **Blu** = Book / Chapter
- 🟣 **Viola** = Galleria immagini
- ⚫ **Grigio** = File → Pagina

### 4. Avvio importazione

Cliccando **"Avvia Importazione"** lo script si connette a BookStack e crea tutta la struttura.

 

## Gestione dei tipi di file

### `.docx`

Viene letto tramite **Microsoft Word COM** (metodo principale) o tramite lettura diretta dello ZIP (fallback). Vengono estratti paragrafi con stile (H1, H2, H3, grassetto, corsivo), tabelle e immagini inline (caricate nella gallery di BookStack).

### `.pdf`

Viene convertito in `.docx` tramite Word (Word 2013+), poi trattato come un normale DOCX. Se la conversione fallisce, il PDF viene caricato come **allegato** e la pagina contiene un pulsante di download.

### `.xlsx` / `.xls`

Viene letto tramite il modulo **ImportExcel** (metodo principale) o tramite **Excel COM** (fallback). Ogni foglio diventa una sezione con titolo e tabella HTML.

### Cartelle con sole immagini

Vengono importate come una **singola pagina galleria**: ogni immagine viene caricata nella gallery di BookStack e visualizzata con didascalia.

 

## Funzioni principali

### Estrazione contenuto

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-funzione-descrizione"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Funzione</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Estrai-Docx`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Legge un `.docx` via Word COM</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Estrai-DocxFallback`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Legge un `.docx` via ZIP (senza Word)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Estrai-Immagini-Docx`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Estrae immagini da un `.docx`</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Estrai-Excel`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Legge un `.xlsx`/`.xls`</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Importa-Pdf-Iframe`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Converte PDF→DOCX o carica come allegato</td></tr></tbody></table>

</div>

### API BookStack

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-funzione-endpoint-de"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Funzione</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Endpoint</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-CreaShelf`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/shelves`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Crea uno Shelf</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-CreaBook`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/books`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Crea un Book</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-CreaChapter`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/chapters`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Crea un Chapter</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-CreaPagina`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/pages`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Crea una Page</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-UploadImmagine`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/image-gallery`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Carica un'immagine</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`BS-UploadAllegato`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`POST /api/attachments`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Carica un allegato</td></tr></tbody></table>

</div>

### Importazione

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-funzione-descrizione-1"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Funzione</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Importa-File`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Gestisce un singolo file (smista per tipo)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Importa-Cartella`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Importa ricorsivamente una cartella come Book</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Importa-CartellaImmagini`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Crea una pagina galleria da una cartella di immagini</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`Importa-Tutto-GUI`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Entry point principale dell'importazione</td></tr></tbody></table>

</div> 

## Utility

<div class="overflow-x-auto w-full px-2 mb-6" id="bkmrk-funzione-descrizione-2"><table class="min-w-full border-collapse text-sm leading-[1.7] whitespace-normal"><thead class="text-left"><tr><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Funzione</th><th class="text-text-100 border-b-0.5 border-border-300/60 py-2 pr-4 align-top font-bold" scope="col">Descrizione</th></tr></thead><tbody><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`NomePulito`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Converte nomi file/cartelle in titoli leggibili (CamelCase → Camel Case, rimuove `_` e `-`)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`EscapeHtml`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Escape dei caratteri HTML speciali (`&`, `<`, `>`)</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`SanitizeHtml`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Rimuove caratteri di controllo e BOM dall'HTML</td></tr><tr><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">`TruncateContent`</td><td class="border-b-0.5 border-border-300/30 py-2 pr-4 align-top">Tronca il contenuto se supera `$MAX_CONTENT_KB`</td></tr></tbody></table>

</div> 

## Statistiche a fine importazione

Al termine, lo script mostra un riepilogo sia in console che in una finestra popup con il conteggio di Shelf, Books, Chapters, Pagine ed Errori.

 

## Note e limitazioni

> ⚠️ Le sottocartelle oltre il secondo livello vengono **"appiattite"** nel Chapter padre: il nome della pagina includerà il percorso relativo (es. `Sottocartella - NomeFile`).

> ⚠️ Il contenuto di ogni pagina viene troncato se supera **10 MB** (modificabile con `$MAX_CONTENT_KB`).

> ⚠️ La conversione PDF→DOCX richiede **Microsoft Word 2013 o superiore** installato sulla macchina che esegue lo script.

> ✅ Lo script gestisce automaticamente i **certificati SSL self-signed** (comune in ambienti interni).

 

## Script


```powershell
# ======================================================================
# BookStack Folder Importer - PowerShell
# Scansiona una cartella e importa Word/PDF/Excel in BookStack
# ======================================================================
#
# Requisiti:
#   - PowerShell 5.1+ o PowerShell 7+ (consigliato)
#   - Microsoft Word installato (per .docx)  OPPURE modulo PnP.PowerShell
#   - Modulo ImportExcel:  Install-Module ImportExcel -Scope CurrentUser
#
# Uso:
#   .\BookstackImporter.ps1
#
#                        -
# CONFIGURAZIONE - modifica questi valori prima di eseguire
#                        -

$BOOKSTACK_URL    = "***URL_BOOKSTACK***"
$API_TOKEN_ID     = "***TOKEN_ID***"
$API_TOKEN_SECRET = "***TOKEN_SECRET***"
$CARTELLA_ROOT    = "\\Cartella\da_importare"

# Opzioni
$DRY_RUN        = $true
$DELAY_TRA_FILE = 300
$MAX_CONTENT_KB = 10240
$ESTENSIONI         = @(".docx", ".pdf", ".xlsx", ".xls")
$ESTENSIONI_IMMAGINI = @(".jpg", ".jpeg", ".png", ".gif", ".webp", ".bmp", ".tiff", ".tif")

#                        -

# Ignora errori SSL self-signed
if ($PSVersionTable.PSVersion.Major -lt 6) {
    Add-Type @"
using System.Net;
using System.Security.Cryptography.X509Certificates;
public class TrustAll : ICertificatePolicy {
    public bool CheckValidationResult(ServicePoint sp, X509Certificate cert, WebRequest req, int problem) {
        return true;
    }
}
"@
    [System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAll
}

# ======================================================================
# LOG
# ======================================================================

function Write-OK   { param($msg) Write-Host "  [OK]   $msg" -ForegroundColor Green }
function Write-Warn { param($msg) Write-Host "  [WARN] $msg" -ForegroundColor Yellow }
function Write-Err  { param($msg) Write-Host "  [ERR]  $msg" -ForegroundColor Red }
function Write-Info { param($msg) Write-Host "  [INFO] $msg" -ForegroundColor Cyan }
function Write-Skip { param($msg) Write-Host "  [SKIP] $msg" -ForegroundColor DarkYellow }

# ======================================================================
# UTILITA'
# ======================================================================

function NomePulito {
    param([System.IO.FileSystemInfo]$item)
    if ($item.PSIsContainer) {
        $name = $item.Name
    } else {
        $name = [System.IO.Path]::GetFileNameWithoutExtension($item.Name)
    }
    # Inserisce spazio prima di una maiuscola preceduta da minuscola (CamelCase -> Camel Case)
    $name = [regex]::Replace($name, '(?<=[a-z])(?=[A-Z])', ' ')
    # Sostituisce trattini, underscore e punti con spazi
    $name = $name -replace '[_\-\.]+',' '
    # Rimuove spazi multipli
    $name = [regex]::Replace($name, '\s+', ' ').Trim()
    # TitleCase
    return (Get-Culture).TextInfo.ToTitleCase($name.ToLower())
}

function EscapeHtml {
    param([string]$text)
    $text = $text -replace '&',  '&amp;'
    $text = $text -replace '<',  '&lt;'
    $text = $text -replace '>',  '&gt;'
    $text = $text -replace '"',  '&quot;'
    $text = $text -replace "'", '&#39;'
    return $text
}

function TruncateContent {
    param([string]$html)
    $bytes    = [System.Text.Encoding]::UTF8.GetBytes($html)
    $maxBytes = $MAX_CONTENT_KB * 1024
    if ($bytes.Length -gt $maxBytes) {
        $truncated = [System.Text.Encoding]::UTF8.GetString($bytes[0..($maxBytes - 1)])
        return $truncated + '<p><em>Contenuto troncato per dimensione massima.</em></p>'
    }
    return $html
}

function SanitizeHtml {
    param([string]$html)
    if ([string]::IsNullOrWhiteSpace($html)) {
        return '<p><em>Contenuto non disponibile.</em></p>'
    }
    # Rimuove caratteri di controllo (null, BEL, BS ecc.) tranne tab/LF/CR
    $html = [regex]::Replace($html, '[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]', '')
    # Rimuove BOM UTF-8
    $html = $html.TrimStart([char]0xFEFF)
    if ([string]::IsNullOrWhiteSpace($html)) {
        return '<p><em>Contenuto non disponibile.</em></p>'
    }
    return $html
}

# ======================================================================
# ESTRAZIONE CONTENUTO
# ======================================================================

function Estrai-Docx {
    param([string]$filePath)

    # Metodo 1: Microsoft Word COM
    try {
        $word = New-Object -ComObject Word.Application -ErrorAction Stop
        $word.Visible      = $false
        $word.DisplayAlerts = 0

        $doc      = $word.Documents.Open($filePath, $false, $true)
        $htmlParts = [System.Collections.Generic.List[string]]::new()

        # Estrai immagini dal DOCX via ZIP (affidabile indipendentemente dal COM)
        $docImmagini = Estrai-Immagini-Docx $filePath
        $imgIdx      = 0  # indice corrente nella lista immagini

        foreach ($para in $doc.Paragraphs) {
            # Se il paragrafo contiene immagini inline, inserisci placeholder prima del testo
            try {
                if ($para.Range.InlineShapes.Count -gt 0) {
                    for ($si = 1; $si -le $para.Range.InlineShapes.Count; $si++) {
                        if ($imgIdx -lt $docImmagini.Count) {
                            $img = $docImmagini[$imgIdx]
                            $htmlParts.Add("<!--BOOKSTACK_IMG:$($img.Path):$($img.Name)-->")
                            $imgIdx++
                        }
                    }
                }
            } catch {}

            $text = $para.Range.Text.Trim() -replace "`r", ''
            if ([string]::IsNullOrWhiteSpace($text)) { continue }

            $escaped   = EscapeHtml $text
            $styleName = $para.Style.NameLocal.ToLower()

            if      ($styleName -match 'heading 1|titolo 1') { $htmlParts.Add("<h1>$escaped</h1>") }
            elseif  ($styleName -match 'heading 2|titolo 2') { $htmlParts.Add("<h2>$escaped</h2>") }
            elseif  ($styleName -match 'heading 3|titolo 3') { $htmlParts.Add("<h3>$escaped</h3>") }
            elseif  ($styleName -match 'heading 4|heading 5') { $htmlParts.Add("<h4>$escaped</h4>") }
            elseif  ($styleName -match 'list|elenco')         { $htmlParts.Add("<li>$escaped</li>") }
            else {
                $isBold   = $para.Range.Bold   -eq -1
                $isItalic = $para.Range.Italic -eq -1
                if      ($isBold -and $isItalic) { $htmlParts.Add("<p><strong><em>$escaped</em></strong></p>") }
                elseif  ($isBold)                { $htmlParts.Add("<p><strong>$escaped</strong></p>") }
                elseif  ($isItalic)              { $htmlParts.Add("<p><em>$escaped</em></p>") }
                else                             { $htmlParts.Add("<p>$escaped</p>") }
            }
        }

        # Immagini rimanenti non associate a paragrafi (es. floating shapes) - aggiungi in fondo
        while ($imgIdx -lt $docImmagini.Count) {
            $img = $docImmagini[$imgIdx]
            $htmlParts.Add("<!--BOOKSTACK_IMG:$($img.Path):$($img.Name)-->")
            $imgIdx++
        }

        # Tabelle
        foreach ($tbl in $doc.Tables) {
            $rows = [System.Collections.Generic.List[string]]::new()
            for ($r = 1; $r -le $tbl.Rows.Count; $r++) {
                $cells = [System.Collections.Generic.List[string]]::new()
                for ($c = 1; $c -le $tbl.Columns.Count; $c++) {
                    try {
                        $cellText = EscapeHtml ($tbl.Cell($r, $c).Range.Text -replace '[`r`a]', '').Trim()
                        $tag      = if ($r -eq 1) { 'th' } else { 'td' }
                        $cells.Add("<$tag>$cellText</$tag>")
                    } catch {
                        $cells.Add('<td></td>')
                    }
                }
                $rows.Add('<tr>' + ($cells -join '') + '</tr>')
            }
            $htmlParts.Add('<table><tbody>' + ($rows -join '') + '</tbody></table>')
        }

        $doc.Close($false)
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($doc)  | Out-Null
        $word.Quit()
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($word) | Out-Null

        $result = $htmlParts -join "`n"
        $result = [regex]::Replace($result, '(?s)((?:<li>.*?</li>\n?)+)', '<ul>$1</ul>')
        return $result

    } catch {
        return Estrai-DocxFallback $filePath
    }
}

function Estrai-DocxFallback {
    param([string]$filePath)
    try {
        Add-Type -AssemblyName System.IO.Compression.FileSystem
        $zip   = [System.IO.Compression.ZipFile]::OpenRead($filePath)
        $entry = $zip.Entries | Where-Object { $_.FullName -eq 'word/document.xml' }

        if (-not $entry) {
            $zip.Dispose()
            return '<p><em>Impossibile leggere il file DOCX</em></p>'
        }

        $stream     = $entry.Open()
        $reader     = New-Object System.IO.StreamReader($stream)
        $xmlContent = $reader.ReadToEnd()
        $reader.Dispose()
        $zip.Dispose()

        $text      = [regex]::Replace($xmlContent, '<[^>]+>', ' ')
        $text      = [regex]::Replace($text, '\s+', ' ').Trim()
        $text      = EscapeHtml $text
        $htmlParts = @()

        foreach ($line in ($text -split '(?<=\.) ')) {
            $line = $line.Trim()
            if ($line.Length -gt 0) { $htmlParts += "<p>$line</p>" }
        }
        return ($htmlParts -join "`n")

    } catch {
        $msg = EscapeHtml $_.Exception.Message
        return "<p><em>Errore lettura DOCX: $msg</em></p>"
    }
}

function Estrai-Immagini-Docx {
    # Estrae immagini dal DOCX (e' uno ZIP) in ordine di apparizione nel documento
    # Restituisce lista ordinata di @{Name; Path; RId; Ext}
    param([string]$filePath)
    $result = [System.Collections.Generic.List[hashtable]]::new()
    try {
        Add-Type -AssemblyName System.IO.Compression.FileSystem
        $zip = [System.IO.Compression.ZipFile]::OpenRead($filePath)

        # Leggi relazioni rId -> file media
        $ridToFile = @{}
        $relEntry  = $zip.Entries | Where-Object { $_.FullName -eq 'word/_rels/document.xml.rels' }
        if ($relEntry) {
            $stream = $relEntry.Open()
            $reader = New-Object System.IO.StreamReader($stream)
            $relXml = $reader.ReadToEnd()
            $reader.Dispose()
            $matches = [regex]::Matches($relXml, 'Id="([^"]+)"[^>]*Type="[^"]*\/image"[^>]*Target="([^"]+)"')
            foreach ($m in $matches) { $ridToFile[$m.Groups[1].Value] = $m.Groups[2].Value }
        }

        # Leggi document.xml per trovare immagini in ordine
        $docXml  = ''
        $docEntry = $zip.Entries | Where-Object { $_.FullName -eq 'word/document.xml' }
        if ($docEntry) {
            $stream = $docEntry.Open()
            $reader = New-Object System.IO.StreamReader($stream)
            $docXml = $reader.ReadToEnd()
            $reader.Dispose()
        }

        # Trova tutti r:embed in ordine (ogni embed = immagine nel documento)
        $counter = 0
        $seen    = @{}
        $embedMatches = [regex]::Matches($docXml, 'r:embed="([^"]+)"')
        foreach ($m in $embedMatches) {
            $rid = $m.Groups[1].Value
            if (-not $ridToFile.ContainsKey($rid)) { continue }
            if ($seen.ContainsKey($rid))            { continue }
            $seen[$rid] = $true
            $counter++

            $mediaPath = 'word/' + $ridToFile[$rid]
            $imgEntry  = $zip.Entries | Where-Object { $_.FullName -eq $mediaPath }
            if (-not $imgEntry) { continue }

            $imgName = 'img{0:D2}' -f $counter
            $ext     = [System.IO.Path]::GetExtension($ridToFile[$rid])
            $tmpPath = [System.IO.Path]::Combine($env:TEMP, "$imgName$ext")

            $inStream  = $imgEntry.Open()
            $outStream = [System.IO.File]::Create($tmpPath)
            $inStream.CopyTo($outStream)
            $outStream.Dispose()
            $inStream.Dispose()

            $result.Add(@{ Name = $imgName; Path = $tmpPath; RId = $rid; Ext = $ext.TrimStart('.').ToLower() })
        }
        $zip.Dispose()
    } catch {
        Write-Warn "Impossibile estrarre immagini da DOCX: $($_.Exception.Message)"
    }
    return $result
}

function BS-UploadImmagine {
    param([string]$filePath, [string]$nome, [int]$pageId)
    $ext      = [System.IO.Path]::GetExtension($filePath).TrimStart('.').ToLower()
    $mimeType = if ($ext -in @('jpg','jpeg')) { 'image/jpeg' } elseif ($ext -eq 'gif') { 'image/gif' } elseif ($ext -eq 'webp') { 'image/webp' } else { 'image/png' }
    $boundary = [System.Guid]::NewGuid().ToString('N')
    $enc      = [System.Text.Encoding]::UTF8
    $CRLF     = "`r`n"

    $parts = [System.Collections.Generic.List[byte[]]]::new()

    # Campo type
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"type`"$CRLF$CRLF"))
    $parts.Add($enc.GetBytes("gallery$CRLF"))
    # Campo uploaded_to
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"uploaded_to`"$CRLF$CRLF"))
    $parts.Add($enc.GetBytes("$pageId$CRLF"))
    # Campo name
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"name`"$CRLF$CRLF"))
    $parts.Add($enc.GetBytes("$nome$CRLF"))
    # File
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"image`"; filename=`"$nome.$ext`"$CRLF"))
    $parts.Add($enc.GetBytes("Content-Type: $mimeType$CRLF$CRLF"))
    $parts.Add([System.IO.File]::ReadAllBytes($filePath))
    $parts.Add($enc.GetBytes($CRLF))
    $parts.Add($enc.GetBytes("--$boundary--$CRLF"))

    # Assembla body
    $totalLen = 0; foreach ($p in $parts) { $totalLen += $p.Length }
    $body = New-Object byte[] $totalLen
    $off  = 0
    foreach ($p in $parts) { [System.Buffer]::BlockCopy($p, 0, $body, $off, $p.Length); $off += $p.Length }

    $resp = Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/image-gallery" `
        -Method POST -Headers (BS-Headers) -Body $body `
        -ContentType "multipart/form-data; boundary=$boundary" -ErrorAction Stop
    return $resp.url
}

# Estrai-Pdf rimossa: i PDF vengono importati come allegati iframe (vedi Importa-Pdf-Iframe)

function Estrai-Excel {
    param([string]$filePath)

    # Metodo 1: modulo ImportExcel
    if (Get-Module -ListAvailable -Name ImportExcel) {
        try {
            Import-Module ImportExcel -ErrorAction Stop
            $htmlParts = [System.Collections.Generic.List[string]]::new()
            $workbook  = Open-ExcelPackage -Path $filePath

            foreach ($ws in $workbook.Workbook.Worksheets) {
                $sheetName = EscapeHtml $ws.Name
                $htmlParts.Add("<h2>Foglio: $sheetName</h2>")
                $dim = $ws.Dimension
                if (-not $dim) { $htmlParts.Add('<p><em>(foglio vuoto)</em></p>'); continue }

                $htmlParts.Add('<table><tbody>')
                for ($r = $dim.Start.Row; $r -le $dim.End.Row; $r++) {
                    $cells = [System.Collections.Generic.List[string]]::new()
                    for ($c = $dim.Start.Column; $c -le $dim.End.Column; $c++) {
                        $val = EscapeHtml ([string]$ws.Cells[$r, $c].Text)
                        $tag = if ($r -eq $dim.Start.Row) { 'th' } else { 'td' }
                        $cells.Add("<$tag>$val</$tag>")
                    }
                    $htmlParts.Add('<tr>' + ($cells -join '') + '</tr>')
                }
                $htmlParts.Add('</tbody></table>')
            }
            Close-ExcelPackage $workbook -NoSave
            return ($htmlParts -join "`n")
        } catch {}
    }

    # Metodo 2: Excel COM
    try {
        $excel = New-Object -ComObject Excel.Application -ErrorAction Stop
        $excel.Visible       = $false
        $excel.DisplayAlerts = $false
        $wb        = $excel.Workbooks.Open($filePath, 0, $true)
        $htmlParts = [System.Collections.Generic.List[string]]::new()

        foreach ($ws in $wb.Worksheets) {
            $sheetName = EscapeHtml $ws.Name
            $htmlParts.Add("<h2>Foglio: $sheetName</h2>")
            $used = $ws.UsedRange
            if (-not $used) { $htmlParts.Add('<p><em>(foglio vuoto)</em></p>'); continue }

            $htmlParts.Add('<table><tbody>')
            for ($r = 1; $r -le $used.Rows.Count; $r++) {
                $cells = [System.Collections.Generic.List[string]]::new()
                for ($c = 1; $c -le $used.Columns.Count; $c++) {
                    $val = EscapeHtml ([string]$used.Cells($r, $c).Text)
                    $tag = if ($r -eq 1) { 'th' } else { 'td' }
                    $cells.Add("<$tag>$val</$tag>")
                }
                $htmlParts.Add('<tr>' + ($cells -join '') + '</tr>')
            }
            $htmlParts.Add('</tbody></table>')
        }

        $wb.Close($false)
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($wb)    | Out-Null
        $excel.Quit()
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($excel) | Out-Null
        return ($htmlParts -join "`n")

    } catch {
        return '<p><em>Per leggere file Excel installa il modulo: Install-Module ImportExcel -Scope CurrentUser</em></p>'
    }
}

function Estrai-Contenuto {
    param([System.IO.FileInfo]$file)
    Write-Info "Estraggo: $($file.Name)"
    switch ($file.Extension.ToLower()) {
        '.docx'                    { return Estrai-Docx  $file.FullName }
        '.pdf'                     { return '<p><em>PDF gestito come allegato iframe.</em></p>' }
        { $_ -in '.xlsx', '.xls' } { return Estrai-Excel $file.FullName }
        default                    { return "<p><em>Formato non supportato: $($file.Extension)</em></p>" }
    }
}

# ======================================================================
# API BOOKSTACK
# ======================================================================

function BS-Headers {
    return @{
        'Authorization' = "Token ${API_TOKEN_ID}:${API_TOKEN_SECRET}"
    }
}

function BS-Get {
    param([string]$endpoint)
    return Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/$endpoint" `
        -Method GET -Headers (BS-Headers) -ErrorAction Stop
}

function ConvertTo-JsonSafe {
    param([hashtable]$obj)
    # JavaScriptSerializer gestisce Unicode meglio di ConvertTo-Json in PS 5.1
    Add-Type -AssemblyName System.Web.Extensions
    $ser = New-Object System.Web.Script.Serialization.JavaScriptSerializer
    $ser.MaxJsonLength = [int]::MaxValue
    $ser.RecursionLimit = 20
    return $ser.Serialize($obj)
}

function BS-Post {
    param([string]$endpoint, [hashtable]$body)
    $json  = ConvertTo-JsonSafe $body
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    return Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/$endpoint" `
        -Method POST -Headers (BS-Headers) -Body $bytes `
        -ContentType 'application/json; charset=utf-8' -ErrorAction Stop
}

function BS-Put {
    param([string]$endpoint, [hashtable]$body)
    $json  = ConvertTo-JsonSafe $body
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    return Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/$endpoint" `
        -Method PUT -Headers (BS-Headers) -Body $bytes `
        -ContentType 'application/json; charset=utf-8' -ErrorAction Stop
}

function BS-TestConnessione {
    try { BS-Get 'books?count=1' | Out-Null; return $true }
    catch { return $false }
}

function BS-CreaShelf {
    param([string]$nome, [string]$desc = '')
    $r = BS-Post 'shelves' @{ name = $nome; description = $desc }
    return $r.id
}

function BS-CreaBook {
    param([string]$nome, [string]$desc = '', [int]$shelfId = 0)
    $r      = BS-Post 'books' @{ name = $nome; description = $desc }
    $bookId = $r.id
    if ($shelfId -gt 0) {
        try {
            $shelf   = BS-Get "shelves/$shelfId"
            $bookIds = @($shelf.books | ForEach-Object { $_.id }) + $bookId
            BS-Put "shelves/$shelfId" @{ books = $bookIds } | Out-Null
        } catch {}
    }
    return $bookId
}

function BS-CreaChapter {
    param([string]$nome, [int]$bookId, [string]$desc = '')
    $r = BS-Post 'chapters' @{ name = $nome; book_id = $bookId; description = $desc }
    return $r.id
}

function BS-CreaPagina {
    param([string]$nome, [string]$contenutoHtml, [int]$bookId = 0, [int]$chapterId = 0)
    $contenutoHtml = SanitizeHtml $contenutoHtml
    $contenutoHtml = TruncateContent $contenutoHtml
    if ($nome.Length -gt 255) { $nome = $nome.Substring(0, 252) + '...' }
    $body = @{ name = $nome; html = $contenutoHtml }
    if ($chapterId -gt 0)    { $body['chapter_id'] = $chapterId }
    elseif ($bookId -gt 0)   { $body['book_id']    = $bookId }
    else                     { throw 'Serve bookId o chapterId' }
    # Debug: verifica body prima dell'invio
    if ([string]::IsNullOrWhiteSpace($nome)) {
        throw 'Nome pagina vuoto - impossibile creare la pagina'
    }
    if ([string]::IsNullOrWhiteSpace($contenutoHtml)) {
        $body['html'] = '<p><em>Contenuto non disponibile.</em></p>'
    }
    try {
        $r = BS-Post 'pages' $body
        return $r.id
    } catch {
        $detail = ''
        if ($_.Exception.Response) {
            try {
                $stream = $_.Exception.Response.GetResponseStream()
                $reader = New-Object System.IO.StreamReader($stream)
                $detail = ' | API: ' + $reader.ReadToEnd()
                $reader.Dispose()
            } catch {}
        }
        throw ($_.Exception.Message + $detail)
    }
}

# ======================================================================
# CARTELLE IMMAGINI
# ======================================================================

function Test-CartellaImmagini {
    # Ritorna $true se la cartella contiene SOLO immagini (nessun doc/pdf/xls e nessuna sottocartella con doc)
    param([string]$cartella)
    $items = Get-ChildItem -LiteralPath $cartella -Recurse -ErrorAction SilentlyContinue |
        Where-Object { -not $_.PSIsContainer -and -not $_.Name.StartsWith('.') }
    if ($items.Count -eq 0) { return $false }
    $nonImg = $items | Where-Object { $_.Extension.ToLower() -notin $ESTENSIONI_IMMAGINI }
    return ($nonImg.Count -eq 0)
}

function Importa-CartellaImmagini {
    param([string]$cartella, [string]$nomePageTitle, [int]$bookId = 0, [int]$chapterId = 0)
    $imgFiles = Get-ChildItem -LiteralPath $cartella -Recurse -ErrorAction SilentlyContinue |
        Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI_IMMAGINI } |
        Sort-Object FullName

    if ($imgFiles.Count -eq 0) {
        Write-Skip "Cartella immagini vuota: $cartella"
        return
    }

    Write-Info "  Cartella immagini: $($imgFiles.Count) file → pagina '$nomePageTitle'"

    # Crea prima la pagina vuota per ottenere il pageId
    $htmlTemp = "<p><em>Caricamento immagini in corso...</em></p>"
    try {
        $pageId = BS-CreaPagina -nome $nomePageTitle -contenutoHtml $htmlTemp -bookId $bookId -chapterId $chapterId
    } catch {
        Write-Err "Impossibile creare pagina immagini '$nomePageTitle': $($_.Exception.Message)"
        $script:stats.errori++
        return
    }

    # Carica ogni immagine e costruisci HTML
    $imgCounter = 1
    $htmlParts  = [System.Collections.Generic.List[string]]::new()
    $htmlParts.Add("<h2>$nomePageTitle</h2>")

    foreach ($imgFile in $imgFiles) {
        $imgName = 'img{0:D2}' -f $imgCounter
        $imgCounter++
        try {
            $imgUrl = BS-UploadImmagine -filePath $imgFile.FullName -nome $imgName -pageId $pageId
            $label  = [System.IO.Path]::GetFileNameWithoutExtension($imgFile.Name)
            $htmlParts.Add("<figure>")
            $htmlParts.Add("  <img src=`"$imgUrl`" alt=`"$label`" style=`"max-width:100%; margin:8px 0;`">")
            $htmlParts.Add("  <figcaption><em>$label</em></figcaption>")
            $htmlParts.Add("</figure>")
            Write-OK "    Immagine caricata: $imgName ($($imgFile.Name))"
        } catch {
            Write-Warn "    Immagine non caricata ($($imgFile.Name)): $($_.Exception.Message)"
            $htmlParts.Add("<p><em>[Immagine non disponibile: $($imgFile.Name)]</em></p>")
        }
    }

    # Aggiorna la pagina con tutte le immagini
    $htmlFinale = $htmlParts -join "`n"
    try {
        $updateBody = @{ html = $htmlFinale }
        $bytes = [System.Text.Encoding]::UTF8.GetBytes((ConvertTo-JsonSafe $updateBody))
        Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/pages/$pageId" `
            -Method PUT -Headers (BS-Headers) -Body $bytes `
            -ContentType 'application/json; charset=utf-8' -ErrorAction Stop | Out-Null
        Write-OK "Pagina immagini: '$nomePageTitle' (ID: $pageId) — $($imgFiles.Count) immagini"
        $script:stats.page++
    } catch {
        Write-Warn "Pagina '$nomePageTitle' creata ma aggiornamento immagini fallito: $($_.Exception.Message)"
        $script:stats.page++
    }
}

# ======================================================================
# STAMPA ALBERO
# ======================================================================

function Stampa-Albero {
    param([string]$cartella, [string]$prefix = '', [int]$livello = 0)

    if ($livello -eq 0) {
        $name = Split-Path -Leaf $cartella
        Write-Host ''
        Write-Host "``[SHELF``] $name" -ForegroundColor Cyan
    }

    $items    = Get-ChildItem -LiteralPath $cartella -ErrorAction SilentlyContinue | Where-Object { -not $_.Name.StartsWith('.') }
    $dirs     = $items | Where-Object {  $_.PSIsContainer } | Sort-Object Name
    $files    = $items | Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } | Sort-Object Name
    $allItems = @($dirs) + @($files)

    for ($i = 0; $i -lt $allItems.Count; $i++) {
        $item      = $allItems[$i]
        $isLast    = ($i -eq $allItems.Count - 1)
        if ($isLast) { $connector = '`-- '; $extPrefix = $prefix + '    ' }
        else         { $connector = '+-- '; $extPrefix = $prefix + '|   ' }

        if ($item.PSIsContainer) {
            if (Test-CartellaImmagini $item.FullName) {
                $nImg = (Get-ChildItem -LiteralPath $item.FullName -Recurse -ErrorAction SilentlyContinue | Where-Object { $_.Extension.ToLower() -in $ESTENSIONI_IMMAGINI }).Count
                Write-Host "$prefix$connector" -NoNewline
                Write-Host $item.Name -NoNewline -ForegroundColor Magenta
                Write-Host "  ``[IMG-PAGE: $nImg immagini``]" -ForegroundColor Magenta
            } else {
                if ($livello -eq 0) { $tag = 'BOOK';    $col = 'Green'  }
                else                { $tag = 'CHAPTER'; $col = 'Yellow' }
                Write-Host "$prefix$connector" -NoNewline
                Write-Host $item.Name -NoNewline -ForegroundColor White
                Write-Host "  [$tag]" -ForegroundColor $col
                    Stampa-Albero $item.FullName $extPrefix ($livello + 1)
            }
        } else {
            $ext = $item.Extension.ToLower()
            $tag = switch ($ext) { '.pdf' { '[PDF]' } '.xlsx' { '[XLS]' } '.xls' { '[XLS]' } default { '[DOC]' } }
            Write-Host "$prefix$connector$tag $($item.Name)" -NoNewline
            Write-Host '  [PAGE]' -ForegroundColor Cyan
        }
    }
}

# ======================================================================
# IMPORTAZIONE
# ======================================================================

$stats = @{ shelf = 0; book = 0; chapter = 0; page = 0; errori = 0 }

function BS-UploadAllegato {
    param([string]$filePath, [string]$nome, [int]$pageId)
    $boundary = [System.Guid]::NewGuid().ToString('N')
    $fileName  = [System.IO.Path]::GetFileName($filePath)
    $enc       = [System.Text.Encoding]::UTF8
    $CRLF      = "`r`n"

    $parts = [System.Collections.Generic.List[byte[]]]::new()

    # Campo name
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"name`"$CRLF$CRLF"))
    $parts.Add($enc.GetBytes("$nome$CRLF"))
    # Campo uploaded_to
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"uploaded_to`"$CRLF$CRLF"))
    $parts.Add($enc.GetBytes("$pageId$CRLF"))
    # File
    $parts.Add($enc.GetBytes("--$boundary$CRLF"))
    $parts.Add($enc.GetBytes("Content-Disposition: form-data; name=`"file`"; filename=`"$fileName`"$CRLF"))
    $parts.Add($enc.GetBytes("Content-Type: application/pdf$CRLF$CRLF"))
    $parts.Add([System.IO.File]::ReadAllBytes($filePath))
    $parts.Add($enc.GetBytes($CRLF))
    $parts.Add($enc.GetBytes("--$boundary--$CRLF"))

    $totalLen = 0; foreach ($p in $parts) { $totalLen += $p.Length }
    $body = New-Object byte[] $totalLen
    $off  = 0
    foreach ($p in $parts) { [System.Buffer]::BlockCopy($p, 0, $body, $off, $p.Length); $off += $p.Length }

    $resp = Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/attachments" `
        -Method POST -Headers (BS-Headers) -Body $body `
        -ContentType "multipart/form-data; boundary=$boundary" -ErrorAction Stop
    return $resp.id
}

function Importa-Pdf-Iframe {
    param([System.IO.FileInfo]$file, [int]$bookId = 0, [int]$chapterId = 0)
    $nome    = NomePulito $file
    $tmpDocx = $null
    $word    = $null

    try {
        Write-Info "  Conversione PDF -> DOCX via Word: $($file.Name)"
        $word = New-Object -ComObject Word.Application -ErrorAction Stop
        $word.Visible       = $false
        $word.DisplayAlerts = 0

        # Word apre il PDF (funziona da Word 2013+)
        $doc     = $word.Documents.Open($file.FullName, $false, $true)
        $tmpDocx = [System.IO.Path]::Combine($env:TEMP, [System.IO.Path]::GetFileNameWithoutExtension($file.Name) + '_conv.docx')
        # 16 = wdFormatXMLDocument (docx)
        $doc.SaveAs([ref]$tmpDocx, [ref]16)
        $doc.Close($false)
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($doc) | Out-Null
        $word.Quit()
        [System.Runtime.InteropServices.Marshal]::ReleaseComObject($word) | Out-Null
        $word = $null
        Write-OK "  Conversione riuscita -> $([System.IO.Path]::GetFileName($tmpDocx))"

        # Estrai contenuto dal DOCX convertito (identico al flusso normale)
        $contenuto = Estrai-Docx $tmpDocx
        if ([string]::IsNullOrWhiteSpace($contenuto)) { $contenuto = '<p><em>Contenuto non disponibile.</em></p>' }

        $imgPlaceholders = [regex]::Matches($contenuto, '<!--BOOKSTACK_IMG:([^:]+):([^-]+)-->')
        $htmlSenzaImg    = [regex]::Replace($contenuto, '<!--BOOKSTACK_IMG:[^>]+-->', '')

        $pageId = BS-CreaPagina -nome $nome -contenutoHtml $htmlSenzaImg -bookId $bookId -chapterId $chapterId
        Write-OK "Pagina PDF->DOCX: '$nome' (ID: $pageId)"
        $script:stats.page++

        if ($imgPlaceholders.Count -gt 0) {
            Write-Info "  Upload $($imgPlaceholders.Count) immagini..."
            foreach ($m in $imgPlaceholders) {
                $imgPath = $m.Groups[1].Value
                $imgName = $m.Groups[2].Value
                try {
                    if (Test-Path $imgPath) {
                        $imgUrl    = BS-UploadImmagine -filePath $imgPath -nome $imgName -pageId $pageId
                        $imgTag    = "<p><img src=`"$imgUrl`" alt=`"$imgName`" style=`"max-width:100%`"></p>"
                        $contenuto = $contenuto -replace [regex]::Escape($m.Value), $imgTag
                        Write-OK "  Immagine caricata: $imgName"
                        Remove-Item $imgPath -Force -ErrorAction SilentlyContinue
                    }
                } catch {
                    Write-Warn "  Immagine '$imgName' non caricata: $($_.Exception.Message)"
                }
            }
            $htmlFinale = [regex]::Replace($contenuto, '<!--BOOKSTACK_IMG:[^>]+-->', '')
            $htmlFinale = SanitizeHtml $htmlFinale
            $upd = @{ html = $htmlFinale }
            $bytes = [System.Text.Encoding]::UTF8.GetBytes((ConvertTo-JsonSafe $upd))
            Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/pages/$pageId" `
                -Method PUT -Headers (BS-Headers) -Body $bytes `
                -ContentType 'application/json; charset=utf-8' -ErrorAction Stop | Out-Null
            Write-OK "  Pagina aggiornata con immagini"
        }

    } catch {
        # Fallback: Word non disponibile o PDF protetto -> carica come allegato con link
        Write-Warn "  Conversione fallita ($($_.Exception.Message)) -- carico come allegato"
        try { if ($word) { $word.Quit() } } catch {}
        try {
            $htmlTemp  = '<p><em>Caricamento PDF...</em></p>'
            $pageId    = BS-CreaPagina -nome $nome -contenutoHtml $htmlTemp -bookId $bookId -chapterId $chapterId
            $script:stats.page++
            $attachId  = BS-UploadAllegato -filePath $file.FullName -nome $file.Name -pageId $pageId
            $attachUrl = "$BOOKSTACK_URL/attachments/$attachId"
            $nomeEsc   = EscapeHtml $file.Name
            $html  = "<h2>$nome</h2>"
            $html += "<p><em>File: $nomeEsc</em></p>"
            $html += "<p><a href=`"$attachUrl`" target=`"_blank`" style=`"display:inline-block;padding:12px 24px;background:#1e3a5f;color:#fff;border-radius:6px;text-decoration:none;font-weight:bold;`">Apri / Scarica PDF</a></p>"
            $upd = @{ html = $html }
            $bytes = [System.Text.Encoding]::UTF8.GetBytes((ConvertTo-JsonSafe $upd))
            Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/pages/$pageId" `
                -Method PUT -Headers (BS-Headers) -Body $bytes `
                -ContentType 'application/json; charset=utf-8' -ErrorAction Stop | Out-Null
            Write-OK "Pagina PDF (allegato): '$nome' (ID: $pageId)"
        } catch {
            Write-Err "Errore import PDF '$nome': $($_.Exception.Message)"
            $script:stats.errori++
        }
    } finally {
        if ($tmpDocx -and (Test-Path $tmpDocx -ErrorAction SilentlyContinue)) {
            Remove-Item $tmpDocx -Force -ErrorAction SilentlyContinue
        }
    }
}

function Importa-File {
    param([System.IO.FileInfo]$file, [int]$bookId = 0, [int]$chapterId = 0)
    $nome = NomePulito $file

    # I PDF vengono caricati come allegati e mostrati in iframe
    if ($file.Extension.ToLower() -eq '.pdf') {
        Importa-Pdf-Iframe -file $file -bookId $bookId -chapterId $chapterId
        return
    }

    try {
        $contenuto = Estrai-Contenuto $file

        $lunghezza = if ($contenuto) { $contenuto.Length } else { 0 }
        Write-Info "  Contenuto estratto: $lunghezza caratteri"

        if ($lunghezza -eq 0 -or $null -eq $contenuto) {
            $contenuto = '<p><em>Contenuto non disponibile o formato non supportato.</em></p>'
        }

        # Estrai placeholder immagini dall'HTML prima di creare la pagina
        $imgPlaceholders = [regex]::Matches($contenuto, '<!--BOOKSTACK_IMG:([^:]+):([^-]+)-->')
        # Rimuovi i placeholder dall'HTML iniziale (verranno sostituiti dopo l'upload)
        $htmlSenzaImg = [regex]::Replace($contenuto, '<!--BOOKSTACK_IMG:[^>]+-->', '')

        # Crea la pagina (senza immagini per ora)
        $pageId = BS-CreaPagina -nome $nome -contenutoHtml $htmlSenzaImg -bookId $bookId -chapterId $chapterId
        Write-OK "Pagina: '$nome' (ID: $pageId)"
        $script:stats.page++

        # Upload immagini e aggiorna HTML
        if ($imgPlaceholders.Count -gt 0) {
            Write-Info "  Upload $($imgPlaceholders.Count) immagini..."
            $htmlConImg = $htmlSenzaImg

            foreach ($m in $imgPlaceholders) {
                $imgPath = $m.Groups[1].Value
                $imgName = $m.Groups[2].Value
                try {
                    if (Test-Path $imgPath) {
                        $imgUrl = BS-UploadImmagine -filePath $imgPath -nome $imgName -pageId $pageId
                        # Inserisci tag img nella posizione del placeholder (gia' rimosso, aggiungiamo a fine paragrafo precedente o come paragrafo)
                        $imgTag = "<p><img src=`"$imgUrl`" alt=`"$imgName`" style=`"max-width:100%`"></p>"
                        # Ricostruiamo: sostituiamo il placeholder originale nel contenuto originale
                        $contenuto = $contenuto -replace [regex]::Escape($m.Value), $imgTag
                        Write-OK "  Immagine caricata: $imgName"
                        Remove-Item $imgPath -Force -ErrorAction SilentlyContinue
                    }
                } catch {
                    Write-Warn "  Immagine '$imgName' non caricata: $($_.Exception.Message)"
                }
            }

            # Aggiorna la pagina con le immagini
            try {
                $htmlFinale = [regex]::Replace($contenuto, '<!--BOOKSTACK_IMG:[^>]+-->', '')
                $htmlFinale = SanitizeHtml $htmlFinale
                $updateBody = @{ html = $htmlFinale }
                $bytes = [System.Text.Encoding]::UTF8.GetBytes((ConvertTo-JsonSafe $updateBody))
                Invoke-RestMethod -Uri "$BOOKSTACK_URL/api/pages/$pageId" `
                    -Method PUT -Headers (BS-Headers) -Body $bytes `
                    -ContentType 'application/json; charset=utf-8' -ErrorAction Stop | Out-Null
                Write-OK "  Pagina aggiornata con immagini"
            } catch {
                Write-Warn "  Impossibile aggiornare pagina con immagini: $($_.Exception.Message)"
            }
        }

        Start-Sleep -Milliseconds $DELAY_TRA_FILE
    } catch {
        Write-Err "Errore pagina '$nome': $($_.Exception.Message)"
        $script:stats.errori++
    }
}

function Importa-Cartella {
    param([string]$cartella, [int]$bookId, [int]$livello = 1)

    $items    = Get-ChildItem -LiteralPath $cartella -ErrorAction SilentlyContinue | Where-Object { -not $_.Name.StartsWith('.') }
    $dirs     = $items | Where-Object {  $_.PSIsContainer } | Sort-Object Name
    $files    = $items | Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } | Sort-Object Name

    foreach ($f in $files) { Importa-File $f -bookId $bookId }

    foreach ($d in $dirs) {
        $nome = NomePulito $d
        if ($livello -eq 1) {
            Write-Host ''
            Write-Host "  Chapter: $nome" -ForegroundColor Yellow
            try {
                $chId = BS-CreaChapter -nome $nome -bookId $bookId
                Write-OK "Chapter: '$nome' (ID: $chId)"
                $script:stats.chapter++
            } catch {
                Write-Err "Errore chapter '$nome': $($_.Exception.Message)"
                $script:stats.errori++
                continue
            }

            $chFiles = Get-ChildItem -LiteralPath $d.FullName -ErrorAction SilentlyContinue |
                Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } |
                Sort-Object Name
            foreach ($f in $chFiles) { Importa-File $f -chapterId $chId }

            $subDirs = Get-ChildItem -LiteralPath $d.FullName -ErrorAction SilentlyContinue |
                Where-Object { $_.PSIsContainer -and -not $_.Name.StartsWith('.') }
            foreach ($sd in $subDirs) {
                $sdNome = NomePulito $sd

                # Cartella con SOLO immagini -> pagina galleria nel chapter
                if (Test-CartellaImmagini $sd.FullName) {
                    Write-Info "  Galleria immagini: '$($sd.Name)' -> pagina '$sdNome'"
                    Importa-CartellaImmagini -cartella $sd.FullName -nomePageTitle $sdNome -chapterId $chId
                    continue
                }

                Write-Info "Sottocartella '$($sd.Name)' piattata nel chapter '$nome'"
                $deepFiles = Get-ChildItem -LiteralPath $sd.FullName -Recurse -ErrorAction SilentlyContinue |
                    Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } |
                    Sort-Object FullName
                foreach ($f in $deepFiles) {
                    $rel      = $f.FullName.Substring($d.FullName.Length + 1)
                    $relParts = $rel -split '\\'
                    $nomePag  = if ($relParts.Count -gt 1) {
                        ($relParts[0..($relParts.Count - 2)] -join ' - ') + ' - ' + (NomePulito $f)
                    } else {
                        NomePulito $f
                    }
                    # Usa Importa-File cosi i PDF vanno via iframe e i DOCX normalmente
                    $fInfo = Get-Item -LiteralPath $f.FullName
                    # Rinomina temporaneamente il nome pulito passandolo come file info e usando chapterId
                    try {
                        if ($f.Extension.ToLower() -eq '.pdf') {
                            Importa-Pdf-Iframe -file $fInfo -chapterId $chId
                        } else {
                            $contenuto = Estrai-Contenuto $fInfo
                            $pageId    = BS-CreaPagina -nome $nomePag -contenutoHtml $contenuto -chapterId $chId
                            Write-OK "Pagina: '$nomePag' (ID: $pageId)"
                            $script:stats.page++
                            Start-Sleep -Milliseconds $DELAY_TRA_FILE
                        }
                    } catch {
                        Write-Err "Errore pagina '$nomePag': $($_.Exception.Message)"
                        $script:stats.errori++
                    }
                }
            }
        }
    }
}

function Importa-Tutto {
    param([string]$rootPath)

    $root = Get-Item -LiteralPath $rootPath
    Write-Host ''
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host "  Avvio importazione: $($root.Name)" -ForegroundColor White
    Write-Host ('=' * 60) -ForegroundColor DarkGray

    $shelfNome = NomePulito $root
    Write-Host ''
    Write-Host "Shelf: $shelfNome" -ForegroundColor Cyan
    $shelfId = BS-CreaShelf -nome $shelfNome -desc "Importato da: $rootPath"
    Write-OK "Shelf: '$shelfNome' (ID: $shelfId)"
    $script:stats.shelf++

    $items      = Get-ChildItem -LiteralPath $rootPath -ErrorAction SilentlyContinue | Where-Object { -not $_.Name.StartsWith('.') }
    $dirs       = $items | Where-Object {  $_.PSIsContainer } | Sort-Object Name
    $filesRoot  = $items | Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } | Sort-Object Name

    if ($filesRoot.Count -gt 0) {
        $nomeBook = "$shelfNome - Generale"
        Write-Host ''
        Write-Host "Book (root): $nomeBook" -ForegroundColor Green
        try {
            $bookId = BS-CreaBook -nome $nomeBook -shelfId $shelfId
            Write-OK "Book: '$nomeBook' (ID: $bookId)"
            $script:stats.book++
            foreach ($f in $filesRoot) { Importa-File $f -bookId $bookId }
        } catch {
            Write-Err "Errore book root: $($_.Exception.Message)"
            $script:stats.errori++
        }
    }

    $generaleBookId = $null  # usato per cartelle-galleria a livello root

    foreach ($d in $dirs) {
        $nome = NomePulito $d

        # Cartella con sole immagini a livello root -> Book "Generale" + pagina galleria
        if (Test-CartellaImmagini $d.FullName) {
            Write-Host ''
            Write-Host "Galleria: $nome" -ForegroundColor Magenta
            # Assicura che esista un book "Generale"
            if (-not $generaleBookId) {
                $nomeBook = "$shelfNome - Generale"
                try {
                    $generaleBookId = BS-CreaBook -nome $nomeBook -shelfId $shelfId
                    Write-OK "Book: '$nomeBook' (ID: $generaleBookId)"
                    $script:stats.book++
                } catch {
                    Write-Err "Errore book Generale: $($_.Exception.Message)"
                    $script:stats.errori++
                    continue
                }
            }
            Importa-CartellaImmagini -cartella $d.FullName -nomePageTitle $nome -bookId $generaleBookId
            continue
        }

        Write-Host ''
        Write-Host "Book: $nome" -ForegroundColor Green
        try {
            $bookId = BS-CreaBook -nome $nome -shelfId $shelfId
            Write-OK "Book: '$nome' (ID: $bookId)"
            $script:stats.book++
        } catch {
            Write-Err "Errore book '$nome': $($_.Exception.Message)"
            $script:stats.errori++
            continue
        }
        Importa-Cartella $d.FullName $bookId 1
    }

    Write-Host ''
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host '  Importazione completata!' -ForegroundColor Green
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host "  Shelf    : $($stats.shelf)"
    Write-Host "  Books    : $($stats.book)"
    Write-Host "  Chapters : $($stats.chapter)"
    Write-Host "  Pagine   : $($stats.page)"   -ForegroundColor Green
    if ($stats.errori -gt 0) {
        Write-Host "  Errori   : $($stats.errori)" -ForegroundColor Red
    }
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host ''
}


# ======================================================================
# INSTALLAZIONE DIPENDENZE
# ======================================================================

function Installa-Dipendenze {
    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    # ── Finestra log installazione ────────────────────────────────────────────
    $frmDep = New-Object System.Windows.Forms.Form
    $frmDep.Text            = 'Installazione dipendenze'
    $frmDep.Size            = New-Object System.Drawing.Size(640, 540)
    $frmDep.StartPosition   = 'CenterScreen'
    $frmDep.FormBorderStyle = 'FixedDialog'
    $frmDep.MaximizeBox     = $false
    $frmDep.BackColor       = [System.Drawing.Color]::FromArgb(245, 247, 250)
    $frmDep.Font            = New-Object System.Drawing.Font('Segoe UI', 9)

    $bannerDep = New-Object System.Windows.Forms.Panel
    $bannerDep.Location  = New-Object System.Drawing.Point(0, 0)
    $bannerDep.Size      = New-Object System.Drawing.Size(640, 52)
    $bannerDep.BackColor = [System.Drawing.Color]::FromArgb(217, 119, 6)
    $frmDep.Controls.Add($bannerDep)

    $lblDepTitle = New-Object System.Windows.Forms.Label
    $lblDepTitle.Text      = 'Installazione dipendenze'
    $lblDepTitle.Location  = New-Object System.Drawing.Point(14, 6)
    $lblDepTitle.Size      = New-Object System.Drawing.Size(600, 26)
    $lblDepTitle.Font      = New-Object System.Drawing.Font('Segoe UI', 13, [System.Drawing.FontStyle]::Bold)
    $lblDepTitle.ForeColor = [System.Drawing.Color]::White
    $bannerDep.Controls.Add($lblDepTitle)

    $lblDepSub = New-Object System.Windows.Forms.Label
    $lblDepSub.Text      = 'Poppler (PDF)  |  ImportExcel (XLSX)  |  NuGet provider'
    $lblDepSub.Location  = New-Object System.Drawing.Point(16, 32)
    $lblDepSub.Size      = New-Object System.Drawing.Size(600, 16)
    $lblDepSub.Font      = New-Object System.Drawing.Font('Segoe UI', 8)
    $lblDepSub.ForeColor = [System.Drawing.Color]::FromArgb(255, 237, 213)
    $bannerDep.Controls.Add($lblDepSub)

    # Log box
    $txtLog = New-Object System.Windows.Forms.RichTextBox
    $txtLog.Location   = New-Object System.Drawing.Point(12, 62)
    $txtLog.Size       = New-Object System.Drawing.Size(606, 390)
    $txtLog.ReadOnly   = $true
    $txtLog.BackColor  = [System.Drawing.Color]::FromArgb(20, 20, 30)
    $txtLog.ForeColor  = [System.Drawing.Color]::FromArgb(180, 220, 180)
    $txtLog.Font       = New-Object System.Drawing.Font('Consolas', 9)
    $txtLog.ScrollBars = 'Vertical'
    $frmDep.Controls.Add($txtLog)

    $btnClose = New-Object System.Windows.Forms.Button
    $btnClose.Text      = 'Chiudi'
    $btnClose.Location  = New-Object System.Drawing.Point(492, 466)
    $btnClose.Size      = New-Object System.Drawing.Size(126, 34)
    $btnClose.BackColor = [System.Drawing.Color]::FromArgb(107, 114, 128)
    $btnClose.ForeColor = [System.Drawing.Color]::White
    $btnClose.FlatStyle = 'Flat'
    $btnClose.FlatAppearance.BorderSize = 0
    $btnClose.Enabled   = $false
    $frmDep.Controls.Add($btnClose)
    $btnClose.Add_Click({ $frmDep.Close() })

    $lblFinalMsg = New-Object System.Windows.Forms.Label
    $lblFinalMsg.Location  = New-Object System.Drawing.Point(12, 472)
    $lblFinalMsg.Size      = New-Object System.Drawing.Size(470, 20)
    $lblFinalMsg.ForeColor = [System.Drawing.Color]::DimGray
    $lblFinalMsg.Text      = ''
    $frmDep.Controls.Add($lblFinalMsg)

    # ── Helper log ────────────────────────────────────────────────────────────
    function Write-Log {
        param([string]$msg, [string]$color = 'normal')
        $col = switch ($color) {
            'ok'      { [System.Drawing.Color]::FromArgb(134, 239, 172) }
            'warn'    { [System.Drawing.Color]::FromArgb(253, 224, 71)  }
            'err'     { [System.Drawing.Color]::FromArgb(252, 165, 165) }
            'info'    { [System.Drawing.Color]::FromArgb(147, 197, 253) }
            'head'    { [System.Drawing.Color]::FromArgb(253, 186, 116) }
            default   { [System.Drawing.Color]::FromArgb(180, 220, 180) }
        }
        $txtLog.SelectionStart  = $txtLog.TextLength
        $txtLog.SelectionLength = 0
        $txtLog.SelectionColor  = $col
        $txtLog.AppendText("$msg`n")
        $txtLog.ScrollToCaret()
        $frmDep.Refresh()
        [System.Windows.Forms.Application]::DoEvents()
    }

    # ── Mostra la finestra ────────────────────────────────────────────────────
    $frmDep.Show()
    $frmDep.BringToFront()
    [System.Windows.Forms.Application]::DoEvents()

    $needsRestart = $false

    Write-Log '============================================' 'head'
    Write-Log '  Verifica e installazione dipendenze'       'head'
    Write-Log '============================================' 'head'
    Write-Log ''

    # ── 1. NuGet provider (serve per Install-Module) ──────────────────────────
    Write-Log '[1/3] NuGet package provider...' 'info'
    try {
        $nuget = Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue
        if (-not $nuget -or $nuget.Version -lt [Version]'2.8.5.201') {
            Write-Log '  Installazione NuGet provider...' 'warn'
            Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Confirm:$false -Scope CurrentUser -ErrorAction Stop | Out-Null
            Write-Log '  NuGet installato correttamente.' 'ok'
        } else {
            Write-Log "  NuGet gia presente (v$($nuget.Version))." 'ok'
        }
    } catch {
        Write-Log "  ERRORE NuGet: $($_.Exception.Message)" 'err'
    }
    Write-Log ''

    # ── 2. Modulo ImportExcel ─────────────────────────────────────────────────
    Write-Log '[2/3] Modulo ImportExcel (lettura XLSX senza Excel)...' 'info'
    try {
        if (Get-Module -ListAvailable -Name ImportExcel) {
            $ver = (Get-Module -ListAvailable -Name ImportExcel | Sort-Object Version -Descending | Select-Object -First 1).Version
            Write-Log "  ImportExcel gia installato (v$ver)." 'ok'
        } else {
            Write-Log '  Installazione ImportExcel...' 'warn'
            Install-Module -Name ImportExcel -Scope CurrentUser -Force -Confirm:$false -AllowClobber -ErrorAction Stop
            Write-Log '  ImportExcel installato correttamente.' 'ok'
        }
    } catch {
        Write-Log "  ERRORE ImportExcel: $($_.Exception.Message)" 'err'
    }
    Write-Log ''

    # ── 3. Poppler (pdftotext.exe) ────────────────────────────────────────────
    Write-Log '[3/3] Poppler per Windows (pdftotext.exe per leggere PDF)...' 'info'

    $popplerCheck = Get-Command 'pdftotext.exe' -ErrorAction SilentlyContinue
    if ($popplerCheck) {
        Write-Log "  Poppler gia nel PATH: $($popplerCheck.Source)" 'ok'
    } else {
        # Percorso di installazione
        $popplerDir  = "$env:LOCALAPPDATA\Poppler"
        $popplerBin  = "$popplerDir\bin"
        $zipPath     = "$env:TEMP\poppler-windows.zip"
        $extractPath = "$env:TEMP\poppler-extract"

        # Cerca se e gia installato ma non nel PATH
        $existingBin = Get-ChildItem "$popplerDir\*\bin\pdftotext.exe" -ErrorAction SilentlyContinue |
                           Select-Object -First 1
        if (-not $existingBin) {
            $existingBin = Get-ChildItem "$env:LOCALAPPDATA\poppler-*\bin\pdftotext.exe" -ErrorAction SilentlyContinue |
                               Select-Object -First 1
        }

        if ($existingBin) {
            $popplerBin = Split-Path $existingBin.FullName
            Write-Log "  Poppler trovato in: $popplerBin" 'warn'
            Write-Log '  Ma non e nel PATH. Aggiorno PATH...' 'warn'
        } else {
            # Download da GitHub releases (poppler-windows)
            Write-Log '  Download Poppler da GitHub...' 'warn'
            Write-Log '  (versione 24.x, ~30 MB — attendere...)' 'warn'

            $apiUrl = 'https://api.github.com/repos/oschwartz10612/poppler-windows/releases/latest'
            try {
                $release  = Invoke-RestMethod -Uri $apiUrl -UseBasicParsing -ErrorAction Stop
                $asset    = $release.assets | Where-Object { $_.name -like '*.zip' } | Select-Object -First 1
                $dlUrl    = $asset.browser_download_url
                $zipName  = $asset.name
                Write-Log "  Release: $($release.tag_name) — File: $zipName" 'info'

                $webClient = New-Object System.Net.WebClient
                $webClient.DownloadFile($dlUrl, $zipPath)
                Write-Log "  Download completato. Estrazione..." 'warn'

                if (Test-Path $extractPath) { Remove-Item $extractPath -Recurse -Force }
                Add-Type -AssemblyName System.IO.Compression.FileSystem
                [System.IO.Compression.ZipFile]::ExtractToDirectory($zipPath, $extractPath)

                # Trova la cartella bin dentro lo zip estratto
                $extractedBin = Get-ChildItem "$extractPath\*\Library\bin\pdftotext.exe" -Recurse -ErrorAction SilentlyContinue |
                                    Select-Object -First 1
                if (-not $extractedBin) {
                    $extractedBin = Get-ChildItem "$extractPath" -Filter 'pdftotext.exe' -Recurse -ErrorAction SilentlyContinue |
                                        Select-Object -First 1
                }

                if ($extractedBin) {
                    $srcBinDir = Split-Path $extractedBin.FullName
                    if (-not (Test-Path $popplerBin)) { New-Item -ItemType Directory -Path $popplerBin -Force | Out-Null }
                    Copy-Item "$srcBinDir\*" $popplerBin -Force
                    Write-Log "  Poppler estratto in: $popplerBin" 'ok'

                    Remove-Item $zipPath     -Force -ErrorAction SilentlyContinue
                    Remove-Item $extractPath -Recurse -Force -ErrorAction SilentlyContinue
                } else {
                    Write-Log '  ERRORE: impossibile trovare pdftotext.exe nello ZIP estratto.' 'err'
                    $popplerBin = $null
                }
            } catch {
                Write-Log "  ERRORE download Poppler: $($_.Exception.Message)" 'err'
                Write-Log '  Scaricalo manualmente da: https://github.com/oschwartz10612/poppler-windows/releases' 'warn'
                $popplerBin = $null
            }
        }

        # Aggiorna PATH utente
        if ($popplerBin -and (Test-Path "$popplerBin\pdftotext.exe")) {
            $currentPath = [System.Environment]::GetEnvironmentVariable('PATH', 'User')
            if ($currentPath -notlike "*$popplerBin*") {
                $newPath = $currentPath.TrimEnd(';') + ';' + $popplerBin
                [System.Environment]::SetEnvironmentVariable('PATH', $newPath, 'User')
                Write-Log "  PATH utente aggiornato con: $popplerBin" 'ok'
                $needsRestart = $true
            } else {
                Write-Log '  Percorso gia nel PATH.' 'ok'
            }
            # Aggiorna PATH sessione corrente
            $env:PATH = $env:PATH.TrimEnd(';') + ';' + $popplerBin
            Write-Log '  PATH sessione corrente aggiornato (subito attivo).' 'ok'
        }
    }

    # ── Riepilogo finale ──────────────────────────────────────────────────────
    Write-Log ''
    Write-Log '============================================' 'head'
    Write-Log '  Installazione completata!'                  'head'
    Write-Log '============================================' 'head'
    Write-Log ''

    $excel  = if (Get-Module -ListAvailable -Name ImportExcel) { 'OK' } else { 'MANCANTE' }
    $poppl  = if (Get-Command 'pdftotext.exe' -ErrorAction SilentlyContinue) { 'OK' } else { 'MANCANTE' }

    Write-Log "  ImportExcel : $excel" (if ($excel -eq 'OK') { 'ok' } else { 'err' })
    Write-Log "  Poppler     : $poppl" (if ($poppl -eq 'OK') { 'ok' } else { 'err' })
    Write-Log ''

    if ($needsRestart) {
        Write-Log '  ATTENZIONE: PATH aggiornato nel registro.' 'warn'
        Write-Log '  Per Poppler, la sessione corrente e gia' 'warn'
        Write-Log '  aggiornata e puo essere usata subito.' 'warn'
        Write-Log '  Per sessioni future il PATH e gia salvato.' 'warn'
        $lblFinalMsg.ForeColor = [System.Drawing.Color]::DarkOrange
        $lblFinalMsg.Text      = 'PATH aggiornato — sessione corrente gia operativa!'
    } else {
        $lblFinalMsg.ForeColor = [System.Drawing.Color]::FromArgb(22, 163, 74)
        $lblFinalMsg.Text      = 'Tutto installato e pronto!'
    }

    $btnClose.Enabled = $true
    $frmDep.ShowDialog() | Out-Null
    $frmDep.Dispose()
}

# ======================================================================
# GUI
# ======================================================================

Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

function Build-TreeNodes {
    param([string]$cartella, [System.Windows.Forms.TreeNode]$parentNode)
    $items = Get-ChildItem -LiteralPath $cartella -ErrorAction SilentlyContinue |
                 Where-Object { -not $_.Name.StartsWith('.') }
    $dirs  = $items | Where-Object {  $_.PSIsContainer } | Sort-Object Name
    $files = $items | Where-Object { -not $_.PSIsContainer -and
                 ($_.Extension.ToLower() -in $ESTENSIONI -or $_.Extension.ToLower() -in $ESTENSIONI_IMMAGINI) } |
                 Sort-Object Name

    foreach ($d in $dirs) {
        if (Test-CartellaImmagini $d.FullName) {
            $nImg = (Get-ChildItem -LiteralPath $d.FullName -Recurse -ErrorAction SilentlyContinue |
                Where-Object { $_.Extension.ToLower() -in $ESTENSIONI_IMMAGINI }).Count
            $node = New-Object System.Windows.Forms.TreeNode("$($d.Name)  ``[GALLERIA: $nImg img``]")
            $node.ForeColor = [System.Drawing.Color]::MediumOrchid
        } else {
            $node = New-Object System.Windows.Forms.TreeNode($d.Name)
            $node.ForeColor = [System.Drawing.Color]::SteelBlue
        }
        $parentNode.Nodes.Add($node) | Out-Null
        Build-TreeNodes $d.FullName $node
    }
    foreach ($f in $files) {
        $ext   = $f.Extension.ToLower()
        $label = switch ($ext) {
            '.pdf'  { "``[PDF``] $($f.Name)" }
            '.docx' { "``[DOC``] $($f.Name)" }
            '.xlsx' { "``[XLS``] $($f.Name)" }
            '.xls'  { "``[XLS``] $($f.Name)" }
            default { "``[IMG``] $($f.Name)" }
        }
        $node           = New-Object System.Windows.Forms.TreeNode($label)
        $node.ForeColor = [System.Drawing.Color]::DimGray
        $parentNode.Nodes.Add($node) | Out-Null
    }
}

function Show-AlberoConferma {
    param([string]$cartella, [string]$shelfName)
    $formTree = New-Object System.Windows.Forms.Form
    $formTree.Text            = "Anteprima struttura - $shelfName"
    $formTree.Size            = New-Object System.Drawing.Size(640, 600)
    $formTree.StartPosition   = 'CenterScreen'
    $formTree.FormBorderStyle = 'FixedDialog'
    $formTree.MaximizeBox     = $false
    $formTree.BackColor       = [System.Drawing.Color]::FromArgb(245, 247, 250)
    $formTree.Font            = New-Object System.Drawing.Font('Segoe UI', 9)

    $lblHeader = New-Object System.Windows.Forms.Label
    $lblHeader.Text      = "Struttura da importare come Shelf: ""$shelfName"""
    $lblHeader.Location  = New-Object System.Drawing.Point(12, 12)
    $lblHeader.Size      = New-Object System.Drawing.Size(600, 20)
    $lblHeader.Font      = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
    $lblHeader.ForeColor = [System.Drawing.Color]::FromArgb(30, 58, 95)
    $formTree.Controls.Add($lblHeader)

    $legend = New-Object System.Windows.Forms.Label
    $legend.Text      = "Blu = Book/Chapter   |   Viola = Galleria immagini   |   Grigio = File pagina"
    $legend.Location  = New-Object System.Drawing.Point(12, 34)
    $legend.Size      = New-Object System.Drawing.Size(600, 18)
    $legend.Font      = New-Object System.Drawing.Font('Segoe UI', 8)
    $legend.ForeColor = [System.Drawing.Color]::Gray
    $formTree.Controls.Add($legend)

    $tv = New-Object System.Windows.Forms.TreeView
    $tv.Location      = New-Object System.Drawing.Point(12, 60)
    $tv.Size          = New-Object System.Drawing.Size(600, 450)
    $tv.BorderStyle   = 'FixedSingle'
    $tv.BackColor     = [System.Drawing.Color]::White
    $tv.ShowLines     = $true
    $tv.ShowPlusMinus = $true
    $tv.Font          = New-Object System.Drawing.Font('Consolas', 9)

    $rootNode           = New-Object System.Windows.Forms.TreeNode("SHELF: $shelfName")
    $rootNode.ForeColor = [System.Drawing.Color]::FromArgb(30, 58, 95)
    $rootNode.NodeFont  = New-Object System.Drawing.Font('Consolas', 9, [System.Drawing.FontStyle]::Bold)
    $tv.Nodes.Add($rootNode) | Out-Null
    Build-TreeNodes $cartella $rootNode
    $rootNode.Expand()
    foreach ($n in $rootNode.Nodes) { $n.Expand() }
    $formTree.Controls.Add($tv)

    $btnOK = New-Object System.Windows.Forms.Button
    $btnOK.Text      = 'Avvia Importazione'
    $btnOK.Location  = New-Object System.Drawing.Point(390, 524)
    $btnOK.Size      = New-Object System.Drawing.Size(150, 34)
    $btnOK.BackColor = [System.Drawing.Color]::FromArgb(22, 163, 74)
    $btnOK.ForeColor = [System.Drawing.Color]::White
    $btnOK.FlatStyle = 'Flat'
    $btnOK.FlatAppearance.BorderSize = 0
    $btnOK.Font         = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
    $btnOK.DialogResult = [System.Windows.Forms.DialogResult]::OK
    $formTree.Controls.Add($btnOK)
    $formTree.AcceptButton = $btnOK

    $btnCancel = New-Object System.Windows.Forms.Button
    $btnCancel.Text      = 'Indietro'
    $btnCancel.Location  = New-Object System.Drawing.Point(260, 524)
    $btnCancel.Size      = New-Object System.Drawing.Size(110, 34)
    $btnCancel.BackColor = [System.Drawing.Color]::FromArgb(107, 114, 128)
    $btnCancel.ForeColor = [System.Drawing.Color]::White
    $btnCancel.FlatStyle = 'Flat'
    $btnCancel.FlatAppearance.BorderSize = 0
    $btnCancel.Font         = New-Object System.Drawing.Font('Segoe UI', 9)
    $btnCancel.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
    $formTree.Controls.Add($btnCancel)
    $formTree.CancelButton = $btnCancel

    $result = $formTree.ShowDialog()
    $formTree.Dispose()
    return ($result -eq [System.Windows.Forms.DialogResult]::OK)
}

function Show-MainForm {
    $form = New-Object System.Windows.Forms.Form
    $form.Text            = 'BookStack Folder Importer v2.0'
    $form.Size            = New-Object System.Drawing.Size(560, 545)
    $form.StartPosition   = 'CenterScreen'
    $form.FormBorderStyle = 'FixedDialog'
    $form.MaximizeBox     = $false
    $form.BackColor       = [System.Drawing.Color]::FromArgb(245, 247, 250)
    $form.Font            = New-Object System.Drawing.Font('Segoe UI', 9)

    $banner = New-Object System.Windows.Forms.Panel
    $banner.Location  = New-Object System.Drawing.Point(0, 0)
    $banner.Size      = New-Object System.Drawing.Size(560, 64)
    $banner.BackColor = [System.Drawing.Color]::FromArgb(30, 58, 95)
    $form.Controls.Add($banner)

    $lblTitle = New-Object System.Windows.Forms.Label
    $lblTitle.Text      = 'BookStack Folder Importer v2.0'
    $lblTitle.Location  = New-Object System.Drawing.Point(16, 10)
    $lblTitle.Size      = New-Object System.Drawing.Size(520, 28)
    $lblTitle.Font      = New-Object System.Drawing.Font('Segoe UI', 14, [System.Drawing.FontStyle]::Bold)
    $lblTitle.ForeColor = [System.Drawing.Color]::White
    $banner.Controls.Add($lblTitle)

    $lblSub = New-Object System.Windows.Forms.Label
    $lblSub.Text      = 'Importa documenti da cartelle di rete in BookStack'
    $lblSub.Location  = New-Object System.Drawing.Point(18, 40)
    $lblSub.Size      = New-Object System.Drawing.Size(520, 18)
    $lblSub.Font      = New-Object System.Drawing.Font('Segoe UI', 8)
    $lblSub.ForeColor = [System.Drawing.Color]::FromArgb(147, 197, 253)
    $banner.Controls.Add($lblSub)

    function New-Field {
        param($parent, $labelText, $defaultVal, $topY)
        $lbl           = New-Object System.Windows.Forms.Label
        $lbl.Text      = $labelText
        $lbl.Location  = New-Object System.Drawing.Point(20, $topY)
        $lbl.Size      = New-Object System.Drawing.Size(510, 18)
        $lbl.Font      = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
        $lbl.ForeColor = [System.Drawing.Color]::FromArgb(55, 65, 81)
        $parent.Controls.Add($lbl)
        $txt              = New-Object System.Windows.Forms.TextBox
        $txt.Text         = $defaultVal
        $txt.Location     = New-Object System.Drawing.Point(20, ($topY + 20))
        $txt.Size         = New-Object System.Drawing.Size(510, 24)
        $txt.BorderStyle  = 'FixedSingle'
        $txt.BackColor    = [System.Drawing.Color]::White
        $parent.Controls.Add($txt)
        return $txt
    }

    $txtUrl     = New-Field $form 'URL BookStack'                                    $BOOKSTACK_URL    80
    $txtTokenId = New-Field $form 'Token ID'                                         $API_TOKEN_ID    130
    $txtTokenSec= New-Field $form 'Token Secret'                                     $API_TOKEN_SECRET 180
    $txtPath    = New-Field $form 'Cartella da importare'                            '\\PLACEHOLDER' 230
    $txtShelf   = New-Field $form 'Nome Shelf (come apparira in BookStack)'          'NOMESHELF'     280

    $grpOpt = New-Object System.Windows.Forms.GroupBox
    $grpOpt.Text      = 'Opzioni'
    $grpOpt.Location  = New-Object System.Drawing.Point(20, 340)
    $grpOpt.Size      = New-Object System.Drawing.Size(510, 50)
    $grpOpt.Font      = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
    $grpOpt.ForeColor = [System.Drawing.Color]::FromArgb(55, 65, 81)
    $form.Controls.Add($grpOpt)

    $chkDry = New-Object System.Windows.Forms.CheckBox
    $chkDry.Text     = 'DRY RUN (anteprima senza importare)'
    $chkDry.Location = New-Object System.Drawing.Point(12, 22)
    $chkDry.Size     = New-Object System.Drawing.Size(340, 20)
    $chkDry.Checked  = $false
    $chkDry.Font     = New-Object System.Drawing.Font('Segoe UI', 9)
    $grpOpt.Controls.Add($chkDry)

    $lblStatus = New-Object System.Windows.Forms.Label
    $lblStatus.Location  = New-Object System.Drawing.Point(20, 456)
    $lblStatus.Size      = New-Object System.Drawing.Size(510, 18)
    $lblStatus.ForeColor = [System.Drawing.Color]::Gray
    $lblStatus.Text      = ''
    $form.Controls.Add($lblStatus)

    $btnNext = New-Object System.Windows.Forms.Button
    $btnNext.Text      = 'Avanti: Anteprima struttura >'
    $btnNext.Location  = New-Object System.Drawing.Point(310, 480)
    $btnNext.Size      = New-Object System.Drawing.Size(220, 36)
    $btnNext.BackColor = [System.Drawing.Color]::FromArgb(30, 58, 95)
    $btnNext.ForeColor = [System.Drawing.Color]::White
    $btnNext.FlatStyle = 'Flat'
    $btnNext.FlatAppearance.BorderSize = 0
    $btnNext.Font      = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
    $form.Controls.Add($btnNext)

    $btnEsci = New-Object System.Windows.Forms.Button
    $btnEsci.Text      = 'Esci'
    $btnEsci.Location  = New-Object System.Drawing.Point(220, 480)
    $btnEsci.Size      = New-Object System.Drawing.Size(80, 36)
    $btnEsci.BackColor = [System.Drawing.Color]::FromArgb(107, 114, 128)
    $btnEsci.ForeColor = [System.Drawing.Color]::White
    $btnEsci.FlatStyle = 'Flat'
    $btnEsci.FlatAppearance.BorderSize = 0
    $btnEsci.Font         = New-Object System.Drawing.Font('Segoe UI', 9)
    $btnEsci.DialogResult = [System.Windows.Forms.DialogResult]::Cancel
    $form.Controls.Add($btnEsci)
    $form.CancelButton = $btnEsci

    # Bottone installa dipendenze
    $btnDeps = New-Object System.Windows.Forms.Button
    $btnDeps.Text      = 'Installa dipendenze per l''import'
    $btnDeps.Location  = New-Object System.Drawing.Point(20, 400)
    $btnDeps.Size      = New-Object System.Drawing.Size(510, 36)
    $btnDeps.BackColor = [System.Drawing.Color]::FromArgb(217, 119, 6)
    $btnDeps.ForeColor = [System.Drawing.Color]::White
    $btnDeps.FlatStyle = 'Flat'
    $btnDeps.FlatAppearance.BorderSize = 0
    $btnDeps.Font      = New-Object System.Drawing.Font('Segoe UI', 9, [System.Drawing.FontStyle]::Bold)
    $form.Controls.Add($btnDeps)

    $btnDeps.Add_Click({
        $form.Hide()
        Installa-Dipendenze
        $form.Show()
    })

    $script:guiResult = $null

    $btnNext.Add_Click({
        $url   = $txtUrl.Text.Trim()
        $tid   = $txtTokenId.Text.Trim()
        $tsec  = $txtTokenSec.Text.Trim()
        $path  = $txtPath.Text.Trim()
        $shelf = $txtShelf.Text.Trim()

        if ([string]::IsNullOrWhiteSpace($url))   { [System.Windows.Forms.MessageBox]::Show('Inserisci URL BookStack.','Errore','OK','Warning'); return }
        if ([string]::IsNullOrWhiteSpace($tid))   { [System.Windows.Forms.MessageBox]::Show('Inserisci Token ID.','Errore','OK','Warning'); return }
        if ([string]::IsNullOrWhiteSpace($tsec))  { [System.Windows.Forms.MessageBox]::Show('Inserisci Token Secret.','Errore','OK','Warning'); return }
        if ([string]::IsNullOrWhiteSpace($shelf)) { [System.Windows.Forms.MessageBox]::Show('Inserisci nome Shelf.','Errore','OK','Warning'); return }
        if (-not (Test-Path -LiteralPath $path -PathType Container)) {
            [System.Windows.Forms.MessageBox]::Show("Cartella non trovata:`n$path",'Errore','OK','Error')
            return
        }

        $script:BOOKSTACK_URL        = $url
        $script:API_TOKEN_ID         = $tid
        $script:API_TOKEN_SECRET     = $tsec
        $script:CARTELLA_ROOT        = $path
        $script:SHELF_NAME_OVERRIDE  = $shelf
        $script:DRY_RUN_GUI          = $chkDry.Checked

        $lblStatus.Text      = 'Analisi struttura cartelle...'
        $lblStatus.ForeColor = [System.Drawing.Color]::DarkOrange
        $form.Refresh()

        $ok = Show-AlberoConferma -cartella $path -shelfName $shelf

        if ($ok) {
            $lblStatus.Text      = 'Avvio importazione...'
            $lblStatus.ForeColor = [System.Drawing.Color]::Green
            $form.Refresh()
            $script:guiResult  = 'GO'
            $form.DialogResult = [System.Windows.Forms.DialogResult]::OK
            $form.Close()
        } else {
            $lblStatus.Text      = 'Tornato al menu principale.'
            $lblStatus.ForeColor = [System.Drawing.Color]::Gray
        }
    })

    $form.ShowDialog() | Out-Null
    $form.Dispose()
    return $script:guiResult
}

# ======================================================================
# MAIN
# ======================================================================

$script:BOOKSTACK_URL       = $BOOKSTACK_URL
$script:API_TOKEN_ID        = $API_TOKEN_ID
$script:API_TOKEN_SECRET    = $API_TOKEN_SECRET
$script:CARTELLA_ROOT       = $CARTELLA_ROOT
$script:SHELF_NAME_OVERRIDE = $null
$script:DRY_RUN_GUI         = $false

$esito = Show-MainForm

if ($esito -ne 'GO') {
    Write-Host 'Operazione annullata.' -ForegroundColor Yellow
    exit 0
}

$BOOKSTACK_URL    = $script:BOOKSTACK_URL
$API_TOKEN_ID     = $script:API_TOKEN_ID
$API_TOKEN_SECRET = $script:API_TOKEN_SECRET
$CARTELLA_ROOT    = $script:CARTELLA_ROOT
$DRY_RUN          = $script:DRY_RUN_GUI

Write-Host ''
Write-Host '+               -+' -ForegroundColor Cyan
Write-Host '|    BookStack Folder Importer v2.0            |' -ForegroundColor Cyan
Write-Host '+               -+' -ForegroundColor Cyan

if (-not (Get-Module -ListAvailable -Name ImportExcel)) {
    Write-Warn 'Modulo ImportExcel non trovato.'
    Write-Host '  Install-Module ImportExcel -Scope CurrentUser' -ForegroundColor White
}

if ($DRY_RUN) {
    Write-Warn 'DRY RUN selezionato - nessuna importazione eseguita.'
    exit 0
}

Write-Host ''
Write-Info "Test connessione a BookStack ($BOOKSTACK_URL)..."
if (-not (BS-TestConnessione)) {
    [System.Windows.Forms.MessageBox]::Show(
        "Impossibile connettersi a BookStack.`nURL: $BOOKSTACK_URL`n`nVerifica URL e token API.",
        'Errore di connessione', 'OK', 'Error') | Out-Null
    exit 1
}
Write-OK 'Connessione OK!'

function Importa-Tutto-GUI {
    param([string]$rootPath, [string]$shelfOverride)
    Write-Host ''
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host "  Avvio importazione: $shelfOverride" -ForegroundColor White
    Write-Host ('=' * 60) -ForegroundColor DarkGray

    $shelfNome = $shelfOverride
    Write-Host "Shelf: $shelfNome" -ForegroundColor Cyan
    $shelfId = BS-CreaShelf -nome $shelfNome -desc "Importato da: $rootPath"
    Write-OK "Shelf: '$shelfNome' (ID: $shelfId)"
    $script:stats.shelf++

    $items     = Get-ChildItem -LiteralPath $rootPath -ErrorAction SilentlyContinue | Where-Object { -not $_.Name.StartsWith('.') }
    $dirs      = $items | Where-Object {  $_.PSIsContainer } | Sort-Object Name
    $filesRoot = $items | Where-Object { -not $_.PSIsContainer -and $_.Extension.ToLower() -in $ESTENSIONI } | Sort-Object Name

    if ($filesRoot.Count -gt 0) {
        $nomeBook = "$shelfNome - Generale"
        Write-Host "Book (root): $nomeBook" -ForegroundColor Green
        try {
            $bkId = BS-CreaBook -nome $nomeBook -shelfId $shelfId
            Write-OK "Book: '$nomeBook' (ID: $bkId)"
            $script:stats.book++
            foreach ($f in $filesRoot) { Importa-File $f -bookId $bkId }
        } catch {
            Write-Err "Errore book root: $($_.Exception.Message)"
            $script:stats.errori++
        }
    }

    $generaleBookId = $null
    foreach ($d in $dirs) {
        $nome = NomePulito $d
        if (Test-CartellaImmagini $d.FullName) {
            Write-Host "Galleria: $nome" -ForegroundColor Magenta
            if (-not $generaleBookId) {
                $nomeBook = "$shelfNome - Generale"
                try {
                    $generaleBookId = BS-CreaBook -nome $nomeBook -shelfId $shelfId
                    Write-OK "Book: '$nomeBook' (ID: $generaleBookId)"
                    $script:stats.book++
                } catch {
                    Write-Err "Errore book Generale: $($_.Exception.Message)"
                    $script:stats.errori++; continue
                }
            }
            Importa-CartellaImmagini -cartella $d.FullName -nomePageTitle $nome -bookId $generaleBookId
            continue
        }
        Write-Host "Book: $nome" -ForegroundColor Green
        try {
            $bkId = BS-CreaBook -nome $nome -shelfId $shelfId
            Write-OK "Book: '$nome' (ID: $bkId)"
            $script:stats.book++
        } catch {
            Write-Err "Errore book '$nome': $($_.Exception.Message)"
            $script:stats.errori++; continue
        }
        Importa-Cartella $d.FullName $bkId 1
    }

    Write-Host ''
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host '  Importazione completata!' -ForegroundColor Green
    Write-Host ('=' * 60) -ForegroundColor DarkGray
    Write-Host "  Shelf    : $($stats.shelf)"
    Write-Host "  Books    : $($stats.book)"
    Write-Host "  Chapters : $($stats.chapter)"
    Write-Host "  Pagine   : $($stats.page)"   -ForegroundColor Green
    if ($stats.errori -gt 0) { Write-Host "  Errori   : $($stats.errori)" -ForegroundColor Red }
    Write-Host ('=' * 60) -ForegroundColor DarkGray

    [System.Windows.Forms.MessageBox]::Show(
        "Importazione completata!`n`n  Shelf    : $($stats.shelf)`n  Books    : $($stats.book)`n  Chapters : $($stats.chapter)`n  Pagine   : $($stats.page)`n  Errori   : $($stats.errori)",
        'BookStack Importer - Completato', 'OK', 'Information') | Out-Null
}

Importa-Tutto-GUI -rootPath $CARTELLA_ROOT -shelfOverride $script:SHELF_NAME_OVERRIDE

```