# Cross-Asset Factor Engine

Atlante delle esposizioni al rischio, inizialmente dedicato alle azioni.

Il progetto cerca di rispondere a una domanda: *quali caratteristiche rendono un
mercato o un settore azionario vulnerabile a un determinato cambiamento economico,
quali investimenti condividono queste vulnerabilità e quali possono compensarle?*

Il lavoro procede per **piccoli studi completi**: un mercato, un rischio, una domanda
circoscritta. Ogni studio parte da una scheda teorica in `studies/`, viene svolto in un
notebook in `notebooks/` e lascia in `src/` il codice che vale la pena riutilizzare.

---

## 1. Prerequisiti

Servono due programmi. **Non serve installare Python a mano**: se ne occupa `uv`.

### Git

Verifica se c'è già:

```powershell
git --version
```

Se manca: [git-scm.com/download/win](https://git-scm.com/download/win), oppure
`winget install Git.Git`.

### uv

`uv` è il gestore di progetto: installa la versione giusta di Python, crea l'ambiente
virtuale, installa le dipendenze alle versioni esatte registrate in `uv.lock`.

```powershell
winget install astral-sh.uv
```

In alternativa, lo script ufficiale di Astral:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**Chiudi e riapri il terminale** dopo l'installazione, altrimenti il comando non viene
trovato. Poi verifica:

```powershell
uv --version
```

---

## 2. Clona il repository

```powershell
git clone <url-del-repo>
cd cross-asset-factor-engine
```

Tutti i comandi che seguono vanno lanciati **dentro questa cartella**.

---

## 3. Crea l'ambiente

```powershell
uv sync
```

Cosa fa, in ordine: scarica Python 3.13 se non ce l'hai (versione fissata in
`.python-version`), crea la cartella `.venv/`, installa le dipendenze alle versioni
**esatte** di `uv.lock`, e installa il progetto stesso in modo che `src/` sia
importabile dai notebook.

Non devi "attivare" l'ambiente: ogni comando si lancia con `uv run`, che usa in
automatico l'ambiente del progetto.

---

## 4. Installa il filtro nbstripout

**Obbligatorio, una volta per ogni copia del repository.**

```powershell
uv run nbstripout --install --attributes .gitattributes
```

### Perché serve

Un notebook `.ipynb` non è il testo che vedi a schermo: è un file JSON che contiene
anche gli **output** (una tabella pandas come blocco HTML, un grafico come immagine PNG
codificata in base64, cioè decine di migliaia di caratteri su una riga sola) e il
contatore `execution_count` di ogni cella.

Senza filtro: `git diff` diventa illeggibile anche per una modifica di una parola, il
repository si gonfia di immagini a ogni commit, e i conflitti finiscono dentro il JSON
lasciando un file che Jupyter non riesce più ad aprire.

`nbstripout` si registra come *git filter*: rimuove output e `execution_count`
**dalla versione che entra nel commit**, lasciando intatto il file aperto sul tuo
schermo. I tuoi grafici restano visibili: non perdi nulla in locale.

### Perché va rifatto da ognuno

Il comando scrive in due posti. La regola "applica il filtro ai file `.ipynb`" va in
`.gitattributes`, che è versionato e quindi arriva con il clone. Ma la definizione del
filtro va in `.git/config`, che è **locale e non si propaga mai**: nessun `git clone`
o `git pull` può installarlo al posto tuo.

È l'errore più comune di questo setup. Se lo salti, tutto sembra funzionare ma
ricominci a committare gli output.

---

## 5. Crea le cartelle dei dati

```powershell
mkdir data\raw, data\processed
```

`data/` è escluso da git di proposito (vedi sezione *Regole*), quindi va ricreato a
mano dopo ogni clone. Le altre cartelle arrivano già con il repository.

---

## 6. Verifica che tutto funzioni

### L'ambiente

```powershell
uv run jupyter lab
```

Si apre JupyterLab nel browser. Crea un notebook di prova in `notebooks/`, esegui
`import pandas as pd` e un grafico qualsiasi con matplotlib: se non ci sono errori,
l'ambiente è a posto.

### Il filtro (fallo davvero: è la parte che si rompe in silenzio)

Con il notebook di prova salvato e contenente almeno un grafico:

```powershell
git add notebooks\prova.ipynb
git diff --cached
```

Quello che devi vedere: **solo il codice**, poche righe leggibili.

Quello che **non** devi vedere: righe lunghissime di lettere e numeri senza senso
(è il PNG in base64), o righe `"execution_count"`. Se compaiono, il filtro non è
installato in questa copia: torna al punto 4.

Poi annulla la prova con `git reset` e cancella il notebook.

---

## 7. Struttura del progetto

```
notebooks/       un notebook per studio, numerato (01_, 02_, ...)
studies/         le schede teoriche in Markdown, stesso numero del notebook
src/             il codice riutilizzabile, importabile dai notebook
data/raw/        i dati come sono stati scaricati — mai modificati (fuori da git)
data/processed/  i dati ripuliti, prodotti da codice a partire da raw/ (fuori da git)
outputs/         grafici e tabelle finali da condividere
```

`pyproject.toml` elenca le dipendenze, `uv.lock` ne fissa le versioni esatte:
entrambi sono versionati e non vanno modificati a mano (si usa `uv add`).

---

## 8. Flusso di lavoro quotidiano

**All'inizio di ogni sessione:**

```powershell
git pull
uv sync
uv run jupyter lab
```

`uv sync` dopo il `pull` serve nel caso l'altro abbia aggiunto una dipendenza.

**Prima di ogni commit:**

1. **Kernel → Restart Kernel and Run All Cells.**
   Verifica che il notebook funzioni davvero eseguito dall'alto in basso, e non solo
   grazie a una variabile definita in una cella che hai poi cancellato. Se si ferma
   con un errore, il notebook non è riproducibile: risolvilo prima di committare.
   *Questo controllo resta a carico tuo: `nbstripout` non lo fa.*
2. Guarda gli output: il Restart verifica che il notebook **giri**, non che il
   risultato sia **giusto**.
3. `git add` e `git commit`. La pulizia degli output avviene da sola.

**Per aggiungere una libreria:** `uv add nome-libreria` (mai `pip install`, che
scriverebbe nell'ambiente senza registrare nulla in `pyproject.toml`). Poi committa
`pyproject.toml` e `uv.lock` insieme.

---

## 9. Regole del repository

**Un notebook per studio, una persona per notebook** finché quello studio è aperto.
`nbstripout` rende i conflitti risolvibili, non inesistenti: il coordinamento evita la
gran parte dei casi a costo zero.

**I file in `data/raw/` sono di sola lettura.** Sono la prova su cui poggia ogni
risultato: se li modifichi, nessuna analisi è più verificabile. I dati stanno fuori da
git perché sono ricostruibili dal codice che li scarica; se una fonte gratuita rischia
di sparire o di cambiare retroattivamente, quel file specifico si committa come
eccezione, annotandolo nella scheda dello studio.

**Il codice usato in due studi esce dal notebook.** Quando stai per copiare un pezzo di
codice da un notebook a un altro, fermati e spostalo in `src/` come funzione con nome,
parametri espliciti e valore di ritorno. È così che il progetto diventa un programma
invece di una pila di notebook.

**La scheda teorica viene prima del notebook.** Domanda precisa, definizione del
rischio, meccanismo ipotizzato, ipotesi da verificare, orizzonte, spiegazioni
alternative, e quale evidenza ci farebbe abbandonare l'ipotesi. Decidere il perimetro
dopo aver visto i numeri non produce un risultato.

---

## 10. Problemi frequenti

**`uv` non viene trovato** — non hai riaperto il terminale dopo l'installazione.

**`git diff` mostra migliaia di righe di base64** — il filtro non è installato in
questa copia: rilancia il punto 4.

**Jupyter dice che il notebook non è JSON valido / "Unreadable Notebook"** — c'è un
conflitto git irrisolto dentro il file. Non modificarlo a caso: se sei in mezzo a una
fusione, `git merge --abort` ti riporta allo stato precedente e ne parlate prima di
riprovare.

**Un `import` da `src/` non funziona nel notebook** — manca l'installazione del
progetto nell'ambiente: rilancia `uv sync`.

**Il notebook non trova pandas anche se è installato** — il kernel selezionato non è
quello del progetto. In alto a destra in JupyterLab, controlla che il kernel punti
all'ambiente `.venv` del progetto.
