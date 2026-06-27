# Controllo del flusso asincrono

L'asincronia è una realtà ineludibile in JavaScript. I callback sono stati il meccanismo storico, ma soffrono di problemi strutturali — inversione del controllo, difficoltà di composizione, gestione degli errori disomogenea. ES6 aggiunge le **Promise** come soluzione al problema della fiducia, e il pattern **generator + Promise** come soluzione al problema della leggibilità.

> Per un'analisi approfondita di callback, Promise e generator asincroni si rimanda a [Async & Performance — Promise](/docs/libro5/03-promises.md) e [Generator](/docs/libro5/04-generators.md). Questo capitolo ne sintetizza l'essenziale e illustra la loro combinazione.

---

## Promise

Una Promise **non sostituisce** i callback — li gestisce attraverso un intermediario affidabile. Si può pensare a una Promise in tre modi complementari:

- **Event listener** che si attiva esattamente una volta, all'completamento di un task
- **Valore futuro** — un contenitore indipendente dal tempo che, una volta risolto, non cambia più
- **Versione asincrona del valore di ritorno** di una funzione sincrona

Una Promise si risolve in uno di due stati finali, ognuno con un singolo valore:
- **fulfilled** (successo) → valore di fulfillment
- **rejected** (errore) → reason (la motivazione del rifiuto)

Una volta risolta, la Promise è immutabile. Tentativi successivi di risolverla di nuovo vengono ignorati.

### Costruzione e utilizzo

```js
var p = new Promise(function(resolve, reject) {
    /* operazione asincrona */
    if (/* successo */) resolve(value);
    else                 reject(reason);
});

p.then(
    function fulfilled(value) { /* gestisci successo */ },
    function rejected(reason) { /* gestisci errore */ }
);

/* shorthand per il solo handler di errore */
p.catch(function rejected(reason) { /* ... */ });
```

`then()` accetta due callback opzionali. Se uno dei due è omesso o non è una funzione, viene usato un placeholder di default che propaga il valore/errore lungo la catena. `then()` e `catch()` restituiscono sempre una **nuova Promise**, permettendo il chaining.

### Chaining

```js
ajax("url1")
.then(function fulfilled(contents) {
    return contents.toUpperCase();      /* valore immediato → avanza nella catena */
})
.then(function fulfilled(data) {
    return ajax("url2?v=" + data);      /* promise → adottata dalla catena */
})
.then(function fulfilled(result) {
    console.log(result);
})
.catch(function rejected(err) {
    /* cattura qualsiasi errore lungo la catena */
});
```

Un'eccezione nel handler `fulfilled` del primo `then` non viene catturata dall'handler `rejected` dello stesso `then` — viene propagata alla Promise restituita da quel `then`, e da lì al successivo `catch` o handler `rejected`.

### Thenable

Qualsiasi oggetto o funzione con un metodo `then()` viene trattato come un **thenable** — una Promise-like che il sistema Promise tenta di adottare. Il problema: i thenable non sono necessariamente affidabili (possono chiamare il callback più volte, non seguire il contratto, ecc.).

La soluzione è passare qualsiasi valore sospetto attraverso `Promise.resolve()`:

```js
/* normalizza: se thenable → ne adotta lo stato; se valore → lo avvolge in una Promise */
Promise.resolve(suspectValue).then(handler);
```

### Promise API

```js
/* crea una Promise già fulfilled */
var p = Promise.resolve(42);

/* crea una Promise già rejected */
var p = Promise.reject("Oops");

/* gate: fulfilled solo se tutti i valori/Promise si fulfillono */
Promise.all([p1, p2, p3])
.then(function(vals) {
    console.log(vals); // [v1, v2, v3] — nello stesso ordine
});
/* se uno si rigetta, l'intera Promise.all si rigetta immediatamente */

/* latch: fulfilled/rejected al primo che completa */
Promise.race([p1, p2, p3])
.then(function(val) { /* valore del primo a completarsi */ });
/* attenzione: Promise.race([]) non si risolve mai */
```

---

## Generator + Promise

Promise e generator si combinano per produrre codice asincrono che appare sincrono. Il pattern:

1. Il generator esegue un'operazione asincrona e fa `yield` della Promise risultante
2. Un runner esterno riceve la Promise, aspetta la sua risoluzione, e riprende il generator con il valore di fulfillment (o inietta un errore con `it.throw()` in caso di rejection)

```js
function *main() {
    var result1 = yield step1();

    try {
        var result2 = yield step2(result1);
    } catch (err) {
        result2 = yield step2Failed(err);
    }

    /* parallelo con Promise.all */
    var results = yield Promise.all([
        step3a(result2),
        step3b(result2),
        step3c(result2)
    ]);

    yield step4(results);
}
```

Il corpo del generator legge come codice sincrono sequenziale — assegnazioni, `try/catch`, espressioni normali — anche se ogni `yield` maschera un'operazione asincrona.

### Il runner

```js
function run(gen) {
    var args = [].slice.call(arguments, 1), it;
    it = gen.apply(this, args);

    return Promise.resolve()
        .then(function handleNext(value) {
            var next = it.next(value);
            return (function handleResult(next) {
                if (next.done) {
                    return next.value;
                }
                return Promise.resolve(next.value)
                    .then(
                        handleNext,
                        function handleErr(err) {
                            return Promise.resolve(it.throw(err))
                                .then(handleResult);
                        }
                    );
            })(next);
        });
}

run(main)
.then(
    function fulfilled() { /* *main() completato */ },
    function rejected(reason) { /* errore non gestito */ }
);
```

Il runner:
1. Avvia il generator
2. Riceve ogni valore `yield`-ato e lo normalizza con `Promise.resolve()`
3. Quando la Promise si fulfilla, chiama `it.next(value)` per riprendere il generator
4. Quando si rigetta, chiama `it.throw(err)` per iniettare l'errore nel generator
5. Quando `done: true`, la Promise del runner si fulfilla con il valore finale

### async/await — la sintassi nativa

Il pattern generator + Promise è così comune che ES2017 lo ha elevato a sintassi di prima classe:

```js
/* generator + run() */
function *main() {
    var x = yield fetch("url");
    return x;
}
run(main);

/* equivalente con async/await */
async function main() {
    var x = await fetch("url");
    return x;
}
main().then(/* ... */);
```

`async function` è fondamentalmente una versione specializzata di un generator controllato dal proprio runner integrato. `await` è `yield` che sa aspettare specificamente le Promise. Si veda il capitolo [ES6 & Beyond — oltre ES6](/docs/libro6/08-oltre-es6.md) per i dettagli.

---

## ⚡ Ripasso veloce

```js
/* === PROMISE === */
var p = new Promise((resolve, reject) => {
    /* ... resolve(val) o reject(reason) */
});

p.then(onFulfilled, onRejected)
 .then(/* catena */)
 .catch(onError);

Promise.resolve(val);           /* normalizza qualsiasi valore */
Promise.reject(reason);
Promise.all([p1, p2, p3]);      /* gate: tutti devono fulfillare */
Promise.race([p1, p2, p3]);     /* latch: primo che completa vince */

/* === GENERATOR + PROMISE === */
function *workflow() {
    try {
        var a = yield asyncOpA();     /* pausa, aspetta Promise */
        var b = yield asyncOpB(a);
        var [c, d] = yield Promise.all([asyncOpC(), asyncOpD()]);
        return c + d;
    } catch (err) {
        /* errori asincroni catturabili con try/catch */
    }
}

run(workflow)
.then(result => console.log(result))
.catch(err => console.error(err));

/* async/await = zucchero sintattico per il pattern sopra */
async function workflow() {
    var a = await asyncOpA();
    return a;
}
```

**Regola pratica**: con più di due step asincroni in sequenza, usare un generator (o `async/await`) invece di una catena di `.then()`. Il codice sincrono-looking è più leggibile e `try/catch` funziona naturalmente.

---

## Domande

<details>
<summary>In cosa consiste il "problema della fiducia" dei callback che le Promise risolvono?</summary>

Con i callback, si consegna una funzione a una libreria o sistema esterno e si spera che venga chiamata al momento giusto, una sola volta, con gli argomenti corretti. Non esiste un contratto formale che lo garantisca — la libreria potrebbe chiamare il callback più volte, mai, in modo sincrono invece che asincrono, o inghiottire gli errori. Le Promise invertono questo controllo: invece di consegnare un callback, si riceve una Promise che rappresenta il risultato futuro. Il consumatore decide quando e come osservare quel risultato tramite `.then()`. Le Promise garantiscono contrattualmente: risoluzione unica (fulfilled o rejected, mai entrambe), comportamento sempre asincrono (anche se il valore è immediato), propagazione degli errori. `Promise.resolve()` permette di normalizzare anche i thenable non fidati, avvolgendoli in una Promise genuina.

</details>

<details>
<summary>Cosa succede se si lancia un'eccezione dentro il handler <code>fulfilled</code> di un <code>.then()</code>?</summary>

L'eccezione non viene catturata dall'handler `rejected` dello *stesso* `then()` — quel handler risponde solo alla rejection della Promise precedente nella catena. L'eccezione viene invece catturata dalla Promise restituita da quel `then()`, e da lì propagata al successivo `catch()` o all'handler `rejected` del `then()` successivo. In pratica, ogni `then()` produce una nuova Promise che rappresenta il risultato del handler — se il handler lancia, quella Promise viene rigettata con l'eccezione come reason. Questo spiega perché è buona pratica aggiungere un `.catch()` alla fine di ogni catena di Promise: senza, i rejection non osservati vengono silenziosamente ignorati.

</details>

<details>
<summary>Come funziona il runner che combina generator e Promise?</summary>

Il runner esegue il generator chiamando `it.next()` in un ciclo. Ogni volta che il generator cede (`yield`) una Promise, il runner aspetta la risoluzione di quella Promise (tramite `.then()`). Se la Promise si fulfilla, il runner chiama `it.next(value)` con il valore di fulfillment, che diventa il risultato dell'espressione `yield` nel generator. Se si rigetta, il runner chiama `it.throw(err)`, iniettando l'eccezione nel generator nel punto esatto in cui era pausato — dove può essere catturata con un normale `try/catch`. Quando il generator completa (`done: true`), il runner fulfilla la propria Promise con il valore finale. Questo meccanismo è esattamente ciò che `async/await` automatizza nativamente: `async function` è un generator con runner integrato, e `await` è `yield` specializzato per le Promise.

</details>

<details>
<summary>Qual è la differenza tra <code>Promise.all()</code> e <code>Promise.race()</code>?</summary>

`Promise.all([p1, p2, p3])` implementa un **gate**: aspetta che *tutte* le Promise nell'array si fulfillino, poi si fulfilla con un array dei valori nello stesso ordine. Se anche solo una si rigetta, l'intera `Promise.all` si rigetta immediatamente con quella reason, ignorando le restanti. È utile per operazioni parallele dove servono tutti i risultati. `Promise.race([p1, p2, p3])` implementa un **latch**: si fulfilla o si rigetta con il primo valore che completa, ignorando le restanti. È utile per timeout (race tra l'operazione reale e una Promise che si rigetta dopo N millisecondi). Attenzione: `Promise.race([])` con array vuoto non si risolve mai — a differenza di `Promise.all([])` che si fulfilla immediatamente con `[]`.

</details>

<details>
<summary>Perché il codice con generator + Promise è considerato più leggibile di una catena di <code>.then()</code>?</summary>

Una catena di `.then()` costringe a strutturare il codice in frammenti separati (un callback per step), rendendo difficile seguire il flusso logico, condividere variabili tra step, e gestire errori in modo uniforme. Con i generator, il codice asincrono si scrive come sequenza lineare di statement — assegnazioni, condizioni, `try/catch` — esattamente come il codice sincrono. Il `yield` è un dettaglio di implementazione quasi invisibile: non cambia la forma del ragionamento. La gestione degli errori con `try/catch` funziona trasparentemente su confini asincroni (una `yield`-ed Promise rigettata lancia nel generator come se fosse un `throw` normale). Per flussi con più di due step in sequenza, o con logica di errore condizionale, il pattern generator è significativamente più manutenibile.

</details>
