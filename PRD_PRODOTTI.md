# PRD_PRODOTTI — Documentazione Tecnica

**Progetto:** DataHub / Area Flow `PRD_PRODOTTI`
**Ambito:** Sincronizzazione dell'anagrafica prodotti tra Microsoft Dynamics 365 Business Central, il data warehouse DataHub, l'applicazione di configurazione commerciale Apparound e il sistema gestionale AS400
**Versione documento:** 1.0

---

## Indice

1. [Introduzione e scopo del progetto](#1-introduzione-e-scopo-del-progetto)
2. [Architettura generale](#2-architettura-generale)
3. [Componenti principali](#3-componenti-principali)
4. [Meccanismi comuni](#4-meccanismi-comuni)
5. [Schema ER delle tabelle](#5-schema-er-delle-tabelle)
6. [Tabelle, stored procedure e function](#6-tabelle-stored-procedure-e-function)
7. [Tabelle di logging/errori](#7-tabelle-di-loggingerrori)
8. [Sequenza esecutiva di una elaborazione](#8-sequenza-esecutiva-di-una-elaborazione)
9. [Interazioni principali fra i componenti](#9-interazioni-principali-fra-i-componenti)
10. [Descrizione dei componenti di ogni package](#10-descrizione-dei-componenti-di-ogni-package)

---

## 1. Introduzione e scopo del progetto

**PRD_PRODOTTI** è l'area del **Flow Framework** (documentato separatamente in `SSIS_Flow_Framework.md`) dedicata alla gestione end-to-end dell'anagrafica prodotti aziendale: prodotti, distinte base, prezzi, bundle commerciali e regole di configurazione.

Il progetto risolve un problema tipico delle aziende manifatturiere con più sistemi verticali: l'anagrafica prodotto "master" vive in **Business Central** (ERP, ex Dynamics NAV), ma deve essere resa disponibile, in forma coerente e aggiornata, a due sistemi consumer con esigenze molto diverse tra loro:

- **Apparound**, la piattaforma CPQ (*Configure, Price, Quote*) utilizzata dalla rete commerciale per configurare offerte e preventivi, che richiede prodotti, prezzi, distinte base "esplose", bundle e regole di compatibilità in formato JSON via API REST;
- **AS400**, il sistema gestionale/produttivo storico, che richiede le distinte base in formato tabellare tramite Linked Server.

Il progetto **PRD_PRODOTTI** orchestra, tramite il Flow Framework, l'intera catena: estrazione dei dati da Business Central, riconciliazione tramite **Change Data Capture** (confronto tra dato importato e dato consolidato, con individuazione delle sole variazioni), propagazione delle variazioni ad Apparound via API e aggiornamento delle distinte base su AS400. In questo modo ciascun sistema riceve solo le informazioni realmente cambiate, riducendo il carico sui sistemi di destinazione e mantenendo un log puntuale di ogni sincronizzazione.

---

## 2. Architettura generale

Il progetto segue un flusso lineare a quattro macro-stadi, tutti orchestrati come fasi dell'area `PRD_PRODOTTI` dal motore **FlowEngineManager.dtsx** del Flow Framework.

![Architettura generale](diagrams_prd_prodotti/01_architettura_generale.png)

**Descrizione del flusso:**

1. **Estrazione dati da Business Central verso DataHub (fasi 1–12).** I package `PRD_01`…`PRD_12` si collegano a Business Central tramite un componente sorgente dedicato (*Dynamics NAV Source*) ed effettuano un caricamento "truncate & load" delle entità rilevanti (articoli, caratteristiche configurabili, distinte base di produzione, listini prezzi, attributi) nelle tabelle dello schema **Staging** (`Staging.BC_*`). Questo stadio produce sempre una **fotografia integrale** e aggiornata dei dati sorgente.

2. **Change Data Capture: da Staging a Products (fasi 21–25).** I package `PRD_21`…`PRD_25` invocano le stored procedure di refresh (`Products.BC_Items_Refresh`, `BC_Prices_Refresh`, `BC_DBase_Refresh`, `BC_Bundle_Refresh`, `BC_Rule_Refresh`). Ciascuna procedura confronta i dati appena importati (schema `Staging`) con lo stato consolidato precedente (schema `Products`), determina le righe nuove/modificate, le applica alle tabelle `Products.*` e, soprattutto, **traccia puntualmente ogni variazione** in apposite tabelle delta (`Staging.BC_*_T`). Questo è il vero meccanismo di CDC del progetto: non un CDC nativo SQL Server, ma un confronto applicativo che produce un log dettagliato di ciò che è cambiato in ciascuna esecuzione (transazione).

3. **Aggiornamento dell'applicazione Apparound (fasi 31–37).** I package `PRD_31`…`PRD_37` leggono esclusivamente le righe **delta** prodotte al punto precedente (tramite le viste `Products.V_Apparound_*` e `Products.V_DBase_Apparound`), le trasformano in JSON e le inviano via **API REST** ad Apparound (autenticazione OAuth con token recuperato dinamicamente, endpoint e credenziali configurati in `Products.Api_Config`). Ogni chiamata viene tracciata in `Products.Api_Log`/`Api_Log_Ok`.

4. **Aggiornamento delle distinte base su AS400 (fase 41).** Il package `PRD_41` invoca `Products.AS400_DBase_Refresh`, che legge la distinta base consolidata (`Products.DBase_BC`, `Products.Prod_Prodotti`) e la scrive sulle tabelle native AS400 (`PRI_COM3.YCOAC00F`, `PRI_COM3.YCODC00F`) tramite **Linked Server** (`DBAS400`) e istruzioni `INSERT INTO OPENQUERY(...)`.

Questa architettura a stadi separati (import → riconciliazione → distribuzione) consente di isolare le responsabilità: l'import non applica logica di business, la riconciliazione non conosce i sistemi di destinazione, e la distribuzione non conosce il formato sorgente Business Central — ogni stadio comunica con il successivo esclusivamente tramite tabelle del database DataHub.

---

## 3. Componenti principali

Il progetto è composto da **24 package SSIS**, orchestrati come fasi dell'area `PRD_PRODOTTI` (si veda `Fasi.csv`), da un insieme di stored procedure e function nello schema `Products`, e da un ampio schema dati suddiviso tra `Staging` (dati grezzi importati) e `Products` (dati anagrafici consolidati, distinte base, prezzi, bundle, regole, log).

![Componenti principali](diagrams_prd_prodotti/02_componenti_principali.png)

| Gruppo di fasi | Package `.dtsx` | Stored procedure / meccanismo | Tabelle principali |
|---|---|---|---|
| **1–12 · Import BC** | `PRD_01_BC_Items` … `PRD_12_BC_CFGRuleRelations` | Data Flow con sorgente *Dynamics NAV*; truncate preliminare | `Staging.BC_Items`, `BC_CFGCharacValues`, `BC_CFGRuleCharacValues`, `BC_CFGItemVarCharacs`, `BC_ProductionBomLines/Header`, `BC_PriceListLine`, `BC_PricesMatrix`, `BC_ItemAttribute`, `BC_ItemAttributeValue`, `BC_ItemAttributeValueMapping`, `BC_CFGRuleRelations` |
| **21–25 · Refresh/CDC** | `PRD_21_BC_Items_Refresh` … `PRD_25_BC_Rule_Refresh` | `Products.BC_Items_Refresh`, `BC_Prices_Refresh`, `BC_DBase_Refresh`, `BC_Bundle_Refresh`, `BC_Rule_Refresh` | `Products.Main*`, `ATS*`, `STK*`, `CnC*`, `Prod_*`, `DBase_BC`, `DBase_Head`, `Bundle_Head/BC`, `Rule_Head/BC/Config`, `Prices`; delta in `Staging.BC_*_T` |
| **31–37 · Integrazione Apparound** | `PRD_31_Apparound_Products` … `PRD_37_Apparound_Rules` | `Products.Apparound_Check_Anomalie`, `Products.Apparound_DBase_Refresh`, chiamate REST via Script Component | `Products.Api_Config`, `Api_Log`, `Api_Log_Ok`, viste `V_Apparound_Products/Prices`, `V_DBase_Apparound`, `V_Main_Clienti` |
| **41 · Refresh AS400** | `PRD_41_AS400_DBase_Refresh` | `Products.AS400_DBase_Refresh` (Linked Server `DBAS400`) | `Products.DBase_BC`, `Prod_Prodotti`, tabelle AS400 `PRI_COM3.YCOAC00F/YCODC00F` |
| **Trasversali** | — | `Products.LogProcess`, `Products.LogErrors`, `Products.TypeOfErrors`, function `Products.fn_GetLevelProducts`, `fn_GetTopLevelProducts`, `fn_BuildDBProductJson` | log applicativo, anomalie dati, calcolo livelli di distinta base |

L'elenco completo, ordinato, delle 24 fasi (con relativo package) è riportato in [Sezione 10](#10-descrizione-dei-componenti-di-ogni-package).

---

## 4. Meccanismi comuni

### 4.1 Logging

Il progetto adotta **due livelli di logging distinti e complementari**:

- **Logging di framework** (ereditato dal Flow Framework): ogni fase (`PRD_xx`) viene tracciata da `Flow.TransazioniFase`, con orari di inizio/fine e riferimento all'`execution_id` SSISDB; eventuali fallimenti tecnici (errori bloccanti dell'Execute SQL Task, timeout, errori di connessione) vengono intercettati dall'Event Handler `OnError` di `FlowEngineManager.dtsx` e registrati in `Flow.TransazioniErrori`.
- **Logging applicativo di progetto**: ogni stored procedure di refresh scrive un log dettagliato passo-passo (conteggio insert/update per ciascuna entità elaborata, esito di commit/rollback) in `Products.LogProcess`, associato alla transazione (`Flow.TransazioniArea.transazione`) e al nome del processo (es. `BC_Items_Refresh`). Questo log applicativo è indipendente dal log tecnico del framework e sopravvive anche quando l'esecuzione si conclude con successo, fornendo una tracciabilità puntuale di *cosa* è stato importato/modificato in ogni transazione.

### 4.2 Gestione degli errori

Anche qui convivono due meccanismi:

- **Errori bloccanti** (violazioni di vincoli, timeout, errori di rete verso Business Central/Apparound/AS400): le stored procedure di refresh e di integrazione sono scritte con blocco `BEGIN TRY / BEGIN CATCH`; in caso di eccezione, la transazione SQL viene sottoposta a `ROLLBACK`, l'errore viene registrato in `Products.LogProcess` e infine viene rilanciato con `THROW`. Il rilancio fa fallire il task SSIS, il che a sua volta fa fallire il package e attiva la gestione errori centralizzata del Flow Framework (registrazione in `Flow.TransazioniErrori`, transazione portata in stato bloccato, notifica e-mail).
- **Anomalie di dato** (non bloccanti: es. un articolo referenziato in una distinta base ma non presente in anagrafica): vengono rilevate durante l'elaborazione e scritte in `Products.LogErrors`, con riferimento al processo e alla transazione, e classificate tramite `Products.TypeOfErrors`. A differenza degli errori bloccanti, queste anomalie **non fermano l'elaborazione**: al termine della procedura, se sono presenti righe in `LogErrors` per quella transazione/processo, viene inviata una **notifica e-mail dedicata** (tramite `msdb.dbo.sp_send_dbmail`, con destinatari/oggetto configurati per fase in `Flow.FasiAnomalie`), distinta dalla notifica generale di errore del framework. Anche l'integrazione con Apparound utilizza lo stesso pattern tramite `Products.Apparound_Check_Anomalie`, che invia una notifica riepilogativa se sono presenti righe di errore in `Products.Api_Log` per la transazione/fase corrente.

> **Nota:** la tabella `Flow.FasiAnomalie`, richiamata dalle stored procedure per determinare destinatari e oggetto delle notifiche di anomalia, non è presente tra gli script DDL forniti; la sua struttura (colonne `area`, `fase_desc`, `to_address`, `cc_address`, `subject`) è stata dedotta dall'utilizzo che ne fanno le stored procedure.

### 4.3 Parametrizzazione

Tutti i package ricevono dal framework, tramite parametro di progetto SSISDB, l'identificativo della **transazione corrente** (`prmTransazione`), che viene passato come parametro `@Transazione` a ogni stored procedure invocata. Questo singolo parametro è la chiave che lega tutte le tabelle delta (`Staging.BC_*_T`), i log applicativi (`LogProcess`, `LogErrors`, `Api_Log`) e le tabelle di runtime del framework (`Flow.TransazioniArea/Fase`), permettendo di ricostruire in ogni momento l'intera catena di elaborazione di una singola esecuzione. Le stored procedure di refresh ricevono inoltre il parametro `@Refresh` (`'Y'`/`'N'`), che consente di eseguire la sola fase di analisi/logging delle differenze senza applicarle fisicamente alle tabelle `Products.*` — utile per verifiche o esecuzioni di controllo.

### 4.4 Connection Manager condivisi

Tutti i package che eseguono direttamente logica T-SQL (fasi 1–12, 21–25, 41) condividono il medesimo connection manager verso il database `DataHub` (`SRVSQL02\DWH.datahub`), oltre al connection manager verso Business Central (per l'estrazione dati) nei package di import. L'accesso ad **AS400** non richiede un connection manager SSIS dedicato: avviene interamente lato server tramite il **Linked Server `DBAS400`**, configurato una sola volta a livello di istanza SQL Server e richiamato dalle stored procedure con `OPENQUERY`. Analogamente, le credenziali e gli endpoint di **Apparound** non sono cablati nei package ma centralizzati nella tabella di configurazione `Products.Api_Config` (chiave applicazione/ambiente), consultata dinamicamente a ogni esecuzione.

### 4.5 Restart

Il progetto **non implementa un proprio meccanismo di restart**: si appoggia interamente su quello del Flow Framework descritto in `SSIS_Flow_Framework.md`. Se una fase fallisce, l'area `PRD_PRODOTTI` (se marcata `restartable = 1` in `Flow.Aree`) può essere rilanciata automaticamente da `FlowRestartArea.dtsx`, che riprende l'esecuzione dalla fase interrotta (checkpoint a livello di `Flow.TransazioniFase`). Poiché ciascuna stored procedure di refresh opera all'interno di una propria transazione SQL con `ROLLBACK` in caso di errore, un riavvio della fase non lascia mai lo schema `Products` in uno stato parzialmente aggiornato: o la fase è stata applicata per intero, o non è stata applicata affatto.

---

## 5. Schema ER delle tabelle

Data l'ampiezza dello schema (oltre 70 tabelle complessive tra `Staging` e `Products`), il diagramma seguente si concentra sulle **entità core del flusso di sincronizzazione**: la catena delta (import → CDC → distribuzione) e le tabelle di logging. Le tabelle anagrafiche di supporto (famiglie, serie, modelli, varianti, caratteristiche tecniche, classi di stock, ecc.) sono descritte a livello di elenco nella [Sezione 6](#6-tabelle-stored-procedure-e-function).

![Schema ER](diagrams_prd_prodotti/03_er_diagram.png)

Lettura del diagramma:

- `BC_Items` (Staging, dato grezzo importato da Business Central) genera, tramite `BC_Items_Refresh`, le righe delta in `BC_Items_T`, secondo lo stesso pattern seguito da tutte le altre coppie import/delta del progetto (`BC_Prices_T`, `BC_DBase_T`, `BC_Bundle_T`/`BC_Bundle_Head_T`, `BC_Rule_T`/`BC_Rule_Head_T`).
- `Prod_Prodotti` è l'anagrafica prodotto consolidata; da essa dipendono la distinta base (`DBase_BC`), il prezzo (`Prices`), i bundle (`Bundle_Head`/`Bundle_BC`) e le regole di configurazione (`Rule_Head`/`Rule_BC`).
- `DBase_BC` (distinta base "piatta", un livello) viene esplosa ricorsivamente in `DBase_Apparound` (distinta base multilivello, con `Level` e `Ordering`), il formato consumato dalle viste di estrazione per Apparound.
- `Api_Config` fornisce le credenziali per autenticare le chiamate tracciate in `Api_Log`; le chiamate concluse con successo vengono inoltre duplicate in `Api_Log_Ok`.
- `LogProcess` è il log applicativo trasversale a tutte le stored procedure; `TypeOfErrors` è l'anagrafica di classificazione degli errori applicativi registrati in `LogErrors`.

---

## 6. Tabelle, stored procedure e function

### 6.1 Schema `Staging` — dati importati da Business Central

Ogni tabella corrisponde a un'entità estratta integralmente ("truncate & load") a ogni esecuzione della relativa fase di import (1–12). Le tabelle con suffisso **`_T`** sono invece le tabelle **delta**, popolate dalle stored procedure di refresh (fasi 21–25) con le sole righe risultate nuove o modificate rispetto allo stato precedente, taggate con `transazione` e, dove previsto, con `rowStatus` (`INSERT`/`UPDATE`) ed `eventTime`.

| Tabella | Alimentata da | Significato |
|---|---|---|
| `Staging.BC_Items` | PRD_01 | Articoli (item) importati da Business Central: codice, descrizione, stato attivo. |
| `Staging.BC_CFGCharacValues` | PRD_02 | Valori delle caratteristiche di configurazione prodotto (characteristics) definite in BC. |
| `Staging.BC_CFGRuleCharacValues` | PRD_03 | Valori delle caratteristiche utilizzate nelle regole di configurazione. |
| `Staging.BC_CFGItemVarCharacs` | PRD_04 | Associazione tra articoli/varianti e caratteristiche configurabili. |
| `Staging.BC_ProductionBomLines` | PRD_05 | Righe (componenti) delle distinte base di produzione. |
| `Staging.BC_ProductionBomHeader` | PRD_06 | Testate delle distinte base di produzione. |
| `Staging.BC_PriceListLine` | PRD_07 | Righe dei listini prezzo. |
| `Staging.BC_PricesMatrix` | PRD_08 | Matrice prezzi (combinazioni articolo/variante/condizione), tabella con il maggior numero di colonne (79) del progetto. |
| `Staging.BC_ItemAttribute` | PRD_09 | Anagrafica attributi articolo. |
| `Staging.BC_ItemAttributeValue` | PRD_10 | Valori ammessi per ciascun attributo articolo. |
| `Staging.BC_ItemAttributeValueMapping` | PRD_11 | Mappatura valore attributo ↔ articolo. |
| `Staging.BC_CFGRuleRelations` | PRD_12 | Relazioni/condizioni tra regole di configurazione (compatibilità, esclusioni). |
| `Staging.BC_Items_T` | BC_Items_Refresh | Delta articoli: righe nuove/modificate, con stato precedente (`...Old`) e nuovo (`...New`) per i principali campi. |
| `Staging.BC_Item_Attributes_T` | BC_Items_Refresh | Delta sugli attributi/caratteristiche di prodotto derivati dagli articoli (macrofamiglie, serie, modelli, ecc.). |
| `Staging.BC_Prices_T` | BC_Prices_Refresh | Delta sui prezzi. |
| `Staging.BC_DBAse_T` | BC_DBase_Refresh | Delta sulla distinta base "piatta" (`Products.DBase_BC`). |
| `Staging.DBAse_Head_T` | BC_DBase_Refresh | Delta sulle testate di distinta base (macchine il cui albero componenti è variato); utilizzata anche da `Apparound_DBase_Refresh` e da `AS400_DBase_Refresh` per limitare l'elaborazione alle sole macchine interessate. |
| `Staging.BC_Bundle_T` / `BC_Bundle_Head_T` | BC_Bundle_Refresh | Delta su righe e testate dei bundle commerciali. |
| `Staging.BC_Rule_T` / `BC_Rule_Head_T` | BC_Rule_Refresh | Delta su righe e testate delle regole di configurazione. |

### 6.2 Schema `Products` — dati anagrafici consolidati

Lo schema `Products` raccoglie, oltre alle tabelle di distribuzione (distinte base, prezzi, bundle, regole), un ricco insieme di **anagrafiche di supporto** organizzate in famiglie coerenti, riconoscibili dal prefisso.

#### 6.2.1 Distinte base, prezzi, bundle, regole (core del flusso)

| Tabella | Significato |
|---|---|
| `Products.Prod_Prodotti` | Anagrafica prodotto consolidata: `CodiceProdotto` (PK), descrizione, `CodiceDerivazione` (tipologia/derivazione del codice), categoria, famiglia, flag `Attivo`. È l'entità centrale da cui dipendono distinta base, prezzo, bundle e regole. |
| `Products.Prod_AS400` | Vista/estratto dell'anagrafica prodotto nel formato richiesto per l'esportazione verso AS400. |
| `Products.Prod_Ghost` | Prodotti "fantasma"/segnaposto usati nella costruzione della distinta base (nodi intermedi non commerciali). |
| `Products.Prod_Categorie`, `Prod_Famiglie`, `Prod_Derivazioni`, `Prod_Attributi`, `Prod_Opzioni`, `Prod_All` | Tabelle di codifica/anagrafica di supporto per categoria, famiglia, derivazione, attributi e opzioni prodotto. |
| `Products.DBase_BC` | Distinta base "piatta" a un livello: `CodiceProdotto`/`CodiceComponente` (PK composta), `Quantity`, `UdM`, `Posizione`, `POVisible` (visibilità nell'ordine fornitore), `Attivo`. Alimentata da `BC_DBase_Refresh`. |
| `Products.DBase_BC_Macchina` | Distinta base specifica per macchina (`CodiceMacchina` + componente), usata come base per la vista `V_DBase_Apparound`. |
| `Products.DBase_Head` | Testate di distinta base: legame `CodiceMainCliente` – `CodiceCnC` – `CodiceMacchina`. |
| `Products.DBase_STD` | Legami di distinta base "standard" (`IDLegame`), variante semplificata di `DBase_BC`. |
| `Products.DBase_Apparound` | Distinta base **esplosa multilivello** (`Level`, `Ordering`) nel formato consumato da Apparound; alimentata da `Apparound_DBase_Refresh`. |
| `Products.DBase_Apparound_Config` | Configurazione dei livelli di esplosione per categoria/famiglia, usata dal motore di esplosione della distinta base. |
| `Products.Prices` / `Prices_save` | Prezzi per prodotto/versione/unità di misura (`Prezzo`, `PrezzoAggiuntivo`); `Prices_save` è una copia di salvataggio con la medesima struttura. |
| `Products.Bundle_Head` | Testata bundle: macchina, prodotto principale, codice bundle, `IDBundle`. |
| `Products.Bundle_BC` | Righe bundle: componenti del bundle con quantità e regola applicata. |
| `Products.Rule_Head` | Testata regola di configurazione: macchina, prodotto, quantità richiesta, elenco codici prodotto coinvolti. |
| `Products.Rule_BC` | Righe regola: componente, quantità minima/massima, condizione, `IDRule`. |
| `Products.Rule_Config` | Variante di configurazione delle regole (stessa struttura di `Rule_BC` con `Regola` obbligatoria). |

#### 6.2.2 Anagrafiche di supporto "Main" (gerarchia commerciale macchina)

| Tabella | Significato |
|---|---|
| `Products.Main` | Anagrafica prodotto vista secondo la gerarchia commerciale: fornitore, serie, modello, variante, macrofamiglia, caratteristica tecnica. |
| `Products.Main_MacroFamiglie` | Macrofamiglie di prodotto. |
| `Products.Main_Fornitori`, `Main_Serie`, `Main_Modelli`, `Main_Varianti` | Gerarchia fornitore → serie → modello → variante. |
| `Products.Main_Macchine` | Anagrafica delle macchine (prodotti "macchina", radice della distinta base). |
| `Products.Main_CaratteristicheTecniche`, `Main_TabellaCaratteristicheTecniche`, `Main_TipoCaratteristicheTecniche` | Caratteristiche tecniche associate a fornitore/serie/modello/variante e relativa tipizzazione. |
| `Products.Main_Clienti` | Vista tabellare di supporto per la vista `V_Main_Clienti` (anagrafica prodotto lato "cliente finale"). |

#### 6.2.3 Anagrafiche di supporto "ATS", "CnC", "STK"

Queste tre famiglie rappresentano viste alternative/parallele della stessa gerarchia prodotto, utilizzate da domini funzionali diversi (rispettivamente: attributi tecnici specifici, cicli e configurazioni CnC, struttura a "kit"/stock):

| Tabella | Significato |
|---|---|
| `Products.ATS`, `ATS_Area`, `ATS_CaratteristicheTecniche`, `ATS_CicliDiLavoro`, `ATS_Fornitori`, `ATS_ListinoFornitori`, `ATS_Modelli`, `ATS_Serie`, `ATS_Varianti` | Gerarchia prodotto "ATS": fornitore → serie → modello → variante, con cicli di lavoro e listini fornitore associati. |
| `Products.CnC`, `Cnc_Fornitori`, `Cnc_Modelli`, `Cnc_Serie` | Gerarchia prodotto semplificata "CnC": fornitore → serie → modello. |
| `Products.STK`, `STK_Classi`, `STK_Specifiche`, `STK_UdM`, `STK_Valori` | Anagrafica prodotto per componenti/kit di stock, classificati per classe, specifica, unità di misura e valore. |

### 6.3 Stored procedure

| Stored procedure | Fase che la invoca | Descrizione |
|---|---|---|
| `Products.BC_Items_Refresh` | PRD_21 | Confronta `Staging.BC_Items` e le tabelle di configurazione articolo con le anagrafiche `Products.Main*`/`ATS*`/`CnC*`/`STK*`/`Prod_*`; applica inserimenti/aggiornamenti (se `@Refresh='Y'`) e produce il delta in `Staging.BC_Items_T` e `BC_Item_Attributes_T`. |
| `Products.BC_Prices_Refresh` | PRD_22 | Confronta `Staging.BC_PriceListLine`/`BC_PricesMatrix` con `Products.Prices`; applica le variazioni e produce il delta in `Staging.BC_Prices_T`. |
| `Products.BC_DBase_Refresh` | PRD_23 | Confronta `Staging.BC_ProductionBomLines/Header` con `Products.DBase_BC`/`DBase_Head`/`DBase_BC_Macchina`; applica le variazioni e produce il delta in `Staging.BC_DBAse_T` e `DBAse_Head_T`. |
| `Products.BC_Bundle_Refresh` | PRD_24 | Confronta i dati bundle importati con `Products.Bundle_Head`/`Bundle_BC`; applica le variazioni e produce il delta in `Staging.BC_Bundle_T`/`BC_Bundle_Head_T`. |
| `Products.BC_Rule_Refresh` | PRD_25 | Confronta i dati regole importati con `Products.Rule_Head`/`Rule_BC`/`Rule_Config`, utilizzando `Products.fn_GetLevelProducts` per determinare i livelli di distinta coinvolti; applica le variazioni e produce il delta in `Staging.BC_Rule_T`/`BC_Rule_Head_T`. |
| `Products.Apparound_Check_Anomalie` | PRD_31…PRD_37 | Verifica se, per la transazione e la fase correnti, esistono righe di errore in `Products.Api_Log`; in caso positivo invia una notifica e-mail riepilogativa (destinatari/oggetto da `Flow.FasiAnomalie`). |
| `Products.Apparound_DBase_Refresh` | PRD_35 | Esplode la distinta base "piatta" (`DBase_BC`/`DBase_Head`) nel formato multilivello richiesto da Apparound, scrivendo il risultato in `Products.DBase_Apparound`, limitandosi alle macchine presenti nel delta `Staging.DBAse_Head_T`. |
| `Products.AS400_DBase_Refresh` | PRD_41 | Legge la distinta base consolidata (`DBase_BC`, `Prod_Prodotti`) per le sole macchine variate (`Staging.DBAse_Head_T`) e la scrive sulle tabelle native AS400 `PRI_COM3.YCOAC00F` (anagrafica) e `PRI_COM3.YCODC00F` (distinta base) tramite Linked Server `DBAS400` e `INSERT INTO OPENQUERY(...)`. |

Tutte le stored procedure di refresh condividono lo stesso pattern implementativo: `BEGIN TRY` → `BEGIN TRAN` → confronto Staging/Products con logging progressivo in una tabella di appoggio → applicazione delle modifiche (se abilitata) → popolamento delle tabelle delta → `COMMIT TRAN` → scrittura del log in `Products.LogProcess` → invio eventuale notifica di anomalia; in caso di eccezione: `ROLLBACK TRAN`, log dell'errore, `THROW`.

### 6.4 Function

| Function | Tipo | Descrizione |
|---|---|---|
| `Products.fn_GetLevelProducts` | Table-valued (CTE ricorsiva) | Dato un `CodiceMacchina` e un `CodiceComponente`, risale ricorsivamente la distinta base (`Products.V_DBase_Apparound`) fino al livello massimo raggiungibile, restituendo la sequenza di prodotti attraversati in formato stringa CSV pronta per l'uso in messaggi/log. Utilizzata da `BC_Rule_Refresh` per determinare i livelli di distinta coinvolti dalle regole di configurazione. |
| `Products.fn_GetTopLevelProducts` | Table-valued (CTE ricorsiva) | Dato un `CodiceProdotto`/`CodiceComponente`, risale la distinta base "piatta" (`Products.DBase_BC`) per individuare i prodotti di livello più alto (macchine, presenti in `Products.Main_Macchine`) che, direttamente o indirettamente, contengono quel componente. |
| `Products.fn_BuildDBProductJson` | Scalare (nvarchar(max), ricorsiva) | Costruisce ricorsivamente la rappresentazione **JSON gerarchica** della distinta base Apparound (`Products.DBase_Apparound`) a partire da un nodo radice, con struttura `{"productCode":...,"enabled":true,"children":[...]}`; usata per generare payload di distinta base annidata da esporre/consultare in formato JSON. |

### 6.5 Viste

| Vista | Utilizzata da | Descrizione |
|---|---|---|
| `Products.V_Apparound_Products` | PRD_31 | Proietta l'anagrafica prodotto (`Products.Main`, gerarchia fornitore/serie/modello/variante, macrofamiglie) nel formato JSON-ready richiesto dall'API "Products" di Apparound (codice, nome, categoria, filtri/attributi commerciali). |
| `Products.V_Apparound_Prices` | PRD_32 | Proietta i prezzi nel formato richiesto dall'API "Prices" di Apparound (modello di pricing, importi one-off/recurring, unità di misura). |
| `Products.V_DBase_Apparound` | PRD_35, `fn_GetLevelProducts` | Costruisce la distinta base "vista Apparound" unendo più livelli (macchina → richiesta speciale, macchina → componenti STK, macchina → "T-Macchina" → prodotti T) in un'unica struttura `CodiceMacchina`/`CodiceProdotto`/`CodiceComponente` uniforme, base per l'esplosione multilivello. |
| `Products.V_Main_Clienti` | PRD_31, `BC_Bundle_Refresh`, `BC_Prices_Refresh` | Combina `Products.Main` con la gerarchia fornitore/serie/modello/variante/macrofamiglia in un'unica riga per prodotto, con descrizione commerciale completa. |

---

## 7. Tabelle di logging/errori

Il progetto utilizza quattro tabelle di logging applicativo, distinte dalle tabelle di runtime del Flow Framework (`Flow.TransazioniArea/Fase/Errori`, documentate in `SSIS_Flow_Framework.md`).

### 7.1 `Products.LogProcess`

Log applicativo sequenziale prodotto da ogni stored procedure di refresh/integrazione: una riga per ogni passo significativo (conteggio insert/update per entità, esito di commit/rollback, eventuale messaggio di errore tecnico).

| Colonna | Tipo | Significato |
|---|---|---|
| `IDLog` (PK) | `int identity` | Identificativo univoco della riga di log. |
| `transazione` | `int` | Transazione Flow a cui si riferisce l'esecuzione (`Flow.TransazioniArea.transazione`). |
| `process` | `nvarchar(50)` | Nome della stored procedure/processo che ha generato la riga (es. `BC_Items_Refresh`). |
| `log_message` | `nvarchar(4000)` | Messaggio descrittivo del passo eseguito (es. `"Main - Macrofamiglia - Insert: 3"`, `"OK - Commit Trans"`, oppure il dettaglio di un'eccezione). |
| `eventTime` | `datetime` | Istante di registrazione del passo. |

### 7.2 `Products.LogErrors`

Anomalie di dato non bloccanti, rilevate durante l'elaborazione (es. riferimenti mancanti, incoerenze tra fonti).

| Colonna | Tipo | Significato |
|---|---|---|
| `IDError` (PK) | `int identity` | Identificativo univoco della riga di anomalia. |
| `Transazione` | `int` | Transazione Flow in cui è stata rilevata l'anomalia. |
| `Process` | `nvarchar(50)` | Processo (stored procedure) che ha rilevato l'anomalia. |
| `Codice` | `nvarchar(100)` | Codice/chiave del dato anomalo (es. codice prodotto o combinazione prodotto-componente). |
| `Messaggio` | `nvarchar(100)` | Descrizione sintetica dell'anomalia. |
| `ErrorNumber` | `int` | Codice di classificazione dell'anomalia, riferito a `Products.TypeOfErrors`. |

### 7.3 `Products.TypeOfErrors`

Anagrafica di classificazione delle tipologie di anomalia registrabili in `LogErrors`.

| Colonna | Tipo | Significato |
|---|---|---|
| `ErrorNumber` (PK) | `int` | Codice numerico del tipo di errore. |
| `Description` | `varchar(100)` | Descrizione della tipologia di errore. |
| `Severity` | `int` | Livello di gravità associato. |

### 7.4 `Products.Api_Log` / `Products.Api_Log_Ok`

Log delle chiamate REST effettuate verso Apparound durante le fasi 31–37.

| Colonna | Tipo | Significato | Presente in |
|---|---|---|---|
| `Transazione` | `int` | Transazione Flow della chiamata. | Api_Log, Api_Log_Ok |
| `AppKey` | `nvarchar(50)` | Applicazione di destinazione (`'Apparound'`). | Api_Log, Api_Log_Ok |
| `ApiKey` | `nvarchar(200)` | Identifica l'operazione/entità chiamata (es. `Products`, `Prices`, `Bundles`, `Rules`). | Api_Log, Api_Log_Ok |
| `LogKey` | `nvarchar(50)` | Chiave del record inviato (es. codice prodotto). | Api_Log, Api_Log_Ok |
| `Request` | `nvarchar(max)` | Corpo della richiesta HTTP (payload JSON) inviata. | solo Api_Log |
| `StatusCode` | `nvarchar(200)` | Codice di esito della chiamata HTTP. | Api_Log, Api_Log_Ok |
| `Response` | `nvarchar(max)` | Corpo della risposta ricevuta dall'API. | solo Api_Log |
| `ExecTime` | `datetime` | Istante di esecuzione della chiamata. | Api_Log, Api_Log_Ok |

`Api_Log` registra **ogni** chiamata (con payload e risposta completi, utile a scopo diagnostico), mentre `Api_Log_Ok` registra in forma sintetica le sole chiamate concluse con successo, per un log leggero delle sincronizzazioni andate a buon fine.

---

## 8. Sequenza esecutiva di una elaborazione

Il diagramma seguente descrive la sequenza tipica di un'esecuzione completa dell'area `PRD_PRODOTTI`, dall'import da Business Central fino all'aggiornamento di Apparound e AS400.

![Sequenza esecutiva](diagrams_prd_prodotti/04_sequenza_esecutiva.png)

**Passi principali:**

1. **Import (fasi 1–12).** Per ciascuna entità Business Central, il relativo package tronca la tabella `Staging.BC_*` corrispondente e la ricarica integralmente dalla sorgente.
2. **Refresh/CDC (fasi 21–25).** Per ciascun dominio (articoli, prezzi, distinta base, bundle, regole), la stored procedure di refresh confronta i dati importati con lo stato consolidato in `Products.*`, applica le variazioni e scrive le righe effettivamente cambiate nelle tabelle delta `Staging.*_T`. In caso di errore bloccante la transazione SQL viene annullata (`ROLLBACK`) e l'errore risale al framework.
3. **Integrazione Apparound (fasi 31–37).** Per ciascun dominio, il package verifica prima eventuali anomalie residue dalla chiamata precedente (`Apparound_Check_Anomalie`), ottiene un token OAuth valido interrogando `Products.Api_Config`, legge le sole righe delta della transazione corrente e le invia via API REST ad Apparound, registrando ogni esito in `Api_Log`/`Api_Log_Ok`.
4. **Refresh AS400 (fase 41).** Il package legge la distinta base consolidata per le macchine variate nella transazione e la scrive sulle tabelle native AS400 tramite Linked Server.

Poiché tutte le fasi sono `managing_code = 1` nella configurazione (`Fasi.csv`) e solo l'ultima fase (41) ha `ending_code = 1`, il Flow Framework esegue l'intera sequenza in un'unica transazione logica: un errore in una qualsiasi fase interrompe l'intera catena in quel punto, lasciando le fasi successive non eseguite fino a un eventuale restart.

---

## 9. Interazioni principali fra i componenti

Il diagramma seguente riassume come i package di progetto, le stored procedure/function e le tabelle interagiscono tra loro, indipendentemente dall'ordine temporale.

![Interazioni principali](diagrams_prd_prodotti/05_interazioni_principali.png)

Punti salienti:

- I package di **import** (1–12) sono gli unici che dialogano con Business Central; scrivono esclusivamente in `Staging.BC_*` e non conoscono né le tabelle `Products.*` né i sistemi di destinazione.
- I package di **refresh** (21–25) sono l'unico punto in cui i dati **entrano** nello schema `Products`; sono anche l'unico punto che scrive sia sul log applicativo (`LogProcess`) sia sulle anomalie di dato (`LogErrors`), e l'unico che genera le tabelle delta consumate a valle.
- I package di **integrazione Apparound** (31–37) non scrivono mai direttamente su `Products.*` (con la sola eccezione indiretta di `PRD_35`, che tramite `Apparound_DBase_Refresh` aggiorna `Products.DBase_Apparound`, una tabella derivata dedicata esclusivamente a quell'integrazione): leggono attraverso le viste `V_Apparound_*`/`V_DBase_Apparound`/`V_Main_Clienti` e scrivono solo sul proprio log (`Api_Log`/`Api_Log_Ok`).
- Il package **AS400** (41) è l'unico che accede a un sistema esterno tramite Linked Server anziché tramite API applicativa, e legge direttamente le tabelle consolidate `Products.*` senza passare da una vista dedicata.
- Le **function** (`fn_GetLevelProducts`, `fn_GetTopLevelProducts`, `fn_BuildDBProductJson`) non sono richiamate direttamente dai package SSIS, ma dalle stored procedure e dalle viste: rappresentano la logica di navigazione ricorsiva della distinta base, riutilizzata in più punti del progetto (calcolo dei livelli per le regole, risalita da componente a macchina, generazione di JSON gerarchico).

---

## 10. Descrizione dei componenti di ogni package

| Fase | Package | Descrizione |
|---|---|---|
| 1 | `PRD_01_BC_Items.dtsx` | Truncate di `Staging.BC_Items`; Data Flow con sorgente *Dynamics NAV* che importa gli articoli da Business Central (con una `Derived Column` di normalizzazione) e li carica via OLE DB Destination. |
| 2 | `PRD_02_BC_CFGCharacValues.dtsx` | Import dei valori delle caratteristiche di configurazione (`Staging.BC_CFGCharacValues`), stesso pattern truncate & load. |
| 3 | `PRD_03_BC_CFGRuleCharacValues.dtsx` | Import dei valori delle caratteristiche usate nelle regole (`Staging.BC_CFGRuleCharacValues`). |
| 4 | `PRD_04_BC_CFGItemVarCharacs.dtsx` | Import dell'associazione articoli/varianti–caratteristiche (`Staging.BC_CFGItemVarCharacs`). |
| 5 | `PRD_05_BC_ProductionBomLines.dtsx` | Import delle righe di distinta base di produzione (`Staging.BC_ProductionBomLines`). |
| 6 | `PRD_06_BC_ProductionBomHeader.dtsx` | Import delle testate di distinta base di produzione (`Staging.BC_ProductionBomHeader`). |
| 7 | `PRD_07_BC_PriceListLine.dtsx` | Import delle righe di listino prezzo (`Staging.BC_PriceListLine`). |
| 8 | `PRD_08_BC_PricesMatrix.dtsx` | Import della matrice prezzi (`Staging.BC_PricesMatrix`), la tabella con il maggior numero di colonne del progetto. |
| 9 | `PRD_09_BC_ItemAttribute.dtsx` | Import dell'anagrafica attributi articolo (`Staging.BC_ItemAttribute`). |
| 10 | `PRD_10_BC_ItemAttributeValue.dtsx` | Import dei valori ammessi per attributo (`Staging.BC_ItemAttributeValue`). |
| 11 | `PRD_11_BC_ItemAttributeValueMapping.dtsx` | Import della mappatura valore attributo–articolo (`Staging.BC_ItemAttributeValueMapping`). |
| 12 | `PRD_12_BC_CFGRuleRelations.dtsx` | Import delle relazioni/condizioni tra regole di configurazione (`Staging.BC_CFGRuleRelations`). |
| 21 | `PRD_21_BC_Items_Refresh.dtsx` | Execute SQL Task che invoca `Products.BC_Items_Refresh @Transazione, @Refresh='Y'`: riconciliazione articoli e generazione delta. |
| 22 | `PRD_22_BC_Prices_Refresh.dtsx` | Execute SQL Task che invoca `Products.BC_Prices_Refresh`: riconciliazione prezzi e generazione delta. |
| 23 | `PRD_23_BC_DBase_Refresh.dtsx` | Execute SQL Task che invoca `Products.BC_DBase_Refresh`: riconciliazione distinta base e generazione delta. |
| 24 | `PRD_24_BC_Bundle_Refresh.dtsx` | Execute SQL Task che invoca `Products.BC_Bundle_Refresh`: riconciliazione bundle e generazione delta. |
| 25 | `PRD_25_BC_Rule_Refresh.dtsx` | Execute SQL Task che invoca `Products.BC_Rule_Refresh`: riconciliazione regole di configurazione e generazione delta. |
| 31 | `PRD_31_Apparound_Products.dtsx` | Verifica anomalie pregresse (`Apparound_Check_Anomalie`); recupera credenziali/URL da `Api_Config` e un token OAuth (`ST Get Token`); Data Flow (`DF Extract Products`) che legge il delta prodotti tramite `V_Apparound_Products`, costruisce il payload JSON (Script Component `SC Set JSonPayload`) e lo invia ad Apparound (`SC Send Product`); instrada l'esito (Conditional Split) verso `Api_Log` (errore) o `Api_Log_Ok` (successo). |
| 32 | `PRD_32_Apparound_Prices.dtsx` | Stessa logica di PRD_31 applicata ai prezzi (`V_Apparound_Prices`, `DF Extract Prices`). |
| 33 | `PRD_33_Apparound_Bundles_Del.dtsx` | Elimina su Apparound i bundle non più attivi/rimossi (`DF Delete Bundles`), con lo stesso schema di autenticazione e logging degli altri package Apparound. |
| 34 | `PRD_34_Apparound_Rules_Del.dtsx` | Elimina su Apparound le regole non più attive/rimosse (`DF Delete Rules`). |
| 35 | `PRD_35_Apparound_Boms.dtsx` | Invoca `Apparound_DBase_Refresh` per esplodere la distinta base multilivello in `Products.DBase_Apparound`, quindi la invia ad Apparound (`DF Extract Boms`). |
| 36 | `PRD_36_Apparound_Bundles.dtsx` | Invia ad Apparound i bundle nuovi (`DF Insert Bundles`) e quelli modificati (`DF Update Bundles`). |
| 37 | `PRD_37_Apparound_Rules.dtsx` | Ciclo completo sulle regole: lettura (`DF Get Rules`), inserimento (`DF Insert Rules`), aggiornamento (`DF Update Rules`) ed eliminazione (`DF Delete Rules`) su Apparound. |
| 41 | `PRD_41_AS400_DBase_Refresh.dtsx` | Execute SQL Task che invoca `Products.AS400_DBase_Refresh @Transazione`: scrittura dell'anagrafica e della distinta base delle macchine variate sulle tabelle native AS400, tramite Linked Server `DBAS400`. |

**Nota sui componenti ricorrenti nei package Apparound (31–37):** tutti condividono la medesima struttura interna, riconoscibile dagli stessi nomi di task: `EST Apparound_Check_Anomalie` (verifica anomalie pregresse), `EST Get Variables` (recupero credenziali/URL da `Products.Api_Config`), `ST Get Token` (Script Task che effettua la richiesta OAuth e restituisce `AuthToken`), seguiti da uno o più Data Flow (`DF ...`) dedicati alle operazioni di estrazione/inserimento/aggiornamento/cancellazione verso l'API Apparound, ciascuno con un componente script per la costruzione del payload JSON e uno per l'invio HTTP, e un Conditional Split che instrada l'esito verso `Api_Log` o `Api_Log_Ok`.

---

*Documento generato a partire dall'analisi dei 24 package SSIS del progetto (cartella `DTSX.zip`), della configurazione delle fasi (`Fasi.csv`), degli script DDL di `Staging.sql`, `Products.sql`, delle stored procedure (`StoredProcedures.sql`), delle viste (`Views.sql`) e delle function (`Functions.sql`). Il progetto opera come area `PRD_PRODOTTI` del Flow Framework, documentato separatamente in `SSIS_Flow_Framework.md`.*
