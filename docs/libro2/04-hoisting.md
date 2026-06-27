# Hoisting

Si tende a pensare che JavaScript esegua il codice riga per riga, dall'alto verso il basso, nell'ordine in cui è scritto. Questa intuizione è sostanzialmente corretta — ma c'è un aspetto sottile nel modo in cui le dichiarazioni vengono gestite che la viola sistematicamente, con conseguenze che confondono chi non ne è consapevole.

## Il problema: uovo o gallina?

Si considerino questi due snippet:

```js
a = 2;
var a;
console.log(a); // cosa stampa?
```

Molti si aspetterebbero `undefined`: `var a` viene dopo l'assegnamento, e sembra naturale che la "ridichiarazione" reimposti la variabile. L'output è invece `2`.

```js
console.log(a); // cosa stampa?
var a = 2;
```

Qui ci si potrebbe aspettare `2` (per analogia con il caso precedente) o un `ReferenceError` (perché `a` sembra usata prima di essere dichiarata). L'output è `undefined`.

Il comportamento apparentemente contraddittorio ha una spiegazione precisa.

## Come funziona: compilazione prima dell'esecuzione

Come visto nel Cap 1, il JavaScript engine compila il codice prima di eseguirlo. Parte del processo di compilazione consiste nel trovare tutte le dichiarazioni e associarle ai rispettivi scope — questo è il cuore del lexical scope.

La conseguenza pratica è che `var a = 2;` non è una singola operazione: l'engine la vede come **due statement separati**:

- `var a;` — dichiarazione, elaborata durante la **fase di compilazione**
- `a = 2;` — assegnamento, lasciato in loco per la **fase di esecuzione**

Il primo snippet viene quindi elaborato come:

```js
/* fase di compilazione */
var a;

/* fase di esecuzione */
a = 2;
console.log(a); // 2
```

Il secondo come:

```js
/* fase di compilazione */
var a;

/* fase di esecuzione */
console.log(a); // undefined — a esiste ma non ha ancora un valore
a = 2;
```

Il modo metaforico di descrivere questo comportamento è che le dichiarazioni vengono "spostate" in cima al loro scope prima che il codice venga eseguito. Questo meccanismo si chiama **hoisting** (sollevamento).

> Solo le dichiarazioni vengono sollevate — non gli assegnamenti né qualsiasi altra logica eseguibile. L'hoisting non riordina il codice: è una conseguenza della separazione tra fase di compilazione e fase di esecuzione.

## Hoisting delle funzioni

Le **function declaration** vengono hoistate integralmente — nome e corpo — il che permette di chiamarle prima della riga in cui sono scritte:

```js
foo();

function foo() {
    console.log(a); // undefined
    var a = 2;
}
```

L'hoisting è **per-scope**: `foo` viene sollevata all'inizio del global scope, e `var a` viene sollevata all'inizio dello scope di `foo`. Il programma viene trattato come:

```js
function foo() {
    var a;          /* sollevata in cima a foo */
    console.log(a); // undefined
    a = 2;
}

foo();
```

Le **function expression**, al contrario, non vengono hoistate — nemmeno quelle con nome:

```js
foo(); // TypeError — foo esiste ma è undefined in questo punto
var foo = function bar() {
    // ...
};
```

La variabile `foo` viene sollevata (ed è `undefined`), ma il valore — la funzione — viene assegnato solo all'esecuzione. Tentare di invocare `undefined` produce un `TypeError`, non un `ReferenceError`. Il nome `bar` della function expression non è nemmeno raggiungibile dallo scope esterno:

```js
foo(); // TypeError
bar(); // ReferenceError
var foo = function bar() { /* ... */ };
```

## Le funzioni prima delle variabili

Quando nello stesso scope convivono dichiarazioni di variabili e di funzioni con lo stesso nome, **le function declaration vengono hoistate prima delle variabili**. La dichiarazione `var` duplicata viene ignorata.

```js
foo(); // 1
var foo;

function foo() { console.log(1); }
foo = function() { console.log(2); };
```

L'engine elabora questo codice come:

```js
function foo() { console.log(1); } /* hoistata per prima */
/* var foo ignorata — foo già dichiarata */

foo(); // 1
foo = function() { console.log(2); };
```

Le dichiarazioni di funzione successive **sovrascrivono** quelle precedenti con lo stesso nome:

```js
foo(); // 3 — l'ultima dichiarazione vince

function foo() { console.log(1); }
var foo = function() { console.log(2); };
function foo() { console.log(3); }
```

Questo comportamento rende le dichiarazioni duplicate in uno stesso scope una fonte di bug silenziosi e difficili da individuare. È buona pratica evitarle sistematicamente.

## Function declaration nei blocchi

Le function declaration all'interno di blocchi condizionali (`if`, `else`) vengono hoistate all'inizio dello scope circostante — non restano confinate al blocco. Questo comportamento non è però garantito dalla specifica e cambia tra ambienti diversi:

```js
foo(); // "b" — ma il comportamento non è affidabile

var a = true;
if (a) {
    function foo() { console.log("a"); }
} else {
    function foo() { console.log("b"); }
}
```

Dichiarare funzioni all'interno di blocchi è una pratica da evitare. Per questo scopo si usano le function expression.

---

## ⚡ Ripasso veloce

**Hoisting** = le dichiarazioni (variabili e funzioni) vengono processate nella fase di compilazione, prima dell'esecuzione. Solo le dichiarazioni vengono sollevate — gli assegnamenti restano al loro posto.

```js
console.log(a); // undefined — a esiste già, ma senza valore
var a = 2;
console.log(a); // 2
```

**Function declaration** → hoistata integralmente (nome + corpo) → chiamabile prima della definizione.

**Function expression** → solo il nome della variabile viene hoistato (come `undefined`) → l'assegnamento rimane al suo posto → invocarla prima dell'assegnamento produce `TypeError`.

```js
foo();            // OK — function declaration hoistata
bar();            // TypeError — bar è undefined in questo punto

function foo() { console.log("foo"); }
var bar = function() { console.log("bar"); };
```

**Priorità**: function declaration > variable declaration. Le `var` duplicate vengono ignorate; le function declaration successive sovrascrivono quelle precedenti.

---

## Domande

<details>
<summary>Perché `var a = 2;` produce `undefined` se si usa `a` prima di quella riga?</summary>

Perché l'engine divide `var a = 2;` in due operazioni separate. Durante la compilazione viene elaborata solo la dichiarazione `var a`, che viene associata allo scope corrente — la variabile esiste già dall'inizio dell'esecuzione, ma il suo valore è `undefined`. L'assegnamento `a = 2` avviene solo quando quella specifica riga viene raggiunta durante l'esecuzione. Accedere a `a` prima di quella riga restituisce quindi `undefined`, non un `ReferenceError`.

</details>

<details>
<summary>Qual è la differenza di hoisting tra una function declaration e una function expression?</summary>

Una function declaration viene hoistata integralmente: sia il nome che il corpo della funzione sono disponibili dall'inizio dell'esecuzione dello scope. È quindi possibile chiamarla prima della riga in cui è scritta. Una function expression, invece, viene trattata come un assegnamento: solo il nome della variabile viene hoistato (come `undefined`), mentre il valore — la funzione stessa — viene assegnato solo quando quella riga viene eseguita. Invocare la variabile prima dell'assegnamento produce un `TypeError` (si sta tentando di chiamare `undefined` come funzione).

</details>

<details>
<summary>Cosa succede quando una function declaration e una `var` hanno lo stesso nome nello stesso scope?</summary>

La function declaration viene hoistata prima della dichiarazione `var`, che viene ignorata come duplicata. La funzione vince. Se dopo la dichiarazione segue un assegnamento (come `var foo = function() {…}`), quell'assegnamento avviene normalmente durante l'esecuzione e sovrascrive il riferimento a `foo`. Le dichiarazioni di funzione successive con lo stesso nome si sovrascrivono a vicenda nell'ordine in cui appaiono nel codice, con l'ultima che ha la precedenza.

</details>

<details>
<summary>L'hoisting "sposta" fisicamente il codice in cima al file?</summary>

No. L'hoisting è una metafora utile ma non una descrizione letterale di ciò che accade. Il codice sorgente non viene riordinato. Quello che succede realmente è che durante la fase di compilazione l'engine analizza tutte le dichiarazioni nello scope e le registra nell'ambiente lessicale prima che inizi l'esecuzione. Quando l'engine inizia a eseguire il codice riga per riga, le variabili e le funzioni dichiarate nello scope esistono già — è questo che crea l'effetto del "sollevamento".

</details>

<details>
<summary>Perché è pericoloso dichiarare funzioni all'interno di blocchi `if` / `else`?</summary>

Perché il comportamento non è definito con precisione dalla specifica ES5 e varia tra motori JavaScript diversi: in molti ambienti la function declaration viene hoistata all'inizio dello scope circostante ignorando la condizione, in altri viene trattata come un blocco condizionale. Il risultato è codice il cui comportamento dipende dall'ambiente di esecuzione e non è prevedibile. La pratica corretta è usare function expression assegnate a variabili dichiarate con `let` o `const` all'interno dei blocchi.

</details>
