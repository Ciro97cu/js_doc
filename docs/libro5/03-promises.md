# Promise

Le Promise risolvono i due problemi fondamentali dei callback: la difficoltà cognitiva del flusso non-lineare e l'inversion of control. Anziché passare la continuation a una terza parte sperando che la chiami correttamente, si riceve indietro un oggetto che rappresenta il completamento futuro dell'operazione — e si decide poi, autonomamente, cosa fare con quel risultato.

---

## Cos'è una Promise

Una Promise è un **future value** — un segnaposto per un valore che non è ancora disponibile ma lo sarà. Come lo scontrino al ristorante fast-food: non è il panino, ma rappresenta il panino futuro. Si può ragionare su quel valore, pianificare cosa fare quando arriva, e accettare che potrebbe non arrivare (panino esaurito = rejection).

Una Promise ha sempre uno e un solo esito finale:
- **fulfilled** (risolta con un valore)
- **rejected** (rifiutata con un motivo)

Una volta risolta, una Promise è **immutabile** — il suo stato non cambia più.

### Future value e time-independence

Il problema con il codice asincrono è che tratta i valori in modo time-sensitive: si assume che siano già disponibili. Le Promise normalizzano il tempo:

```js
/* senza Promise — il valore potrebbe non essere pronto */
var x, y = 2;
console.log(x + y); /* NaN se x non è ancora disponibile */

/* con Promise — si lavora con future values indipendentemente dal tempo */
function add(xPromise, yPromise) {
    return Promise.all([xPromise, yPromise])
        .then(function(values) {
            return values[0] + values[1];
        });
}

add(fetchX(), fetchY())
    .then(function(sum) {
        console.log(sum); /* 42, quando entrambi i valori sono pronti */
    });
```

### Completion event — uninversion of control

Con i callback si passa la continuation a un'altra parte. Con le Promise si riceve indietro un "evento di completamento" a cui ci si può sottoscrivere:

```js
/* pattern callback — cedere il controllo */
foo(42, myCallback);

/* pattern Promise — ricevere il controllo */
var p = foo(42);
p.then(bar);    /* bar è notificato quando foo completa */
p.then(baz);    /* baz è notificato indipendentemente */
```

La stessa Promise può essere osservata da parti diverse del codice — senza che si interferiscano tra loro. Questo è l'**uninversion of control**: il controllo torna al chiamante.

### Il costruttore revealing constructor

```js
var p = new Promise(function(resolve, reject) {
    /* logica asincrona... */
    setTimeout(function() {
        resolve(42);   /* fulfillment con valore 42 */
        /* oppure: reject("motivo dell'errore") */
    }, 1000);
});

p.then(
    function fulfilled(val) { console.log(val); },  /* 42 */
    function rejected(err)  { console.error(err); }
);
```

La funzione passata al costruttore viene eseguita **immediatamente** (non in modo async). `resolve()` e `reject()` sono le funzioni di risoluzione. La differenza: `reject()` rifiuta sempre; `resolve()` può fare entrambe — se gli si passa una Promise o thenable, ne adotta lo stato.

---

## Thenable duck typing

Qualsiasi oggetto con un metodo `.then()` viene considerato un **thenable** dal sistema delle Promise. Non è necessario che sia `instanceof Promise`.

```js
/* duck typing check usato internamente */
if (p !== null && (typeof p === "object" || typeof p === "function") && typeof p.then === "function") {
    /* trattato come thenable */
}
```

Il lato oscuro: se un oggetto ha accidentalmente `.then()` (per esempio tramite prototype chain), verrà trattato come thenable con conseguenze imprevedibili. `Promise.resolve()` normalizza qualsiasi thenable in una Promise affidabile.

---

## Promise Trust — le garanzie built-in

Le Promise risolvono per design i 5 problemi dell'inversion of control:

### 1. Mai chiamato troppo presto (no Zalgo)

I callback registrati con `.then()` vengono **sempre** chiamati in modo asincrono — anche se la Promise è già risolta al momento della registrazione. Non esiste un `.then()` sincrono. Le Promise usano la Job queue (Cap. 1), quindi il callback parte al prossimo tick.

```js
var p = new Promise(function(resolve) { resolve(42); });
p.then(function(v) { console.log(v); }); /* asincrono — mai sincrono */
console.log("questo stampa prima di 42");
```

### 2. Mai chiamato troppo tardi

Quando `resolve()` o `reject()` vengono chiamati, tutti i `.then()` registrati vengono schedulati nella Job queue nell'ordine in cui sono stati registrati. Nessuna catena sincrona può interferire con l'ordine di invocazione. L'output `A B C` nell'esempio seguente è garantito:

```js
p.then(function() {
    p.then(function() { console.log("C"); });
    console.log("A");
});
p.then(function() { console.log("B"); });
/* A B C — C non può precedere B */
```

### 3. Mai non chiamato

Se sia il fulfillment handler sia il rejection handler sono registrati, uno dei due viene sempre chiamato (anche in caso di errore JS all'interno della Promise). Per il caso "Promise che non si risolve mai", si usa `Promise.race()` con un timeout:

```js
function timeoutPromise(delay) {
    return new Promise(function(resolve, reject) {
        setTimeout(function() { reject("Timeout!"); }, delay);
    });
}

Promise.race([
    foo(),                   /* l'operazione vera */
    timeoutPromise(3000)     /* timeout di 3 secondi */
]).then(fulfilled, rejected);
```

### 4. Chiamato esattamente una volta

Una Promise può essere risolta **una sola volta**. Se `resolve()` o `reject()` vengono chiamati più volte, solo la prima ha effetto — le successive vengono silenziosamente ignorate.

### 5. Errori non inghiottiti

Le eccezioni JS lanciate all'interno del costruttore della Promise vengono intercettate e convertite in rejection:

```js
var p = new Promise(function(resolve, reject) {
    foo.bar(); /* foo non esiste — TypeError */
    resolve(42); /* mai raggiunto */
});

p.then(
    function fulfilled() { /* mai raggiunto */ },
    function rejected(err) { console.log(err); /* TypeError */ }
);
```

Se l'eccezione avviene in un fulfillment handler, non viene "persa" — finisce come rejection sulla Promise successiva della catena.

### Trustable Promise — `Promise.resolve()`

`Promise.resolve()` è l'idioma per normalizzare qualsiasi valore (immediato, thenable, o Promise) in una Promise affidabile:

```js
Promise.resolve(42);        /* Promise fulfilled con 42 */
Promise.resolve(p);         /* se p è già una Promise, la ritorna identica */
Promise.resolve(thenable);  /* unwrappa il thenable fino al valore finale */
```

Avvolgere `Promise.resolve()` attorno al return di qualsiasi funzione è una best practice per eliminare il nondeterminismo sync/async:

```js
/* invece di: */
foo(42).then(handler);

/* preferire: */
Promise.resolve(foo(42)).then(handler);
```

---

## Chain flow — catena di Promise

Ogni `.then()` restituisce una **nuova Promise**. Questo permette di costruire catene asincrone:

```js
Promise.resolve(21)
    .then(function(v) {
        return v * 2;        /* fulfilla la prossima Promise con 42 */
    })
    .then(function(v) {
        console.log(v);      /* 42 */
    });
```

Per introdurre asincronia in un passaggio della catena, si restituisce una Promise dal handler — essa viene **unwrappata** automaticamente, e la catena aspetta la sua risoluzione:

```js
Promise.resolve(21)
    .then(function(v) {
        return new Promise(function(resolve) {
            setTimeout(function() { resolve(v * 2); }, 100);
        });
    })
    .then(function(v) {
        console.log(v); /* 42 — dopo 100ms */
    });
```

Un caso pratico con richieste Ajax sequenziali:

```js
function request(url) {
    return new Promise(function(resolve, reject) {
        ajax(url, resolve);
    });
}

request("http://url.1/")
    .then(function(response1) {
        return request("http://url.2/?v=" + response1);
    })
    .then(function(response2) {
        console.log(response2);
    });
```

### Propagazione degli errori nella catena

Gli errori propagano lungo la catena fino al primo rejection handler. Un rejection handler che restituisce un valore (non rilancia) "resetta" la catena allo stato di fulfillment:

```js
request("http://url.1/")
    .then(function(response1) {
        foo.bar();    /* TypeError — salta allo step successivo con rejection */
        return request("http://url.2/?v=" + response1);
    })
    .then(
        function fulfilled(response2) { /* mai raggiunto */ },
        function rejected(err) {
            console.log(err); /* TypeError */
            return 42;        /* resetta la catena: il prossimo .then riceve 42 */
        }
    )
    .then(function(msg) {
        console.log(msg); /* 42 */
    });
```

`catch(handler)` è shorthand per `.then(null, handler)`.

### Terminologia: resolve, fulfill, reject

- **`resolve()`** nel costruttore: può portare a fulfillment o rejection (se gli si passa una Promise rejected). È corretto chiamarlo `resolve`.
- **`reject()`** nel costruttore: rifiuta sempre, non fa unwrap.
- **`fulfilled`/`rejected`**: i nomi corretti per i callback di `.then()`.
- **`Promise.resolve()`** e **`Promise.reject()`**: shorthand statici.

---

## Gestione degli errori

`try/catch` non funziona con codice asincrono:

```js
function foo() {
    setTimeout(function() { baz.bar(); }, 100); /* errore asincrono */
}
try {
    foo(); /* non cattura l'errore — avviene dopo */
} catch(err) { /* mai eseguito */ }
```

Le Promise usano la separazione in due handler (`.then(fulfilled, rejected)` o `.catch()`). Il problema è il **pit of despair**: gli errori si inghiottono silenziosamente se nessuno osserva la Promise risultante dal `.then()`.

```js
var p = Promise.resolve(42);
p.then(function fulfilled(msg) {
    console.log(msg.toLowerCase()); /* TypeError — msg è number */
    /* l'errore va nella Promise restituita da .then(), non in questo rejected */
}, function rejected(err) {
    /* mai eseguito — questo handler è per p, già fulfilled a 42 */
});
```

La best practice è terminare ogni catena con `.catch()`:

```js
somePromiseChain
    .then(step1)
    .then(step2)
    .catch(handleErrors); /* cattura qualsiasi errore lungo la catena */
```

Il limite: se `handleErrors` lancia a sua volta, quell'errore non viene catchato. Non esiste una soluzione perfetta in ES6 — alcuni browser riportano le Promise rejected non osservate nella console tramite garbage collection.

---

## Pattern con le Promise

### `Promise.all([..])` — gate

Attende che **tutte** le Promise siano fulfilled. Se una sola viene rejected, la Promise principale viene subito rejected (scartando tutte le altre).

```js
var p1 = request("http://url.1/");
var p2 = request("http://url.2/");

Promise.all([p1, p2])
    .then(function(msgs) {
        /* msgs[0] = risultato di p1, msgs[1] = risultato di p2 */
        return request("http://url.3/?v=" + msgs.join(","));
    })
    .then(function(msg) { console.log(msg); })
    .catch(handleErrors);
```

Array vuoto → si fulfilla immediatamente.

### `Promise.race([..])` — latch

La **prima** Promise che si risolve (fulfilled o rejected) determina l'esito. Le altre vengono ignorate (ma continuano a girare in background — le Promise non si cancellano).

```js
Promise.race([
    foo(),
    timeoutPromise(3000)
]).then(fulfilled, rejected);
```

Array vuoto → non si risolve mai (footgun da evitare).

### Pattern aggiuntivi (non ES6 native)

| Pattern | Semantica |
|---|---|
| `none([..])` | Come `all`, ma con fulfillment e rejection invertiti |
| `any([..])` | Si fulfilla al primo fulfilled, ignora i rejected |
| `first([..])` | Come race, ma solo sul primo fulfilled |
| `last([..])` | Solo l'ultimo fulfilled vince |

---

## API di riferimento

```js
/* costruttore */
var p = new Promise(function(resolve, reject) {
    /* resolve(value) oppure reject(reason) */
});

/* shorthand statici */
var p1 = Promise.resolve(42);        /* Promise già fulfilled */
var p2 = Promise.reject("motivo");   /* Promise già rejected */
var p3 = Promise.resolve(thenable);  /* normalizza thenable */

/* metodi sull'istanza */
p.then(onFulfilled, onRejected); /* restituisce nuova Promise */
p.catch(onRejected);             /* = p.then(null, onRejected) */

/* statici di coordinazione */
Promise.all([p1, p2, p3]);       /* gate — attende tutti */
Promise.race([p1, p2, p3]);      /* latch — primo che vince */
```

---

## Limiti delle Promise

**Errori in catena silenziosamente inghiottiti**: senza un `.catch()` finale, gli errori propagano in silenzio lungo tutta la catena.

**Single value**: ogni Promise porta un solo valore. Per più valori, si usa un array/oggetto, oppure si espongono più Promise separate + `Promise.all()`.

**Single resolution**: una Promise si risolve una sola volta — non è adatta a eventi ripetuti (click, stream). Per eventi multipli si crea una nuova Promise chain per ogni evento.

**Inertia — promisifying**: il codice legacy con callback deve essere avvolto:

```js
/* Promise.wrap = promisify helper */
if (!Promise.wrap) {
    Promise.wrap = function(fn) {
        return function() {
            var args = [].slice.call(arguments);
            return new Promise(function(resolve, reject) {
                fn.apply(null, args.concat(function(err, v) {
                    if (err) reject(err);
                    else resolve(v);
                }));
            });
        };
    };
}

var request = Promise.wrap(ajax); /* promisory di ajax */
request("http://url.1/").then(handler);
```

**Non cancellabili**: una Promise non può essere cancellata dall'esterno — per design, per preservare l'immutabilità. La cancellazione appartiene a un livello di astrazione superiore (es. sequenza di Promise).

**Performance**: le Promise sono leggermente più lente dei callback nudi (overhead della Job queue, dei trust check, dell'immutabilità). Il tradeoff è quasi sempre vantaggioso — ottimizzare via callback solo in hot path misurati.

---

## ⚡ Ripasso veloce

**Promise = future value immutabile**: si risolve una sola volta (fulfilled o rejected), il suo stato non cambia più.

**Uninversion of control**: invece di cedere il callback, si riceve una Promise. Chi la osserva decide come reagire.

**5 garanzie built-in**: sempre async (no Zalgo), ordine preservato, errori non persi, chiamata esatta una volta, gestione eccezioni automatica.

**Chain flow**: ogni `.then()` restituisce nuova Promise. Restituire un valore la fulfilla; restituire una Promise la unwrappa (aspetta). Gli errori propagano fino al primo rejection handler.

**`Promise.all`** = gate (tutti devono passare). **`Promise.race`** = latch (primo che passa vince).

**`Promise.resolve(x)`**: normalizza qualsiasi valore in Promise affidabile.

```js
/* costruzione */
var p = new Promise((resolve, reject) => {
    setTimeout(() => resolve(42), 100);
});

/* catena */
p.then(v => v * 2)
 .then(v => console.log(v)) /* 84 */
 .catch(err => console.error(err));

/* gate */
Promise.all([p1, p2]).then(([v1, v2]) => console.log(v1, v2));

/* latch + timeout */
Promise.race([p, timeoutPromise(3000)]).then(ok, fail);

/* promisify */
var request = Promise.wrap(ajax);
request("/api/data").then(handler);
```

---

## Domande

<details>
<summary>Cos'è l'uninversion of control e perché le Promise la realizzano?</summary>

Con i callback si cede il controllo sulla continuation del programma a una terza parte — quella terza parte decide quando e quante volte chiamare il callback. L'inversion of control è il problema di fiducia che ne deriva. Le Promise invertono questa relazione: invece di passare un callback, si riceve indietro un oggetto (la Promise) che rappresenta il completamento futuro. Il codice che reagisce al risultato (`.then()`, `.catch()`) viene registrato sulla Promise stessa, non ceduto all'operazione esterna. La terza parte risolve (o rifiuta) la Promise, ma è il chiamante a decidere cosa farne — e le garanzie built-in assicurano che avvenga in modo predictabile.

</details>

<details>
<summary>Perché i callback registrati con `.then()` sono sempre asincroni, anche se la Promise è già risolta?</summary>

È una garanzia fondamentale del design delle Promise per prevenire il problema Zalgo (comportamento sync-or-async nondeterministico, vedi Cap. 2). Anche se si chiama `.then()` su una Promise già fulfilled, il callback viene sempre messo nella Job queue e eseguito al prossimo tick — mai in modo sincrono. Questo rende il comportamento uniforme e prevedibile indipendentemente dallo stato corrente della Promise. Non servono più `setTimeout(fn, 0)` per forzare l'asincronia.

</details>

<details>
<summary>Cosa succede quando si restituisce una Promise da un handler `.then()`?</summary>

Quando il fulfillment (o rejection) handler di un `.then()` restituisce una Promise, quella Promise viene **unwrappata** automaticamente: la Promise successiva nella catena adotta il suo stato e valore finali. Se la Promise restituita è pending, la catena si "ferma" ad aspettare la sua risoluzione prima di procedere. Se è già resolved, il prossimo `.then()` viene notificato al prossimo tick. Questo meccanismo permette di costruire catene di operazioni async sequenziali senza annidare Promise manualmente.

</details>

<details>
<summary>Qual è la differenza tra `Promise.all()` e `Promise.race()`?</summary>

`Promise.all([..])` implementa il pattern **gate**: aspetta che **tutte** le Promise dell'array siano fulfilled prima di fulfillarsi a sua volta, con un array dei valori in ordine di posizione (non di arrivo). Se anche una sola viene rejected, la Promise principale viene subito rejected. `Promise.race([..])` implementa il **latch**: si fulfilla (o rifiuta) non appena la **prima** Promise dell'array si risolve in qualsiasi modo — le altre vengono ignorate. Attenzione: `race([])` con array vuoto non si risolve mai; `all([])` si fulfilla immediatamente.

</details>

<details>
<summary>Perché le Promise non si possono cancellare e perché è una scelta corretta?</summary>

Le Promise sono progettate per essere **immutabili dall'esterno** una volta create: chiunque le osservi riceve lo stesso risultato, senza che nessun osservatore possa interferire con gli altri. Permettere la cancellazione esterna violerebbe questa garanzia — un consumatore potrebbe cancellare una Promise che un altro consumatore sta ancora aspettando, riproducendo i problemi di trust dei callback. La cancellazione è un concetto che appartiene al livello di **sequenza** di operazioni, non alla singola Promise. Librerie come `asynquence` offrono sequenze cancellabili costruite sopra le Promise, mantenendo l'immutabilità della singola Promise.

</details>
