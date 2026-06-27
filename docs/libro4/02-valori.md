# Valori

Array, stringhe e numeri sono i mattoni fondamentali di qualsiasi programma. JavaScript ha caratteristiche peculiari su ciascuno di questi tipi che possono sorprendere chi arriva da altri linguaggi.

## Array

In JS gli array sono contenitori generici: possono tenere valori di qualsiasi tipo, compresi altri array. Non è necessario pre-dimensionarli — si dichiarano vuoti e si aggiungono valori liberamente:

```js
var a = [1, "2", [3]];
a.length;      // 3
a[2][0] === 3; // true
```

**Array sparsi**: lasciare slot vuoti tra gli indici è legale ma produce comportamenti ambigui. Lo slot vuoto si legge come `undefined`, ma non è la stessa cosa di uno slot esplicitamente impostato a `undefined`:

```js
var a = [];
a[0] = 1;
a[2] = [3]; /* a[1] non esiste */
a[1];       // undefined — ma lo slot non esiste
a.length;   // 3
```

**Chiavi stringa**: gli array sono oggetti, quindi accettano proprietà stringa, ma queste non influenzano `length`. Attenzione: se la stringa-chiave è convertibile a numero intero, viene interpretata come indice:

```js
var a = [];
a["foobar"] = 2;
a.length;    // 0 — non conta
a["13"] = 42;
a.length;    // 14 — interpretato come indice!
```

### Array-like

Oggetti con indici numerici e `.length` (come `arguments` o NodeList) non sono veri array. Si convertono con:

```js
var arr = Array.prototype.slice.call(arguments);
/* oppure ES6: */
var arr = Array.from(arguments);
```

---

## Stringhe

Le stringhe assomigliano agli array di caratteri ma **non lo sono**. La differenza fondamentale: le stringhe sono **immutabili**, gli array no.

```js
var a = "foo";
var b = ["f","o","o"];

a[1] = "O"; a; // "foo" — invariata
b[1] = "O"; b; // ["f","O","o"] — modificata
```

I metodi stringa che sembrerebbero modificarla in-place restituiscono invece nuove stringhe:

```js
var c = a.toUpperCase();
a; // "foo" — invariata
c; // "FOO"
```

Si possono "prendere in prestito" metodi array non-mutanti sulle stringhe:

```js
var c = Array.prototype.join.call(a, "-"); // "f-o-o"
var d = Array.prototype.map.call(a, v => v.toUpperCase() + ".").join(""); // "F.O.O."
```

I metodi mutanti (come `reverse()`) non funzionano su stringhe perché l'immutabilità lo impedisce. Per invertire una stringa semplice, il workaround classico è:

```js
var c = a.split("").reverse().join(""); // "oof"
/* attenzione: non funziona con caratteri Unicode multibyte */
```

---

## Numeri

JavaScript ha un solo tipo numerico — `number` — che comprende sia interi che decimali in virgola mobile. L'implementazione segue lo standard **IEEE 754** a doppia precisione (64-bit), lo stesso usato dalla maggior parte dei linguaggi moderni.

### Sintassi letterale

```js
var a = 42;      /* intero */
var b = 42.3;    /* decimale */
var c = .42;     /* 0 iniziale opzionale */
var d = 42.;     /* 0 finale opzionale — valido ma da evitare */
var e = 5E10;    /* notazione esponenziale: 50000000000 */
var f = 0xf3;    /* esadecimale: 243 */
var g = 0o363;   /* ottale ES6: 243 */
var h = 0b11110011; /* binario ES6: 243 */
```

Metodi su literal numerici: il `.` è ambiguo (parte del numero o property accessor?). Soluzioni:

```js
// 42.toFixed(3) → SyntaxError: il . è inglobato nel numero
(42).toFixed(3);  // "42.000" ✓
42..toFixed(3);   // "42.000" ✓ (primo . = parte del numero, secondo = accessor)
```

### Piccoli decimali e floating-point

```js
0.1 + 0.2 === 0.3; // false — 0.30000000000000004
```

Non è un bug JS: è il comportamento di IEEE 754 in tutti i linguaggi. Per comparare decimali con tolleranza si usa `Number.EPSILON` (ES6):

```js
function closeEnough(a, b) {
    return Math.abs(a - b) < Number.EPSILON;
}
closeEnough(0.1 + 0.2, 0.3); // true
```

### Range safe degli interi

Il massimo intero rappresentabile con precisione garantita è 2⁵³ − 1 = `9007199254740991`, disponibile come `Number.MAX_SAFE_INTEGER`. Per ID a 64 bit provenienti da database si usa la rappresentazione stringa.

```js
Number.isInteger(42.0); // true
Number.isInteger(42.3); // false
Number.isSafeInteger(Number.MAX_SAFE_INTEGER); // true
Number.isSafeInteger(Math.pow(2, 53));         // false
```

---

## Valori speciali

### `undefined` e `null`

Sono i soli valori dei rispettivi tipi. Semantica comune:
- `null` — valore vuoto, slot che aveva un valore e non ce l'ha più.
- `undefined` — valore mancante, slot che non ha ancora avuto un valore.

`null` è una keyword, non può essere riassegnato. `undefined` è (purtroppo) un identificatore: in non-strict mode è tecnicamente assegnabile — da non fare mai.

**`void`**: produce `undefined` da qualsiasi espressione senza modificarla. Utile nei rari casi in cui si vuole scartare un valore di ritorno:

```js
var a = 42;
void a; // undefined — a rimane 42

/* uso pratico: ignorare il return value di setTimeout */
return void setTimeout(doSomething, 100);
```

### `NaN` — "numero non valido"

`NaN` (Not a Number) è il risultato di un'operazione aritmetica fallita. Il tipo resta `"number"`:

```js
var a = 2 / "foo"; // NaN
typeof a;          // "number"
```

`NaN` è l'unico valore in JS che **non è uguale a se stesso**:

```js
NaN === NaN; // false — proprietà unica
```

Il global `isNaN()` ha un bug storico: restituisce `true` per qualsiasi non-numero, non solo per `NaN`:

```js
isNaN("foo"); // true — sbagliato: "foo" non è NaN, è solo non-un-numero
```

Si usa sempre `Number.isNaN()` (ES6) o il polyfill basato sulla peculiarità di auto-disuguaglianza:

```js
Number.isNaN(2 / "foo"); // true
Number.isNaN("foo");     // false ✓

/* polyfill: NaN è l'unico valore dove n !== n */
Number.isNaN = function(n) { return n !== n; };
```

### `Infinity` e `-Infinity`

Divisione per zero non lancia eccezioni in JS — produce `Infinity`:

```js
1 / 0;   // Infinity
-1 / 0;  // -Infinity
Infinity / Infinity; // NaN — operazione indefinita
```

Una volta raggiunto l'infinito non si torna indietro: qualsiasi operazione tra infinito e un numero finito resta infinita.

### `-0` (zero negativo)

JS ha due zeri: `0` e `-0`. Il motivo pratico: rappresentare con un solo valore sia la magnitudine (zero) che la direzione (segno negativo, per esempio in animazioni).

```js
var a = 0 / -3; // -0
var b = 0 * -3; // -0

/* -0 mente nelle stringhe e nei confronti */
String(-0);  // "0" — deceptive
-0 === 0;    // true — deceptive
-0 > 0;      // false

/* rilevazione corretta */
function isNegZero(n) {
    n = Number(n);
    return (n === 0) && (1 / n === -Infinity);
}
isNegZero(-0); // true
isNegZero(0);  // false
```

### `Object.is()` — uguaglianza assoluta

ES6 introduce `Object.is()` per confronti senza le eccezioni di `NaN` e `-0`:

```js
Object.is(NaN, NaN); // true
Object.is(-0, 0);    // false
Object.is(-0, -0);   // true
```

Da usare solo per questi casi edge — `===` rimane più efficiente e idiomatico negli altri contesti.

---

## Value vs Reference

In JS non esistono puntatori. Il tipo del **valore** determina se viene copiato o passato per riferimento — non la sintassi, non la dichiarazione.

**Primitivi** (`null`, `undefined`, `string`, `number`, `boolean`, `symbol`): sempre **value-copy**. Ogni assegnamento o passaggio crea una copia indipendente.

**Compound values** (oggetti, array, funzioni): sempre **reference-copy**. Il riferimento punta al valore condiviso — non a un'altra variabile.

```js
/* primitivo: copia del valore */
var a = 2;
var b = a;
b++;
a; // 2 — invariato

/* compound: copia del riferimento */
var c = [1, 2, 3];
var d = c;
d.push(4);
c; // [1, 2, 3, 4] — modificato anche c
```

**Riassegnare un riferimento non cambia l'originale**:

```js
var a = [1, 2, 3];
var b = a;
b = [4, 5, 6]; /* rimpiazza il riferimento di b, non a */
a; // [1, 2, 3] — invariato
```

**Con i parametri delle funzioni**: passare un array a una funzione passa una copia del riferimento. Mutare il contenuto dell'array è visibile all'esterno; riassegnare il parametro no:

```js
function foo(x) {
    x.push(4);   /* visibile fuori: a diventa [1,2,3,4] */
    x = [4,5,6]; /* non visibile fuori: sostituisce solo il riferimento locale */
    x.push(7);
}
var a = [1, 2, 3];
foo(a);
a; // [1, 2, 3, 4] — non [4,5,6,7]
```

Per modificare il contenuto in-place, usare `x.length = 0` + `push`:

```js
function foo(x) {
    x.length = 0;
    x.push(4, 5, 6, 7);
}
var a = [1, 2, 3];
foo(a);
a; // [4, 5, 6, 7] ✓
```

Per passare un array **per valore** (impedire modifiche dall'esterno): copiarlo prima con `slice()`:

```js
foo(a.slice()); /* foo riceve un nuovo array — a è al sicuro */
```

---

## ⚡ Ripasso veloce

**Array**: contenitori generici, indici numerici. Le chiavi stringa non contano per `length`. Gli slot vuoti (sparse array) sono leciti ma ambigui.

**Stringhe**: immutabili. I metodi non modificano in-place. Si possono prendere in prestito metodi array non-mutanti.

**Numeri**: IEEE 754 double precision. `0.1 + 0.2 !== 0.3` — usare `Number.EPSILON` per confronti decimali. Max safe integer: `Number.MAX_SAFE_INTEGER` (2⁵³ − 1).

**Valori speciali**:
- `NaN`: tipo `"number"`, mai uguale a se stesso. Usare `Number.isNaN()`, non `isNaN()`.
- `-0`: uguale a `0` con `===`, ma rilevabile con `1/n === -Infinity`.
- `Object.is()`: uguaglianza assoluta per i casi edge di `NaN` e `-0`.

**Value vs Reference**: il tipo del valore decide. Primitivi → copia. Oggetti/array → riferimento condiviso.

```js
/* NaN check corretto */
Number.isNaN(2 / "foo"); // true
Number.isNaN("foo");     // false

/* -0 check */
(n === 0) && (1 / n === -Infinity); // true solo per -0

/* Object.is per casi edge */
Object.is(NaN, NaN); // true
Object.is(-0, 0);    // false
```

---

## Domande

<details>
<summary>Perché gli array sparsi sono problematici in JavaScript?</summary>

Un array sparso ha slot "vuoti" — indici che non sono mai stati assegnati esplicitamente. Il valore letto da uno slot vuoto è `undefined`, ma non è lo stesso di uno slot a cui è stato assegnato `undefined`. La differenza emerge con metodi come `map()`, `forEach()` o `filter()`: questi iterano solo sugli slot che esistono effettivamente, saltando i vuoti. Il risultato è che `[1,,3].map(x => x * 2)` produce `[2, empty, 6]` — lo slot vuoto viene saltato silenziosamente. Gli array sparsi portano a comportamenti imprevedibili e vanno evitati.

</details>

<details>
<summary>Perché non si può invertire una stringa con `Array.prototype.reverse.call(str)`?</summary>

Perché le stringhe sono **immutabili**: non possono essere modificate in-place. `reverse()` è un metodo mutante che modifica il valore originale sul posto. Applicarlo a una stringa (tramite borrowing) non lancia un errore, ma non ha effetto sulla stringa originale — restituisce un oggetto `String` wrapper per la stringa originale non modificata. Il workaround standard (`split("").reverse().join("")`) funziona perché converte la stringa in un array di caratteri (mutable), inverte l'array, e ricrea la stringa. Non funziona però con caratteri Unicode multibyte.

</details>

<details>
<summary>Perché `isNaN("foo")` restituisce `true` ed è considerato un bug?</summary>

Il global `isNaN()` interpreta il suo argomento troppo letteralmente: controlla se il valore "non è un numero" — includendo qualsiasi valore non numerico, non solo il valore speciale `NaN`. `"foo"` non è un numero, quindi `isNaN("foo")` restituisce `true`, anche se `"foo"` non è `NaN`. `Number.isNaN()` (ES6) è la versione corretta: verifica prima che il tipo sia `"number"`, e poi che il valore sia effettivamente `NaN`. In alternativa si può sfruttare la proprietà unica di `NaN` — è l'unico valore non uguale a se stesso: `function isNaN(n) { return n !== n; }`.

</details>

<details>
<summary>Perché `-0 === 0` è `true` in JavaScript, e quando è importante distinguerli?</summary>

È una scelta della specifica: gli operatori di confronto (`==` e `===`) trattano `-0` come uguale a `0`. Anche `String(-0)` restituisce `"0"`. La distinzione è nascosta intenzionalmente nella maggior parte dei contesti. Diventa rilevante in applicazioni dove il segno di un valore numerico codifica informazioni aggiuntive — per esempio, in un'animazione dove `0` significa "fermo" e il segno indica l'ultima direzione di movimento. Se si perde il segno dello zero, si perde anche l'informazione sulla direzione. Per rilevare `-0` si usa `(n === 0) && (1/n === -Infinity)`, oppure `Object.is(n, -0)` (ES6).

</details>

<details>
<summary>Qual è la differenza tra passare per value-copy e passare per reference-copy, e come si controlla in JS?</summary>

In JS non si controlla la modalità di passaggio con la sintassi — la decide il **tipo del valore**. I primitivi (`number`, `string`, `boolean`, `null`, `undefined`, `symbol`) vengono sempre copiati per valore: la funzione riceve una copia indipendente e non può modificare l'originale. Gli oggetti e gli array vengono passati per reference-copy: la funzione riceve una copia del riferimento allo stesso valore condiviso, può quindi mutare il contenuto dell'oggetto/array (visibile all'esterno), ma non può rimpiazzare l'originale riassegnando il parametro. Per "passare un array per valore" si copia esplicitamente prima: `foo(a.slice())`. Per "passare un primitivo per riferimento" si lo avvolge in un oggetto: `foo({ val: 42 })`.

</details>
