# Collezioni

ES6 aggiunge al linguaggio strutture dati native che prima andavano simulate con array e oggetti: **TypedArray**, **Map**, **WeakMap**, **Set** e **WeakSet**.

---

## TypedArray

Un TypedArray non è un array di un "tipo di valori" nel senso dei tipi JS — è un'interfaccia indicizzata su un buffer di dati binari grezzi. Il "tipo" nel nome indica la *view* (interpretazione) da applicare ai bit sottostanti.

### `ArrayBuffer` e view

```js
var buf = new ArrayBuffer(32); // buffer di 32 byte (256 bit), tutti a zero
buf.byteLength;                // 32

/* layer di una view sopra il buffer */
var arr = new Uint16Array(buf);
arr.length;  // 16 — 256 bit / 16 bit per elemento
```

Lo stesso buffer può avere più view contemporaneamente, e ogni view vede gli stessi bit reinterpretati:

```js
var buf = new ArrayBuffer(2);
var view8  = new Uint8Array(buf);
var view16 = new Uint16Array(buf);

view16[0] = 3085;
view8[0].toString(16); // "d"  (byte basso)
view8[1].toString(16); // "c"  (byte alto)
```

### Endianness

L'ordine dei byte dipende dalla piattaforma. Su un sistema little-endian (la maggior parte dei browser moderni), il byte meno significativo viene memorizzato per primo. Se si scambiano dati binari tra sistemi diversi, è necessario verificare l'endianness:

```js
var littleEndian = (function() {
    var buf = new ArrayBuffer(2);
    new DataView(buf).setInt16(0, 256, true);
    return new Int16Array(buf)[0] === 256;
})();
```

`DataView` permette accesso granulare con endianness esplicita — utile quando il producer e il consumer del buffer potrebbero differire.

### Costruttori disponibili

```
Int8Array, Uint8Array, Uint8ClampedArray
Int16Array, Uint16Array
Int32Array, Uint32Array
Float32Array, Float64Array
```

Forme di costruzione:

```js
new Uint8Array(buf);              // view su buffer esistente
new Uint8Array(buf, offset, len); // view con offset e lunghezza
new Uint8Array(8);                // nuovo buffer di 8 byte
new Uint8Array([10, 20, 30]);     // da array/iterable
new Uint8Array(otherTypedArr);    // copia da un altro TypedArray
```

### Comportamento

I TypedArray supportano la maggior parte dei metodi di `Array.prototype` (`map`, `filter`, `forEach`, `join`, ecc.), ma **non** i metodi mutanti come `push`, `splice`, `concat`. La lunghezza è fissa.

**Gotcha importante** — overflow silenzioso: se il valore eccede il range del tipo, viene troncato (wrap-around):

```js
var a = new Uint8Array([10, 20, 30]);
var b = a.map(v => v * v);
b; // [100, 144, 132]  — 400 e 900 hanno fatto overflow in 8 bit

/* soluzione: promuovere a un tipo più grande prima della map */
var b = Uint16Array.from(a, v => v * v);
b; // [100, 400, 900]
```

`sort()` su TypedArray usa il confronto numerico per default (non lessicografico come sugli Array normali):

```js
[10, 1, 2].sort();                 // [1, 10, 2]  — lessicografico
new Uint8Array([10, 1, 2]).sort(); // [1, 2, 10]  — numerico
```

---

## Map

Gli oggetti JS funzionano come mappe chiave/valore, ma con un limite: le chiavi devono essere stringhe (o Symbol). Qualsiasi altro valore viene convertito in stringa, causando collisioni inattese:

```js
var m = {};
var x = { id: 1 }, y = { id: 2 };
m[x] = "foo";
m[y] = "bar";
m[x]; // "bar" — entrambi stringificano a "[object Object]"
```

`Map` risolve il problema: qualsiasi valore — inclusi oggetti — può essere chiave, con identità per riferimento.

```js
var m = new Map();
var x = { id: 1 }, y = { id: 2 };

m.set(x, "foo");
m.set(y, "bar");
m.get(x);   // "foo"
m.get(y);   // "bar"

m.has(x);   // true
m.size;     // 2
m.delete(y);
m.clear();
```

### Inizializzazione da iterable

Il costruttore accetta un iterable di coppie `[chiave, valore]`:

```js
var m = new Map([
    [x, "foo"],
    [y, "bar"]
]);

/* copiare una Map */
var m2 = new Map(m);  // m è iterable e produce entries
```

### Iteratori

```js
var keys    = [...m.keys()];
var vals    = [...m.values()];
var entries = [...m.entries()]; // array di [chiave, valore]

for (var [k, v] of m) { /* default iterator = entries */ }
```

L'ordine di iterazione è l'ordine di inserimento.

---

## WeakMap

WeakMap è una variante di Map dove le **chiavi sono tenute debolmente** — se l'oggetto-chiave non ha più riferimenti nel codice, la voce nella WeakMap viene rimossa automaticamente dal garbage collector (GC).

```js
var m = new WeakMap();
var x = { id: 1 };
m.set(x, "foo");
m.has(x);     // true

x = null;     // { id: 1 } diventa GC-eligible; la voce in m verrà rimossa
```

Differenze rispetto a Map:
- Le chiavi **devono essere oggetti** (no primitive)
- Non ha `size`, `clear()`, né iteratori (non è ispezionabile)
- Solo: `set()`, `get()`, `has()`, `delete()`

La debolezza riguarda solo le **chiavi**, non i valori:

```js
var m = new WeakMap();
var key = { id: 1 }, val = { id: 2 };
m.set(key, val);
key = null; // la voce diventa GC-eligible, val incluso
val = null; // val sarebbe comunque GC-eligible solo se key lo è già
```

**Caso d'uso tipico**: associare dati privati a oggetti DOM o oggetti che non si controllano direttamente. Quando il DOM element viene rimosso, la voce nella WeakMap scompare automaticamente senza bisogno di cleanup manuale.

---

## Set

Un Set è una collezione di valori **unici** — i duplicati vengono ignorati silenziosamente.

```js
var s = new Set();
var x = { id: 1 }, y = { id: 2 };

s.add(x);
s.add(y);
s.add(x);    // duplicato ignorato
s.size;      // 2

s.has(x);    // true
s.delete(y);
s.clear();
```

Inizializzazione da iterable:

```js
var s = new Set([x, y]);

/* deduplicare un array */
var arr = [1, 2, 3, 4, "1", 2, 4, "5"];
var uniques = [...new Set(arr)];
uniques; // [1, 2, 3, 4, "1", "5"]
```

L'unicità usa un confronto simile a `Object.is()` — senza coercizione: `1` e `"1"` sono valori distinti.

### Iteratori

```js
var s = new Set([x, y]);

[...s.keys()];    // [x, y]   — stesso di values()
[...s.values()];  // [x, y]
[...s.entries()]; // [[x,x], [y,y]]  — chiave e valore sono lo stesso elemento

for (var v of s) { /* default iterator = values */ }
```

---

## WeakSet

WeakSet è a Set quello che WeakMap è a Map: i **valori sono tenuti debolmente**. Se un oggetto nel WeakSet non ha più riferimenti, viene rimosso automaticamente.

```js
var s = new WeakSet();
var x = { id: 1 }, y = { id: 2 };

s.add(x);
s.add(y);
x = null; // { id: 1 } diventa GC-eligible
```

- I valori devono essere **oggetti** (no primitive)
- Solo: `add()`, `has()`, `delete()`
- Nessun `size`, `clear()`, né iteratori

**Caso d'uso tipico**: tenere traccia di quali oggetti sono stati "visitati" o "processati", senza impedire il loro GC quando non servono più.

---

## ⚡ Ripasso veloce

```js
/* === TYPEDARRAY === */
var buf = new ArrayBuffer(16);    // buffer grezzo di 16 byte
var ints = new Int32Array(buf);   // 4 elementi da 32 bit
var bytes = new Uint8Array(buf);  // 16 elementi da 8 bit (stessi bit!)
ints[0] = 1000;
/* sort numerico per default, no push/splice/concat */
/* overflow silenzioso: valore > range → troncato */

/* === MAP === */
var m = new Map([[keyObj, "val"]]);
m.set(obj, data);  m.get(obj);  m.has(obj);
m.delete(obj);     m.size;      m.clear();
for (var [k, v] of m) { }       // ordine di inserimento
[...m.keys()]; [...m.values()]; [...m.entries()];

/* === WEAKMAP === */
var wm = new WeakMap();          // chiavi: solo oggetti, tenute debolmente
wm.set(domEl, privateData);      // se domEl viene rimosso → voce GC-ta
/* no size, no clear, no iteratori */

/* === SET === */
var s = new Set([1, 2, 2, 3]);   // {1, 2, 3}
s.add(4);  s.has(2);  s.delete(2);
var uniques = [...new Set(arr)]; // deduplicazione array
/* unicità senza coercizione: 1 !== "1" */

/* === WEAKSET === */
var ws = new WeakSet();          // valori: solo oggetti, tenuti debolmente
ws.add(obj);  ws.has(obj);      // no size, no iteratori
```

---

## Domande

<details>
<summary>Cos'è un <code>ArrayBuffer</code> e qual è il suo rapporto con i TypedArray?</summary>

Un `ArrayBuffer` è un blocco di memoria binaria grezza — semplicemente una sequenza di byte inizializzati a zero, senza alcuna struttura o tipo. Non è direttamente leggibile o scrivibile. Per interagire con i dati, si crea una *view* sopra il buffer tramite uno dei costruttori TypedArray (`Uint8Array`, `Int32Array`, `Float64Array`, ecc.). La view interpreta i bit del buffer come se fossero elementi di quel tipo (8-bit unsigned integers, 32-bit signed integers, ecc.). Lo stesso buffer può avere più view simultanee — modificando i dati tramite una view, le modifiche sono visibili attraverso tutte le altre view, perché condividono gli stessi bit sottostanti. Questa architettura permette di manipolare dati binari complessi (audio, video, WebGL, rete) in modo efficiente.

</details>

<details>
<summary>Perché <code>Map</code> risolve un problema che gli oggetti normali non possono risolvere?</summary>

Gli oggetti JS convertono automaticamente qualsiasi chiave in stringa tramite `toString()`. Questo significa che oggetti distinti come `{ id: 1 }` e `{ id: 2 }` stringificano entrambi a `"[object Object]"` e finiscono per essere la stessa chiave nell'oggetto — una sovrascrive l'altra. `Map` usa l'identità per riferimento come criterio per le chiavi oggetto: `x` e `y` sono chiavi distinte anche se hanno la stessa struttura, perché sono riferimenti a oggetti diversi in memoria. Questo rende `Map` la struttura corretta ogni volta che si ha bisogno di associare dati aggiuntivi a un oggetto senza modificare l'oggetto stesso — ad esempio per caching, metadata, o dati privati.

</details>

<details>
<summary>Qual è la differenza tra <code>Map</code> e <code>WeakMap</code>, e quando si preferisce l'una all'altra?</summary>

La differenza fondamentale è nel trattamento delle chiavi rispetto al garbage collector. In una `Map` normale, la chiave-oggetto è tenuta con un riferimento forte: anche se tutte le altre parti del codice hanno rimosso i riferimenti all'oggetto, la `Map` lo mantiene vivo in memoria. In una `WeakMap`, le chiavi sono tenute debolmente: se l'oggetto non ha altri riferimenti nel programma, il GC può rimuoverlo e la voce nella `WeakMap` scompare automaticamente. Si usa `WeakMap` quando la chiave è un oggetto che non si controlla direttamente (es. un DOM element) e si vuole che i dati associati vengano eliminati automaticamente quando l'oggetto viene distrutto. Il prezzo da pagare: `WeakMap` non è ispezionabile — non ha `size`, `clear()`, né iteratori.

</details>

<details>
<summary>Come funziona l'unicità in un <code>Set</code> e come si può sfruttare per deduplicare un array?</summary>

Un `Set` considera due valori identici se e solo se sono uguali secondo un algoritmo simile a `Object.is()` — senza coercizione. Quindi `1` e `"1"` sono valori distinti, `-0` e `0` invece sono trattati come uguali. Aggiungere un valore già presente non ha effetto e non genera errori. Per deduplicare un array, si costruisce un `Set` con l'array come iterable e si espande di nuovo in array: `[...new Set(arr)]`. L'ordine di inserimento viene preservato. Attenzione: i valori oggetto sono uguali solo se sono lo stesso riferimento in memoria — due oggetti `{ id: 1 }` distinti sono considerati diversi anche se hanno la stessa struttura.

</details>

<details>
<summary>Cosa distingue <code>WeakMap</code> da <code>WeakSet</code> e quali sono i rispettivi casi d'uso?</summary>

`WeakMap` associa a ogni chiave-oggetto un valore arbitrario — è una mappa chiave→valore dove le chiavi sono tenute debolmente. `WeakSet` è invece una collezione di oggetti unici senza alcun valore associato — serve solo a rispondere alla domanda "questo oggetto è presente nel set?". Il caso d'uso tipico di `WeakMap` è associare dati privati o metadata a oggetti esterni (DOM elements, oggetti di librerie terze) senza modificarli, con cleanup automatico quando l'oggetto viene rimosso. Il caso d'uso tipico di `WeakSet` è tenere traccia degli oggetti già processati — ad esempio in un algoritmo di visita di grafi — senza impedire il loro GC quando non servono più. Entrambi accettano solo oggetti come chiavi/valori, e nessuno dei due è iterabile o ha `size`.

</details>
