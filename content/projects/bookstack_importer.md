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

 
```
