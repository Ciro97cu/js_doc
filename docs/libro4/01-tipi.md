# Tipi

Molti sviluppatori descrivono JavaScript come un linguaggio "senza tipi" perché le variabili non ne hanno uno fisso. Ma la specifica ES5.1 è chiara: i tipi esistono, sono associati ai **valori** — non alle variabili. Capire questa distinzione è il punto di partenza per comprendere la coercion (conversione di tipo), uno degli aspetti più fraintesi di JS.

## I sette tipi built-in

JavaScript definisce sette tipi built-in:

- `null`
- `undefined`
- `boolean`
- `number`
- `string`
- `object`
- `symbol` (aggiunto in ES6)

Tutti eccetto `object` sono **primitivi**.

L'operatore `typeof` permette di ispezionare il tipo di un valore e restituisce sempre una stringa:

```js
typeof undefined     === "undefined"; // true
typeof true          === "boolean";   // true
typeof 42            === "number";    // true
typeof "42"          === "string";    // true
typeof { life: 42 }  === "object";    // true
typeof Symbol()      === "symbol";    // true
```

**Eccezione storica — `null`**: `typeof null` restituisce `"object"`, non `"null"`. È un bug del linguaggio che esiste da quasi trent'anni e non verrà mai corretto perché troppo codice esistente dipende da questo comportamento. Per verificare se un valore è `null`:

```js
var a = null;
(!a && typeof a === "object"); // true — unico modo sicuro
```

`null` è il solo valore primitivo **falsy** che produce `"object"` da `typeof`.

**Funzioni**: `typeof function(){}` restituisce `"function"`, non `"object"`. Le funzioni sono però un sottotipo di `object` — oggetti "chiamabili" con una proprietà interna `[[Call]]`. Come tutti gli oggetti, possono avere proprietà; per esempio `a.length` è il numero di parametri formali dichiarati.

**Array**: `typeof [1,2,3]` restituisce `"object"`. Gli array sono anch'essi un sottotipo di `object` — oggetti con indici numerici e `.length` auto-aggiornato.

---

## I valori hanno tipi, le variabili no

In JavaScript, una variabile è solo un contenitore: può tenere un valore di qualsiasi tipo in qualsiasi momento. Il tipo appartiene al **valore**, non alla variabile.

```js
var a = 42;
typeof a; // "number"

a = true;
typeof a; // "boolean"
```

`typeof a` non chiede "che tipo ha la variabile `a`?" ma "che tipo ha il valore che `a` contiene adesso?".

```js
typeof typeof 42; // "string"
/* typeof 42 → "number" (stringa)
   typeof "number" → "string" */
```

---

## `undefined` vs "undeclared"

Sono due concetti distinti che JavaScript (purtroppo) tende a confondere.

**`undefined`**: variabile dichiarata ma senza valore assegnato. L'engine le assegna automaticamente il valore speciale `undefined`.

**Undeclared** (non dichiarata): variabile mai dichiarata nello scope accessibile. Accedervi produce `ReferenceError`.

```js
var a;
a;  // undefined — dichiarata, senza valore
b;  // ReferenceError: b is not defined
```

Il messaggio "b is not defined" è fuorviante: non significa "b è undefined", significa "b non è stata dichiarata". La confusione terminologica è un difetto storico di JS.

`typeof` aggrava la confusione: restituisce `"undefined"` sia per variabili dichiarate-senza-valore sia per variabili mai dichiarate:

```js
var a;
typeof a; // "undefined"
typeof b; // "undefined" — b non è mai stata dichiarata, ma nessun errore!
```

Questo comportamento — non lanciare errore su una variabile non dichiarata — è però una **safety guard** utile in certi contesti.

---

## Il safety guard di `typeof`

In ambienti browser dove più script condividono il namespace globale, può essere necessario verificare l'esistenza di una variabile globale senza sapere se è stata dichiarata:

```js
/* pericoloso: lancia ReferenceError se DEBUG non esiste */
if (DEBUG) { console.log("debug mode"); }

/* sicuro: typeof non lancia errore su variabili non dichiarate */
if (typeof DEBUG !== "undefined") { console.log("debug mode"); }
```

La stessa tecnica serve per feature detection (polyfill):

```js
if (typeof atob === "undefined") {
    atob = function() { /* implementazione custom */ };
}
```

**Alternativa per globali nel browser**: le variabili globali sono proprietà dell'oggetto `window`. Accedere a una proprietà inesistente su un oggetto non lancia errori:

```js
if (window.DEBUG) { /* sicuro in browser */ }
```

Non funziona però in ambienti non-browser (Node.js), dove `window` non esiste. `typeof` è l'approccio universale.

**Dependency injection**: un'alternativa al safety guard è ricevere la dipendenza come parametro esplicito:

```js
function doSomethingCool(FeatureXYZ) {
    var helper = FeatureXYZ || function() { /* default */ };
    helper();
}
```

Nessun approccio è universalmente corretto — si sceglie in base al contesto.

---

## ⚡ Ripasso veloce

**7 tipi built-in**: `null`, `undefined`, `boolean`, `number`, `string`, `object`, `symbol`. Tutti tranne `object` sono primitivi.

**`typeof null === "object"`**: bug storico, non verrà corretto. Per testare `null`: `(!a && typeof a === "object")`.

**Variabili senza tipo, valori con tipo**: `typeof` ispeziona il valore corrente nel contenitore, non il contenitore stesso.

**`undefined` ≠ undeclared**: `undefined` è un valore; "undeclared" è l'assenza di dichiarazione. `typeof` restituisce `"undefined"` per entrambi — safety guard intenzionale, confusione terminologica no.

```js
/* safety guard pattern */
if (typeof FeatureXYZ !== "undefined") {
    /* usa FeatureXYZ in sicurezza */
}

/* null check corretto */
var x = null;
x === null;                        // true — confronto diretto
(!x && typeof x === "object");     // true — alternativa con typeof
```

| Espressione | Risultato |
|-------------|-----------|
| `typeof undefined` | `"undefined"` |
| `typeof null` | `"object"` ← bug |
| `typeof 42` | `"number"` |
| `typeof "42"` | `"string"` |
| `typeof true` | `"boolean"` |
| `typeof Symbol()` | `"symbol"` |
| `typeof {}` | `"object"` |
| `typeof []` | `"object"` |
| `typeof function(){}` | `"function"` |

---

## Domande

<details>
<summary>Perché si dice che in JavaScript i valori hanno tipi ma le variabili no?</summary>

Perché le variabili JS sono contenitori non tipizzati: una stessa variabile può contenere un numero, poi una stringa, poi un booleano, senza nessuna dichiarazione di tipo e senza errori. Il tipo appartiene al **valore** contenuto, non al contenitore. `typeof a` non interroga la variabile `a` ma il valore che `a` detiene in quell'istante. In linguaggi staticamente tipizzati come TypeScript o Java, la variabile ha un tipo fisso assegnato alla dichiarazione; in JS questa restrizione non esiste.

</details>

<details>
<summary>Perché `typeof null` restituisce `"object"` e come si verifica correttamente un valore null?</summary>

È un bug della prima implementazione di JavaScript, mai corretto per ragioni di retrocompatibilità: troppo codice esistente dipende da questo comportamento, e "correggerlo" romperebbe quei programmi. Per verificare se un valore è `null` si usa un confronto diretto (`val === null`) oppure, quando si deve anche usare `typeof`, la condizione composta `(!val && typeof val === "object")` — che sfrutta il fatto che `null` è falsy, mentre tutti gli oggetti veri sono truthy.

</details>

<details>
<summary>Qual è la differenza tra `undefined` e "undeclared", e perché `typeof` le tratta allo stesso modo?</summary>

`undefined` è un valore valido che una variabile dichiarata può contenere: significa "dichiarata ma senza valore assegnato". Una variabile "undeclared" è una variabile che non è mai stata dichiarata nello scope accessibile — accedervi direttamente lancia `ReferenceError`. `typeof` restituisce `"undefined"` per entrambe i casi: è una scelta progettuale (o un difetto) che evita un errore nel caso undeclared, fornendo un "safety guard". La confusione è aggravata dai messaggi di errore dei browser ("b is not defined") che usano terminologia fuorviante.

</details>

<details>
<summary>Quando e perché usare il safety guard di `typeof` per verificare l'esistenza di una variabile?</summary>

Si usa quando si vuole controllare se una variabile o feature esiste senza sapere se è stata dichiarata — tipicamente in: feature detection per polyfill (`if (typeof atob === "undefined")`), librerie da includere in contesti sconosciuti dove certe dipendenze potrebbero mancare, script browser che condividono il namespace globale. L'alternativa `window.varName` funziona solo in browser; il safety guard `typeof` è universale (funziona anche in Node.js e altri ambienti). La dependency injection (passare la dipendenza come parametro) è un'alternativa più esplicita e testabile.

</details>

<details>
<summary>Le funzioni e gli array sono tipi distinti in JavaScript?</summary>

No. Sia le funzioni che gli array sono **sottotipi di `object`**: `typeof []` e `typeof function(){}` restituiscono rispettivamente `"object"` e `"function"` — ma le funzioni sono oggetti con la proprietà interna `[[Call]]` che le rende invocabili, e gli array sono oggetti con indici numerici e `.length` automatico. `typeof` restituisce `"function"` per le funzioni (non `"object"`) come convenienza, ma la specifica le classifica come sottotipo di object. Per verificare se qualcosa è un array si usa `Array.isArray(val)`, non `typeof`.

</details>
