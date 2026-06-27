# Program Performance

I pattern asincroni trattati nei capitoli precedenti — callback, Promise, generator — migliorano le prestazioni permettendo di sovrapporre operazioni che altrimenti si bloccherebbero a vicenda. Ma l'asincronia rimane vincolata a un singolo thread. Questo capitolo copre tre meccanismi di livello superiore che portano il parallelismo reale in JavaScript.

---

## Web Workers

JavaScript è single-threaded — ma il *runtime* non lo è necessariamente. I **Web Workers** sono una funzionalità del browser (non del linguaggio JS in sé) che permette di avviare thread separati, ognuno con la propria istanza del JS engine, in cui far girare codice indipendentemente dal thread UI principale. Si tratta di **task parallelism**: si suddivide il programma in pezzi che girano in parallelo.

```js
/* thread principale */
var w1 = new Worker("http://some.url.1/mycoolworker.js");
```

Il file JS puntato dall'URL viene caricato in un thread separato e gira come programma autonomo. Questo tipo di Worker si chiama *Dedicated Worker* (dedicato a una sola connessione).

### Comunicazione via messaggi

I Worker non condividono scope, variabili globali né accesso al DOM con il thread principale — nessuno stato condiviso, nessun mutex. La comunicazione avviene esclusivamente tramite eventi di tipo "message":

```js
/* thread principale — invia e riceve */
w1.addEventListener("message", function(evt) {
    console.log(evt.data);  // messaggio dal Worker
});
w1.postMessage("ciao Worker");

/* Worker — mycoolworker.js */
addEventListener("message", function(evt) {
    // elabora evt.data
    postMessage("risposta dal Worker");
});
```

Per terminare un Worker immediatamente: `w1.terminate()`. Non viene dato al Worker alcun tempo per completare o fare cleanup — equivale a chiudere una scheda del browser.

### Ambiente del Worker

Dentro un Worker non è disponibile: DOM, variabili globali del thread principale, localStorage. Sono disponibili: Ajax (`XMLHttpRequest`), WebSocket, timer, `navigator`, `location`, `JSON`, `applicationCache`. Si possono caricare script aggiuntivi in modo sincrono:

```js
/* dentro il Worker */
importScripts("foo.js", "bar.js");
/* blocca l'esecuzione del Worker finché i file non sono caricati ed eseguiti */
```

Un Worker può a sua volta creare sub-Worker (supporto variabile tra browser).

### Trasferimento dei dati

Quando si passa un oggetto tramite `postMessage()`, il browser applica di default un **structured clone**: copia l'oggetto sull'altro lato (incluse strutture con riferimenti circolari). Funziona, ma duplica la memoria.

Per dati di grandi dimensioni esiste l'opzione migliore dei **Transferable Objects**: il *possesso* dell'oggetto viene trasferito, non copiato — zero copia di memoria. L'oggetto originale diventa inaccessibile nel mittente.

```js
/* trasferisce foo.buffer al Worker senza copia */
postMessage(foo.buffer, [foo.buffer]);
/* foo.buffer è ora vuoto/inaccessibile nel thread principale */
```

Tipici Transferable: `ArrayBuffer`, `MessagePort`, `ImageBitmap`. Browser senza supporto degradano silenziosamente al structured clone.

### Shared Workers

Se più tab della stessa pagina devono comunicare con lo stesso Worker, si usa un **SharedWorker** — un'istanza condivisa tra più connessioni:

```js
var w1 = new SharedWorker("http://some.url.1/mycoolworker.js");
```

Ogni connessione è identificata da un **port** (oggetto porta). La comunicazione passa attraverso di esso:

```js
/* lato chiamante */
w1.port.addEventListener("message", handleMessages);
w1.port.postMessage("qualcosa");
w1.port.start();

/* dentro lo SharedWorker */
addEventListener("connect", function(evt) {
    var port = evt.ports[0];
    port.addEventListener("message", function(evt) {
        port.postMessage(/* risposta */);
    });
    port.start();
});
```

Uno SharedWorker rimane vivo finché esiste almeno una porta connessa; un Dedicated Worker termina non appena la connessione con il suo creatore si chiude.

### Polyfilling dei Worker

Se l'ambiente non supporta Worker, non è possibile simulare il parallelismo reale. Si può però creare un polyfill che usa `setTimeout()` per simulare l'asincronia (non il parallelismo) e che espone l'API Worker. La limitazione principale: non si può simulare `importScripts()` in modo sincrono.

---

## SIMD

**SIMD** (*Single Instruction, Multiple Data*) è una forma di **data parallelism**: invece di parallelizzare pezzi di logica (come i Worker), si parallelizzano operazioni matematiche su più valori simultaneamente, sfruttando le istruzioni vettoriali delle CPU moderne.

Un vettore SIMD tipico contiene 4 elementi da 32 bit (128 bit totali — la dimensione nativa delle unità SIMD in molte CPU):

```js
var v1 = SIMD.float32x4(3.14159, 21.0, 32.3, 55.55);
var v2 = SIMD.float32x4(2.1, 3.2, 4.3, 5.4);

SIMD.float32x4.mul(v1, v2);
// [ 6.597339, 67.2, 138.89, 299.97 ]
// le 4 moltiplicazioni avvengono in parallelo a livello CPU

var v3 = SIMD.int32x4(10, 101, 1001, 10001);
var v4 = SIMD.int32x4(10, 20, 30, 40);
SIMD.int32x4.add(v3, v4);
// [ 20, 121, 1031, 10041 ]
```

Le operazioni disponibili includono: aritmetiche (`mul`, `add`, `sub`, `div`, `abs`, `sqrt`), logiche (`and`, `or`, `xor`), confronto (`equal`, `greaterThan`), shuffle e shift. I benefici in termini di prestazioni sono evidenti per applicazioni data-intensive: analisi di segnali, operazioni su matrici grafiche, elaborazione audio/immagini.

SIMD era in fase di proposta durante la scrittura di YDKJS; nella pratica odierna l'accesso a istruzioni SIMD avviene principalmente tramite WebAssembly.

---

## asm.js

**asm.js** è un sottoinsieme del linguaggio JavaScript progettato per essere altamente ottimizzabile dai motori JS. Non introduce nuova sintassi: definisce uno stile di codice JS standard che, se riconosciuto dall'engine, riceve ottimizzazioni aggressive a basso livello.

L'obiettivo principale non è la scrittura manuale: asm.js è pensato come *target di compilazione* per strumenti come Emscripten (che trasla C/C++ in JavaScript).

### Type hints con `|0`

Il principale ostacolo all'ottimizzazione in JS è il tracking dei tipi per la coercione. asm.js usa operatori bitwise per segnalare all'engine il tipo inteso di una variabile:

```js
/* hint: b è sempre un intero a 32 bit */
var b = a | 0;

/* hint: la somma è un intero a 32 bit */
(a + b) | 0;
```

Un engine asm.js-aware interpreta `| 0` come garanzia di tipo intero e salta il tracking della coercione. In un engine normale il codice funziona ugualmente, senza ottimizzazioni speciali.

### Moduli asm.js

Un modulo asm.js è una funzione che riceve tre parametri predefiniti e usa `"use asm"` come direttiva:

```js
function fooASM(stdlib, foreign, heap) {
    "use asm";

    /* stdlib: namespace delle API standard (es. window) */
    var arr = new stdlib.Int32Array(heap);

    function foo(x, y) {
        x = x | 0;          // type hint: intero
        y = y | 0;
        var i = 0;
        var p = 0;
        var sum = 0;
        var count = ((y | 0) - (x | 0)) | 0;

        for (i = x | 0; (i | 0) < (y | 0); p = (p + 8) | 0, i = (i + 1) | 0) {
            arr[p >> 3] = (i * (i + 1)) | 0;
        }
        for (i = 0, p = 0; (i | 0) < (count | 0); p = (p + 8) | 0, i = (i + 1) | 0) {
            sum = (sum + arr[p >> 3]) | 0;
        }
        return +(sum / count);  // + converte a float per il return
    }

    return { foo: foo };
}

/* heap: buffer pre-allocato — nessuna allocazione dinamica dentro il modulo */
var heap = new ArrayBuffer(0x1000);
var foo = fooASM(window, null, heap).foo;
foo(10, 20);    // 233
```

I tre parametri del modulo:
- **`stdlib`**: namespace delle API standard richieste (di solito `window`)
- **`foreign`**: riferimenti a funzioni JS esterne al modulo
- **`heap`**: un `ArrayBuffer` pre-allocato che il modulo usa come memoria propria

Il vantaggio dell'heap pre-allocato è eliminare allocazione dinamica e garbage collection all'interno del modulo — le due operazioni più costose in JS. Tutto il lavoro avviene su memoria già riservata.

asm.js è adatto a task intensivi e circoscritti (calcoli matematici pesanti, elaborazione grafica, fisica dei giochi). Non è uno strumento di ottimizzazione generale per qualsiasi codice JS.

---

## ⚡ Ripasso veloce

**Tre tecniche per performance oltre l'asincronia:**

| Tecnica | Tipo di parallelismo | Caso d'uso |
|---|---|---|
| Web Workers | Task (thread separati) | Calcoli pesanti off-thread, comunicazione Ajax |
| SIMD | Data (istruzioni vettoriali CPU) | Elaborazione numerica massiva |
| asm.js | Ottimizzazione dell'engine | Target di compilazione, calcoli intensivi |

```js
/* Web Worker — spawn e messaggi */
var w1 = new Worker("worker.js");
w1.addEventListener("message", function(evt) { /* riceve */ });
w1.postMessage({ data: payload });

/* Transferable Object — zero copia */
var buf = new ArrayBuffer(1024 * 1024);
w1.postMessage(buf, [buf]);           // buf ora inaccessibile qui

/* SharedWorker — condiviso tra tab */
var sw = new SharedWorker("shared.js");
sw.port.start();
sw.port.postMessage("connessione");

/* asm.js type hint — segnala intero 32-bit all'engine */
var x = someValue | 0;
var result = (a + b) | 0;
```

**Regole pratiche:**
- I Worker non condividono scope: comunicazione solo via `postMessage()`
- `terminate()` è brutale — nessun cleanup
- Usare Transferable Objects per dati grandi (evitare la copia)
- asm.js non si scrive a mano: è output di compilatori come Emscripten

---

## Domande

<details>
<summary>Perché i Web Worker non possono condividere variabili con il thread principale?</summary>

La condivisione di stato tra thread è il problema fondamentale della programmazione multithread: richiede meccanismi di sincronizzazione come mutex e lock per evitare race condition, che aggiungono complessità elevata e sono fonte di bug difficilissimi da riprodurre. La scelta di isolare completamente i Worker — nessuna variabile globale condivisa, nessun DOM, nessun accesso allo scope esterno — elimina alla radice questa classe di problemi. Il canale di comunicazione a messaggi (`postMessage`/`addEventListener("message")`) è l'unica interfaccia ammessa, e garantisce che ogni scambio di dati sia esplicito e controllato.

</details>

<details>
<summary>Qual è la differenza tra structured clone e Transferable Objects, e quando usare l'uno o l'altro?</summary>

Il **structured clone** copia l'oggetto da un thread all'altro: l'originale rimane disponibile nel mittente, ma si paga il costo di duplicare la memoria (e dell'eventuale garbage collection successiva). I **Transferable Objects** trasferiscono la *proprietà* del buffer senza copiare i dati: l'originale diventa inaccessibile nel mittente (diventa un buffer vuoto), ma il costo di trasferimento è essenzialmente zero indipendentemente dalla dimensione. La scelta pratica: usare structured clone per oggetti piccoli o quando il mittente deve continuare a usare i dati; usare Transferable (tipicamente `ArrayBuffer`) per dati grandi (immagini, audio, array numerici) dove la copia sarebbe costosa.

</details>

<details>
<summary>In cosa differisce SIMD dai Web Workers come forma di parallelismo?</summary>

I Web Workers implementano **task parallelism**: pezzi distinti di logica del programma girano su thread separati. SIMD implementa **data parallelism**: una singola operazione viene applicata simultaneamente a più valori, sfruttando le istruzioni vettoriali della CPU (tipicamente 4 valori da 32 bit in parallelo su vettori a 128 bit). SIMD non crea thread aggiuntivi: opera all'interno del singolo thread, mappando le operazioni direttamente alle istruzioni hardware. I casi d'uso sono diversi: Worker per task complessi e autonomi, SIMD per operazioni matematiche ripetute su grandi array di numeri (grafica 3D, elaborazione audio, machine learning nel browser).

</details>

<details>
<summary>Perché <code>| 0</code> è una type hint valida per asm.js e cosa cambia a runtime?</summary>

L'operatore `|` (OR bitwise) con `0` come secondo operando non ha effetto sul valore numerico intero (ogni bit OR 0 resta invariato), ma ha l'effetto collaterale di convertire il risultato a un intero a 32 bit. In JS standard, ogni operazione bitwise forza la conversione dell'operando a `Int32` e produce un risultato `Int32`. Un engine asm.js-aware interpreta la presenza di `| 0` come una dichiarazione implicita: "questa variabile/espressione è sempre un intero a 32 bit". L'engine può quindi ottimizzare aggressivamente, saltando il tracking dinamico del tipo e la gestione della coercione. Lo stesso codice gira correttamente in qualsiasi JS engine standard — `| 0` produce sempre il risultato corretto — ma solo i motori asm.js-aware ne traggono beneficio in termini di prestazioni.

</details>

<details>
<summary>Per quale motivo asm.js è pensato come target di compilazione e non per la scrittura manuale?</summary>

Il codice asm.js è estremamente verboso e basso livello: ogni variabile deve avere type hint espliciti, tutta la memoria temporanea va allocata in anticipo su un heap pre-dichiarato, e qualsiasi operazione deve rispettare i vincoli del sottoinsieme ottimizzabile. Scrivere e mantenere manualmente del codice in questo stile è simile a scrivere assembly language a mano — noioso, error-prone e difficile da debuggare. Strumenti come Emscripten prendono codice C/C++ ad alte prestazioni, già scritto con gli strumenti e le astrazioni adatte a quel livello, e lo traducono automaticamente in JavaScript conforme asm.js. In questo modo si ottengono i benefici dell'ottimizzazione senza il costo della manutenzione manuale.

</details>
