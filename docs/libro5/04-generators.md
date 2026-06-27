# Generators

I generator sono il terzo pilastro dell'asincronia in ES6, dopo callback e Promise. Il loro potere sta in una capacità apparentemente impossibile: una funzione che si mette in pausa a metà della sua esecuzione, cede il controllo all'esterno, e poi riprende da dove si era fermata — mantenendo intatto il proprio stato interno.

---

## Rompere il run-to-completion

Nei capitoli precedenti si è visto che ogni funzione JS, una volta avviata, viene eseguita fino alla fine senza che nessun altro codice possa interromperla. I generator rompono questa regola.

```js
var x = 1;

function *foo() {
    x++;
    yield;          // pausa!
    console.log("x:", x);
}

function bar() { x++; }

var it = foo();     // crea l'iterator, non esegue ancora *foo()
it.next();          // avvia *foo() fino al yield → x diventa 2
bar();              // x diventa 3
it.next();          // riprende *foo() → stampa "x: 3"
```

`foo()` non esegue il generator: restituisce un **iterator** (oggetto di controllo). Solo `it.next()` avvia e poi riprende l'esecuzione. La parola chiave `yield` segnala un punto di pausa cooperativa — il generator cede volontariamente il controllo.

Un generator è una funzione speciale che può avviarsi e fermarsi una o più volte, senza necessariamente mai completarsi.

---

## Input, output e messaggi bidirezionali

Un generator rimane una funzione: accetta argomenti e può restituire un valore.

```js
function *foo(x, y) {
    return x * y;
}
var it = foo(6, 7);
it.next().value;    // 42
```

`it.next()` restituisce sempre un oggetto `{ done: boolean, value: any }`. Quando il generator esegue `return`, `done` diventa `true` e `value` contiene il valore restituito.

### Il canale bidirezionale: yield ↔ next

`yield` e `next()` formano un sistema di messaggi a due vie: `yield expr` invia un valore verso l'esterno; `it.next(val)` invia un valore verso l'interno, che diventa il risultato dell'espressione `yield` correntemente in pausa.

```js
function *foo(x) {
    var y = x * (yield "Hello");    // yield invia "Hello" fuori
    return y;
}

var it = foo(6);
var res = it.next();        // primo next() — non passare nulla
res.value;                  // "Hello"
res = it.next(7);           // 7 è il risultato del yield sospeso
res.value;                  // 42 (6 * 7)
```

**Regola fondamentale**: il primo `next()` avvia il generator fino al primo `yield` — non c'è nessun `yield` in pausa a cui passare un valore, quindi qualsiasi argomento al primo `next()` viene silenziosamente ignorato. Si deve sempre avviare un generator con `next()` senza argomenti.

Il motivo dello "sfasamento" (N yield → N+1 next) è che ogni `next()` risponde alla domanda posta dal `yield` precedente. L'ultimo `next()` riceve risposta dal `return` finale (o da un `return undefined` implicito).

---

## Istanze multiple

Ogni chiamata a `foo()` crea un'istanza indipendente del generator con il proprio stato interno. Due istanze dello stesso generator possono girare in parallelo e persino condividere variabili esterne:

```js
function *foo() {
    var x = yield 2;
    z++;
    var y = yield (x * z);
    console.log(x, y, z);
}

var z = 1;
var it1 = foo();
var it2 = foo();

var val1 = it1.next().value;            // 2
var val2 = it2.next().value;            // 2
val1 = it1.next(val2 * 10).value;       // x=20, z=2 → 40
val2 = it2.next(val1 * 5).value;        // x=200, z=3 → 600
it1.next(val2 / 2);                     // y=300 → "20 300 3"
it2.next(val1 / 4);                     // y=10  → "200 10 3"
```

Questo interleaving manuale mostra che i generator possono riprodurre — in modo controllato e cooperativo — scenari di concorrenza che con le normali funzioni non sarebbero possibili.

---

## Generare valori: iteratori e iterabili

L'origine del nome "generator" sta nel loro uso come produttori di sequenze di valori. Il pattern classico con closure:

```js
var gimmeSomething = (function() {
    var nextVal;
    return function() {
        if (nextVal === undefined) nextVal = 1;
        else nextVal = (3 * nextVal) + 6;
        return nextVal;
    };
})();
```

Si può esprimere con l'interfaccia standard degli **iterator** (oggetti con metodo `next()` che restituisce `{ done, value }`):

```js
var something = (function() {
    var nextVal;
    return {
        [Symbol.iterator]: function() { return this; },
        next: function() {
            if (nextVal === undefined) nextVal = 1;
            else nextVal = (3 * nextVal) + 6;
            return { done: false, value: nextVal };
        }
    };
})();
```

Con un generator la stessa logica diventa molto più semplice:

```js
function *something() {
    var nextVal;
    while (true) {
        if (nextVal === undefined) nextVal = 1;
        else nextVal = (3 * nextVal) + 6;
        yield nextVal;
    }
}
```

Il `while (true)` non blocca il thread perché il generator si sospende a ogni `yield`. Il loop `for..of` consuma automaticamente un iterable (oggetto con `Symbol.iterator`) chiamando `next()` finché `done` non è `true`:

```js
for (var v of something()) {
    console.log(v);
    if (v > 500) break;
}
// 1 9 33 105 321 969
```

**Distinzione**: un **iterator** ha il metodo `next()`; un **iterable** ha `Symbol.iterator` che restituisce un iterator. Un generator produce un iterator che è anche iterable (ha `Symbol.iterator` che ritorna `this`).

### Fermare un generator

Quando `for..of` termina con `break`, `return` o un'eccezione non gestita, invia automaticamente un segnale di terminazione all'iterator. Si può inviare lo stesso segnale manualmente con `it.return(val)`:

```js
function *something() {
    try {
        var nextVal;
        while (true) {
            if (nextVal === undefined) nextVal = 1;
            else nextVal = (3 * nextVal) + 6;
            yield nextVal;
        }
    }
    finally {
        console.log("cleaning up!");
    }
}

var it = something();
for (var v of it) {
    console.log(v);
    if (v > 500) {
        console.log(it.return("Hello World").value);
        // no `break` needed — done:true stops the loop
    }
}
// 1 9 33 105 321 969
// cleaning up!
// Hello World
```

Il blocco `finally` viene sempre eseguito, sia per terminazione normale che per terminazione esterna — utile per cleanup di risorse (connessioni DB, file descriptor).

---

## Generator e asincronia

Il vero potere dei generator emerge quando si combina il meccanismo di pausa con il flusso async. Il classico approccio con callback:

```js
function foo(x, y, cb) {
    ajax("http://some.url.1/?x=" + x + "&y=" + y, cb);
}

foo(11, 31, function(err, text) {
    if (err) console.error(err);
    else console.log(text);
});
```

Con un generator si può esprimere lo stesso flusso in modo sincrono-looking:

```js
function foo(x, y) {
    ajax("http://some.url.1/?x=" + x + "&y=" + y, function(err, data) {
        if (err) it.throw(err);
        else it.next(data);
    });
}

function *main() {
    try {
        var text = yield foo(11, 31);
        console.log(text);
    }
    catch (err) {
        console.error(err);
    }
}

var it = main();
it.next();  // avvia il generator
```

Il meccanismo: `yield foo(11,31)` chiama `foo()` (che avvia la richiesta Ajax), poi sospende il generator. Quando Ajax risponde, il callback chiama `it.next(data)` per riprendere il generator — e `data` diventa il valore assegnato a `text`. In caso di errore, `it.throw(err)` lo inietta nel generator come se fosse lanciato dal `yield`: il `try..catch` lo cattura normalmente.

Risultato: **gestione degli errori sincrona (`try..catch`) su codice asincrono**. Questo è il salto qualitativo rispetto ai callback.

---

## Generator + Promise

Il pattern precedente funziona ma rinuncia alle garanzie di trustability delle Promise. La combinazione ottimale è: il generator `yield`a una Promise, e un runner esterno gestisce la risoluzione.

```js
function foo(x, y) {
    return request("http://some.url.1/?x=" + x + "&y=" + y);
    // request() restituisce una Promise
}

function *main() {
    try {
        var text = yield foo(11, 31);
        console.log(text);
    }
    catch (err) {
        console.error(err);
    }
}
```

Il codice dentro `*main()` non cambia: non sa se sta yielding una Promise o qualcos'altro. Il runner esterno si occupa di tutto:

```js
function run(gen) {
    var args = [].slice.call(arguments, 1), it;
    it = gen.apply(this, args);

    return Promise.resolve()
        .then(function handleNext(value) {
            var next = it.next(value);
            return (function handleResult(next) {
                if (next.done) return next.value;
                return Promise.resolve(next.value)
                    .then(handleNext, function handleErr(err) {
                        return Promise.resolve(it.throw(err))
                            .then(handleResult);
                    });
            })(next);
        });
}

run(main);
```

`run()` avanza automaticamente il generator: quando riceve una Promise (`next.value`), aspetta che si risolva e poi richiama il generator con il valore (o inietta l'errore con `it.throw(err)`). Il loop si ripete fino a `done: true`.

### ES2017: async/await

Questo pattern — generator che yielda Promise, runner che le gestisce — è talmente utile che ES2017 lo ha integrato nella sintassi:

```js
async function main() {
    try {
        var text = await foo(11, 31);
        console.log(text);
    }
    catch (err) {
        console.error(err);
    }
}

main();  // nessun run() necessario
```

`async function` equivale a un generator con runner integrato. `await` equivale a `yield` su una Promise. `async function` restituisce automaticamente una Promise che si risolve al completamento della funzione.

### Concorrenza in un generator

Il "sync-looking" dei generator può portare a scrivere richieste sequenziali dove potrebbero essere parallele:

```js
/* SUBOTTIMALE: r1 e r2 attendono in serie */
function *foo() {
    var r1 = yield request("http://some.url.1");
    var r2 = yield request("http://some.url.2");
    var r3 = yield request("http://some.url.3/?v=" + r1 + "," + r2);
    console.log(r3);
}
```

Soluzione: avviare entrambe le Promise prima di yieldarle.

```js
/* OTTIMALE: r1 e r2 in parallelo */
function *foo() {
    var p1 = request("http://some.url.1");  // avvia subito
    var p2 = request("http://some.url.2");  // avvia subito
    var r1 = yield p1;                      // attendi p1
    var r2 = yield p2;                      // attendi p2 (già avviata)
    var r3 = yield request("http://some.url.3/?v=" + r1 + "," + r2);
    console.log(r3);
}
run(foo);
```

Oppure, con `Promise.all`:

```js
function *foo() {
    var results = yield Promise.all([
        request("http://some.url.1"),
        request("http://some.url.2")
    ]);
    var [r1, r2] = results;
    var r3 = yield request("http://some.url.3/?v=" + r1 + "," + r2);
    console.log(r3);
}
run(foo);
```

La regola generale: nascondere la logica Promise concorrente in funzioni helper, tenendo il generator pulito con semplici `yield` lineari.

---

## Generator delegation: `yield *`

Un generator può delegare l'iterazione a un altro generator (o qualsiasi iterable) con la sintassi `yield *`:

```js
function *foo() {
    console.log("`*foo()` starting");
    yield 3;
    yield 4;
    console.log("`*foo()` finished");
}

function *bar() {
    yield 1;
    yield 2;
    yield *foo();   // delega a *foo()
    yield 5;
}

var it = bar();
it.next().value;    // 1
it.next().value;    // 2
it.next().value;    // "*foo()* starting" → 3
it.next().value;    // 4
it.next().value;    // "*foo()* finished" → 5
```

`yield *foo()` trasferisce il controllo dell'iterator a `*foo()`. L'iterator esterno (`it`) non si accorge della transizione — vede semplicemente una sequenza continua di valori. Quando `*foo()` termina, il controllo torna a `*bar()`.

### Messaggi e delegazione

La delegazione è trasparente anche per i messaggi bidirezionali: i valori inviati via `next()` raggiungono il generator delegato; il valore `return` del generator delegato diventa il risultato dell'espressione `yield *` nel delegante.

Anche le eccezioni si propagano trasparentemente in entrambe le direzioni attraverso la catena di delegazione.

### Delegazione asincrona

Con il pattern Promise + `run()`, `yield *` elimina la necessità di annidare chiamate a `run()`:

```js
function *foo() {
    var r2 = yield request("http://some.url.2");
    var r3 = yield request("http://some.url.3/?v=" + r2);
    return r3;
}

function *bar() {
    var r1 = yield request("http://some.url.1");
    var r3 = yield *foo();   // invece di: yield run(foo)
    console.log(r3);
}

run(bar);
```

`yield *` delega il controllo dell'iteration — non solo la composizione — rendendo il codice più leggibile e la struttura più simmetrica alle normali chiamate di funzione.

---

## Thunk

Un **thunk** è una funzione che, senza parametri, invoca un'altra funzione con i parametri già predisposti — rinviandone l'esecuzione al momento della chiamata del thunk stesso.

```js
function foo(x, y, cb) {
    setTimeout(function() { cb(x + y); }, 1000);
}

/* thunkify produce una "thunkory" (factory di thunk) */
function thunkify(fn) {
    return function() {
        var args = [].slice.call(arguments);
        return function(cb) {
            args.push(cb);
            return fn.apply(null, args);
        };
    };
}

var fooThunkory = thunkify(foo);
var fooThunk = fooThunkory(3, 4);

fooThunk(function(sum) { console.log(sum); });  // 7 (dopo 1s)
```

La simmetria con le Promise è notevole: una *thunkory* produce thunk esattamente come una *promisory* (`Promise.wrap()`) produce Promise. Entrambi rappresentano una domanda su un valore futuro.

Un generator può yieldar thunk invece di Promise — e un `run()` esteso può gestire entrambi — ma i thunk non offrono nessuna delle garanzie di trustability delle Promise. La raccomandazione è: preferire `yield pr` (Promise) a `yield th` (thunk), anche se un runner può supportare entrambi.

---

## Pre-ES6: transpilazione manuale

I generator sono nuova sintassi ES6 e non possono essere polyfillati come le Promise. La soluzione è la transpilazione: un transpiler trasforma il codice generator in una state machine ES5 equivalente usando closure.

```js
/* generator originale */
function *foo(url) {
    try {
        var val = yield request(url);
        console.log(val);
    }
    catch (err) {
        console.log("Oops:", err);
        return false;
    }
}
```

Transpilato manualmente diventa:

```js
function foo(url) {
    var state;
    var val;

    function process(v) {
        switch (state) {
            case 1:
                console.log("requesting:", url);
                return request(url);
            case 2:
                val = v;
                console.log(val);
                return;
            case 3:
                var err = v;
                console.log("Oops:", err);
                return false;
        }
    }

    return {
        next: function(v) {
            if (!state) { state = 1; return { done: false, value: process() }; }
            state = 2;
            return { done: true, value: process(v) };
        },
        throw: function(e) {
            state = 3;
            return { done: true, value: process(e) };
        }
    };
}
```

Ogni `yield` diventa un cambio di stato. Le variabili del generator diventano variabili nella closure esterna. La state machine avanza di un passo a ogni `next()`. I transpiler come Regenerator automatizzano questa trasformazione, rendendo i generator utilizzabili anche in ambienti pre-ES6.

---

## ⚡ Ripasso veloce

**Sintassi base:**

```js
function *gen() {
    var x = yield "primo";   // invia "primo", attende un valore
    return x * 2;
}

var it = gen();
it.next().value;        // "primo" — avvia fino al yield
it.next(21).value;      // 42 — x=21, return 42, done:true
```

**Punti chiave:**

- `function *nome()` dichiara un generator; `nome()` restituisce un iterator, non esegue ancora
- `yield` sospende il generator e invia un valore fuori; `it.next(val)` riprende e invia un valore dentro
- Sempre N+1 `next()` rispetto ai `yield`: il primo avvia, ogni successivo risponde al yield precedente, l'ultimo riceve risposta da `return`
- `yield *iterable` delega l'iterazione a un altro generator o iterable
- `it.throw(err)` inietta un'eccezione nel punto di yield; `it.return(val)` termina il generator
- Combinazione con Promise: il generator `yield`a Promise, un runner le gestisce → codice async con aspetto sync + trustability Promise
- `async/await` (ES2017) è questa stessa combinazione nella sintassi del linguaggio

```js
/* pattern async: generator + Promise */
function *main() {
    try {
        var result = yield fetch("http://api.example.com/data");
        console.log(result);
    } catch (err) {
        console.error(err);
    }
}
run(main);  // runner che gestisce le Promise yieldate

/* equivalente ES2017 */
async function main() {
    try {
        var result = await fetch("http://api.example.com/data");
        console.log(result);
    } catch (err) {
        console.error(err);
    }
}
main();
```

---

## Domande

<details>
<summary>Perché <code>foo()</code> non esegue il generator? Cosa restituisce invece?</summary>

Chiamare `foo()` su un generator non avvia l'esecuzione: restituisce un **iterator** — un oggetto con i metodi `next()`, `throw()`, e `return()` — che rappresenta un "controllo remoto" per il generator. L'esecuzione vera inizia solo alla prima chiamata `it.next()`. Questo design permette di passare l'iterator in giro, avviarlo in un secondo momento, o non avviarlo affatto. Ogni chiamata a `foo()` crea un'istanza indipendente con il proprio stato interno, quindi due iterator dello stesso generator non condividono scope locale.

</details>

<details>
<summary>Perché il canale <code>yield</code>/<code>next()</code> è "sfasato" di uno, e cosa risponde all'ultimo <code>next()</code>?</summary>

Il primo `next()` avvia il generator e lo porta fino al primo `yield` — a quel punto non c'è ancora nessun `yield` in pausa a cui passare un valore, quindi qualsiasi argomento viene ignorato. Il secondo `next(val)` fornisce il valore che diventa il risultato dell'espressione `yield` correntemente in pausa, e avanza fino al prossimo `yield`. Quindi con N yield servono N+1 chiamate a `next()`. L'ultimo `next()` — quello senza un `yield` corrispondente — riceve risposta dall'istruzione `return` del generator (o da `return undefined` implicito).

</details>

<details>
<summary>Come si ottiene codice async con aspetto sincrono usando generator + Promise?</summary>

Si fa in modo che il generator `yield`i una Promise invece di un valore diretto. Un runner esterno (come la funzione `run()`) riceve la Promise yieldata, aspetta che si risolva, e poi riprende il generator con il valore di fulfillment tramite `it.next(val)` — oppure inietta l'errore con `it.throw(err)`. Dal punto di vista del codice dentro il generator, tutto sembra sincrono: si scrive `var text = yield foo(11, 31)` e si ottiene il valore direttamente, senza callback. Il `try..catch` cattura sia errori sincroni che asincroni. Questo pattern è esattamente ciò che `async/await` implementa nativamente in ES2017.

</details>

<details>
<summary>Cosa fa <code>yield *foo()</code> e in cosa differisce da una normale chiamata a funzione?</summary>

`yield *foo()` è la sintassi di **yield-delegation**: trasferisce il controllo dell'iterator dal generator corrente all'iterator prodotto da `*foo()`. Tutti i valori yielded da `*foo()` passano trasparentemente verso l'esterno; tutti i valori inviati via `next()` dall'esterno passano trasparentemente verso `*foo()`; tutte le eccezioni si propagano in entrambe le direzioni. Il valore `return` di `*foo()` diventa il risultato dell'espressione `yield *foo()` nel generator delegante. A differenza di una normale chiamata di funzione, `yield *` può sospendersi nei punti `yield` del generator delegato, mantenendo il flusso asincrono. Si può delegare a qualsiasi iterable, non solo a generator.

</details>

<details>
<summary>Qual è la differenza tra richieste sequenziali e parallele in un generator?</summary>

In un generator con `run()`, il pattern `var r1 = yield request(url1); var r2 = yield request(url2)` esegue le due richieste in serie: la seconda parte solo dopo che la prima ha risposto. Per eseguirle in parallelo si devono avviare entrambe le Promise prima di yieldarle: `var p1 = request(url1); var p2 = request(url2); var r1 = yield p1; var r2 = yield p2;`. In questo modo entrambe le richieste Ajax partono immediatamente; i due `yield` attendono semplicemente la risoluzione delle Promise già in volo. In alternativa, `yield Promise.all([request(url1), request(url2)])` produce lo stesso risultato in modo più esplicito (gate pattern: entrambe devono risolvere).

</details>
