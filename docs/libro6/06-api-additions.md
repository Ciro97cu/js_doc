# Aggiunte alle API

ES6 aggiunge metodi statici e di prototype a `Array`, `Object`, `Math`, `Number` e `String`. La maggior parte è polyfillabile.

---

## Array

### `Array.of()`

Il costruttore `Array(n)` con un singolo argomento numerico crea un array di `n` slot vuoti — un comportamento sorprendente. `Array.of()` lo sostituisce come forma funzionale del costruttore:

```js
Array(3);           // [ , , ,]  — 3 slot vuoti
Array.of(3);        // [3]       — array con un elemento
Array.of(1, 2, 3);  // [1, 2, 3]
```

Utile anche nelle subclass: `MyCoolArray.of(3)` produce un'istanza di `MyCoolArray` con `length: 1`, senza il bug del costruttore base.

### `Array.from()`

Converte array-like (oggetti con proprietà numeriche e `length`) e iterable in array reali — rimpiazza il pattern `Array.prototype.slice.call(arrayLike)`:

```js
var arrayLike = { length: 3, 0: "foo", 1: "bar" };
Array.from(arrayLike); // ["foo", "bar", undefined]

/* copia di un array */
Array.from(arr);

/* da iterable */
Array.from(new Set([1, 2, 3]));

/* array di undefined (no slot vuoti) */
Array.from({ length: 4 }); // [undefined, undefined, undefined, undefined]
```

**Differenza critica**: `Array.from()` non produce mai slot vuoti — i slot mancanti diventano `undefined` esplicito.

Accetta un secondo argomento — callback di mapping applicata durante la conversione:

```js
Array.from({ length: 4 }, (val, idx) => idx * 2); // [0, 2, 4, 6]
Array.from(uint8arr, v => v * v);                  // utile per TypedArray con overflow
```

### `copyWithin(target, start[, end])`

Copia una porzione dell'array in un'altra posizione **nello stesso array**, sovrascrivendo i valori presenti. Non cambia la lunghezza dell'array. Argomenti negativi sono relativi alla fine:

```js
[1, 2, 3, 4, 5].copyWithin(3, 0);       // [1, 2, 3, 1, 2]  — copia da 0 a posizione 3
[1, 2, 3, 4, 5].copyWithin(3, 0, 1);    // [1, 2, 3, 1, 5]  — copia solo l'indice 0
[1, 2, 3, 4, 5].copyWithin(0, -2);      // [4, 5, 3, 4, 5]  — -2 = indice 3
```

### `fill(value[, start, end])`

Riempie l'array (o una sua parte) con un valore specificato:

```js
new Array(4).fill(undefined);               // [undefined, undefined, undefined, undefined]
[null, null, null, null].fill(42, 1, 3);    // [null, 42, 42, null]
```

### `find()` e `findIndex()`

`indexOf()` usa `===` e non è personalizzabile. `find()` e `findIndex()` accettano una callback di matching e restituiscono rispettivamente il valore trovato e il suo indice:

```js
var a = [1, 2, 3, 4, 5];

a.find(v => v == "2");       // 2  (trova il primo match truthy della callback)
a.find(v => v == 7);         // undefined

a.findIndex(v => v == "2");  // 1
a.findIndex(v => v == 7);    // -1

/* utile per oggetti complessi */
var points = [{ x: 10, y: 20 }, { x: 30, y: 40 }];
points.find(p => p.x % 3 === 0 && p.y % 4 === 0); // { x: 30, y: 40 }
```

Regola: usare `some()` per boolean, `find()` per il valore, `findIndex()` per l'indice con logica custom, `indexOf()` per uguaglianza stretta.

### `entries()`, `values()`, `keys()`

Array espone gli stessi iteratori di Map e Set:

```js
var a = [1, 2, 3];
[...a.values()];   // [1, 2, 3]   — default iterator
[...a.keys()];     // [0, 1, 2]
[...a.entries()];  // [[0,1], [1,2], [2,3]]
```

---

## Object

### `Object.is()`

Confronto di uguaglianza più preciso di `===`, senza le eccezioni per `NaN` e `-0`:

```js
NaN === NaN;              // false
Object.is(NaN, NaN);      // true  ← corretto

0 === -0;                 // true
Object.is(0, -0);         // false ← corretto
```

Non è un sostituto di `===` per uso generale. Va preferito quando serve identificare specificamente `NaN` o `-0`.

### `Object.getOwnPropertySymbols()`

Recupera le proprietà con chiave Symbol di un oggetto (invisibili a `for..in` e `Object.keys()`):

```js
var o = {
    foo: 42,
    [Symbol("bar")]: "hello"
};
Object.getOwnPropertySymbols(o); // [Symbol(bar)]
```

### `Object.setPrototypeOf()`

Imposta il `[[Prototype]]` di un oggetto dopo la sua creazione:

```js
var o1 = { foo() { console.log("foo"); } };
var o2 = {};
Object.setPrototypeOf(o2, o1);
o2.foo(); // "foo"  — delega a o1
```

Equivale a `__proto__` nei literal, ma applicato post-creazione. Cambiare il prototype dopo la creazione è possibile ma da evitare salvo casi necessari.

### `Object.assign(target, ...sources)`

Copia le proprietà **proprie ed enumerabili** (inclusi Symbol enumerabili) da uno o più source verso target, tramite semplice assegnazione `=`. Restituisce il target:

```js
var target = {};
var o1 = { a: 1 }, o2 = { b: 2 };
Object.assign(target, o1, o2);
target; // { a: 1, b: 2 }
```

**Cosa non viene copiato**: proprietà non-enumerabili, proprietà non-proprie (ereditate), Symbol non-enumerabili. Le proprietà vengono copiate **per valore**, non duplicando i descriptor — una proprietà read-only nella source diventa scrivibile nel target.

Idioma comune per creare un oggetto con prototype personalizzato e proprietà proprie:

```js
var o2 = Object.assign(Object.create(o1), {
    bar: 42
});
```

---

## Math

ES6 aggiunge utility matematiche per trigonometria iperbolica, aritmetica precisa e operazioni bit-level:

| Categoria | Funzioni |
|---|---|
| Trigonometria iperbolica | `Math.sinh()`, `Math.cosh()`, `Math.tanh()`, `Math.asinh()`, `Math.acosh()`, `Math.atanh()` |
| Geometria | `Math.hypot()` — radice della somma dei quadrati (teorema di Pitagora generalizzato) |
| Aritmetica | `Math.cbrt()` (radice cubica), `Math.log2()`, `Math.log10()`, `Math.log1p()`, `Math.expm1()` |
| Integer/bit | `Math.clz32()` (leading zero bits in 32-bit), `Math.imul()` (moltiplicazione intera 32-bit) |
| Meta | `Math.sign()` (segno del numero), `Math.trunc()` (parte intera), `Math.fround()` (arrotondamento a float32) |

---

## Number

### Costanti statiche

```js
Number.EPSILON;           // 2^-52 — margine minimo di differenza tra due float
Number.MAX_SAFE_INTEGER;  // 2^53 - 1 — massimo intero rappresentabile senza ambiguità
Number.MIN_SAFE_INTEGER;  // -(2^53 - 1)
```

### Funzioni statiche

**`Number.isNaN()`** — versione corretta del globale `isNaN()`: non fa coercizione, quindi distingue correttamente `NaN` da stringhe non-numeriche:

```js
isNaN("NaN");        // true  — sbagliato: la stringa viene coerced a NaN
Number.isNaN("NaN"); // false — corretto
Number.isNaN(NaN);   // true
```

**`Number.isFinite()`** — versione senza coercizione del globale `isFinite()`:

```js
isFinite("42");        // true  — la stringa viene coerced a 42
Number.isFinite("42"); // false — nessuna coercizione
Number.isFinite(42);   // true
Number.isFinite(NaN);  // false
Number.isFinite(Infinity); // false
```

**`Number.isInteger()`** — verifica se il valore è un intero (senza parte decimale non nulla):

```js
Number.isInteger(4);    // true
Number.isInteger(4.0);  // true  — 4.0 === 4 in JS
Number.isInteger(4.2);  // false
Number.isInteger(NaN);  // false
Number.isInteger(Infinity); // false
```

Per `isFloat()` non basta `!Number.isInteger()` perché `NaN` e `Infinity` non sono interi ma neanche float:

```js
function isFloat(x) {
    return Number.isFinite(x) && !Number.isInteger(x);
}
```

**`Number.isSafeInteger()`** — verifica che sia un intero nel range `[MIN_SAFE_INTEGER, MAX_SAFE_INTEGER]`:

```js
Number.isSafeInteger(Math.pow(2, 53) - 1);  // true
Number.isSafeInteger(Math.pow(2, 53));       // false
```

`Number.parseInt()` e `Number.parseFloat()` sono alias degli omonimi globali — spostati su `Number` per coerenza con il principio di ridurre le dipendenze dall'oggetto globale.

---

## String

### Funzioni Unicode

Già descritte nel capitolo Sintassi, qui per completezza:

```js
String.fromCodePoint(0x1d49e);      // "𝒞"
"ab𝒞d".codePointAt(2).toString(16); // "1d49e"
"é".normalize();               // "é" — combina e + combining accent
"é".normalize().length;        // 1
```

### `String.raw`

Tag built-in per template literal che restituisce la stringa raw senza interpretare le escape:

```js
String.raw`\ta${str}d\xE9`;  // "\tabcd\xE9" — letteralmente backslash-t, non tab
```

### `repeat(n)`

```js
"foo".repeat(3);   // "foofoofoo"
"-".repeat(10);    // "----------"
```

### `startsWith()`, `endsWith()`, `includes()`

Sostituiscono i pattern con `indexOf()` per i casi comuni:

```js
var s = "step on no pets";

s.startsWith("step on");    // true
s.startsWith("on", 5);      // true  — inizia la ricerca dall'indice 5
s.endsWith("no pets");      // true
s.endsWith("no", 10);       // true  — considera solo i primi 10 caratteri
s.includes("on");           // true
s.includes("on", 6);        // false — inizia la ricerca dall'indice 6
```

---

## ⚡ Ripasso veloce

```js
/* ARRAY */
Array.of(3);                    // [3]  — no bug del costruttore
Array.from(arrayLike, mapper);  // da array-like/iterable, con mapping opzionale
arr.find(fn);   arr.findIndex(fn);  // ricerca con callback custom
arr.fill(val, start, end);
arr.copyWithin(target, start, end);
[...arr.entries()]; [...arr.keys()]; [...arr.values()];

/* OBJECT */
Object.is(NaN, NaN);  // true  — NaN === NaN
Object.is(0, -0);     // false — 0 !== -0
Object.assign(target, ...sources); // shallow copy, enumerable+own
Object.getOwnPropertySymbols(obj); // symbol props
Object.setPrototypeOf(obj, proto);

/* MATH */
Math.trunc(4.9);  // 4   — parte intera senza arrotondamento
Math.sign(-5);    // -1  — segno: -1 | 0 | 1
Math.cbrt(27);    // 3   — radice cubica
Math.hypot(3, 4); // 5   — Pitagora

/* NUMBER */
Number.isNaN("foo");        // false  — no coercizione
Number.isFinite("42");      // false  — no coercizione
Number.isInteger(4.0);      // true
Number.isSafeInteger(2**53); // false
Number.EPSILON;              // 2^-52
Number.MAX_SAFE_INTEGER;     // 2^53 - 1

/* STRING */
"abc".repeat(3);                            // "abcabcabc"
"hello world".includes("world");            // true
"hello world".startsWith("hello");          // true
"hello world".endsWith("world");            // true
String.raw`\n`;                             // "\\n" — no interpretazione escape
```

---

## Domande

<details>
<summary>Perché <code>Array.of()</code> esiste se si può già usare la sintassi literal <code>[...]</code>?</summary>

La sintassi literal `[3]` crea un array con un elemento, mentre `Array(3)` crea un array con 3 slot vuoti — un comportamento sorprendente del costruttore `Array`. `Array.of()` risolve questo caso edge quando si ha bisogno della forma funzionale del costruttore, ad esempio: (1) in un callback che deve avvolgere i propri argomenti in un array senza sapere quanti ne riceverà; (2) nelle subclass di `Array` — `MyCoolArray.of(3)` crea correttamente un'istanza di `MyCoolArray` con `length: 1`, dove `new MyCoolArray(3)` erediterebbe il bug del costruttore base producendo 3 slot vuoti.

</details>

<details>
<summary>Qual è la differenza tra <code>Array.from()</code> e <code>Array.prototype.slice.call()</code> per convertire array-like in array?</summary>

Entrambi convertono array-like in array reali, ma con una differenza importante: `slice()` preserva gli slot vuoti (empty slots), mentre `Array.from()` non li produce mai — un slot mancante diventa `undefined` esplicito. In pratica: `Array.from({length: 3})` produce `[undefined, undefined, undefined]` (tre valori reali), mentre `Array.apply(null, {length: 3})` o `new Array(3)` producono un array con slot vuoti che si comportano in modo inconsistente con `map()`, `filter()` e altri metodi. Inoltre `Array.from()` accetta un secondo argomento di mapping, eliminando la necessità di un `.map()` separato, e funziona con qualsiasi iterable (non solo array-like).

</details>

<details>
<summary>Cosa copia esattamente <code>Object.assign()</code> e cosa invece non copia?</summary>

`Object.assign()` copia le proprietà che sono **proprie** (non ereditate dalla catena dei prototype) **ed enumerabili** dell'oggetto source. Questo include le proprietà Symbol enumerabili. Non copia: proprietà non-enumerabili, proprietà ereditate (nel prototype), Symbol non-enumerabili. La copia avviene tramite semplice assegnazione `=` — non viene duplicato il property descriptor. Di conseguenza, una proprietà read-only (`writable: false`) nella source diventa una normale proprietà scrivibile nel target. È una copia superficiale (shallow): i valori oggetto vengono copiati per riferimento, non clonati. Per lo stesso motivo non è un modo affidabile per clonare oggetti complessi.

</details>

<details>
<summary>Perché <code>Number.isNaN()</code> si comporta diversamente dal globale <code>isNaN()</code>?</summary>

La funzione globale `isNaN()` coerisce il suo argomento a numero prima di verificare se è `NaN`. Questo significa che `isNaN("foo")` restituisce `true` — non perché `"foo"` sia `NaN`, ma perché `Number("foo")` produce `NaN`. `Number.isNaN()` non fa coercizione: verifica se il valore passato è letteralmente il valore `NaN`, e restituisce `false` per qualsiasi altro valore incluse stringhe, `undefined`, o `null`. La versione globale è difettosa per design — esisteva prima che si capisse il problema. `Number.isNaN()` è la versione corretta da preferire. Analogamente, `Number.isFinite()` non coerce l'argomento, dove il globale `isFinite("42")` restituirebbe `true`.

</details>

<details>
<summary>Qual è la differenza tra <code>find()</code>, <code>findIndex()</code>, <code>indexOf()</code> e <code>some()</code> per cercare in un array?</summary>

I quattro metodi rispondono a domande diverse. `some(fn)` risponde a "esiste almeno un elemento che soddisfa la condizione?" → restituisce boolean. `find(fn)` risponde a "qual è il primo elemento che soddisfa la condizione?" → restituisce il valore, o `undefined`. `findIndex(fn)` risponde a "a quale indice si trova?" → restituisce l'indice, o `-1`. `indexOf(val)` risponde a "a quale indice si trova questo valore specifico?" → usa `===` senza callback. La differenza chiave tra `indexOf` e `findIndex`: `indexOf` usa uguaglianza stretta senza possibilità di personalizzazione; `findIndex` accetta una callback arbitraria, permettendo match su oggetti complessi, valori approssimati, o criteri multipli. Non ha senso usare `findIndex(fn) !== -1` (quello che fa `some`), né `arr[arr.findIndex(fn)]` (quello che fa `find`).

</details>
