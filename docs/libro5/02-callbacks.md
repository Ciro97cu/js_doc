# Callbacks

I callback sono l'unità fondamentale dell'asincronia in JavaScript. Ogni funzione che viene eseguita "dopo" — in risposta a un evento, a una risposta Ajax, allo scadere di un timer — è un callback. Per quanto a lungo abbiano servito da workhorse del codice async, soffrono di due gravi limitazioni strutturali che le astrazioni successive (Promise, generator, async/await) cercano di risolvere.

---

## Continuations

Un callback è la **continuation** del programma: la seconda metà che viene eseguita dopo il gap temporale.

```js
// A
ajax("http://some.url.1", function(data) {
    // C
});
// B
```

`A` e `B` avvengono adesso. `C` avviene dopo, quando Ajax ha una risposta. Il callback incapsula tutto ciò che deve accadere "dopo".

---

## Il problema cognitivo: il cervello è sequenziale

Il cervello umano pianifica in modo **sequenziale e bloccante**: "prima A, poi B, poi C". I callback esprimono l'asincronia in modo non-lineare — il flusso salta tra funzioni diverse, spesso definite lontano l'una dall'altra nel codice.

Questa divergenza cognitiva è il primo problema fondamentale dei callback: il codice è difficile da leggere, ragionare, debuggare e mantenere non perché la nesting sia brutta in sé, ma perché il flusso esecutivo non corrisponde al modo in cui il cervello traccia la sequenza degli eventi.

### Callback hell

Il termine "callback hell" (a volte "pyramid of doom" per la forma a triangolo del nesting) indica codice async con molti callback annidati:

```js
listen("click", function handler(evt) {
    setTimeout(function request() {
        ajax("http://some.url.1", function response(text) {
            if (text == "hello") {
                handler();
            } else if (text == "world") {
                request();
            }
        });
    }, 500);
});
```

Eppure il problema non è l'indentazione. Spostare i callback in funzioni named non aiuta:

```js
listen("click", handler);

function handler()  { setTimeout(request, 500); }
function request()  { ajax("http://some.url.1", response); }
function response(text) {
    if (text == "hello")       handler();
    else if (text == "world")  request();
}
```

Il codice non è annidato, ma per capire il flusso bisogna ancora saltare da una funzione all'altra. La difficoltà è strutturale, non visiva.

Un ulteriore problema: se `doA` è async e `doD` è async, l'ordine di esecuzione è:

```js
doA(function() {
    doC();
    doD(function() {
        doF();
    });
    doE();
});
doB();

/* Ordine reale: A → B → C → D → E → F */
```

Ma se per qualche condizione `doA` diventasse sync, l'ordine cambierebbe — e il codice che sembrava corretto si rompe silenziosamente.

I callback hardcodano la sequenza: lo step 2 chiama lo step 3, lo step 3 chiama lo step 4. Non c'è spazio per retry, fork del flusso, gestione di errori diversificata — ogni eventualità va aggiunta manualmente, rendendo il codice fragile e non riutilizzabile.

---

## Inversion of control — il problema di fiducia

Il secondo problema, più sottile e pericoloso, è l'**inversion of control** (inversione di controllo).

```js
// A
ajax("..", function(..) {
    // C — la seconda metà del programma
});
// B
```

`A` e `B` sono sotto il controllo diretto del programma. `C` è ceduto ad `ajax()` — che potrebbe essere una libreria di terze parti. Esiste un contratto implicito: si assume che `ajax()` chiamerà il callback **esattamente una volta**, al momento giusto, con i parametri corretti.

### Un caso reale: cinque chiamate invece di una

Si consideri un checkout di e-commerce che chiama un'utility di analytics di terze parti passando un callback con `chargeCreditCard()`:

```js
analytics.trackPurchase(purchaseData, function() {
    chargeCreditCard();
    displayThankyouPage();
});
```

Se il provider di analytics, per un bug in un deploy sperimentale, chiama il callback 5 volte invece di una, il cliente viene addebitato 5 volte. La correzione ovvia è un latch:

```js
var tracked = false;
analytics.trackPurchase(purchaseData, function() {
    if (!tracked) {
        tracked = true;
        chargeCreditCard();
        displayThankyouPage();
    }
});
```

Ma questo non risolve tutte le possibili misbehavior di un sistema di terze parti. La lista completa di cosa può andare storto:

- Callback chiamato **troppo presto** (prima che lo stato sia pronto)
- Callback chiamato **troppo tardi** (o mai)
- Callback chiamato **troppo poche o troppe volte**
- Callback chiamato senza i parametri necessari
- Eccezioni interne silenziose che impediscono la chiamata

E il problema non riguarda solo il codice di terze parti. Anche il proprio codice va difeso nello stesso modo — come si fa con gli input delle funzioni:

```js
/* troppo fiduciosa */
function addNumbers(x, y) { return x + y; }
addNumbers(21, "21"); /* "2121" — coercion indesiderata */

/* difensiva */
function addNumbers(x, y) {
    if (typeof x != "number" || typeof y != "number") {
        throw Error("Bad parameters");
    }
    return x + y;
}
```

Lo stesso principio "trust, but verify" dovrebbe applicarsi a ogni callback — ma i callback non forniscono strumenti built-in per farlo. Tutto deve essere costruito manualmente, a ogni utilizzo.

---

## Tentativi di salvare i callback

Alcune convenzioni cercano di mitigare i problemi, senza risolverli.

### Split callback (success / error)

```js
function success(data) { console.log(data); }
function failure(err)  { console.error(err); }

ajax("http://some.url.1", success, failure);
```

È il pattern usato anche dall'API ES6 Promise. Ma non risolve le invocazioni multiple né garantisce che venga chiamato uno (e uno solo) tra i due. Aggiunge verbosità senza risolvere i trust issue fondamentali.

### Error-first style (stile Node.js)

Il primo argomento del callback è riservato a un oggetto errore; se non c'è errore è `null`/falsy, il secondo argomento contiene il dato:

```js
function response(err, data) {
    if (err) { console.error(err); }
    else      { console.log(data); }
}
ajax("http://some.url.1", response);
```

È convenzione standard in Node.js. Non previene invocazioni multiple né garantisce che il callback venga chiamato.

### Protezione dal "mai chiamato" — `timeoutify`

```js
function timeoutify(fn, delay) {
    var intv = setTimeout(function() {
        intv = null;
        fn(new Error("Timeout!"));
    }, delay);

    return function() {
        if (intv) {
            clearTimeout(intv);
            fn.apply(this, arguments);
        }
    };
}

ajax("http://url.1", timeoutify(foo, 500));
```

Se il callback non viene chiamato entro 500ms, viene chiamato con un errore di timeout. Funziona, ma è boilerplate da ripetere ovunque.

### Il problema Zalgo — sync-or-async nondeterminism

Il caso più insidioso: una utility che a volte chiama il callback in modo sincrono (es. cache hit) e a volte in modo asincrono (es. rete). Il comportamento del programma dipende dall'ordine di esecuzione:

```js
function result(data) { console.log(a); }
var a = 0;

ajax("..pre-cached-url..", result);
a++;

/* Se il callback è sincrono: stampa 0 */
/* Se il callback è asincrono: stampa 1 */
```

Il nome colloquiale per questo anti-pattern è **Zalgo**. La regola: un'API non deve mai essere sync in certi casi e async in altri — deve scegliere uno e rispettarlo sempre. La soluzione è avvolgere in un `asyncify`:

```js
function asyncify(fn) {
    var orig_fn = fn;
    var intv = setTimeout(function() {
        intv = null;
        if (fn) fn();
    }, 0);
    fn = null;

    return function() {
        if (intv) {
            /* il callback è arrivato prima del timer — serializzalo */
            fn = orig_fn.bind.apply(
                orig_fn,
                [this].concat([].slice.call(arguments))
            );
        } else {
            orig_fn.apply(this, arguments);
        }
    };
}

ajax("..pre-cached-url..", asyncify(result));
/* result viene sempre chiamato in modo asincrono */
```

Ogni soluzione è possibile, ma richiede boilerplate costoso e specifico. I callback non offrono nulla di nativo per risolvere questi problemi.

---

## ⚡ Ripasso veloce

**Due problemi fondamentali dei callback:**

1. **Cognitive mismatch**: il cervello pianifica sequenzialmente; i callback esprimono flussi non-lineari. Il problema non è il nesting visivo — è la difficoltà di tracciare mentalmente il flusso async.

2. **Inversion of control**: passare un callback a una terza parte significa cedere il controllo sulla continuazione del programma. Nessuna garanzia built-in su quante volte verrà chiamato, quando, con quali argomenti.

**Pattern di mitigazione (tutti parziali):**
- Split callback (success/error) — verboso, non previene invocazioni multiple
- Error-first style (Node.js) — convenzione, non garanzia
- `timeoutify` — protegge dal "mai chiamato"
- `asyncify` — previene il nondeterminismo sync/async (Zalgo)

```js
/* error-first style */
ajax("http://url.1", function(err, data) {
    if (err) { /* gestisci errore */ return; }
    /* usa data */
});

/* latch per invocazioni multiple */
var done = false;
thirdParty(data, function() {
    if (done) return;
    done = true;
    /* logica critica */
});

/* timeoutify — mai "dimenticato" */
ajax("http://url.1", timeoutify(callback, 3000));

/* sempre async — mai Zalgo */
ajax("http://url.1", asyncify(callback));
```

La soluzione vera a questi problemi non è un'altra convenzione sui callback — sono le **Promise** (Cap. 3), che invertono l'inversione di controllo.

---

## Domande

<details>
<summary>Perché il callback hell è un problema cognitivo, non solo visivo?</summary>

Il problema non è l'indentazione o il nesting: anche spostando i callback in funzioni named separate si elimina la piramide visiva ma non la difficoltà. Il problema reale è che per ragionare sul flusso async si deve saltare avanti e indietro tra funzioni sparse nel codice, ricostruire mentalmente la sequenza temporale degli eventi in un ordine non corrispondente all'ordine testuale. Il cervello pianifica sequenzialmente (A poi B poi C), ma i callback esprimono dipendenze temporali che si leggono trasversalmente. Aggiungere gestione degli errori, retry e fork del flusso peggiora esponenzialmente questa complessità cognitiva.

</details>

<details>
<summary>Cos'è l'inversion of control e perché è pericolosa nei callback?</summary>

L'inversion of control si verifica quando si passa un callback a una funzione esterna (specialmente di terze parti), cedendole il controllo su quando e come eseguire la continuation del programma. Si crea un contratto implicito non garantito: si assume che il callback venga chiamato esattamente una volta, al momento giusto, con i parametri corretti. Se la parte esterna non rispetta questo contratto — callback chiamato troppe volte, mai, troppo presto, con parametri sbagliati, con eccezioni silenziose — il programma si comporta in modo imprevedibile. E non esiste nessuna primitiva built-in nel sistema dei callback per difendersi da questi scenari.

</details>

<details>
<summary>Qual è il problema del pattern sync-or-async (Zalgo) e come si evita?</summary>

Un'API che a volte chiama il callback in modo sincrono (es. cache hit) e a volte in modo asincrono (es. rete) introduce nondeterminismo: il comportamento del codice che segue il `ajax()` dipende da condizioni runtime invisibili. Il programma stampa valori diversi, esegue passi in ordine diverso, si comporta in modo non riproducibile. La regola è "mai Zalgo": un'API deve scegliere se essere sempre sincrona o sempre asincrona e rispettare questa scelta incondizionatamente. Per forzare l'asincronia si usa il pattern `asyncify`: si avvolge il callback in modo che, anche se l'utility lo chiama in modo sincrono, l'esecuzione effettiva viene posticipata con `setTimeout(fn, 0)`.

</details>

<details>
<summary>Perché l'error-first style (Node.js) non risolve i trust issue dei callback?</summary>

L'error-first style è una convenzione che standardizza il modo in cui gli errori vengono comunicati (`function(err, data)`), ma non fornisce garanzie sul comportamento dell'API chiamante. Non previene: callback mai chiamato (nessun errore, nessun dato — silenzio totale), callback chiamato più volte (l'`if (err)` viene eseguito ripetutamente), comportamento sync-or-async nondeterministico. È utile come convenzione per la leggibilità e l'uniformità del codice, ma rimane solo una convenzione — non un meccanismo di enforcement. I problemi strutturali dell'inversion of control rimangono intatti.

</details>

<details>
<summary>Perché le Promise sono una soluzione migliore ai problemi dei callback?</summary>

Le Promise invertono l'inversione di controllo. Invece di passare il callback alla terza parte e aspettare che la chiami, si riceve indietro una Promise — un oggetto che rappresenta il completamento futuro (o il fallimento) dell'operazione. Il codice che reagisce al risultato (`.then()`, `.catch()`) viene registrato sulla Promise stessa, non ceduto alla terza parte. Questo sposta il controllo: è il programma che decide quando e come consumare il risultato, non la libreria esterna. Le Promise garantiscono: risoluzione una sola volta (immutable), notifica sempre asincrona (Job queue), propagazione degli errori, componibilità. Tutto ciò che con i callback richiedeva boilerplate manuale è built-in.

</details>
