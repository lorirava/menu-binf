# Menu "e' Borg in Festa 2026"

Due deliverable che devono restare allineati:

| file | cos'è |
|------|-------|
| `Menu Borg in Festa 2026.html` | menu **interattivo** per il telefono: si tocca un piatto, si costruisce la lista, si mostra alla cassa |
| `Menu festa con prezzi.pdf` | menu **stampato**, 1 pagina A4 fronte, generato dall'HTML |

> ## ⚠️ Regola fissa
> **A ogni modifica dell'HTML va rigenerato `Menu festa con prezzi.pdf`.**
> Non è un passo opzionale né una cosa da proporre a fine task: fa parte della
> modifica. Il comando è nella sezione "Rigenerare il PDF".

Non c'è build, non c'è package manager, non è un repo git. L'HTML si apre con
doppio click (`file://`) e funziona offline: font, logo e QR sono incorporati.

---

## 1. Struttura del file: è un bundle, non un HTML normale

`Menu Borg in Festa 2026.html` è un **artifact Claude Design impacchettato**.
Aprirlo e modificarlo con Edit/sed sul markup visibile NON funziona: il markup
vero è una stringa JSON escapizzata dentro un tag `<script>`.

| riga | contenuto |
|------|-----------|
| 1–374 | loader: CSS di caricamento, thumbnail SVG, unpacker JS |
| 375 | `<script type="__bundler/manifest">` — asset base64 (logo, QR, font woff2). Riga 376 = ~5,2 MB |
| 379 | `<script type="__bundler/ext_resources">` — React 18.3.1 UMD + react-dom |
| 383 | `<script type="__bundler/page_order">` — `[]` |
| 387 | `<script type="__bundler/template">` — **riga 388 = la pagina vera**, ~120 KB, come stringa JSON |

All'avvio il loader ricostruisce gli asset come blob/`data:` URI e sostituisce
gli uuid nel template (`src="87047bee-…"` → l'immagine). La sostituzione è
globale (`split(uuid).join(url)`), quindi **lo stesso asset può essere
referenziato più volte** — è così che logo e QR compaiono sia nella scheda
interattiva sia nel foglio di stampa.

### Workflow di modifica (obbligatorio)

**1. Estrai il template** (riga 388 = indice 387):

```python
import io, json
lines = io.open('Menu Borg in Festa 2026.html', encoding='utf-8').read().split('\n')
io.open('/tmp/template.html','w',encoding='utf-8').write(json.loads(lines[387]))
```

**2. Modifica `/tmp/template.html`** — è HTML leggibile e indentato.

**3. Reimpacchetta:**

```python
import io, json
tpl = io.open('/tmp/template.html', encoding='utf-8').read()
src = 'Menu Borg in Festa 2026.html'
lines = io.open(src, encoding='utf-8').read().split('\n')
lines[387] = json.dumps(tpl).replace('</', '<\\u002F')   # ← escape indispensabile
io.open(src,'w',encoding='utf-8').write('\n'.join(lines))
```

Il `.replace('</', '<\\u002F')` è **indispensabile**: senza, un `</script>`
dentro la stringa chiuderebbe il tag contenitore e romperebbe il file. `/` è
`/` in JSON, quindi `json.loads` restituisce il testo identico.

**4. Verifica il roundtrip** prima di considerare finito:

```python
json.loads(open('Menu Borg in Festa 2026.html',encoding='utf-8').read().split('\n')[387]) == tpl
```

### Regole per non rompere il bundle

- Non riformattare l'intero file: tocca **solo** la riga 387.
- Non toccare la riga 376 (manifest): 5 MB di base64, va lasciata intatta.
- Le sostituzioni sul template vanno fatte con un helper che **asserisce
  l'unicità** dell'ancora, altrimenti si modifica il pezzo sbagliato in silenzio:

  ```python
  def sub1(old, new):
      global s
      assert s.count(old) == 1, (s.count(old), old[:80])
      s = s.replace(old, new)
  ```
- Backup del file prima di iniziare (`cp` nella scratchpad).

---

## 2. Il framework interno (`x-dc` / DCLogic)

Micro-sintassi template su React. Convenzioni da rispettare aggiungendo markup:

| sintassi | significato |
|----------|-------------|
| `{{ nome }}` | binding a un valore restituito da `renderVals()` |
| `sc-camel-on-click="{{ handler }}"` | `onClick` (idem `sc-camel-on-input` → `onInput`) |
| `sc-camel-view-box="0 0 24 24"` | `viewBox` sugli SVG |
| `ref="{{ listRef }}"` | ref React |
| `style-hover="background:…"` | stile applicato all'hover |
| `<sc-if value="{{ open }}">` | rendering condizionale |
| `<sc-for list="{{ items }}" as="it">` | loop, con `{{ it.name }}` all'interno |

La logica è in fondo al template, in `<script type="text/x-dc">`:
`class Component extends DCLogic`. Ogni valore/handler usato nel markup deve
essere restituito da `renderVals()`.

### Stato e comportamento della scheda interattiva

- Ordine in `localStorage`, chiave **`binf-menu-2026-order`**
  (`{ "nome piatto": quantità }`); i prezzi si rileggono dal DOM via `data-price`.
- Click su una riga → `+1`; il pannello in basso permette `+`/`–`/svuota.
- `filter()` applica ricerca e categoria scrivendo `style.display` **inline**.

### Attributi dati della scheda interattiva

| attributo | uso |
|-----------|-----|
| `data-shell` | contenitore della scheda interattiva — **è l'aggancio che la stampa nasconde** |
| `data-row` + `data-name` + `data-price` | elemento ordinabile: 51 nodi = 37 righe-piatto + 14 bottoni-variante |
| `data-disp="inline-flex"` | marca i 14 bottoni-variante (ripristino del display dopo il filtro) |
| `data-badge` | pallino `×N` della quantità |
| `data-group` | piatto con varianti: riga piatto + riga bottoni (6 gruppi) |
| `data-section` / `data-screen-label` | sezione e categoria per i chip filtro |
| `data-cat` | chip filtro |
| `data-act` | pulsanti `inc` / `dec` / `clear` del pannello lista |

Piatti effettivi: 37 righe singole + 6 gruppi = **43**.

---

## 3. La stampa: foglio dedicato, non il layout mobile riadattato

**Approccio, da non cambiare senza motivo:** la stampa NON è il layout
interattivo compresso con regole `@media print`. Quella strada è stata provata e
scartata — produce un menu che sembra una pagina web schiacciata.

Nell'HTML c'è un **secondo blocco di markup**, `.ps` / `data-print-sheet`,
fratello di `[data-shell]` dentro `<x-dc>`, impaginato apposta per la carta:

```
schermo :  [data-shell] visibile   ·  .ps  display:none
stampa  :  [data-shell] display:none  ·  .ps  visibile
```

Lo scambio è tutto qui, nel `<style>` in coda a `<helmet>`:

```css
.ps{display:none}
@media print{
  @page{size:A4 portrait;margin:10mm 9mm}
  [data-shell]{display:none!important}
  .ps{display:block; …}
}
```

Il resto del blocco `@media print` sono gli stili `.ps-*` del foglio: sono
definiti **solo** dentro la media query, quindi non possono sporcare la vista a
schermo.

### Anatomia del foglio

| classe | elemento |
|--------|----------|
| `.ps-top` `.ps-logo` `.ps-t1` `.ps-t2` `.ps-qr` | testata: logo + "MENU 2026" + sottotitolo, QR Satispay a destra |
| `.ps-don` `.ps-don-t` `.ps-don-b` `.ps-don-p` | fascia blu donazione con pillola prezzo a destra |
| `.ps-cols` | corpo su **2 colonne bilanciate** |
| `.ps-sec` `.ps-h` `.ps-hr` | sezione: titolo Anton + filetto arancio |
| `.ps-it` | **riga piatto + descrizione, blocco indivisibile** (`break-inside:avoid`) |
| `.ps-i` `.ps-nm` `.ps-l` `.ps-pr` `.ps-de` `.ps-a` | nome · leader puntinato · prezzo · descrizione · allergeni |
| `.ps-v` `.ps-s` | tag **VEG** (verde `#3f7d3a`) e **SG** (rosso `#c0392b`) |
| `.ps-no` `.ps-bx` | nota di sezione semplice · riquadro crema con bordo arancio |
| `.ps-leg` `.ps-leg-w` `.ps-leg-g` | legenda allergeni a tutta larghezza sotto le colonne |
| `.ps-ft` | riga di chiusura |

Dettagli che sono lì per un motivo:

- **`.ps-cols` usa il bilanciamento predefinito, non `column-fill:auto`.** Con
  `auto` la prima colonna si allunga per tutta la pagina e spinge la legenda a
  pagina 2.
- **`.ps-it` avvolge riga e descrizione.** Senza, la descrizione si stacca dal
  piatto a fine colonna.
- **`.ps-h` ha `break-after:avoid`** così un titolo di sezione non resta orfano.
- **`print-color-adjust:exact`** fa uscire fondi e colori anche con "Grafica di
  sfondo" disattivata nel dialogo di Chrome.
- Nella scheda interattiva c'è il pulsante **"Stampa il menu"** nel footer
  (`{{ onPrint }}` → `window.print()`). `Ctrl/Cmd+P` fa lo stesso.

### ⚠️ Il contenuto è duplicato: vanno aggiornati tutti e due

Piatti e prezzi esistono **due volte**: nelle righe `data-row` della scheda
interattiva e nel foglio `.ps`. Cambiare un prezzo in un solo posto è l'errore
più facile e più dannoso di questo progetto.

Alcune differenze però sono **volute** — sono scelte editoriali della versione
stampata, non disallineamenti da correggere:

| scheda interattiva | foglio di stampa |
|--------------------|------------------|
| icone SVG spiga/foglia | tag testuali `SG` / `VEG` |
| 4 righe acqua (naturale/frizzante × 1/2 lt e 1,5 lt) | 2 righe "naturale o frizzante" |
| 6 righe vino (rosso/bianco × bicchiere, 1/2 lt, 1 lt) | nota "Rosso o bianco, allergene 5" + 3 righe |
| bottoni-variante parmigiano e salse | nota di sezione / descrizione |
| box arancio "Mozzarella 100% latte di bufala" | descrizione sotto "Caprese" |
| nota "Prezzi in euro, servizio incluso…" nel footer | sottotitolo "Prezzi in euro. Ordina alle casse." in testata |

**Controllo dell'allineamento dei prezzi** — da eseguire dopo ogni modifica ai
piatti. Confronta 30 delle 37 voci del foglio; le 7 scoperte sono esattamente gli
accorpamenti della tabella qui sopra, ed è atteso che restino fuori. Esce con
codice 1 se trova un disallineamento.

```python
import io, json, re, html, sys
SRC = 'Menu Borg in Festa 2026.html'
t = json.loads(io.open(SRC, encoding='utf-8').read().split('\n')[387])

# scheda interattiva: il suffisso " · variante" non fa parte del nome del piatto
inter = {}
for n, p in re.findall(r'data-name="([^"]*)"\s+data-price="([^"]*)"', t):
    inter.setdefault(n.split(' · ')[0], float(p))

def nome(frag):                       # via i tag VEG/SG e i numeri allergene
    frag = re.sub(r'<(b|i)\b[^>]*>.*?</\1>', '', frag)
    return html.unescape(re.sub('<[^>]+>', '', frag)).strip()

stampa = {nome(n): float(p.replace(',', '.'))
          for n, p in re.findall(r'class="ps-nm">(.*?)</span><span class="ps-l">'
                                 r'</span><span class="ps-pr">€ ([\d,]+)', t)}

comuni = sorted(k for k in stampa if k in inter)
bad = [(k, inter[k], stampa[k]) for k in comuni if inter[k] != stampa[k]]
print('foglio: %d voci — confrontate: %d' % (len(stampa), len(comuni)))
for k, a, b in bad:
    print('  DISALLINEATO: %s — scheda %s ≠ stampa %s' % (k, a, b))
print('solo nel foglio (attesi 7): %d' % len([k for k in stampa if k not in inter]))
sys.exit(1 if bad else 0)
```

Due dettagli da tenere presenti quando si scrive codice che tocca questi prezzi:
il separatore prima dell'importo è un **thin space U+2009** (non uno spazio
normale) e il `·` delle varianti è circondato da spazi normali. Una `replace`
con lo spazio sbagliato non fallisce: semplicemente non trova nulla e fa credere
che sia tutto a posto.

### Aggiungere o modificare un piatto

1. Nella scheda: copia una riga `data-row` della stessa sezione, adatta
   `data-name` (univoco: è la chiave del localStorage), `data-price` (punto
   decimale) e il prezzo mostrato (`€ 7,50`, virgola e **thin space** U+2009).
2. Nel foglio `.ps`: aggiungi il `<div class="ps-it">` corrispondente.
3. Rigenera il PDF e **controlla che sia ancora 1 pagina**.

Il foglio sta in una pagina con circa 1 cm di margine di sicurezza in fondo.
Se aggiungendo voci va a 2 pagine, le leve nell'ordine in cui conviene toccarle:
`.ps-i{padding}` (3,2px) → `.ps-nm{font-size}` (11,8px) e `.ps-pr` (12,6px) →
`.ps-h{margin}`. Il foglio è pensato per **1 pagina A4 fronte**: questo vincolo
viene prima dell'estetica delle singole misure.

---

## 4. Rigenerare il PDF

```bash
cd /home/lorenzo.rava/personal/binf2026-menu
google-chrome --headless --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --print-to-pdf="Menu festa con prezzi.pdf" --no-pdf-header-footer \
  "file://$PWD/Menu Borg in Festa 2026.html"

pdfinfo "Menu festa con prezzi.pdf" | grep -iE '^Pages|^Page size'   # atteso: 1, A4
```

`--print-to-pdf` ignora l'impostazione "stampa sfondi", ma il risultato resta
rappresentativo perché è attivo `print-color-adjust: exact`.

Poi **guarda il risultato**, non fermarti al conteggio pagine:

```bash
pdftoppm -png -r 120 "Menu festa con prezzi.pdf" /tmp/pg    # e apri /tmp/pg-1.png
```

Cose che il conteggio pagine non intercetta: descrizioni staccate dal piatto a
fine colonna, titoli di sezione orfani, colonne molto sbilanciate, legenda
spezzata.

### Attenzione: il PDF è binario

Se il PDF viene copiato o trasferito in modalità testo si corrompe in modo
silenzioso: ogni byte non-ASCII diventa `EF BF BD` (U+FFFD), il file si gonfia e
`startxref` punta nel vuoto. È già successo una volta. Verifica veloce:

```bash
head -c 16 "Menu festa con prezzi.pdf" | xxd     # riga 2 deve essere binaria, non EF BF BD
```

---

## 5. Verifica dell'HTML

Non ci sono test: si verifica renderizzando. Chrome è in `/usr/bin/google-chrome`.

```bash
F="file:///home/lorenzo.rava/personal/binf2026-menu/Menu Borg in Festa 2026.html"

# vista mobile: NON deve cambiare quando si tocca la stampa
google-chrome --headless --disable-gpu --no-sandbox --virtual-time-budget=12000 \
  --window-size=430,6400 --screenshot=screen.png "$F"

# il template si è montato davvero
google-chrome --headless --disable-gpu --no-sandbox --virtual-time-budget=12000 \
  --dump-dom "$F" > dom.html
grep -c '{{' dom.html          # DEVE essere 0: binding non risolti
```

Controlla **sempre entrambe le rese**, stampa e mobile: sono due impaginati
diversi nello stesso file ed è facile sistemarne uno rompendo l'altro. Il mobile
è l'uso principale.

---

## 6. Design

| | |
|---|---|
| blu scuro (fondo pagina, testo) | `#12314f` |
| blu (titoli, prezzi, bordi) | `#1b4e7e` |
| arancio (accento, filetti, CTA) | `#f2a02d` |
| crema (fondo scheda) | `#faf4e6` |
| grigio testo secondario | `#5c6b7a` |
| verde vegetariano | `#3f7d3a` |
| rosso senza glutine | `#c0392b` |
| titoli | **Anton** (uppercase, letter-spacing) |
| testo | **Lora** (corsivo per le descrizioni) |

Allergeni: **1** glutine, **2** uova, **3** latte, **4** sedano, **5** solfiti,
**6** frutta a guscio, **7** soia.

Sezioni nell'ordine: donazione Commenda · antipasti · primi · griglia e secondi ·
contorni e piadine · dolci · bevande · vini.

---

## 7. Convenzioni

- Testi, etichette e commenti **in italiano**.
- Nessuna dipendenza esterna, nessun CSS in file separati: il valore del
  deliverable è essere **un singolo file autoportante che funziona offline**.
