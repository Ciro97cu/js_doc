# Grammatica

La **grammatica** di JavaScript descrive come gli elementi sintattici si combinano in programmi validi. Non è sinonimo di sintassi: la sintassi sono le regole formali su quale sequenza di caratteri è ammessa; la grammatica aggiunge le regole semantiche su come quei costrutti si interpretano in contesto. Molti comportamenti sorprendenti di JS derivano dalla grammatica, non dalla sintassi.

---

## Statement ed expression

Ogni programma JS è composto di **statement** (istruzioni) e **expression** (espressioni). La distinzione è analoga a frasi e sintagmi in italiano: ogni expression si valuta in un singolo valore; ogni statement è un'istruzione completa.

```js
var a = 3 * 6; /* declaration statement: dichiara a, assegna 18 */
a;             /* expression statement: l'expression "a" vale 18 */
```

`3 * 6` è un'expression (vale `18`). `var a = 3 * 6` è uno statement. `a` da solo è sia expression che expression statement.

### Completion value degli statement

Ogni statement ha un **completion value** — anche se è solo `undefined`. Il `var` statement restituisce `undefined` (per spec); un blocco `{ }` restituisce il completion value dell'ultimo statement contenuto:

```js
/* nella console del browser, digitare: */
var b;
if (true) { b = 4 + 38; }
/* la console riporta 42 — il completion value del blocco if */
```

Non esiste sintassi per assegnare il completion value di uno statement a una variabile. `eval()` lo permette, ma va evitato. Una proposta ES7 (`do { }` expression) formalizzerebbe questo pattern — al momento è solo curiosità.

### Side effect delle expression

La maggior parte delle expression è pura: calcola un valore senza modificare lo stato. Ma alcune hanno **side effect**:

**`++` e `--`**: postfisso (`a++`) restituisce il valore corrente e poi incrementa; prefisso (`++a`) incrementa prima e poi restituisce il nuovo valore.

```js
var a = 42;
var b = a++; /* b = 42, poi a diventa 43 */
var c = ++a; /* a diventa 44, poi c = 44 */
```

`(a++)` non cambia il comportamento: le parentesi non creano un nuovo contesto di valutazione. Per catturare il valore post-incremento si usa l'operatore virgola:

```js
var a = 42, b;
b = (a++, a); /* prima valuta a++ (side effect), poi a → b = 43 */
```

**`delete`**: rimuove una proprietà da un oggetto. Restituisce `true` se l'operazione è valida, `false` altrimenti.

```js
var obj = { a: 42 };
delete obj.a; // true
obj.a;        // undefined
```

**`=`**: l'assegnamento restituisce il valore assegnato, il che consente assegnamenti concatenati:

```js
var a, b, c;
a = b = c = 42; /* a = (b = (c = 42)) — right-associative */

/* pattern utile: assignment dentro condizione */
function vowels(str) {
    var matches;
    if (str && (matches = str.match(/[aeiou]/g))) {
        return matches; /* matches già assegnato */
    }
}
```

---

## Regole contestuali

Diversi costrutti JS usano la stessa sintassi per cose diverse a seconda del contesto.

### Curly braces `{ }`

Le stesse `{ }` significano cose diverse:

| Contesto | Significato |
|---|---|
| `var a = { }` | object literal |
| Statement autonomo `{ }` | blocco di codice |
| ES6 `var { a, b } = obj` | destructuring |
| ES6 `function foo({ a, b })` | destructuring parametro |

Un blocco autonomo `{ }` è valido JS — raro ma utile con `let` per scope limitato.

### Label

All'interno di un blocco, la sintassi `label: statement` definisce un'etichetta. Sono consentiti `break label` e `continue label` per saltare a/da loop etichettati:

```js
/* break etichettato — esce dal loop esterno */
outer: for (var i = 0; i < 4; i++) {
    for (var j = 0; j < 4; j++) {
        if (i * j >= 3) {
            break outer; /* esce dal for "outer", non solo dal for interno */
        }
    }
}

/* continue etichettato — salta all'iterazione successiva del loop esterno */
outer: for (var i = 0; i < 4; i++) {
    for (var j = 0; j < 4; j++) {
        if (j == i) continue outer;
        console.log(i, j);
    }
}
```

I labeled loop sono rari e spesso evitabili con funzioni. Se si usano, commentare bene l'intento.

### JSON non è JS valido

`{"a":42}` inserito come statement JS viene interpretato come blocco con un label `"a"` — ma le label non possono avere le virgolette, quindi è un errore. JSON è un sottoinsieme della **sintassi** JS, ma non della **grammatica** (non è un programma JS autonomo valido).

### Il gotcha `[] + {}` vs `{} + []`

```js
[] + {};  // "[object Object]"
{} + [];  // 0
```

Nel primo caso, `{}` è nel contesto di un operand del `+` → è un object literal → stringa `"[object Object]"`. Nel secondo, `{}` è all'inizio dello statement → blocco vuoto (nessuna azione) → `+[]` coerce l'array a numero → `0`.

### `else if` non esiste

Non esiste una clausola `else if` in JS. `if` e `else` possono omettere le `{ }` se il corpo è un singolo statement. Quindi:

```js
if (a) { /* ... */ }
else if (b) { /* ... */ }
else { /* ... */ }

/* equivale esattamente a: */
if (a) { /* ... */ }
else {
    if (b) { /* ... */ }
    else { /* ... */ }
}
```

Il pattern `else if` è idiomatico e accettato — ma tecnicamente è un `else` con un singolo `if` come body.

---

## Precedenza degli operatori

Quando un'expression contiene più operatori, la **precedenza** (operator precedence) determina quale si valuta prima. L'**associatività** determina come si raggruppano operatori con la stessa precedenza.

Ordine di precedenza (dal più alto al più basso, sezione rilevante):

1. `++`/`--` postfisso
2. `!`, `~`, `+`/`-` unari, `++`/`--` prefisso
3. `*`, `/`, `%`
4. `+`, `-`
5. `<`, `>`, `<=`, `>=`
6. `==`, `!=`, `===`, `!==`
7. `&&`
8. `||`
9. `? :`
10. `=`, `+=`, `-=`, …
11. `,`

**`&&` ha precedenza maggiore di `||`**:

```js
false && true || true;      // true — come (false && true) || true
true || false && false;     // true — come true || (false && false)
```

**`&&` e `||` prima di `? :`**:

```js
a && b || c ? c || b ? a : c && b : a
/* valutato come: */
((a && b) || c) ? ((c || b) ? a : (c && b)) : a
```

### Short-circuit

`&&` e `||` valutano il secondo operando solo se necessario:

```js
opts && opts.cool;       /* se opts è falsy, opts.cool non viene valutato */
opts.cache || primeCache(); /* se opts.cache è truthy, primeCache() non viene chiamato */
```

### Associatività

`&&` e `||` sono **left-associative**: `a && b && c` si raggruppa come `(a && b) && c`. Il risultato osservabile è lo stesso di `a && (b && c)`, ma la direzione di raggruppamento conta per operatori con side effect.

`? :` e `=` sono **right-associative**:

```js
/* ternario — right-associative */
a ? b : c ? d : e;
/* equivale a: */
a ? b : (c ? d : e);     /* non a: (a ? b : c) ? d : e */

/* assegnamento — right-associative */
a = b = c = 42;
/* equivale a: */
a = (b = (c = 42));
```

### Quando usare le parentesi

Affidarsi alla precedenza va bene per espressioni comuni (`a && b && c`). Per espressioni complesse o non ovvie, le parentesi esplicite migliorano la leggibilità e vanno preferite.

---

## ASI — Automatic Semicolon Insertion

JS inserisce automaticamente `;` in certi punti quando il parser incontrerebbe altrimenti un errore. L'ASI agisce solo **in presenza di newline** — mai in mezzo a una riga.

I casi più rilevanti dove l'ASI interviene:

- Fine di uno statement che non ha `;` ma la riga successiva non è una continuazione valida
- Dopo `do { } while(cond)` — il `;` finale è richiesto dalla grammatica ma spesso dimenticato
- Dopo `return`, `break`, `continue`, `yield` seguiti da newline: il parser inserisce `;` subito

```js
function foo(a) {
    if (!a) return     /* ASI inserisce ; qui */
    a *= 2;            /* questa riga non è il valore di return */
    /* ... */
}
```

Per mantenere un return su più righe, la parentesi aperta deve stare sulla stessa riga di `return`:

```js
function foo(a) {
    return (
        a * 2 + 3 / 12
    );
}
```

Il dibattito su usare o meno i `;` sistematicamente è irrilevante qui: la posizione raccomandata è scrivere i `;` dove la grammatica li richiede e non fare affidamento sull'ASI come meccanismo di default, perché l'ASI è formalmente un meccanismo di **correzione di errori sintattici**, non un'abbreviazione stilistica.

---

## Errori

JS classifica gli errori in due categorie:

**Early errors** (errori in compilazione): rilevati prima che il codice esegua. Non sono catchabili con `try..catch`. Esempi:
- Sintassi regex invalida: `var a = /+foo/;`
- Assegnamento a un non-identificatore: `42 = a;`
- In strict mode: parametri duplicati, proprietà duplicate in object literal

```js
function foo(a, b, a) { }       /* ok in non-strict */
function bar(a, b, a) { "use strict"; } /* Error! in strict */
```

**Runtime errors**: si verificano durante l'esecuzione e sono catchabili con `try..catch`. Includono `TypeError`, `ReferenceError`, ecc.

---

## TDZ — Temporal Dead Zone

ES6 introduce il concetto di **Temporal Dead Zone**: la zona temporale in cui una variabile `let` o `const` esiste nello scope ma non è ancora inizializzata. Accedere a una variabile in TDZ lancia `ReferenceError`.

```js
{
    a = 2;   /* ReferenceError! a è in TDZ */
    let a;
}
```

A differenza delle variabili non dichiarate, `typeof` non ha l'eccezione di sicurezza per le TDZ:

```js
{
    typeof x;  /* undefined — variabile non dichiarata: safe */
    typeof b;  /* ReferenceError! — b in TDZ: non safe */
    let b;
}
```

Anche i parametri di default ES6 hanno il loro TDZ: un default che riferisce a un parametro successivo lancia errore.

```js
function foo(a = 42, b = a + b + 5) { } /* Error! b in TDZ quando si valuta il suo default */
```

---

## `arguments` e parametri default

In ES5, il parametro named e lo slot corrispondente in `arguments` sono **linked** se l'argomento viene passato:

```js
function foo(a) {
    a = 42;
    console.log(arguments[0]); /* 42 — linkati se l'argomento è stato passato */
}
foo(2);   /* 42 */
foo();    /* undefined — nessun linkaggio se omesso */
```

In strict mode questo linkaggio non esiste — `arguments[0]` rimane il valore originale passato, non riflette le modifiche al parametro named.

Con i parametri default ES6, `arguments` riflette solo gli argomenti effettivamente passati (non i default applicati):

```js
function foo(a = 42, b = a + 1) {
    console.log(arguments.length, a, b, arguments[0], arguments[1]);
}
foo();          /* 0  42  43  undefined  undefined */
foo(10);        /* 1  10  11  10         undefined */
foo(10, null);  /* 2  10  null 10        null      */
```

Regola pratica: non mescolare accesso a `arguments[n]` e al parametro named corrispondente nella stessa funzione. Preferire i rest parameter ES6 (`...args`) a `arguments`.

---

## `try..finally`

Il blocco `finally` **esegue sempre**, indipendentemente da come termina `try` (o `catch`). Viene eseguito prima che il completion value venga restituito al chiamante.

```js
function foo() {
    try {
        return 42; /* return "schedula" il valore, ma finally va prima */
    }
    finally {
        console.log("Hello"); /* eseguito prima che il return si concretizzi */
    }
}
console.log(foo());
// Hello
// 42
```

Lo stesso vale per `throw` in `try`:

```js
function foo() {
    try { throw 42; }
    finally { console.log("Hello"); }
}
// Hello
// Uncaught Exception: 42
```

Un'eccezione lanciata in `finally` **sovrascrive** il completion value precedente (sia `return` sia `throw` da `try`/`catch`):

```js
function foo() {
    try { return 42; }
    finally { throw "Oops!"; } /* sovrascrive il return 42 */
}
/* Uncaught Exception: Oops! */
```

Un `return` esplicito in `finally` sovrascrive il `return` di `try`:

```js
function baz() {
    try { return 42; }
    finally { return "Hello"; } /* sovrascrive */
}
baz(); /* "Hello" */
```

Un `finally` senza `return` lascia stare il `return` di `try` — l'omissione di `return` in `finally` non equivale a `return undefined`.

`continue` e `break` si comportano analogamente a `return` — il `finally` si esegue prima che il salto si materializzi. Combinare `finally` con `labeled break` per cancellare un `return` è legale ma da non fare mai.

---

## `switch`

Lo `switch` è una forma abbreviata della catena `if..else if..else`. Il confronto tra l'expression di test e ciascun `case` usa **`===`** (strict equality, nessuna coercion).

```js
switch (a) {
    case 2:
        /* eseguito se a === 2 */
        break;
    case 42:
        /* eseguito se a === 42 */
        break;
    default:
        /* fallback */
}
```

**Hack con `switch(true)`**: per usare `==` o espressioni booleane nei `case`, si può scrivere `switch(true)` e mettere l'expression nel `case`:

```js
var a = "42";
switch (true) {
    case a == 42:
        console.log("42 or '42'");
        break;
}
```

Il `case` valuta l'espressione, e il risultato viene confrontato (con `===`) con `true`. Attenzione: se l'expression restituisce un valore truthy ma non `true`, il match fallisce:

```js
var a = "hello world";
switch (true) {
    case (a || 10 == 10):
        /* non raggiunto: (a || 10==10) è "hello world", non true */
        break;
    default:
        console.log("Oops");
}
/* fix: case !!(a || 10 == 10): */
```

**Fallthrough**: senza `break`, l'esecuzione continua nel `case` successivo. Utile per raggruppare casi, pericoloso se non intenzionale.

**`default`**: opzionale e non deve necessariamente essere l'ultimo. Se non viene trovato nessun `case`, il `default` viene eseguito — e senza `break`, continua verso i `case` successivi anche se già "saltati" nella fase di matching.

---

## ⚡ Ripasso veloce

**Statement vs expression**: le expression si valutano in un valore; gli statement sono istruzioni. Ogni statement ha un completion value.

**Side effect**: `++`/`--`, `delete`, `=` hanno side effect oltre al valore restituito. `a++` restituisce il valore corrente, poi incrementa; `++a` incrementa prima.

**Curly braces**: `{}` cambia significato in base al contesto (object literal, code block, destructuring). Lo stesso per `else if` — non è una clausola autonoma.

**Precedenza**: `&&` > `||` > `? :` > `=` > `,`. `? :` e `=` sono right-associative; `&&` e `||` left-associative.

**ASI**: inserimento automatico di `;` come correzione di errori del parser. Attenzione a `return`/`break`/`continue` seguiti da newline — il `;` viene inserito immediatamente.

**TDZ**: le variabili `let`/`const` non sono accessibili prima della loro dichiarazione nello scope — anche se `typeof` di solito è sicuro, non lo è in TDZ.

**`finally`**: esegue sempre. Un `return`/`throw` in `finally` sovrascrive quello in `try`.

**`switch`**: usa `===`. Per coercion, hack con `switch(true)` + espressione nel `case`.

```js
/* side effect di a++ vs ++a */
var a = 42;
var b = a++; /* b = 42, a = 43 */
var c = ++a; /* a = 44, c = 44 */

/* finally sempre eseguito */
function foo() {
    try { return 42; }
    finally { console.log("eseguito"); }
}
foo(); /* "eseguito", poi ritorna 42 */

/* switch con coercion */
switch (true) {
    case a == "42": /* usa == */
        break;
}

/* TDZ */
{ typeof b; /* ReferenceError! */ let b; }
```

---

## Domande

<details>
<summary>Qual è la differenza tra `a++` e `++a`, e perché `(a++)` non cattura il valore incrementato?</summary>

`a++` (postfisso) restituisce il valore corrente di `a` e poi esegue l'incremento come side effect. `++a` (prefisso) esegue prima l'incremento e poi restituisce il nuovo valore. `(a++)` **non** cambia il comportamento: le parentesi non definiscono un nuovo contesto di valutazione posticipato — racchiudono semplicemente l'expression `a++`, che si valuta comunque restituendo il valore pre-incremento. Per ottenere il valore post-incremento in un'assegnazione, si usa l'operatore virgola: `b = (a++, a)` — che prima esegue `a++` come side effect e poi legge il nuovo `a`.

</details>

<details>
<summary>Perché `{} + []` dà `0` mentre `[] + {}` dà `"[object Object]"`?</summary>

La differenza è come l'engine interpreta `{}`. Quando `{}` è il primo token di uno statement, viene interpretato come **blocco di codice vuoto** (non come object literal) — e i blocchi non producono valore, quindi non partecipano al `+`. Quello che resta è `+[]`, dove l'operatore `+` unario coerce `[]` a numero: `[]` → `""` → `0`. Nella forma `[] + {}`, invece, `{}` non è all'inizio dello statement, quindi viene interpretato come object literal; il `+` binario lo coerce a stringa `"[object Object]"` e concatena con `""` (da `[]`).

</details>

<details>
<summary>Perché `typeof` è sicuro per variabili non dichiarate ma non per variabili in TDZ?</summary>

La safety guard di `typeof` su variabili non dichiarate è un'eccezione storica della specifica: invece di lanciare `ReferenceError`, restituisce `"undefined"`. Questa eccezione però non si estende alle variabili in Temporal Dead Zone. Una variabile `let` o `const` in TDZ **esiste** nello scope (è stata rilevata in compilazione), ma non è ancora inizializzata — e la specifica definisce che qualsiasi accesso a essa (incluso `typeof`) deve lanciare `ReferenceError`. Quindi `typeof x` (non dichiarata) → `"undefined"` safe; `typeof b` con `let b` più in basso nello stesso blocco → `ReferenceError`.

</details>

<details>
<summary>Perché il blocco `finally` può essere problematico con `return`?</summary>

`finally` garantisce l'esecuzione del proprio codice prima che il completion value di `try` (o `catch`) venga propagato. Se `finally` contiene un `return` esplicito, quel valore **sovrascrive** il `return` del blocco `try`. Se `finally` contiene un `throw`, quell'eccezione sovrascrive sia i `return` sia i `throw` precedenti. La sola omissione di `return` in `finally` non equivale a `return undefined` — lascia stare il valore del `try`. Questo crea comportamenti controintuitivi: il programmatore scrive `return 42` in `try`, ma il valore effettivo restituito dalla funzione può essere diverso se `finally` interferisce.

</details>

<details>
<summary>Perché `switch` usa `===` e come si può usare `==` con lo switch?</summary>

La specifica definisce che il confronto tra il valore del test (`switch(a)`) e ciascuna espressione `case` usa l'algoritmo di strict equality (`===`), quindi nessuna coercion. Per aggirarlo, si può scrivere `switch(true)` e mettere un'espressione con `==` (o qualsiasi altra condizione) nel `case`: l'engine confronta con `===` il risultato dell'espressione `case` con `true`. Attenzione: se l'espressione produce un valore truthy ma non esattamente `true` (per esempio `"hello world"`), il confronto con `===` fallisce. La soluzione è forzare il booleano: `case !!(expr):`.

</details>
