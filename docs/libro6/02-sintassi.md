# Sintassi ES6

ES6 introduce un numero consistente di nuove forme sintattiche. Questo capitolo le percorre tutte, dalla dichiarazione di variabili a blocco fino ai Symbol, passando per destructuring, arrow function, template literal e molto altro.

---

## Dichiarazioni block-scoped: `let` e `const`

Prima di ES6 l'unica unità di scope in JS era la funzione. ES6 aggiunge il **block scope**: qualsiasi coppia `{ }` crea scope per le variabili dichiarate con `let` o `const`.

### `let`

```js
var a = 2;
{
    let a = 3;
    console.log(a); // 3
}
console.log(a);     // 2
```

Buona pratica: dichiarare tutte le variabili `let` in cima al blocco, per evitare il **TDZ (Temporal Dead Zone)** — l'intervallo tra l'inizio del blocco e la riga della dichiarazione, in cui la variabile esiste ma non è ancora inizializzata:

```js
{
    console.log(a); // undefined (var è hoistata)
    console.log(b); // ReferenceError! — b è in TDZ
    var a;
    let b;
}
```

Attenzione: `typeof` **non è sicuro** per variabili `let`/`const` in TDZ:

```js
typeof b; // ReferenceError! (c'è un let b più avanti nel blocco)
let b;
```

#### `let` nel `for`

Il caso più importante di `let`: nella testata di un `for`, ogni iterazione ottiene la propria copia della variabile — risolve il classico problema di closure nei loop:

```js
/* con var: tutte le closure chiudono sulla stessa i */
for (var i = 0; i < 5; i++) { funcs.push(() => i); }
funcs[3](); // 5 — sbagliato

/* con let: ogni iterazione ha la propria i */
for (let i = 0; i < 5; i++) { funcs.push(() => i); }
funcs[3](); // 3 — corretto
```

### `const`

`const` crea un binding read-only dopo l'inizializzazione. L'inizializzazione è obbligatoria alla dichiarazione.

```js
const a = 2;
a = 3;          // TypeError!

const arr = [1, 2, 3];
arr.push(4);    // OK — il contenuto è mutabile
arr = [];       // TypeError! — il binding è immutabile
```

`const` vincola il **binding** (la variabile), non il **valore**. Un oggetto o array dichiarato `const` rimane liberamente modificabile al suo interno. Usare `const` come segnale di intento — "questo non cambierà" — non come protezione tecnica.

### Funzioni block-scoped

In ES6 le function declaration dentro un blocco sono scoped a quel blocco:

```js
{
    foo();              // funziona (hoisted nel blocco)
    function foo() {}
}
foo();                  // ReferenceError
```

---

## Spread e Rest (`...`)

L'operatore `...` ha due comportamenti opposti a seconda del contesto.

**Spread** — espande un iterable nei suoi valori individuali:

```js
function foo(x, y, z) { console.log(x, y, z); }

foo(...[1, 2, 3]);          // 1 2 3  (sostituisce apply)

var a = [2, 3, 4];
var b = [1, ...a, 5];       // [1,2,3,4,5]  (sostituisce concat)
```

**Rest/gather** — raccoglie i parametri rimanenti in un array reale (sostituisce `arguments`):

```js
function foo(x, y, ...z) {
    console.log(x, y, z);
}
foo(1, 2, 3, 4, 5); // 1 2 [3,4,5]

function bar(...args) {
    args.shift();           // args è un vero Array
    console.log(...args);
}
```

---

## Default Parameter Values

```js
function foo(x = 11, y = 31) {
    console.log(x + y);
}

foo();                  // 42
foo(5, 6);              // 11
foo(0, 42);             // 42  (0 non è undefined)
foo(5, undefined);      // 36  (undefined attiva il default)
foo(5, null);           // 5   (null coerces a 0, non attiva il default)
```

Il default scatta solo quando l'argomento è assente o `undefined` — non per `null`, `0`, `""`. Regola: `undefined` = mancante.

I default possono essere **espressioni** o chiamate di funzione, valutate *lazily* (solo se necessario):

```js
function bar(val) { return y + val; }

function foo(x = y + 3, z = bar(x)) {
    console.log(x, z);
}

var y = 5;
foo();          // 8, 13
foo(10);        // 10, 15
```

I parametri default vivono in un proprio scope — un riferimento nel default expression cerca prima negli altri parametri, poi nel outer scope. Questo produce TDZ se un default fa riferimento a un parametro non ancora inizializzato.

---

## Destructuring

Il destructuring (de-strutturazione) permette di scomporre array e oggetti direttamente in variabili, eliminando variabili temporanee.

### Array destructuring

```js
var [a, b, c] = [1, 2, 3];
console.log(a, b, c);   // 1 2 3

/* skip elementi */
var [, b] = [1, 2, 3];  // b = 2

/* rest in destructuring */
var [x, ...rest] = [1, 2, 3, 4];
console.log(x, rest);   // 1 [2,3,4]

/* swap senza temp */
var x = 10, y = 20;
[y, x] = [x, y];
console.log(x, y);      // 20 10
```

### Object destructuring

```js
var { x, y, z } = { x: 4, y: 5, z: 6 };
console.log(x, y, z);   // 4 5 6

/* rinomina: source: target */
var { x: bam, y: baz } = { x: 4, y: 5 };
console.log(bam, baz);  // 4 5 — x e y non esistono
```

**Gotcha fondamentale** — la sintassi nell'object destructuring è invertita rispetto agli object literal:
- Object literal: `target: source` → `{ a: valoreDiA }`
- Object destructuring: `source: target` → `{ x: bam }` significa "prendi `x`, mettilo in `bam`"

Senza var/let/const, il `{ }` va avvolto in `( )` per non essere interpretato come blocco:

```js
var a, b;
({ a, b } = { a: 1, b: 2 });
```

### Default values nel destructuring

```js
var [a = 3, b = 6, d = 12] = [1, 2];
console.log(a, b, d);   // 1 2 12

var { x = 5, w = 20 } = { x: 4 };
console.log(x, w);      // 4 20

/* rinomina + default */
var { x: X = 10 } = {};
console.log(X);         // 10
```

### Nested destructuring

```js
var [[a, b], c] = [[1, 2], 3];
var { x: { y: { z: w } } } = { x: { y: { z: 6 } } };
console.log(w); // 6

/* flatten namespace */
var { model: { User } } = App;
```

### Destructuring nei parametri

```js
function foo([x, y]) { console.log(x, y); }
foo([1, 2]);    // 1 2

function bar({ x, y }) { console.log(x, y); }
bar({ y: 1, x: 2 }); // 2 1  (named args pattern)

/* combinazioni */
function f({ x = 10 } = {}, { y } = { y: 10 }) {
    console.log(x, y);
}
f();            // 10 10
f({}, {});      // 10 undefined  — il default { y: 10 } vale solo se l'arg è undefined
```

---

## Object Literal Extensions

### Concise properties e methods

```js
var x = 2, y = 3;
var o = { x, y };  // invece di { x: x, y: y }

var o = {
    foo() { /* ... */ },        // invece di foo: function() {}
    *bar() { /* ... */ },       // concise generator
    get id() { return this._id++; },
    set id(v) { this._id = v; }
};
```

**Gotcha dei concise methods**: sono funzioni anonime — non hanno un self-reference lessicale. Non usarli se la funzione deve fare ricorsione o unbinding su eventi:

```js
/* ROTTO: something è anonima, return something(...) fallisce */
runSomething({
    something(x, y) {
        if (x > y) return something(y, x); // ReferenceError!
    }
});

/* CORRETTO: nome lessicale disponibile */
runSomething({
    something: function something(x, y) {
        if (x > y) return something(y, x); // OK
    }
});
```

### Computed property names

```js
var prefix = "user_";
var o = {
    [prefix + "foo"]: function() { /* ... */ },
    [Symbol.toStringTag]: "my object"
};
```

### `__proto__` e `super`

```js
var o2 = {
    __proto__: o1,   /* imposta [[Prototype]] inline */
    foo() {
        super.foo(); /* super usabile solo in concise methods */
    }
};

Object.setPrototypeOf(o2, o1);  /* alternativa */
```

---

## Template Literal

I template literal (backtick `` ` ``) sono in realtà **interpolated string literals**: stringhe con interpolazione di espressioni JS inline.

```js
var name = "Kyle";
var greeting = `Hello ${name}!`;  // "Hello Kyle!"

/* multiline nativo */
var text = `Now is the time
for all good men`;

/* espressioni qualsiasi */
var out = `${upper("warm")} welcome to ${upper(`${who}s`)}!`;
```

Lo scope delle espressioni `${}` è lessicale — risolto nel punto in cui compare il template literal.

### Tagged string literals

Un *tag* è una funzione invocata prima della composizione finale della stringa:

```js
function tag(strings, ...values) {
    /* strings: array delle parti statiche */
    /* values: array delle espressioni valutate */
    return strings.reduce((s, v, i) =>
        s + (i > 0 ? values[i - 1] : "") + v, "");
}

var desc = "awesome";
var text = tag`Everything is ${desc}!`;
// strings = ["Everything is ", "!"]
// values  = ["awesome"]
```

`strings.raw` contiene le versioni non-processed (escape sequenze non interpretate):

```js
String.raw`Hello\nWorld`; // "Hello\nWorld" (non newline)
```

---

## Arrow Functions (`=>`)

```js
var foo = (x, y) => x + y;
var f1 = () => 12;
var f2 = x => x * 2;
var f3 = (x, y) => {
    var z = x * 2 + y;
    return z;
};
```

Le arrow function sono sempre **function expression anonime** — nessuna forma declaration, nessun nome lessicale per la ricorsione.

### Il vero scopo: `this` lessicale

L'attrattiva principale non è la brevità sintattica ma il fatto che le arrow function **non hanno un proprio `this`**: ereditano il `this` dello scope circostante (lexical `this`).

```js
/* problema classico — this perde il contesto nel callback */
var controller = {
    makeRequest: function() {
        var self = this;  /* hack */
        btn.addEventListener("click", function() {
            self.makeRequest(); /* deve usare self */
        });
    }
};

/* con arrow — this è lessicale */
var controller = {
    makeRequest: function() {
        btn.addEventListener("click", () => {
            this.makeRequest(); /* this è quello di makeRequest */
        });
    }
};
```

Stesso principio per `arguments` e `super`: le arrow function li ereditano dallo scope esterno invece di averne di propri.

### Quando usare `=>`

- Funzione inline breve (una sola espressione) passata come callback, senza `this` reference interna
- Funzione interna che deve ereditare il `this` del metodo esterno (sostituisce `var self = this` o `.bind(this)`)

### Quando NON usare `=>`

- Metodi di oggetto/classe che usano `this` — il `this` lessicale punta allo scope sbagliato
- Funzioni che hanno bisogno di nome lessicale (ricorsione, event unbinding)
- Funzioni lunghe e multistatement — la brevità non compensa la perdita di chiarezza

---

## `for..of`

`for..of` itera sui **valori** prodotti da un iterable (oggetto con `Symbol.iterator`):

```js
var a = ["a", "b", "c"];
for (var val of a) { console.log(val); }  // "a" "b" "c"

/* vs for..in che itera le chiavi */
for (var idx in a) { console.log(idx); }  // 0 1 2
```

Iterabili built-in: Array, String, Generator, Map, Set, TypedArray. I plain object non hanno `Symbol.iterator` per default. `break`/`continue`/`return` chiamano automaticamente `iterator.return()` per cleanup.

---

## Regular Expression: nuovi flag

### Flag `u` (Unicode)

Tratta i caratteri astral (oltre BMP, cioè oltre U+FFFF) come singola entità anziché due surrogate:

```js
/^.-clef/.test("𝄞-clef");   // false — 𝄞 conta come 2 char
/^.-clef/u.test("𝄞-clef");  // true  — 𝄞 conta come 1 char
```

### Flag `y` (Sticky)

Ancora il match esattamente alla posizione di `lastIndex` — nessun "move ahead":

```js
var re = /foo/y, str = "++foo++";
re.lastIndex = 2;
re.test(str);    // true — "foo" è a posizione 2
re.lastIndex;    // 5   — aggiornato dopo il match
re.test(str);    // false — non c'è "foo" a posizione 5
re.lastIndex;    // 0   — resettato dopo fallimento
```

Differenza da `g`: con `g` l'engine cerca avanti liberamente; con `y` deve trovare il match esattamente a `lastIndex`.

### `re.flags` e miglioramenti al costruttore

```js
/foo/ig.flags;          // "gi"  (ordine: gimuy)

var re1 = /foo*/y;
var re2 = new RegExp(re1, "ig");  // override dei flag — valido in ES6
re2.flags;              // "gi"
```

---

## Number Literal Extensions

ES6 standardizza le forme ottale e binaria, e aggiunge il prefisso binario:

```js
var dec = 42;
var oct = 0o52;    // ottale — prima era 052 (non standard)
var hex = 0x2a;    // esadecimale
var bin = 0b101010; // binario — nuovo

/* tutte e quattro equivalgono a 42 */
Number("0o52");    // 42
Number("0x2a");    // 42
Number("0b101010"); // 42

/* conversione inversa */
(42).toString(8);  // "52"
(42).toString(2);  // "101010"
```

---

## Unicode

### Escape e code point

```js
/* pre-ES6: solo BMP (\uXXXX) */
"☃";       // ☃

/* ES6: code point escaping per tutti i code point */
"\u{1D11E}";    // 𝄞  (astral symbol)
```

### Operazioni su stringhe Unicode

Le stringhe JS usano internamente UTF-16. I simboli astral occupano due code unit:

```js
"𝄞".length;     // 2  (due surrogate)
[..."𝄞"].length; // 1  (spread usa l'iterator Unicode-aware)
```

Per lunghezza corretta con combining marks (es. `"e" + U+0301` = `"é"`):

```js
"é".normalize().length; // 1
```

Metodi Unicode-aware:

```js
"ab\u{1d49e}d".codePointAt(2); // code point dell'astral char
String.fromCodePoint(0x1d49e);  // "𝒞"
```

---

## Symbol

ES6 introduce un nuovo tipo primitivo: il **Symbol**. Ogni Symbol è un valore unico e non duplicabile.

```js
var sym = Symbol("descrizione opzionale");
typeof sym;     // "symbol"

/* NON usare new — non è un costruttore */
```

I Symbol non sono oggetti e non hanno forma literal. La stringa passata a `Symbol()` è solo una descrizione per il debugging.

### Uso principale: chiavi collision-free

```js
const EVT_LOGIN = Symbol("event.login");
evthub.listen(EVT_LOGIN, handler);  /* garantito unico */
```

### Symbol registry globale

`Symbol.for()` crea o recupera un Symbol dal registry globale, identificandolo per descrizione:

```js
const S1 = Symbol.for("app.token");
const S2 = Symbol.for("app.token");
S1 === S2;  // true — stesso Symbol dal registry

Symbol.keyFor(S1);  // "app.token"
```

### Symbol come proprietà di oggetto

Le proprietà con chiave Symbol non compaiono in `for..in`, `Object.keys()` o `JSON.stringify()`:

```js
var o = {
    foo: 42,
    [Symbol("bar")]: "hidden-ish"
};

Object.getOwnPropertyNames(o);    // ["foo"]
Object.getOwnPropertySymbols(o);  // [Symbol(bar)]
```

### Symbol predefiniti (built-in)

ES6 include Symbol predefiniti che controllano comportamenti dell'engine:

| Symbol | Controlla |
|---|---|
| `Symbol.iterator` | `for..of`, spread, destructuring |
| `Symbol.toStringTag` | `Object.prototype.toString()` output |
| `Symbol.toPrimitive` | Coercione a primitivo |

---

## ⚡ Ripasso veloce

```js
/* let/const block scope */
{ let a = 1; }  // a non esiste fuori
const arr = []; arr.push(1);  // OK — const vincola il binding, non il valore

/* spread / rest */
foo(...[1, 2, 3]);            // spread in chiamata
function bar(x, ...rest) {}   // rest = array dei rimanenti

/* default params */
function f(x = 11) {}         // undefined attiva il default, null no

/* destructuring */
var [a, , b] = [1, 2, 3];     // a=1, b=3
var { x: y } = { x: 42 };     // y=42 (source: target)
var [head, ...tail] = arr;    // rest in destructuring
function foo({ x = 0 } = {}) {}  // oggetto opzionale con default

/* concise object */
var o = { x, y, foo() {}, *bar() {}, get id() {}, [expr]: val };

/* template literal */
`Hello ${name}!`   // interpolazione
tag`str ${val}`    // tagged: tag riceve (strings, ...values)

/* arrow function */
const add = (x, y) => x + y;
arr.map(v => v * 2);
/* this lessicale — NO per metodi, NO per ricorsione */

/* for..of */
for (var v of [1, 2, 3]) {}   // valori, non indici
for (var c of "hello") {}      // caratteri

/* regex */
/./u   // Unicode-aware
/./y   // sticky (lastIndex-relative)

/* numeri */
0o52; 0x2a; 0b101010;  // 42 in ottale, hex, binario

/* Symbol */
const key = Symbol("desc");
typeof key;         // "symbol"
obj[key] = "val";   // non appare in for..in
Symbol.for("id");   // registry globale
```

---

## Domande

<details>
<summary>Qual è la differenza tra <code>var</code>, <code>let</code> e <code>const</code> in termini di scope e comportamento?</summary>

`var` ha scope di funzione (o globale se al top level) e viene hoistata con valore `undefined`. `let` ha scope di blocco (`{ }`) e soffre di TDZ: è dichiarata alla creazione del blocco ma non inizializzata finché non si raggiunge la sua dichiarazione — accedervi prima è un ReferenceError. `const` è come `let` (block-scoped, TDZ) ma il binding è read-only dopo l'inizializzazione, che è obbligatoria. `const` non congela il valore: un oggetto o array dichiarato `const` è ancora mutabile al suo interno. La regola pratica: usare `const` come segnale di intento per valori che non verranno riassegnati, `let` per quelli che cambieranno, `var` solo in rari casi di backward compatibility.

</details>

<details>
<summary>Perché le arrow function sono state introdotte e quando NON andrebbero usate?</summary>

Il motivo principale dell'introduzione non è la brevità sintattica ma il **lexical `this`**: le arrow function non hanno un proprio `this` — ereditano il `this` dello scope circostante. Questo risolve il pattern `var self = this` o `.bind(this)` per callback e inner function che devono accedere al contesto del metodo esterno. Non andrebbero usate quando: (1) la funzione ha bisogno del proprio `this` dinamico (metodi di oggetto/classe invocati su `this` dell'oggetto), (2) la funzione ha bisogno di auto-riferimento lessicale per ricorsione o event unbinding (le arrow sono sempre anonime), (3) la funzione è lunga e multistatement (la brevità non migliora la leggibilità). Anche `arguments` e `super` sono lessicali nelle arrow — ereditati dall'esterno.

</details>

<details>
<summary>Cosa significa che nel object destructuring la sintassi è "source: target" invece di "target: source"?</summary>

Nei normali object literal, la sintassi è `target: source` — ad esempio `{ a: valoreDiA }` significa "la proprietà `a` riceve il valore di `valoreDiA`". Nell'object destructuring, l'ordine è invertito: `source: target` — `{ x: bam }` significa "prendi il valore della proprietà `x` (source) e assegnalo alla variabile `bam` (target)". Si può pensarlo così: nei literal il token sinistro `:` è il nome della proprietà nell'oggetto risultante; nel destructuring, il token sinistro `:` è il nome della proprietà nell'oggetto sorgente da cui leggere. La forma shorthand `{ x }` è valida in entrambi i contesti e omette sempre il token sinistro: il nome coincide con quello della proprietà.

</details>

<details>
<summary>Qual è la differenza tra il flag <code>y</code> (sticky) e il flag <code>g</code> (global) nelle regular expression?</summary>

Entrambi usano `lastIndex` per le successive match, ma con comportamenti diversi. Con `g`, `exec()` aggiorna `lastIndex` dopo ogni match ma l'engine è libero di avanzare nel testo per trovare il prossimo match — non deve essere esattamente a `lastIndex`. Con `y`, il match deve trovarsi **esattamente** alla posizione indicata da `lastIndex`: nessun avanzamento libero. Se non c'è match a quella posizione, il test fallisce e `lastIndex` viene resettato a 0. Questo rende `y` adatto al parsing di stringhe con struttura prevedibile, dove dopo ogni match `lastIndex` viene automaticamente posizionato al punto giusto per il match successivo. Un'altra differenza: `str.match(re_g)` con flag `g` restituisce tutti i match in un array, mentre con `y` restituisce un match alla volta.

</details>

<details>
<summary>Cosa sono i Symbol e perché sono stati aggiunti in ES6?</summary>

I Symbol sono un nuovo tipo primitivo che produce valori **unici e non duplicabili**. Ogni chiamata a `Symbol()` produce un valore diverso da tutti gli altri, anche se creati con la stessa descrizione. Il problema che risolvono è quello delle "magic string": usare stringhe come chiavi speciali o event name rischia collisioni accidentali se qualcuno usa la stessa stringa per un altro scopo. Con Symbol, il valore è garantito unico — non può collidere. Come proprietà di oggetto, i Symbol non appaiono in `for..in`, `Object.keys()` o `JSON.stringify()`, ma sono visibili tramite `Object.getOwnPropertySymbols()`. `Symbol.for()` e `Symbol.keyFor()` gestiscono un registry globale per Symbol condivisi tra moduli. I Symbol built-in (`Symbol.iterator`, `Symbol.toStringTag`, ecc.) permettono di agganciare i comportamenti del JS engine a oggetti custom.

</details>
