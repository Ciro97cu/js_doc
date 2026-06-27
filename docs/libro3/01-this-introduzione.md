# this: introduzione

`this` è probabilmente il meccanismo più frainteso di JavaScript. È una keyword speciale definita automaticamente nello scope di ogni funzione, ma a cosa fa riferimento dipende da regole che non sono intuitive — e che molti sviluppatori evitano di capire davvero, preferendo aggirare il problema.

Questo capitolo demolisce le due false credenze più comuni su `this` e stabilisce il principio fondamentale su cui si costruisce tutto il capitolo successivo.

## Perché esiste `this`?

`this` esiste per permettere alle funzioni di essere **riutilizzate con contesti diversi** senza dover passare esplicitamente un riferimento all'oggetto corrente.

```js
function identify() {
    return this.name.toUpperCase();
}

function speak() {
    var greeting = "Ciao, sono " + identify.call(this);
    console.log(greeting);
}

var me = { name: "Kyle" };
var you = { name: "Reader" };

identify.call(me);  // "KYLE"
identify.call(you); // "READER"
speak.call(me);     // "Ciao, sono KYLE"
speak.call(you);    // "Ciao, sono READER"
```

Le stesse funzioni `identify` e `speak` vengono usate con due oggetti diversi senza duplicare il codice. L'alternativa è passare il contesto come parametro esplicito:

```js
function identify(context) {
    return context.name.toUpperCase();
}
function speak(context) {
    var greeting = "Ciao, sono " + identify(context);
    console.log(greeting);
}
identify(you); // "READER"
speak(me);     // "Ciao, sono KYLE"
```

Questa alternativa funziona, ma diventa verbosa e scomoda quando il pattern si complica — in particolare con prototype chain e sistemi di oggetti. `this` fornisce un modo implicito ed elegante di trasportare il contesto.

---

## Le due false credenze

### Falsa credenza 1 — `this` riferisce alla funzione stessa

L'interpretazione grammaticale più immediata di `this` all'interno di una funzione porta a pensare che faccia riferimento alla funzione stessa. Sbagliato.

Si consideri un tentativo di usare `this` per contare quante volte una funzione viene chiamata:

```js
function foo(num) {
    console.log("foo: " + num);
    this.count++; // ERRORE: this non è foo
}

foo.count = 0;
for (var i = 0; i < 10; i++) {
    if (i > 5) foo(i);
}
// foo: 6, foo: 7, foo: 8, foo: 9

console.log(foo.count); // 0 — non è cambiata!
```

`foo.count` rimane `0`. Dentro la funzione, `this.count` non punta a `foo.count`: punta a un `count` di un altro oggetto — in questo caso una variabile globale `count` creata accidentalmente (con valore `NaN`, perché `undefined++` è `NaN`).

La risposta sbagliata è evitare il problema usando una variabile esterna:

```js
var data = { count: 0 };
function foo(num) {
    console.log("foo: " + num);
    data.count++; // funziona, ma bypassa il problema invece di capirlo
}
```

Questo funziona, ma non è comprensione — è elusione. Un'alternativa corretta è riferirsi alla funzione tramite il suo **nome lessicale**:

```js
function foo(num) {
    console.log("foo: " + num);
    foo.count++; // riferimento esplicito all'oggetto funzione via nome
}
foo.count = 0;
```

Oppure, se si vuole davvero capire `this`, usarlo correttamente con `call()`:

```js
function foo(num) {
    console.log("foo: " + num);
    this.count++;
}
foo.count = 0;
for (var i = 0; i < 10; i++) {
    if (i > 5) {
        foo.call(foo, i); // forza this a essere foo — corretto
    }
}
console.log(foo.count); // 4
```

### Falsa credenza 2 — `this` riferisce allo scope lessicale della funzione

Il secondo malinteso è che `this` permetta di accedere alle variabili locali della funzione, come se fosse un ponte verso il lexical scope. Non è così.

Lo scope lessicale è interno all'engine — non è un oggetto accessibile dal codice JavaScript.

```js
function foo() {
    var a = 2;
    this.bar(); // tenta di usare this per raggiungere bar
}
function bar() {
    console.log(this.a); // tenta di raggiungere `a` di foo tramite this
}
foo(); // ReferenceError: a is not defined
```

Il codice sbaglia in due punti. `this.bar()` funziona per caso (ne vedremo il motivo nel Cap 2), ma il vero errore è nell'assunzione: non esiste un "bridge" tra gli scope lessicali di `foo` e `bar` tramite `this`. **Non è possibile usare `this` per fare lookup nel lexical scope.** Ogni volta che si sente l'impulso di farlo, è il segnale che si sta confondendo due meccanismi distinti.

---

## Cos'è davvero `this`?

`this` **non** è determinato da dove la funzione è dichiarata — è determinato da **come la funzione viene chiamata**.

`this` è un **runtime binding**: viene creato nel momento in cui la funzione viene invocata, come parte dell'**execution context** (contesto di esecuzione — il record che l'engine crea per ogni invocazione di funzione, contenente informazioni sul call-site, i parametri, e il valore di `this`). Il valore di `this` per tutta la durata dell'esecuzione dipende esclusivamente dal **call-site** (il punto nel codice da cui la funzione viene chiamata).

Questo è il principio che regola tutto: non dove la funzione è scritta, ma come e dove viene chiamata.

Il capitolo successivo mostrerà le quattro regole che determinano quale valore assume `this` in base al call-site.

---

## ⚡ Ripasso veloce

`this` **non** è:
- un riferimento alla funzione stessa
- un riferimento allo scope lessicale della funzione

`this` **è** un binding creato a runtime nel momento dell'invocazione. Il valore dipende interamente dal **call-site** — da come la funzione viene chiamata.

```js
function foo() {
    console.log(this.a);
}

var obj = { a: 2, foo: foo };

obj.foo(); // 2 — this è obj, perché foo è chiamata come metodo di obj
```

Dichiarare la funzione non dice nulla su `this`. Solo l'invocazione conta.

---

## Domande

<details>
<summary>Perché `this` non si riferisce alla funzione stessa, anche se grammaticalmente sembra logico?</summary>

In inglese (e in italiano) il pronome "this" usato all'interno di un discorso si riferisce normalmente al soggetto. Ma in JavaScript `this` non è un riferimento lessicale alla funzione in cui appare — è un binding creato a runtime basato sul call-site. Quando si scrive `this.count++` dentro `foo`, `this` non è `foo`: è un oggetto determinato da come `foo` viene invocata. Se `foo` viene chiamata come funzione semplice (`foo()`), `this` in non-strict mode è il global object — e `this.count++` modifica (o crea) una variabile globale `count`, non `foo.count`. Per riferirsi alla funzione dall'interno si usa il suo nome lessicale (`foo.count++`) oppure si controlla esplicitamente il binding con `call(foo, ...)`.

</details>

<details>
<summary>Perché non è possibile usare `this` come bridge tra scope lessicali?</summary>

Lo scope lessicale è una struttura interna del JavaScript engine — non è un oggetto accessibile dal codice. `this` è un binding a runtime che punta a un oggetto: può accedere alle proprietà di quell'oggetto, ma non alle variabili locali di nessuna funzione. Tentare di usare `this.bar()` per raggiungere una funzione e poi `this.a` per leggere una variabile locale della funzione chiamante confonde due meccanismi completamente distinti: il lookup lessicale (che segue la struttura di annidamento del codice sorgente) e il `this` binding (che dipende da come le funzioni vengono chiamate a runtime). Non esiste nessun bridge tra i due.

</details>

<details>
<summary>Qual è la differenza fondamentale tra lexical scope e `this` binding?</summary>

Il lexical scope è determinato a **write-time** (compile-time): dipende da dove le funzioni sono dichiarate nel codice sorgente, e rimane fisso per tutta la durata del programma. Il `this` binding è determinato a **runtime**: dipende da come la funzione viene invocata in quel momento specifico. La stessa funzione può avere `this` che punta a oggetti diversi a seconda di come viene chiamata — metodo su un oggetto, funzione libera, con `call()`/`apply()`, con `new`. È questa flessibilità che rende `this` potente — e che lo rende confuso senza una comprensione esplicita delle regole.

</details>

<details>
<summary>Se `this` non è la funzione stessa, come si può riferire una funzione a se stessa dall'interno?</summary>

Ci sono due approcci corretti. Il primo è usare il nome lessicale della funzione come identificatore — se la funzione si chiama `foo`, si scrive `foo.count++` invece di `this.count++`. Questo funziona ovunque la funzione sia accessibile tramite quel nome. Il secondo, nel caso si voglia effettivamente usare `this` per puntare alla funzione, è usare `call()` o `apply()` passando la funzione stessa come primo argomento: `foo.call(foo, argomento)`. In questo modo `this` inside `foo` sarà davvero `foo`. Il pattern `arguments.callee` era un'alternativa storica per le funzioni anonime, ma è deprecato e non va usato.

</details>

<details>
<summary>Cos'è il call-site e perché è l'unica cosa che conta per `this`?</summary>

Il call-site è il punto nel codice da cui una funzione viene invocata — non dove è dichiarata, ma dove compare la chiamata (`foo()`, `obj.foo()`, `foo.call(obj)`, ecc.). Quando l'engine esegue una funzione, crea un execution context che include il valore di `this`. Questo valore viene determinato analizzando il call-site secondo un insieme preciso di regole (che il capitolo 2 esplora in dettaglio). Due invocazioni della stessa funzione con call-site diversi possono produrre valori di `this` completamente diversi. Per capire `this` in un dato contesto si deve guardare dove e come la funzione è stata chiamata, non dove è stata definita.

</details>
