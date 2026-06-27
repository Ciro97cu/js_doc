# Metaprogrammazione

La metaprogrammazione è programmazione che ha come oggetto il comportamento del programma stesso: codice che ispeziona se stesso, si modifica, o altera i comportamenti predefiniti del linguaggio. ES6 aggiunge strumenti potenti in questa direzione: inferenza dei nomi di funzione, meta property, Well-Known Symbol, Proxy, Reflect, feature testing e TCO.

---

## Nomi di funzione

Le funzioni hanno una proprietà `name` usata per il debugging (stack trace). In ES5 era non standard; ES6 la standardizza e aggiunge regole di **inferenza**: anche funzioni anonime ricevono un `name` dedotto dal contesto.

```js
var abc = function() {};
abc.name;            // "abc"  — inferito dall'assegnazione

var o = {
    foo() {},                    // name: "foo"
    bar: () => {},               // name: "bar"
    bam: function() {},          // name: "bam"
    get qux() {},                // name: "get qux"
    ["b" + "iz"]: function() {}  // name: "biz"
};

(function() {}).name;            // ""   — IIFE anonima: nessuna inferenza
window.foo = function() {};      // ""   — assegnazione globale: nessuna inferenza
var x = o.foo.bind(o);          // name: "bound foo"
export default function() {}    // name: "default"
```

`name` è configurabile ma non scrivibile per default. Si può modificare con `Object.defineProperty()`.

---

## Meta property: `new.target`

`new.target` è una meta property disponibile in qualsiasi constructor. Punta al constructor invocato direttamente con `new`, anche se l'esecuzione è risalita a un parent tramite `super()`. Fuori da un constructor, vale `undefined`.

```js
class Parent {
    constructor() {
        if (new.target === Parent) {
            console.log("Parent istanziato direttamente");
        } else {
            console.log("Subclass istanziata:", new.target.name);
        }
    }
}
class Child extends Parent {}

new Parent(); // "Parent istanziato direttamente"
new Child();  // "Subclass istanziata: Child"
```

Utile per: rilevare se una classe "astratta" viene istanziata direttamente, accedere a proprietà statiche della classe figlia dall'interno del parent.

---

## Well-Known Symbol

I Well-Known Symbol (WKS) sono Symbol predefiniti che controllano comportamenti interni del JS engine. Impostarli sugli oggetti permette di personalizzare tali comportamenti.

### `Symbol.iterator`

Controlla il protocollo di iterazione (v. capitoli Sintassi e Organizzazione). Si può sovrascrivere per personalizzare il comportamento di `for..of`, spread e destructuring su qualsiasi oggetto:

```js
var arr = [4, 5, 6, 7, 8, 9];

arr[Symbol.iterator] = function*() {
    var idx = 1;
    do { yield this[idx]; } while ((idx += 2) < this.length);
};

for (var v of arr) { console.log(v); }
// 5 7 9  — solo indici dispari
```

### `Symbol.toStringTag` e `Symbol.hasInstance`

Controllano rispettivamente `toString()` e `instanceof`:

```js
function Foo(greeting) { this.greeting = greeting; }

Foo.prototype[Symbol.toStringTag] = "Foo";
Object.defineProperty(Foo, Symbol.hasInstance, {
    value: function(inst) { return inst.greeting == "hello"; }
});

var a = new Foo("hello"), b = new Foo("world");
b[Symbol.toStringTag] = "cool";

a.toString();    // "[object Foo]"
String(b);       // "[object cool]"
a instanceof Foo; // true
b instanceof Foo; // false  — controllato da Symbol.hasInstance
```

### `Symbol.species`

Controlla quale constructor usano i metodi built-in (come `slice()`, `map()`) quando devono creare nuove istanze in una subclass:

```js
class MyCoolArray extends Array {
    static get [Symbol.species]() { return Array; } // forza uso del parent
}
var a = new MyCoolArray(1, 2, 3);
a.slice(1) instanceof MyCoolArray; // false — usa Array, non MyCoolArray
```

### `Symbol.toPrimitive`

Controlla la coercione dell'oggetto a primitivo, ricevendo un hint (`"number"`, `"string"`, `"default"`):

```js
var arr = [1, 2, 3, 4, 5];
arr + 10;  // "1,2,3,4,510"  — comportamento default

arr[Symbol.toPrimitive] = function(hint) {
    if (hint == "default" || hint == "number") {
        return this.reduce((acc, v) => acc + v, 0);
    }
};
arr + 10;  // 25  — somma degli elementi
```

### Symbol per le Regular Expression

Quattro WKS controllano i metodi di `String.prototype` che accettano regex:

| Symbol | Usato da |
|---|---|
| `Symbol.match` | `String.prototype.match()` — anche per il check `isRegExp` |
| `Symbol.replace` | `String.prototype.replace()` |
| `Symbol.search` | `String.prototype.search()` |
| `Symbol.split` | `String.prototype.split()` |

Per impedire che un oggetto venga trattato come regex, si può impostare `Symbol.match = false`.

### `Symbol.isConcatSpreadable`

```js
var a = [1, 2, 3], b = [4, 5, 6];
b[Symbol.isConcatSpreadable] = false;
[].concat(a, b); // [1, 2, 3, [4, 5, 6]]  — b non viene espanso
```

---

## Proxy

Un Proxy avvolge un oggetto (il *target*) intercettando le operazioni eseguite su di esso tramite *handler* (anche detti *trap*). Ogni trap ha un corrispondente in `Reflect`.

```js
var obj = { a: 1 };
var pobj = new Proxy(obj, {
    get(target, key, context) {
        console.log("accesso:", key);
        return Reflect.get(target, key, context);
    }
});

obj.a;   // 1  — nessun trap
pobj.a;  // "accesso: a" → 1
```

### Trap disponibili

| Trap | Triggera su |
|---|---|
| `get` | accesso a proprietà (`.`, `[]`) |
| `set` | assegnazione di proprietà |
| `deleteProperty` | `delete obj.prop` |
| `apply` | invocazione della funzione target |
| `construct` | `new target(...)` |
| `has` | operatore `in` |
| `ownKeys` | `Object.keys()`, `Reflect.ownKeys()` |
| `getOwnPropertyDescriptor` | `Object.getOwnPropertyDescriptor()` |
| `defineProperty` | `Object.defineProperty()` |
| `getPrototypeOf` | `Object.getPrototypeOf()`, `instanceof` |
| `setPrototypeOf` | `Object.setPrototypeOf()` |
| `preventExtensions` / `isExtensible` | omonimi Object/Reflect |

### Proxy revocabile

```js
var { proxy: pobj, revoke: prevoke } = Proxy.revocable(obj, handlers);

pobj.a;   // funziona normalmente
prevoke();
pobj.a;   // TypeError — il proxy è stato revocato
```

Utile per passare un riferimento controllato a un'altra parte dell'applicazione, con la possibilità di invalidarlo in seguito.

### Pattern: proxy-first

Il proxy è l'interfaccia primaria; l'oggetto reale è protetto:

```js
var messages = [];
var messages_proxy = new Proxy(messages, {
    get(target, key) {
        if (typeof target[key] == "string") {
            return target[key].replace(/[^\w]/g, ""); // filtra punteggiatura
        }
        return target[key];
    },
    set(target, key, val) {
        if (typeof val == "string") {
            val = val.toLowerCase();
            if (target.indexOf(val) == -1) target.push(val); // solo stringhe uniche
        }
        return true;
    }
});
```

### Pattern: proxy-last

Il proxy è nel `[[Prototype]]` chain come fallback. Il codice interagisce con l'oggetto reale; il proxy gestisce solo le proprietà non trovate:

```js
var catchall = new Proxy({}, {
    get(target, key, context) {
        return function() { context.speak(key + "!"); };
    }
});
var greeter = { speak(who = "someone") { console.log("hello", who); } };
Object.setPrototypeOf(greeter, catchall);

greeter.speak();      // "hello someone"  — trovato su greeter
greeter.everyone();   // "hello everyone!" — non trovato, proxy intercetta
```

### Pattern: "no such property"

Protezione contro accessi a proprietà non definite — in versione proxy-last:

```js
var pobj = new Proxy({}, {
    get() { throw "No such property/method!"; },
    set() { throw "No such property/method!"; }
});
var obj = { a: 1, foo() { console.log("a:", this.a); } };
Object.setPrototypeOf(obj, pobj);

obj.a = 3;   // OK
obj.foo();   // OK
obj.b = 4;   // Error: No such property/method!
```

### Limiti

Non tutte le operazioni sono trappable: `typeof`, `String()`, `+""`, `==`, `===` tra oggetti non attivano trap. Esistono anche invarianti che non possono essere violate (es. `isExtensible` deve restituire boolean).

---

## Reflect API

`Reflect` è un plain object (come `Math`) che espone funzioni corrispondenti a tutte le operazioni interne JS — esattamente i metodi intercettabili dai Proxy:

```js
Reflect.get(obj, "foo");           // come obj.foo
Reflect.set(obj, "foo", 42);       // come obj.foo = 42
Reflect.deleteProperty(obj, "foo"); // come delete obj.foo
Reflect.has(obj, "foo");           // come "foo" in obj
Reflect.apply(fn, thisObj, args);  // come fn.apply(thisObj, args)
Reflect.construct(Foo, args);      // come new Foo(...args)
Reflect.ownKeys(obj);              // prop string + symbol, ordine garantito
```

Differenza da `Object.*`: i metodi `Reflect.*` lanciano un errore se il target non è un oggetto; gli analoghi `Object.*` tentano di coercirlo.

### Ordinamento delle proprietà

`Reflect.ownKeys()` (e `Object.getOwnPropertyNames()` / `Object.getOwnPropertySymbols()`) garantiscono l'ordine:

1. Integer index in ordine numerico crescente
2. Proprietà stringa in ordine di creazione
3. Symbol in ordine di creazione

```js
var o = {};
o[Symbol("c")] = "yay";
o[2] = true; o[1] = true;
o.b = "ok"; o.a = "cool";

Reflect.ownKeys(o); // [1, 2, "b", "a", Symbol(c)]
```

`Object.keys()`, `for..in` e `JSON.stringify()` condividono un ordinamento implementation-dependent (non garantito dalla spec) — non fare affidamento su di esso.

---

## Feature Testing

Il feature testing è metaprogrammazione: il codice sonda il proprio ambiente di esecuzione per determinare come comportarsi.

**Test di API** (pattern standard):

```js
if (!Number.isNaN) {
    Number.isNaN = function(x) { return x !== x; };
}
```

**Test di sintassi** — non si può usare `try/catch` perché il parser blocca prima dell'esecuzione. Si usa `new Function()` per compilare dinamicamente:

```js
try {
    new Function("(() => {})");
    ARROW_FUNCS_ENABLED = true;
} catch (err) {
    ARROW_FUNCS_ENABLED = false;
}
```

**Split delivery**: in base ai risultati dei test si carica la versione ES6 nativa oppure quella transpilata. È l'approccio più maturo: ogni utente riceve esattamente il codice ottimale per il suo browser.

---

## Tail Call Optimization (TCO)

Ogni chiamata di funzione alloca un nuovo stack frame. Con la ricorsione profonda, la catena di stack frame può causare `RangeError: Maximum call stack size exceeded`.

Una **tail call** è una chiamata di funzione che è l'ultima operazione di una funzione — il risultato viene restituito direttamente senza ulteriore elaborazione. In questo caso l'engine può *riutilizzare* il frame corrente invece di crearne uno nuovo.

```js
"use strict";
function foo(x) { return x * 2; }

/* NON tail call — c'è ancora la somma 1 + da fare dopo foo() */
function bar(x) { return 1 + foo(x); }

/* tail call — foo() è l'ultima cosa */
function bar(x) {
    if (x > 10) return foo(x);
    return bar(x + 1);
}
```

ES6 richiede che tutte le Proper Tail Call (PTC) vengano ottimizzate in strict mode. Il risultato: ricorsione illimitata senza crescita dello stack.

### Riscrivere per TCO — pattern accumulatore

```js
/* non ottimizzabile: (x/2) + foo(x-1) richiede lo stack di foo */
function foo(x) {
    if (x <= 1) return 1;
    return (x / 2) + foo(x - 1); // RangeError con x grande
}

/* ottimizzabile: _foo() è sempre in tail position */
var foo = (function() {
    function _foo(acc, x) {
        if (x <= 1) return acc;
        return _foo((x / 2) + acc, x - 1); // PTC
    }
    return function(x) { return _foo(1, x); };
})();

foo(123456); // 3810376848.5
```

### Alternativa: trampolining

Per ambienti senza TCO, si può usare un loop che chiama ripetutamente funzioni che restituiscono la prossima funzione parziale (invece di chiamarsi ricorsivamente):

```js
function trampoline(res) {
    while (typeof res == "function") { res = res(); }
    return res;
}

var foo = (function() {
    function _foo(acc, x) {
        if (x <= 1) return acc;
        return function partial() { return _foo((x / 2) + acc, x - 1); };
    }
    return function(x) { return trampoline(_foo(1, x)); };
})();

foo(123456); // 3810376848.5  — nessuna crescita dello stack
```

Il trampolining è riutilizzabile e non richiede TCO — funziona in qualsiasi engine.

---

## ⚡ Ripasso veloce

```js
/* === FUNCTION NAME === */
var fn = function() {};
fn.name;             // "fn"  — inferito
(function() {}).name; // ""   — nessuna inferenza

/* === new.target === */
class Foo {
    constructor() {
        if (new.target === Foo) { /* diretta */ }
        else { /* subclass */ }
    }
}

/* === WELL-KNOWN SYMBOL === */
obj[Symbol.iterator] = function*() { yield 1; yield 2; };
obj[Symbol.toStringTag] = "MyType";  // → "[object MyType]"
obj[Symbol.toPrimitive] = (hint) => hint === "number" ? 42 : "str";
Ctor[Symbol.hasInstance] = (inst) => /* custom instanceof logic */;
Arr[Symbol.species] = () => Array;   // slice/map producono Array, non subclass

/* === PROXY === */
var p = new Proxy(target, {
    get(target, key, ctx) { return Reflect.get(target, key, ctx); },
    set(target, key, val, ctx) { return Reflect.set(target, key, val, ctx); }
});
var { proxy, revoke } = Proxy.revocable(target, handlers);
revoke(); // da quel momento: TypeError su qualsiasi accesso

/* === REFLECT === */
Reflect.get(obj, "key");
Reflect.set(obj, "key", val);
Reflect.has(obj, "key");           // come "key" in obj
Reflect.ownKeys(obj);              // ordine garantito: int → string → symbol
Reflect.apply(fn, thisArg, args);
Reflect.construct(Ctor, args);

/* === TCO === */
"use strict";
function _foo(acc, x) {
    if (x <= 1) return acc;
    return _foo((x / 2) + acc, x - 1); // PTC → TCO in ES6
}
```

---

## Domande

<details>
<summary>Cos'è un Proxy in ES6 e quali operazioni può intercettare?</summary>

Un Proxy è un oggetto speciale che si interpone davanti a un altro oggetto (il *target*) intercettando le operazioni fondamentali eseguite su di esso tramite funzioni handler chiamate *trap*. Le operazioni intercettabili includono: lettura di proprietà (`get`), scrittura (`set`), cancellazione (`deleteProperty`), invocazione della funzione target (`apply`), costruzione con `new` (`construct`), enumerazione delle proprietà (`ownKeys`), verifica dell'esistenza (`has`), lettura/definizione di descriptor (`getOwnPropertyDescriptor`, `defineProperty`), lettura/impostazione del prototype, e controllo dell'estensibilità. Il proxy può modificare, bloccare, loggare o redirigere qualsiasi di queste operazioni. Ogni trap ha un corrispondente in `Reflect` che esegue il comportamento default — così il pattern tipico è intercettare, fare qualcosa di custom, poi delegare a `Reflect`.

</details>

<details>
<summary>Qual è la differenza tra il pattern "proxy-first" e "proxy-last"?</summary>

Nel pattern **proxy-first**, il proxy è l'interfaccia principale con cui il codice interagisce — l'oggetto reale è nascosto. Tutti gli accessi passano prima dal proxy, che applica validazioni, trasformazioni o protezioni. Nel pattern **proxy-last**, l'oggetto reale è l'interfaccia principale, e il proxy è posizionato nel suo `[[Prototype]]` chain come fallback. Il codice usa l'oggetto direttamente; il proxy interviene solo quando una proprietà non viene trovata risalendo la catena. Il proxy-last è più semplice perché non deve distinguere se la proprietà esiste o meno — se l'operazione arriva al proxy, è già certo che non è stata trovata altrove.

</details>

<details>
<summary>Cosa sono i Well-Known Symbol e perché sono utili per la metaprogrammazione?</summary>

I Well-Known Symbol (WKS) sono Symbol predefiniti da ES6 che l'engine usa internamente per controllare comportamenti fondamentali: `Symbol.iterator` per il protocollo di iterazione, `Symbol.toPrimitive` per la coercione a primitivo, `Symbol.toStringTag` per l'output di `toString()`, `Symbol.hasInstance` per `instanceof`, `Symbol.species` per la costruzione di nuove istanze nei metodi built-in. Prima di ES6, questi comportamenti erano fissi e non personalizzabili. I WKS li espongono come "ganci" (hook) che si possono impostare sugli oggetti per sovrascrivere il comportamento predefinito. È metaprogrammazione perché si sta modificando come il linguaggio stesso (i suoi operatori e costrutti) si comporta con gli oggetti definiti nel proprio codice.

</details>

<details>
<summary>Cos'è una Proper Tail Call e perché ES6 introduce la TCO?</summary>

Una Proper Tail Call (PTC) è una chiamata di funzione che si trova in posizione di "tail" — è l'ultima operazione della funzione, e il suo risultato viene restituito direttamente senza ulteriori elaborazioni. `return foo(x)` è una PTC; `return 1 + foo(x)` non lo è, perché c'è ancora una somma da fare dopo che `foo` ritorna. La Tail Call Optimization (TCO) è la tecnica per cui il JS engine, invece di allocare un nuovo stack frame per una PTC, riutilizza quello corrente — perché non c'è più nessun stato da preservare. Il risultato è che la ricorsione in tail position non consuma stack: si può ricorrere profondamente quanto necessario senza rischiare `RangeError`. ES6 richiede TCO in strict mode. Per le funzioni che non sono già in tail form, il pattern con accumulatore permette di ristrutturarle per renderle elegibili.

</details>

<details>
<summary>Perché il test di feature per la sintassi ES6 non funziona con <code>try/catch</code> normale, e qual è la soluzione?</summary>

Il problema è che JS compila l'intero file prima di eseguirlo. Se il file contiene sintassi che l'engine non conosce (es. arrow function in un engine pre-ES6), il parser lancia un errore di sintassi prima ancora che il codice venga eseguito — quindi il `try/catch` non ha modo di catturarlo. La soluzione è usare `new Function()` che compila dinamicamente una stringa di codice solo quando viene chiamata, non al momento del parsing del file principale. Il test diventa: `try { new Function("(() => {})"); OK = true; } catch (e) { OK = false; }`. Questo permette di determinare a runtime se una feature sintattica è supportata. La tecnica più sofisticata che ne deriva è la *split delivery*: in base ai test, il bootstrapper carica la versione ES6 nativa del codice oppure quella transpilata, ottimizzando per ogni tipo di browser.

</details>
