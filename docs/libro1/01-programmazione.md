# Introduzione alla programmazione

Un programma — detto anche source code o semplicemente codice — è un insieme di istruzioni che dicono al computer quali operazioni eseguire. In JavaScript è possibile scrivere codice direttamente nella console di sviluppo del browser, senza bisogno di file separati.

Le regole che definiscono il formato valido delle istruzioni si chiamano **sintassi** del linguaggio, esattamente come la grammatica definisce la struttura corretta di una frase in italiano.

## Statement ed espressioni

Un **statement** (istruzione) è un gruppo di parole, numeri e operatori che compie un'azione specifica. In JavaScript si presenta così:

```js
a = b * 2;
```

`a` e `b` sono variabili (contenitori in cui memorizzare valori). Il `2` è un valore letterale. `=` e `*` sono operatori. La maggior parte degli statement termina con `;`.

Gli statement sono composti da una o più **espressioni**. Un'espressione è qualsiasi riferimento a una variabile, a un valore, o a una combinazione di variabili, valori e operatori. Nell'istruzione precedente ci sono quattro espressioni:

- `2` — espressione letterale
- `b` — espressione variabile (recupera il valore corrente)
- `b * 2` — espressione aritmetica
- `a = b * 2` — espressione di assegnamento

## Esecuzione di un programma

Il codice scritto dagli sviluppatori non è direttamente comprensibile dal computer. Un componente speciale — chiamato **interpreter** (interprete) o **compiler** (compilatore) — traduce il codice in istruzioni macchina.

JavaScript è spesso descritto come linguaggio interpretato, ma non è del tutto preciso: il JavaScript engine (JavaScript engine, cioè il componente del browser o dell'ambiente di esecuzione che gestisce il codice) lo compila appena prima di eseguirlo, in un processo chiamato **just-in-time compilation**. Il risultato è lo stesso di una traduzione riga per riga, ma con prestazioni significativamente migliori.

## Output e input

Il modo più comune per produrre output durante lo sviluppo è `console.log()`:

```js
var b = 21;
console.log(b); // 21
```

`console` è un oggetto messo a disposizione dall'ambiente di esecuzione; `log` è una funzione (detta **metodo**) di quell'oggetto.

Per raccogliere input in modo rapido — utile in fase di apprendimento — esiste `prompt()`:

```js
var eta = prompt("Quanti anni hai?");
console.log(eta);
```

## Operatori

Gli operatori permettono di compiere azioni su variabili e valori. I più usati in JavaScript:

| Categoria | Operatori | Esempio |
|---|---|---|
| Assegnamento | `=` | `a = 2` |
| Aritmetici | `+` `-` `*` `/` | `a * 3` |
| Assegnamento composto | `+=` `-=` `*=` `/=` | `a += 2` |
| Incremento/decremento | `++` `--` | `a++` |
| Accesso a proprietà | `.` | `console.log()` |
| Uguaglianza | `==` `===` `!=` `!==` | `a == b` |
| Confronto | `<` `>` `<=` `>=` | `a <= b` |
| Logici | `&&` `\|\|` | `a \|\| b` |

## Valori e tipi

I valori presenti direttamente nel codice sorgente si chiamano **literal** (letterali). JavaScript ha tipi primitivi integrati:

- `number` — numeri, usati per operazioni matematiche (`42`, `3.14`)
- `string` — testo, delimitato da `"..."` o `'...'` (`"ciao"`)
- `boolean` — vero o falso (`true`, `false`)

```js
"Sono una stringa";
42;
true;
```

### Coercizione di tipo

Quando si confrontano o si combinano valori di tipo diverso, JavaScript può convertirli automaticamente: questo meccanismo si chiama **coercizione implicita** (implicit coercion).

```js
var a = "42";
var b = Number(a); // coercizione esplicita: stringa → numero
console.log(a);    // "42"
console.log(b);    // 42

"99.99" == 99.99   // true — JS converte la stringa in numero
```

La coercizione implicita è una caratteristica del linguaggio, non un difetto: una volta comprese le regole che la governano, diventa uno strumento prevedibile. Il Libro IV di questa serie la analizza in dettaglio.

## Commenti

I commenti sono porzioni di testo ignorate dall'engine, scritte esclusivamente per chi legge il codice. In JavaScript esistono due forme:

```js
// commento su singola riga

/* commento
   su più righe */

var a = /* valore arbitrario */ 42;
```

Un buon commento spiega il **perché** di una scelta, non cosa fa il codice (che dovrebbe già comunicarlo con nomi chiari).

## Variabili

Una variabile è un contenitore simbolico che può cambiare valore nel tempo — da qui il nome. Si dichiara con `var` (o con `let`/`const` da ES6):

```js
var amount = 99.99;
amount = amount * 2;
console.log(amount);           // 199.98

amount = "$" + String(amount);
console.log(amount);           // "$199.98"
```

Nell'esempio, `amount` contiene prima un numero, poi una stringa. Questo è possibile perché JavaScript usa la **tipizzazione dinamica** (dynamic typing): una variabile può contenere qualsiasi tipo di valore senza vincoli dichiarativi.

### Costanti

Quando si vuole che un valore non cambi durante l'esecuzione, si usa una costante. Per convenzione, le costanti dichiarate con `var` si scrivono in MAIUSCOLO con underscore. Da ES6 in poi, la keyword `const` impedisce la riassegnazione:

```js
const TAX_RATE = 0.08;  // 8% - tentare di riassegnarla causa un errore

var amount = 99.99;
amount = amount + (amount * TAX_RATE);
console.log(amount.toFixed(2)); // "107.99"
```

`toFixed(2)` è un metodo disponibile sui valori numerici che restituisce una stringa con il numero arrotondato a due decimali.

## Blocchi

Un blocco raggruppa più statement tra parentesi graffe `{ }`. I blocchi non richiedono `;` alla fine e compaiono quasi sempre associati a un'altra struttura di controllo:

```js
var amount = 99.99;

if (amount > 10) {
    amount = amount * 2;
    console.log(amount); // 199.98
}
```

## Condizionali

La struttura `if` esegue un blocco solo se la condizione tra parentesi è vera (truthy). Si può aggiungere un ramo `else` per il caso contrario:

```js
var saldo = 302.13;
var importo = 199.98;

if (importo < saldo) {
    console.log("Acquisto confermato!");
} else {
    console.log("Saldo insufficiente.");
}
```

Quando la condizione non è già un booleano, JavaScript la converte automaticamente: i valori `0`, `""`, `null`, `undefined`, `NaN` e `false` diventano `false` (detti **falsy**); tutti gli altri diventano `true` (**truthy**).

## Cicli

I cicli (loop) ripetono un blocco di statement finché una condizione resta vera. Le forme principali sono `while`, `do..while` e `for`:

```js
/* while: la condizione viene verificata prima di ogni iterazione */
var i = 0;
while (i <= 9) {
    console.log(i);
    i = i + 1;
}
// 0 1 2 3 4 5 6 7 8 9

/* for: forma compatta con inizializzazione, condizione e aggiornamento */
for (var j = 0; j <= 9; j = j + 1) {
    console.log(j);
}
// 0 1 2 3 4 5 6 7 8 9
```

Il `for` è preferibile quando si conta un intervallo preciso: raggruppa inizializzazione, condizione e aggiornamento nella stessa riga, rendendo il codice più leggibile. L'istruzione `break` interrompe anticipatamente qualsiasi ciclo.

## Funzioni

Una funzione è un blocco di codice con un nome, che può essere richiamato più volte. Può ricevere argomenti (parametri) e restituire un valore:

```js
function calcolaTassa(importo) {
    return importo * 0.08;
}

function formattaImporto(importo) {
    return "$" + importo.toFixed(2);
}

var totale = 99.99;
totale = totale + calcolaTassa(totale);
console.log(formattaImporto(totale)); // "$107.99"
```

Le funzioni non servono solo quando si deve ripetere del codice: organizzare statement correlati in una funzione con un nome significativo rende il codice più leggibile anche se viene chiamata una sola volta.

### Scope

Lo **scope** (ambito) è l'insieme delle variabili accessibili in un determinato punto del codice. In JavaScript ogni funzione crea il proprio scope: le variabili dichiarate al suo interno non sono visibili dall'esterno.

```js
function unaFunzione() {
    var a = 1;
    console.log(a); // 1
}

function altraFunzione() {
    var a = 2;      // è una variabile diversa: stesso nome, scope diverso
    console.log(a); // 2
}

unaFunzione();
altraFunzione();
```

Gli scope possono essere annidati: il codice in uno scope interno ha accesso alle variabili degli scope esterni, ma non viceversa.

```js
function esterna() {
    var a = 1;

    function interna() {
        var b = 2;
        console.log(a + b); // 3 — accede sia a `a` che a `b`
    }

    interna();
    console.log(a);         // 1 — non ha accesso a `b`
}

esterna();
```

Questa caratteristica — detta **lexical scope** — è il meccanismo alla base delle closure, trattate nel Libro II.

---

## ⚡ Ripasso veloce

**Statement ed espressioni**: un statement è un'istruzione completa (`a = b * 2;`); un'espressione è qualsiasi parte che produce un valore (`b * 2`, `a`).

**Engine e compilazione**: l'engine JavaScript compila il codice just-in-time prima di eseguirlo — non lo interpreta riga per riga.

**Tipi primitivi**: `number`, `string`, `boolean` sono i tipi di base. I valori senza variabile sono detti literal.

**Coercizione**: JavaScript converte automaticamente i tipi quando necessario (implicit coercion). `==` la permette; `===` la impedisce.

```js
"5" == 5    // true  (coercizione implicita)
"5" === 5   // false (nessuna coercizione)
```

**`const` vs `var`**: `const` impedisce la riassegnazione e segnala l'intenzione di non modificare il valore. `var` è permissivo.

**Scope**: ogni funzione crea il proprio scope. Il codice interno può accedere alle variabili degli scope più esterni; il contrario non è possibile.

```js
function esterna() {
    var x = 10;
    function interna() {
        console.log(x); // 10 — accesso allo scope esterno
    }
    interna();
}
```

---

## Domande

<details>
<summary>Qual è la differenza tra statement ed espressione?</summary>

Un statement è un'istruzione completa che compie un'azione (es. `a = b * 2;`). Un'espressione è qualsiasi porzione di codice che produce un valore: può essere una variabile (`b`), un literal (`2`), un'operazione aritmetica (`b * 2`) o un assegnamento (`a = b * 2`). Uno statement è tipicamente composto da una o più espressioni.

</details>

<details>
<summary>In che senso JavaScript "compila" il codice se viene definito un linguaggio interpretato?</summary>

La distinzione tradizionale tra linguaggi interpretati e compilati non si applica bene a JavaScript. Il JavaScript engine (come V8 di Chrome) non esegue il codice riga per riga: lo analizza, lo compila in bytecode ottimizzato e poi lo esegue — tutto poco prima dell'esecuzione effettiva (just-in-time compilation). Il risultato visibile è simile all'interpretazione, ma le prestazioni sono molto superiori.

</details>

<details>
<summary>Cos'è la coercizione implicita e quando si attiva?</summary>

La coercizione implicita è la conversione automatica di un valore da un tipo a un altro, attivata da JavaScript quando un'operazione richiede un tipo diverso da quello fornito. Ad esempio, `"99.99" == 99.99` converte la stringa in numero prima di confrontarli. Si attiva con l'operatore `==`, nelle condizioni degli `if` (dove i valori vengono convertiti in boolean), e in alcune operazioni aritmetiche con stringhe.

</details>

<details>
<summary>Qual è la differenza tra `var`, `let` e `const`?</summary>

`var` dichiara una variabile con scope di funzione e può essere riassegnata. `let` (introdotto con ES6) dichiara una variabile con scope di blocco e può essere riassegnata. `const` (introdotto con ES6) dichiara un binding con scope di blocco che non può essere riassegnato dopo la dichiarazione — se si tenta la riassegnazione in strict mode viene generato un errore. `const` non rende il valore immutabile: se il valore è un oggetto, le sue proprietà possono comunque essere modificate.

</details>

<details>
<summary>Perché il codice all'interno di una funzione non è accessibile dall'esterno?</summary>

Perché JavaScript usa il lexical scope: ogni funzione crea il proprio scope, che racchiude le variabili dichiarate al suo interno. Il codice esterno non ha accesso a quelle variabili. Il contrario, invece, è possibile: il codice all'interno di una funzione ha accesso alle variabili dichiarate negli scope più esterni (scope annidati). Questo meccanismo è alla base delle closure.

</details>
