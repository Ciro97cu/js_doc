# Natives

I **natives** sono le funzioni costruttore built-in di JavaScript: `String()`, `Number()`, `Boolean()`, `Array()`, `Object()`, `Function()`, `RegExp()`, `Date()`, `Error()`, `Symbol()`. Sono disponibili come costruttori ma il loro uso diretto è quasi sempre sconsigliato.

## Object wrapper e autoboxing

Chiamando `new String("abc")` non si ottiene una stringa — si ottiene un **oggetto** wrapper attorno alla stringa primitiva:

```js
var a = new String("abc");
typeof a;                            // "object" — non "string"
a instanceof String;                 // true
Object.prototype.toString.call(a);   // "[object String]"
```

I primitivi (`string`, `number`, `boolean`) non hanno proprietà o metodi propri. Quando si accede a `.length` o `.toUpperCase()` su un literal, l'engine esegue automaticamente il **boxing** (wrapping temporaneo nel corrispondente oggetto): crea un `String` object, invoca il metodo, poi lo scarta.

```js
var a = "abc";
a.length;        // 3 — autoboxing implicito
a.toUpperCase(); // "ABC"
```

Non conviene creare esplicitamente gli object wrapper per "pre-ottimizzare": i browser ottimizzano già i casi comuni con i primitivi, e usare la forma oggetto risulta più lenta.

### `[[Class]]` interno

Tutti i valori di tipo `"object"` hanno una classificazione interna `[[Class]]` visibile tramite:

```js
Object.prototype.toString.call([1,2,3]);    // "[object Array]"
Object.prototype.toString.call(/regex/i);   // "[object RegExp]"
Object.prototype.toString.call(null);       // "[object Null]"
Object.prototype.toString.call(undefined);  // "[object Undefined]"
/* i primitivi vengono autoboxati prima della chiamata */
Object.prototype.toString.call("abc");      // "[object String]"
Object.prototype.toString.call(42);         // "[object Number]"
```

### Gotcha: `Boolean` wrapper è sempre truthy

```js
var a = new Boolean(false);
if (!a) {
    console.log("Oops"); // non viene mai eseguito!
}
```

L'oggetto wrapper `Boolean(false)` è un oggetto — e tutti gli oggetti sono **truthy**. Il valore primitivo che racchiude (`false`) è irrilevante per la valutazione booleana.

### Unboxing con `valueOf()`

Per estrarre il primitivo da un wrapper si usa `valueOf()`:

```js
var a = new String("abc");
var b = new Number(42);
a.valueOf(); // "abc"
b.valueOf(); // 42
```

L'unboxing avviene anche implicitamente quando il wrapper è usato in un contesto che richiede un primitivo (coercizione, Cap 4):

```js
var a = new String("abc");
var b = a + ""; // b è la stringa primitiva "abc"
typeof a;       // "object"
typeof b;       // "string"
```

---

## Natives come costruttori — quando evitarli

### `Array()`

```js
var a = new Array(1, 2, 3); // [1, 2, 3] — uguale a [1,2,3]
var b = new Array(3);       /* PERICOLO: non [3], ma array con length=3 e slot vuoti */
```

`Array(3)` con un solo argomento numerico non crea un array con il valore `3` — crea uno **sparse array** con `length` 3 e nessuno slot reale. Il comportamento è inconsistente:

```js
var a = new Array(3);
var b = [undefined, undefined, undefined];

a.join("-");  // "--" — funziona (join itera su length)
b.join("-");  // "--"

a.map((v, i) => i); // [empty × 3] — fallisce (map salta slot vuoti)
b.map((v, i) => i); // [0, 1, 2]  — funziona
```

Per creare un array di `undefined` reali (non slot vuoti):

```js
var a = Array.apply(null, { length: 3 });
a; // [undefined, undefined, undefined]
```

**Regola**: non usare mai `Array(n)` con un solo argomento numerico.

### `Object()`, `Function()`, `RegExp()`

Tutte preferire nella forma literal:

```js
/* preferire sempre il literal */
var obj = { foo: "bar" };             /* vs new Object() */
var fn  = function(a) { return a*2; }; /* vs new Function("a","return a*2") */
var rx  = /^a*b+/g;                   /* vs new RegExp("^a*b+","g") */
```

`RegExp()` ha un'unica ragione d'uso legittima: costruire pattern regex dinamicamente a runtime:

```js
var name = "Kyle";
var pattern = new RegExp("\\b(?:" + name + ")+\\b", "ig");
```

### `Date()` e `Error()`

Questi due **non hanno forma literal** — il costruttore è necessario:

```js
/* timestamp Unix corrente */
var ts = Date.now(); /* ES5 — preferita */
/* polyfill pre-ES5: */
if (!Date.now) { Date.now = function() { return (new Date()).getTime(); }; }

/* oggetto errore — cattura lo stack trace */
function foo(x) {
    if (!x) throw new Error("x non fornito");
}
```

`Date()` senza `new` restituisce una stringa con data/ora corrente — non un oggetto Date.

Gli oggetti `Error` catturano il contesto dello stack di esecuzione nel momento della creazione (proprietà `.stack`). Oltre a `Error`, esistono sottotipi specifici: `TypeError`, `ReferenceError`, `SyntaxError`, ecc. — usati automaticamente dall'engine, raramente necessari manualmente.

### `Symbol()`

I **symbol** (ES6) sono valori primitivi unici usati come chiavi di proprietà prive di rischi di collisione:

```js
var mysym = Symbol("my own symbol"); /* SENZA new — lancerebbe errore */
typeof mysym;    // "symbol"
mysym.toString(); // "Symbol(my own symbol)"

var obj = {};
obj[mysym] = "foobar";
Object.getOwnPropertySymbols(obj); // [Symbol(my own symbol)]
```

I symbol non sono privati (sono enumerabili con `getOwnPropertySymbols`), ma convenzionalmente segnalano proprietà interne/speciali — sostituendo il pattern del prefisso `_`. Non sono oggetti: sono scalari primitivi.

---

## Native prototypes

Ogni native costruttore ha il proprio `.prototype` con i metodi specifici del sottotipo:

- `String.prototype` → `indexOf()`, `charAt()`, `slice()`, `toUpperCase()`, `trim()`, …
- `Number.prototype` → `toFixed()`, `toPrecision()`, `toExponential()`, …
- `Array.prototype` → `push()`, `pop()`, `map()`, `filter()`, `concat()`, …
- `Function.prototype` → `call()`, `apply()`, `bind()`, …

Tutti i valori del relativo tipo accedono a questi metodi tramite prototype delegation (autoboxing incluso).

Una curiosità: alcuni prototype hanno tipi particolari:

```js
typeof Function.prototype;        // "function" — è una funzione vuota!
Function.prototype();             // nessun errore — funzione no-op
RegExp.prototype.toString();      // "/(?:)/" — regex vuota
Array.isArray(Array.prototype);   // true — è un array!
```

### Prototypes come valori default

Questa peculiarità permette un pattern utile per default "gratuiti":

```js
function isThisCool(vals, fn, rx) {
    vals = vals || Array.prototype;    /* default: array vuoto */
    fn   = fn   || Function.prototype; /* default: funzione no-op */
    rx   = rx   || RegExp.prototype;   /* default: regex vuota */
    return rx.test(vals.map(fn).join(""));
}
isThisCool(); // true — nessun errore
```

I prototype sono già creati una volta sola al boot del runtime — nessuna allocazione aggiuntiva. **Attenzione**: usarli come default va bene solo se non vengono mai modificati; mutare `Array.prototype` ha effetti globali.

---

## ⚡ Ripasso veloce

**Autoboxing**: l'engine converte automaticamente primitivi in wrapper temporanei per accedere a metodi. Non creare wrapper manualmente con `new String()`, `new Number()`, ecc.

**`new Boolean(false)` è truthy**: il wrapper oggetto è sempre truthy, indipendentemente dal primitivo che contiene.

**`Array(n)` con un solo numero**: crea uno sparse array, non un array con il valore `n`. Da evitare assolutamente.

**`Date` e `Error`**: gli unici natives senza forma literal. `Date.now()` per il timestamp; `new Error(msg)` per catturare lo stack trace.

**`Symbol()`**: senza `new`. Chiavi uniche per proprietà speciali/interne.

```js
/* boxing implicito — corretto */
"abc".toUpperCase(); // "ABC"

/* boxing esplicito — da evitare */
new String("abc").toUpperCase(); // "ABC" ma inutilmente verboso e lento

/* unboxing */
new String("abc").valueOf(); // "abc"

/* Boolean gotcha */
if (new Boolean(false)) { /* si esegue sempre — l'oggetto è truthy */ }

/* Array(n) gotcha */
new Array(3).map((v,i) => i); // [empty × 3] — map non itera

/* Symbol */
var sym = Symbol("desc");
typeof sym; // "symbol"
```

---

## Domande

<details>
<summary>Cosa restituisce `new String("abc")` e perché non va usato direttamente?</summary>

Restituisce un **oggetto** (`typeof` → `"object"`) che racchiude la stringa primitiva `"abc"`, non la stringa stessa. Questo causa comportamenti inaspettati: l'oggetto è sempre truthy (anche se wrappa `false` o `""` o `0`), i confronti con `===` falliscono (`new String("abc") === "abc"` → `false`), e il debug è più complesso. L'engine fa già l'autoboxing automaticamente quando serve accedere a metodi o proprietà — non è mai necessario creare wrapper manualmente. La forma corretta è sempre il literal primitivo: `"abc"`, `42`, `true`.

</details>

<details>
<summary>Perché `new Array(3)` è pericoloso?</summary>

Perché `Array()` con un singolo argomento numerico non crea un array contenente quel numero — crea un array con `length` impostato a quel numero ma senza slot reali (sparse array). Il comportamento è incoerente tra i metodi: `join()` funziona iterando su `length`, ma `map()`, `filter()`, `forEach()` saltano gli slot vuoti perché non esistono. Il risultato è codice che "funziona" in certi casi e fallisce silenziosamente in altri. Per creare un array di `undefined` reali: `Array.apply(null, { length: 3 })`.

</details>

<details>
<summary>Qual è l'unico caso d'uso legittimo di `new RegExp()`?</summary>

Costruire pattern regex dinamicamente a runtime, quando il pattern non è noto in anticipo al momento della scrittura del codice. Per esempio, costruire una regex che cerchi un nome variabile: `new RegExp("\\b" + username + "\\b", "ig")`. Per tutti gli altri casi, la forma literal `/pattern/flags` è preferita sia per leggibilità sia per performance (l'engine può precompilare e cachare le regex literal prima dell'esecuzione).

</details>

<details>
<summary>Perché `Function.prototype` è una funzione vuota e come si può usare questo fatto?</summary>

È una peculiarità dei native prototypes: il prototype di ogni costruttore ha il tipo "giusto" per quel costruttore. `Function.prototype` è una funzione vuota, `Array.prototype` è un array vuoto, `RegExp.prototype` è una regex vuota. Si può sfruttare questo come valore default "gratuito" per parametri opzionali: `fn = fn || Function.prototype` garantisce che `fn` sia sempre chiamabile anche se non fornito, senza creare una nuova funzione anonima ad ogni chiamata. **Attenzione**: non modificare mai questi prototypes — qualsiasi modifica ha effetti globali su tutti i valori di quel tipo.

</details>

<details>
<summary>In cosa si differenziano `Symbol` dagli altri tipi primitivi?</summary>

I `Symbol` sono primitivi scalari unici creati con `Symbol("descrizione")` — senza `new` (lancerebbe `TypeError`). Sono unici per design: due chiamate `Symbol("foo")` producono simboli diversi. Vengono usati come chiavi di proprietà per evitare collisioni accidentali — specialmente utili per proprietà "interne" o per i protocolli built-in di ES6 come `Symbol.iterator`. Non sono privati (`Object.getOwnPropertySymbols()` li rivela), ma segnalano convenzionalmente che una proprietà è speciale. A differenza degli altri primitivi, non hanno un native costruttore usabile con `new`.

</details>
