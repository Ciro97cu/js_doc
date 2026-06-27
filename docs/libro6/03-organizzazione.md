# Organizzazione del codice

ES6 introduce quattro strumenti principali per organizzare il codice in modo più chiaro e componibile: **iterator**, **generator**, **module** e **class**. Questo capitolo li percorre tutti, con attenzione ai dettagli di interfaccia e ai casi d'uso.

---

## Iterator

Un iterator è un pattern strutturato per consumare dati o operazioni un elemento alla volta. ES6 lo standardizza come interfaccia implicita adottata da tutte le strutture dati built-in e implementabile liberamente nel proprio codice.

### Interfacce

L'interfaccia `Iterator` richiede un metodo obbligatorio e due opzionali:

```
Iterator
  next() — obbligatorio: restituisce un IteratorResult
  return() — opzionale: segnala stop anticipato, restituisce IteratorResult
  throw() — opzionale: segnala un errore all'iterator

IteratorResult
  value — valore corrente (undefined se assente)
  done  — boolean: true quando l'iterator è esaurito

Iterable
  [Symbol.iterator]() — produce un Iterator
```

Un oggetto è **iterable** se espone `Symbol.iterator`. Un iterator è anche iterable se il suo `Symbol.iterator` restituisce `this`.

### `next()` e il protocollo base

```js
var arr = [1, 2, 3];
var it = arr[Symbol.iterator]();

it.next(); // { value: 1, done: false }
it.next(); // { value: 2, done: false }
it.next(); // { value: 3, done: false }
it.next(); // { value: undefined, done: true }
```

Regola chiave: l'ultimo valore utile viene restituito con `done: false` — la chiamata successiva restituisce `done: true` senza un valore significativo. Questo perché `for..of` scarta il valore quando `done` è `true`.

### `return()` e `throw()`

`return(x)` segnala all'iterator che il consumer ha finito anticipatamente. È chiamato automaticamente da `for..of` e dallo spread operator in caso di `break`. Restituisce `{ value: x, done: true }` e di norma esegue cleanup.

`throw(x)` inietta un'eccezione nell'iterator alla sua posizione corrente, usato principalmente con i generator (v. sezione successiva).

### Iterator come iterable — uso con `for..of`

Per usare un iterator direttamente con `for..of`, basta aggiungere `[Symbol.iterator]() { return this; }`:

```js
var it = {
    [Symbol.iterator]() { return this; },
    next() { /* ... */ }
};

for (var v of it) { console.log(v); }
```

### Iterator personalizzati

Si può costruire qualsiasi iterator aderendo all'interfaccia:

```js
/* Iterator infinito per la serie di Fibonacci */
var Fib = {
    [Symbol.iterator]() {
        var n1 = 1, n2 = 1;
        return {
            [Symbol.iterator]() { return this; },
            next() {
                var current = n2;
                n2 = n1;
                n1 = n1 + current;
                return { value: current, done: false };
            },
            return(v) {
                console.log("Sequenza abbandonata.");
                return { value: v, done: true };
            }
        };
    }
};

for (var v of Fib) {
    console.log(v);
    if (v > 50) break; // triggera return()
}
// 1 1 2 3 5 8 13 21 34 55
// Sequenza abbandonata.
```

### Consumo degli iterator

Oltre a `for..of`, gli iterator sono consumati da:

```js
var a = [1, 2, 3, 4, 5];

/* spread — esaurisce l'iterator */
foo(...a);             // passa 1,2,3,4,5 come argomenti
var b = [0, ...a, 6]; // [0,1,2,3,4,5,6]

/* destructuring — consuma parzialmente */
var it = a[Symbol.iterator]();
var [x, y] = it;        // consuma i primi due: x=1, y=2
var [z, ...w] = it;     // consuma il resto: z=3, w=[4,5]
it.next();              // { value: undefined, done: true }
```

---

## Generator

Un generator è una funzione che può **pausarsi** nel mezzo dell'esecuzione e **riprendere** in seguito. Ogni pausa/ripresa è un'opportunità per scambiare valori in entrambe le direzioni.

### Sintassi

```js
function *foo() {
    var x = yield 10;
    console.log(x);
}
```

La posizione dell'`*` rispetto a `function` è stilistica. All'interno di object literal si usa la forma concisa:

```js
var o = {
    *bar() { /* ... */ }
};
```

### Esecuzione: il generator produce un iterator

Chiamare un generator **non esegue il corpo** — restituisce un iterator che controlla l'esecuzione:

```js
var it = foo();    // crea l'iterator, nulla eseguito
it.next();         // { value: 10, done: false } — pausa a yield
it.next("ciao");   // stampa "ciao", { value: undefined, done: true }
```

**Regola N+1**: ci sarà sempre una chiamata `next()` in più rispetto al numero di `yield`. La prima `next()` avvia il generator fino al primo `yield` (senza passare valori — non c'è ancora nessuno `yield` in attesa). Ogni successiva `next(val)` completa lo `yield` corrente sostituendolo con `val`.

### `yield *` — delega

`yield *iterable` trasferisce il controllo a un altro iterable fino al suo esaurimento. Il valore di ritorno dell'iterable delegato diventa il valore dell'espressione `yield *`:

```js
function *foo() {
    yield 1; yield 2; yield 3;
    return 4;
}
function *bar() {
    var x = yield *foo();
    console.log("x:", x);
}
for (var v of bar()) { console.log(v); }
// 1 2 3
// x: 4
```

### Terminazione anticipata: `return()` e `throw()`

```js
var it = foo();
it.next();           // { value: 1, done: false }
it.return(42);       // { value: 42, done: true } — termina il generator
it.next();           // { value: undefined, done: true }
```

La clause `finally` viene sempre eseguita prima della terminazione, anche in caso di `return()` o `throw()` esterni.

### Error handling

Gli errori attraversano i confini dei generator — sia in entrata (`it.throw()`) sia in uscita (`throw` interno al generator) — e passano anche attraverso la delega `yield *`:

```js
function *foo() {
    try { yield 1; } catch (err) { console.log(err); }
    yield 2;
    throw "errore da foo";
}
var it = foo();
it.next();             // { value: 1, done: false }
it.throw("e1");        // catturato da catch: "e1", { value: 2, done: false }
try { it.next(); }
catch (err) { console.log(err); }  // "errore da foo"
```

### Transpilazione pre-ES6: state machine

Un generator si può rappresentare come una closure con una variabile di stato e uno `switch`:

```js
/* generator originale */
function *foo() {
    var x = yield 42;
    console.log(x);
}

/* equivalente pre-ES6 */
function foo() {
    var state = 0, x;
    function nextState(v) {
        switch (state) {
            case 0: state++; return 42;     // yield 42
            case 1: state++; x = v; console.log(x); return undefined;
        }
    }
    return {
        next(v) {
            var ret = nextState(v);
            return { value: ret, done: (state == 2) };
        }
    };
}

var it = foo();
it.next();      // { value: 42, done: false }
it.next(10);    // 10, { value: undefined, done: true }
```

---

## Module

Il pattern module — basato su funzioni esterne con closure che espongono una public API — è il pattern di organizzazione più importante in JS. ES6 lo eleva a costrutto di prima classe con `import`/`export`.

### Prima di ES6: IIFE e CommonJS

```js
/* singleton con IIFE */
var me = (function Hello(name) {
    function greeting() { console.log("Hello " + name + "!"); }
    return { greeting };
})("Kyle");

me.greeting(); // Hello Kyle!
```

Format come AMD, UMD, CommonJS erano varianti di questo pattern.

### Caratteristiche dei moduli ES6

- **File-based**: un modulo = un file. Non esiste un modo standard per combinare più moduli in un unico file.
- **Singleton**: esiste una sola istanza del modulo. Ogni `import` ottiene un riferimento alla stessa istanza.
- **API statica**: gli export di un modulo sono definiti staticamente — non si possono aggiungere o rimuovere a runtime (vantaggio: l'engine può analizzarli e ottimizzarli staticamente).
- **Live binding**: gli export sono binding (riferimenti) alle variabili interne, non copie. Se il modulo aggiorna una variabile esportata, chi ha importato vede il nuovo valore.
- **Hoisting degli import**: le dichiarazioni `import` sono hoisted in cima al modulo — si può chiamare una funzione importata prima della riga `import`.
- `import`/`export` devono trovarsi sempre al top-level del modulo — non dentro `if`, funzioni, o blocchi.

### Export

```js
/* named export inline */
export function foo() { /* ... */ }
export var awesome = 42;

/* named export in blocco (preferibile per leggibilità) */
function foo() { /* ... */ }
var bar = [1, 2, 3];
export { foo, bar };

/* alias nell'export */
export { foo as bar };   // esportato come "bar", foo resta privato

/* default export — al massimo uno per modulo */
export default function foo() { /* ... */ }

/* forma alternativa — binding alla variabile, non al valore */
export { foo as default };
```

**Gotcha critico**: `export default foo` esporta il valore corrente di `foo`, non il binding. Se `foo` viene riassegnato dopo l'export, l'import continua a vedere il valore originale. `export { foo as default }` esporta invece il binding live.

Si possono re-esportare binding da un altro modulo senza importarli localmente:

```js
export { foo, bar } from "baz";
export * from "baz";
```

### Import

```js
/* named import */
import { foo, bar } from "foo";

/* rinomina */
import { foo as theFooFunc } from "foo";

/* default import */
import foo from "foo";
import { default as foo } from "foo"; /* equivalente */

/* default + named */
import FOOFN, { bar, baz as BAZ } from "foo";

/* namespace import — tutto il modulo in un oggetto */
import * as foo from "foo";
foo.bar();  foo.x;  foo.baz();

/* side-effect only — esegue il modulo senza importare nulla */
import "foo";
```

I binding importati sono **immutabili**: qualsiasi tentativo di riassegnarli lancia TypeError. Il modulo sorgente può però aggiornare i propri valori interni, e i binding importati rifletteranno automaticamente i nuovi valori.

### Dipendenze circolari

ES6 supporta import circolari grazie all'analisi statica: l'engine scansiona gli export di tutti i moduli prima di eseguirne uno. I binding live garantiscono che i riferimenti incrociati funzionino correttamente una volta che entrambi i moduli sono stati caricati.

### Caricamento dinamico

Al di fuori dei moduli (es. in uno script normale) o per import condizionali, si può usare:

```js
Reflect.Loader.import("foo")
    .then(function(foo) { foo.bar(); });
```

Restituisce una Promise. Da evitare nei moduli normali perché impedisce l'ottimizzazione statica.

---

## Class

ES6 introduce `class` come syntactic sugar sul sistema a prototype. Non cambia il modello sotto — prototype chain, `instanceof`, constructor function — ma offre una sintassi più leggibile per esprimere gerarchie.

### Sintassi base

```js
class Foo {
    constructor(a, b) {
        this.x = a;
        this.y = b;
    }
    gimmeXY() {
        return this.x * this.y;
    }
}

var f = new Foo(5, 15);
f.gimmeXY(); // 75
```

Equivale (approssimativamente) a:

```js
function Foo(a, b) { this.x = a; this.y = b; }
Foo.prototype.gimmeXY = function() { return this.x * this.y; };
```

**Differenze importanti rispetto a `function`:**
- Deve essere invocato con `new` — non esiste `Foo.call(obj)`
- Non è hoisted — deve essere dichiarato prima dell'istanziazione
- Non crea una proprietà globale sull'oggetto `window`
- I metodi sono non-enumerabili
- Non ci sono virgole tra i membri del corpo

### `extends` e `super`

```js
class Bar extends Foo {
    constructor(a, b, c) {
        super(a, b);    /* obbligatorio prima di usare this */
        this.z = c;
    }
    gimmeXYZ() {
        return super.gimmeXY() * this.z;
    }
}

var b = new Bar(5, 15, 25);
b.gimmeXYZ(); // 1875
```

`extends` collega `Bar.prototype` a `Foo.prototype` (e `Bar` stessa a `Foo` per i metodi `static`). `super` in un constructor riferisce la funzione costruttore del parent; in un metodo riferisce il prototype del parent.

**Gotcha di `super`**: è **staticamente bound** alla gerarchia dichiarata — non è dinamico come `this`. Se si "prende in prestito" un metodo di una classe e lo si invoca in un contesto diverso (tramite `.call()` o `.apply()`), il `this` verrà reindirizzato dinamicamente, ma `super` no: continuerà a puntare alla gerarchia originale del metodo.

### Subclass constructor e regola `super()` prima di `this`

In una subclass, `this` non è disponibile finché non si chiama `super()`. La ragione è che in ES6 il constructor del parent è responsabile della creazione dell'istanza `this`. Invertire l'ordine lancia un ReferenceError.

Se il constructor è omesso, la subclass ne ottiene uno di default che è equivalente a `constructor(...args) { super(...args); }`.

### Estendere i built-in

ES6 `class` permette per la prima volta di estendere correttamente i built-in come `Array` ed `Error`, incluso il loro comportamento speciale (aggiornamento automatico di `length`, cattura dello stack trace, ecc.):

```js
class MyCoolArray extends Array {
    first() { return this[0]; }
    last() { return this[this.length - 1]; }
}
var a = new MyCoolArray(1, 2, 3);
a.first(); // 1
a.length;  // 3

class Oops extends Error {
    constructor(reason) { super(); this.oops = reason; }
}
throw new Oops("qualcosa è andato storto"); // con stack trace completo
```

### Metodi `static`

I metodi `static` vengono aggiunti direttamente alla funzione costruttore (non al suo prototype), e vengono ereditati dalle subclass attraverso il loro chain `[[Prototype]]`:

```js
class Foo {
    static cool() { console.log("cool"); }
}
class Bar extends Foo {
    static awesome() { super.cool(); console.log("awesome"); }
}

Foo.cool();   // "cool"
Bar.cool();   // "cool"   — ereditato
Bar.awesome(); // "cool" + "awesome"

var b = new Bar();
b.cool;       // undefined — static non sul prototype
```

`Symbol.species` usato come getter statico permette a una classe derivata di segnalare quale constructor usare quando il parent deve creare nuove istanze:

```js
class MyCoolArray extends Array {
    static get [Symbol.species]() { return Array; }
}
var a = new MyCoolArray(1, 2, 3);
var b = a.map(v => v * 2);
b instanceof MyCoolArray; // false
b instanceof Array;       // true
```

### `new.target`

`new.target` è una meta property disponibile dentro qualsiasi constructor. Punta al constructor invocato direttamente con `new`, anche se l'esecuzione è risalita a un parent tramite `super()`:

```js
class Foo {
    constructor() { console.log("Foo:", new.target.name); }
}
class Bar extends Foo {
    constructor() { super(); console.log("Bar:", new.target.name); }
}
new Foo(); // Foo: Foo
new Bar(); // Foo: Bar — new.target rispetta il call-site originale
           // Bar: Bar
```

---

## ⚡ Ripasso veloce

```js
/* === ITERATOR === */
var it = arr[Symbol.iterator]();
it.next(); // { value: v, done: false/true }
it.return(x); // stop anticipato
/* iterable = ha Symbol.iterator; iterator = ha next() */
/* for..of scarta il value dell'ultimo { done: true } */

/* === GENERATOR === */
function *gen() { var x = yield 10; }
var it = gen();
it.next();     // { value: 10, done: false } — avvia fino al yield
it.next(5);    // x = 5, { value: undefined, done: true }
/* yield *iterable — delega; il return value della delega diventa il valore dell'espressione */

/* === MODULE === */
/* foo.js */
export function bar() {}
export default class Baz {}
var priv = "non esportato";

/* main.js */
import Baz, { bar } from "./foo.js";   // default + named
import * as foo from "./foo.js";        // namespace
import { bar as b } from "./foo.js";    // alias

/* i binding importati sono live (riflettono aggiornamenti del modulo) */
/* e immutabili dal lato dell'importatore */

/* === CLASS === */
class Animal {
    constructor(name) { this.name = name; }
    speak() { return `${this.name} fa rumore`; }
}
class Dog extends Animal {
    constructor(name) { super(name); }  /* super() prima di this */
    speak() { return super.speak() + " — abbaia"; }
    static create(n) { return new Dog(n); }
}
Dog.create("Rex").speak(); // "Rex fa rumore — abbaia"
```

---

## Domande

<details>
<summary>Qual è la differenza tra un <em>iterator</em> e un <em>iterable</em>, e perché la distinzione è importante?</summary>

Un **iterable** è qualsiasi oggetto che espone un metodo `[Symbol.iterator]()` che, quando chiamato, restituisce un **iterator**. L'iterator è l'oggetto con il metodo `next()` che produce i valori uno alla volta. La distinzione è importante perché i costrutti ES6 (`for..of`, spread, destructuring) richiedono un *iterable*, non direttamente un *iterator*. Per rendere un iterator direttamente consumabile da `for..of`, basta aggiungergli `[Symbol.iterator]() { return this; }` — così diventa sia iterable che iterator. Gli Array, le String, i Map, i Set e i generator sono tutti iterabili built-in.

</details>

<details>
<summary>Perché in un generator ci sono sempre N+1 chiamate <code>next()</code> rispetto al numero di <code>yield</code>?</summary>

Perché la prima chiamata a `next()` *avvia* il generator dalla sua posizione iniziale (paused) fino al primo `yield`, ma non c'è ancora nessun `yield` in attesa di un valore — quindi l'argomento passato alla prima `next()` viene ignorato. Ogni `yield expr` svolge due ruoli: *invia* un valore verso l'esterno (il `value` nel risultato), e *attende* un valore dall'esterno (la risposta della `next()` successiva). Se ci sono 3 `yield`, ci vogliono 4 `next()`: la prima per partire, poi una per completare ciascuno dei 3 `yield`. Questo può sembrare asimmetrico, ma ha senso guardandolo dall'interno: ogni `yield` "fa una domanda" e aspetta la risposta nella `next()` successiva.

</details>

<details>
<summary>Cosa significa che i moduli ES6 hanno una "static API" e che i loro export sono "live binding"?</summary>

"Static API" significa che gli export di un modulo sono determinati staticamente al momento della definizione del file — non si possono aggiungere o rimuovere nuovi export a runtime. Questo permette all'engine di analizzare le dipendenze senza eseguire il codice, abilitando ottimizzazioni e rilevamento precoce di errori. "Live binding" significa che quando si esporta una variabile, non si esporta una *copia* del suo valore ma un *riferimento* (binding) alla variabile stessa. Se il modulo aggiorna la variabile dopo l'export, chi ha importato vede il valore aggiornato. Questo è diverso dal modo in cui funzionavano i moduli CommonJS (copia del valore). Il lato "read-only" per l'importatore: chi importa non può modificare il binding, ma il modulo sorgente sì.

</details>

<details>
<summary>Perché in un constructor di subclass ES6 è obbligatorio chiamare <code>super()</code> prima di usare <code>this</code>?</summary>

In ES6, quando si usa `extends`, è il constructor del parent che è responsabile della creazione e inizializzazione dell'istanza `this`. La subclass "eredita" quell'istanza. Se si cerca di usare `this` prima di chiamare `super()`, l'engine lancia un ReferenceError perché `this` non è ancora stato creato. Pre-ES6, il comportamento era opposto: la funzione derivata creava il proprio `this` e poi chiamava `Foo.call(this)` per applicare l'inizializzazione del parent. In ES6, la responsabilità è invertita. Se un constructor viene omesso in una subclass, viene automaticamente generato un constructor di default che fa `super(...args)`, propagando tutti gli argomenti al parent.

</details>

<details>
<summary>Cosa significa che <code>super</code> è "statically bound" in ES6 e quali problemi crea?</summary>

Mentre `this` è risolto dinamicamente in base al call-site, `super` è risolto staticamente al momento della definizione della classe — è vincolato alla gerarchia dichiarata. Se si "prende in prestito" un metodo di `ChildB` (che estende `ParentB`) e lo si invoca nel contesto di un'istanza di `ChildA`, il `this` verrà reindirizzato dinamicamente tramite `.call()` o `.apply()`, ma il `super` dentro quel metodo continuerà a puntare a `ParentB` — non a `ParentA`. Non esiste una soluzione ES6 a questa limitazione. La conseguenza pratica è che i metodi con `super` non sono portabili tra gerarchie diverse. Per questo si consiglia di usare `class`/`extends`/`super` solo con gerarchie statiche e di evitare il borrowing di metodi tra classi non correlate.

</details>
