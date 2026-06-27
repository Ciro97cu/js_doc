# Benchmarking & Tuning

Questo capitolo affronta le prestazioni a livello micro — singole espressioni o statement — e soprattutto la capacità di misurarle correttamente. Il problema principale non è "X è più veloce di Y?" ma "come si sa davvero se è più veloce, e conta davvero?"

---

## Il benchmark naïve è sbagliato

L'approccio più comune per misurare la velocità di un'operazione:

```js
var start = Date.now();
/* operazione da testare */
var end = Date.now();
console.log("Duration:", end - start);
```

È fondamentalmente inaffidabile per diversi motivi.

**Precisione del timer**: su alcune piattaforme (es. vecchie versioni di Windows/IE) il timer si aggiorna ogni 15ms, non ogni 1ms. Se l'operazione dura meno di 15ms, si ottiene 0 — che non significa "istantaneo".

**Singolo campione**: anche se la durata è 4ms, non si sa se quella è la durata normale o se c'era un'interferenza del sistema (garbage collection, I/O, processo OS) proprio in quel momento. Un singolo campione ha confidenza vicina a zero.

**Ottimizzazioni dell'engine**: il JS engine potrebbe ottimizzare il codice isolato del test in modi che non si verificherebbero in un programma reale — rendendo il risultato artificialmente ottimistico.

### Ripetizione e statistica

Aggiungere un loop non è sufficiente. La media aritmetica di N iterazioni è distorta dagli outlier e non fornisce misura dell'affidabilità. Serve:

- **Campioni multipli** per stimare la varianza
- **Durata del ciclo** calibrata sulla precisione del timer: con timer da 15ms, ogni ciclo deve durare almeno 750ms per scendere sotto l'1% di margine d'errore; con timer da 1ms bastano 50ms
- **Metriche statistiche**: deviazione standard, varianza, margine d'errore

Costruire tutto questo da zero richiede competenze statistiche non banali. La soluzione è usare uno strumento già corretto.

---

## Benchmark.js

**Benchmark.js** (di John-David Dalton e Mathias Bynens) è una libreria di benchmarking statisticamente corretta. Va usata al posto di qualsiasi soluzione artigianale.

```js
function foo() {
    /* operazione da testare */
}

var bench = new Benchmark(
    "foo test",   // nome del test
    foo,          // funzione da testare
    { /* opzioni aggiuntive */ }
);

bench.hz;               // operazioni al secondo
bench.stats.moe;        // margin of error
bench.stats.variance;   // varianza tra i campioni
```

Per confrontare due approcci X e Y si definiscono due test in una *suite* — Benchmark.js li esegue in parallelo e compara le statistiche.

Benchmark.js può anche essere integrato nei pipeline CI/CD come test di regressione delle prestazioni: si confrontano le prestazioni attuali con quelle di una versione precedente su percorsi critici dell'applicazione.

### Setup e teardown: una trappola frequente

Benchmark.js supporta funzioni `setup` e `teardown`. È fondamentale capire **quando** vengono eseguite: non a ogni iterazione del test, ma a ogni *ciclo* (la struttura esterna che raggruppa molte iterazioni).

```js
/* INTENZIONE: a inizia come "x" a ogni iterazione */
/* REALTÀ: a inizia come "x" a ogni ciclo, poi cresce */
setup: function() { var a = "x"; }
test:  function() { a = a + "w"; b = a.charAt(1); }
```

Se il test concatena caratteri a `a`, dopo molte iterazioni `a` diventa una stringa molto lunga — non la stringa corta immaginata. Lo stesso vale per modifiche al DOM: se si aggiunge un elemento figlio a ogni iterazione, il genitore non è mai "pulito" tra un'iterazione e l'altra, solo tra un ciclo e l'altro.

---

## Il contesto è tutto

Un benchmark che mostra "X è il 20% più veloce di Y" può essere statisticamente corretto e allo stesso tempo irrilevante.

Si considerino X a 10.000.000 op/s e Y a 8.000.000 op/s: X impiega 100ns per operazione, Y 125ns — una differenza di 25ns. Il cervello umano non percepisce intervalli sotto i 100ms, che è un milione di volte più lento di quei 100ns. Per percepire la differenza cumulativa tra X e Y si dovrebbe eseguire l'operazione 4-5 milioni di volte in sequenza continua — un caso raro nella pratica.

La conclusione: **la maggior parte dei micro-benchmark su operazioni rapide è irrilevante** per decisioni architetturali reali. I miti più persistenti — `++x` vs `x++`, `i < len` vs `i < x.length` — rientrano tutti in questa categoria.

---

## Ottimizzazioni dell'engine: il codice che si scrive ≠ il codice che gira

I motori JS moderni (V8, SpiderMonkey, JavaScriptCore) applicano trasformazioni agressive al codice prima di eseguirlo: inlining di costanti, eliminazione del codice morto, unrolling della ricorsione.

```js
var foo = 41;
(function(){ (function(){ (function(baz){
    var bar = foo + baz;
})(1); })(); })();
```

Un engine potrebbe notare che `foo` viene usato in un solo posto e mai modificato, e trasformarlo in:

```js
(function(){ (function(){ (function(baz){
    var bar = 41 + baz;  // foo inlined
})(1); })(); })();
```

Oppure, data la funzione ricorsiva:

```js
function factorial(n) {
    if (n < 2) return 1;
    return n * factorial(n - 1);
}
factorial(5);   // l'engine potrebbe sostituire con la costante 120
```

L'engine potrebbe calcolare il risultato a compile-time, o riscrivere la ricorsione come loop. Se si fa un benchmark per decidere se usare `n * factorial(n-1)` o `n *= factorial(--n)`, si misura qualcosa che l'engine potrebbe già non eseguire affatto.

La regola: **il codice JS è un suggerimento all'engine, non un'istruzione letterale**. Ossessionarsi per `++i` vs `i++` o per cachare `x.length` è in quasi tutti i casi tempo sprecato — e in V8, cachare `x.length` può persino peggiorare le prestazioni.

---

## I motori non sono tutti uguali

Ogni engine — V8 (Chrome/Node.js), SpiderMonkey (Firefox), JavaScriptCore (Safari) — è spec-compliant ma applica strategie di ottimizzazione diverse e talvolta opposte.

Esempi di consigli specifici per V8 che circolano nella community:
- Non passare l'oggetto `arguments` ad altre funzioni — causa de-ottimizzazione
- Isolare `try..catch` in funzioni proprie — alcuni motori non ottimizzano funzioni che contengono `try..catch`

Questi consigli possono essere corretti per V8 in un certo momento, ma:
- Potrebbero essere sbagliati in altri motori
- V8 stesso potrebbe cambiare comportamento nelle versioni successive (è già successo con `join()` vs `+` per concatenazione di stringhe)

La storia della concatenazione è istruttiva: per anni la best practice era usare `array.join("")` invece di `+` per motivi di prestazioni interni all'engine. Poi i motori ottimizzarono `+` (il pattern più diffuso) e tutto quel codice `join("")` divenne subottimale. È il rischio dell'ottimizzazione "cowpath" — seguire la strada battuta da un'ottimizzazione che poi scompare.

La posizione corretta: **non ottimizzare per un singolo engine**. Se si trova un bug di prestazioni in un browser specifico, segnalarlo agli sviluppatori del browser — i browser si aggiornano molto più frequentemente di quanto si faceva in passato.

---

## jsPerf.com

Benchmark.js misura il codice nell'ambiente corrente. Per conclusioni generalizzabili servono risultati da **molti ambienti diversi**: Chrome desktop, Chrome mobile, Firefox, Safari, Node.js, dispositivi con batteria scarica.

**jsPerf.com** è una piattaforma web che usa Benchmark.js per raccogliere risultati da molti utenti su ambienti diversi e li visualizza in grafici aggregati. Si definisce un test con due o più casi, si pubblica l'URL, e chiunque può contribuire con i propri risultati.

### Errori comuni nei test jsPerf

**Loop innestati nel test**: Benchmark.js già ripete il test migliaia di volte. Aggiungere un `for` dentro il test case introduce rumore e confonde cosa si sta misurando.

**Setup che non resetta lo stato**: come visto sopra, il setup viene eseguito per ciclo, non per iterazione.

**Confronto mele-arance — caso 1**: un comparatore di sort inline crea una nuova funzione a ogni iterazione, aggiungendo costi estranei:

```js
/* SBAGLIATO: mySort viene ricreata a ogni iterazione */
x.sort(function mySort(a, b) { return a - b; });

/* CORRETTO: mySort pre-dichiarata nel page setup */
x.sort(mySort);
```

**Confronto mele-arance — caso 2**: i due casi producono risultati diversi, invalidando il confronto:

```js
/* Case 1: ordine lessicografico → [-14, 0, 0, 12, 18, 2.9, 3] */
x.sort();
/* Case 2: ordine numerico → [-14, 0, 0, 2.9, 3, 12, 18] */
x.sort(function(a, b) { return a - b; });
```

Se i due casi producono output diversi, qualsiasi risultato di prestazioni è privo di senso.

**Asimmetrie sottili**:

```js
/* Case 1: x ha un valore (un'assegnazione in più) */
var x = false; var y = x ? 1 : 2;
/* Case 2: x è undefined (nessuna assegnazione) */
var x;         var y = x ? 1 : 2;

/* Corretto: entrambi hanno un'assegnazione */
var x = false;     var y = x ? 1 : 2;
var x = undefined; var y = x ? 1 : 2;
```

---

## Microperformance: quando conta davvero

Ossessionarsi per microperformance su percorsi non critici è sbagliato. Ma Knuth, spesso citato fuori contesto, diceva:

> *"Programmers waste enormous amounts of time thinking about the speed of noncritical parts of their programs [...] We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. **Yet we should not pass up our opportunities in that critical 3%.**"*

La regola pratica: se il codice è su un **percorso critico** — animazioni, loop hot, UI reattiva — si deve ottimizzare senza riserve. Se non lo è, la leggibilità ha la precedenza.

Per un percorso critico che deve convertire una stringa in numero:

```js
var x = "42";

var y = x / 2;          // coercione implicita — rapida
var y = parseInt(x, 10) / 2;  // parsing completo — più lento
var y = Number(x) / 2;  // funzione — leggibile, simile a +x
var y = +x / 2;         // unario + — rapido, comune
var y = (x | 0) / 2;   // bitwise OR — rapido, ma tronca i decimali
```

`parseInt()` fa più lavoro (parsing carattere per carattere) ed è generalmente più lento — ma è l'unico adatto se la stringa può essere "42px". Le altre opzioni sono comparabili; la scelta va fatta su leggibilità, non su microottimizzazioni.

---

## Tail Call Optimization (TCO)

ES6 introduce un requisito di ottimizzazione che il linguaggio **impone** ai motori: la **Tail Call Optimization** (ottimizzazione delle chiamate in coda).

Una **tail call** (chiamata in coda) è una chiamata a funzione che si trova nell'ultima posizione di un'altra funzione — dopo la quale non c'è nient'altro da fare se non restituire il risultato:

```js
function foo(x) { return x; }

function bar(y) {
    return foo(y + 1);  // tail call: bar() finisce qui
}

function baz() {
    return 1 + bar(40); // NON tail call: c'è ancora "+ 1" da fare
}
```

Normalmente ogni chiamata di funzione aggiunge uno **stack frame** (blocco di memoria per gestire il contesto della chiamata). Con chiamate ricorsive profonde, lo stack cresce fino a esaurire la memoria.

Con TCO, un engine che riconosce una tail call **riusa il frame dello stack corrente** invece di crearne uno nuovo. Per la ricorsione questo è trasformativo: invece di stack frames proporzionali alla profondità, se ne usa sempre solo uno.

```js
/* ricorsione classica — non TCO-friendly */
function factorial(n) {
    if (n < 2) return 1;
    return n * factorial(n - 1);   /* c'è ancora "n *" da fare */
}

/* TCO-friendly: la chiamata ricorsiva è in tail position */
function factorial(n) {
    function fact(n, res) {
        if (n < 2) return res;
        return fact(n - 1, n * res);  /* tail call: nient'altro dopo */
    }
    return fact(n, 1);
}

factorial(5);   // 120
```

La differenza: nella versione TCO-friendly il risultato parziale viene accumulato nel parametro `res` invece che nello stack, permettendo all'engine di riusare il frame. Algoritmi che prima erano impraticabili in JS per i limiti dello stack diventano fattibili.

ES6 richiede TCO nei motori (non lo lascia opzionale) precisamente perché senza di essa certi algoritmi ricorsivi non sono praticamente implementabili — non è solo un'ottimizzazione di velocità, ma un'abilitazione di pattern altrimenti impossibili.

---

## ⚡ Ripasso veloce

**Benchmarking:**
- `Date.now()` diff = metodo sbagliato — imprecisione del timer, campione singolo, ottimizzazioni artificiose
- Usare **Benchmark.js** — statisticamente corretto: campioni multipli, margine d'errore, varianza
- `setup`/`teardown` si eseguono per **ciclo**, non per iterazione

**jsPerf pitfalls:**
- No loop innestati nel test (Benchmark.js già itera)
- Casi che producono output diversi → confronto invalido
- Funzioni inline nel test → costo aggiuntivo non voluto

**Microperformance:**
- L'engine riscrive il codice (inlining, dead-code removal, loop unrolling)
- `++x` vs `x++`, cachare `x.length` → quasi sempre irrilevante
- Non ottimizzare per un singolo engine — le ottimizzazioni cambiano

**Big picture:**
- Ottimizzare solo percorsi critici; leggibilità prima di tutto altrove
- Knuth: prematura ottimizzazione è male sul 97% non-critico — ma il 3% critico va ottimizzato senza riserve

**TCO:**
```js
/* tail call: foo() in posizione finale, nient'altro dopo */
function bar(y) { return foo(y + 1); }

/* TCO-friendly factorial con accumulatore */
function factorial(n) {
    function fact(n, res) {
        if (n < 2) return res;
        return fact(n - 1, n * res);  /* tail call */
    }
    return fact(n, 1);
}
```

---

## Domande

<details>
<summary>Perché misurare le prestazioni con <code>Date.now()</code> su un singolo run è inaffidabile?</summary>

Ci sono tre problemi fondamentali. Il primo è la precisione del timer: alcune piattaforme aggiornano il timestamp ogni 15ms o più, quindi operazioni rapide risultano tutte con durata 0 o 15ms, indipendentemente dalla durata reale. Il secondo è il campionamento singolo: un singolo run può essere distorto da interferenze esterne (garbage collection, scheduling OS, I/O) che non riflettono il comportamento tipico. Il terzo è il rischio di ottimizzazioni artificiali: il JS engine può riconoscere che il codice isolato del test è banale e ottimizzarlo (inlining, eliminazione del codice morto) in modi che non si verificherebbero in un programma reale. La soluzione è usare Benchmark.js, che esegue molti cicli di molte iterazioni, calcola media, varianza e margine d'errore in modo statisticamente corretto.

</details>

<details>
<summary>Cosa significa che "il contesto è tutto" nella valutazione di un micro-benchmark?</summary>

Significa che anche un risultato statisticamente corretto può essere irrilevante. Se X gira a 10 milioni di op/s e Y a 8 milioni, X impiega 100ns e Y 125ns — una differenza di 25ns. Il cervello umano non percepisce intervalli sotto i 100ms, cioè 4 milioni di volte più lento di quella differenza. Per far accumulare 100ms di differenza tra X e Y si dovrebbero eseguire consecutivamente circa 4 milioni di operazioni — uno scenario raro nella pratica. La domanda giusta non è "X è più veloce di Y?" ma "questa differenza ha impatto misurabile sull'esperienza utente nel contesto reale del mio programma?"

</details>

<details>
<summary>Perché ottimizzare per le caratteristiche interne di un singolo JS engine è rischioso?</summary>

Perché i motori cambiano. La concatenazione di stringhe con `array.join("")` era più rapida di `+` in certi motori anni fa, e molti sviluppatori adottarono quella pratica come best practice. Quando i motori ottimizzarono `+` (il pattern più diffuso, la cosiddetta "cowpath"), tutto quel codice `join("")` divenne subottimale. Lo stesso rischio vale per qualsiasi hack specifico a V8, SpiderMonkey o JavaScriptCore: le ottimizzazioni cambiano tra versioni, e il codice contorto scritto per sfruttarle potrebbe sopravvivere alla problematica che doveva risolvere, diventando esso stesso un debt. Inoltre, un'ottimizzazione vera per un engine può essere controproducente in un altro.

</details>

<details>
<summary>Cos'è una tail call e perché TCO è importante specialmente per la ricorsione?</summary>

Una **tail call** è una chiamata a funzione che si trova nell'ultima posizione di un'altra funzione: dopo la sua esecuzione non c'è nient'altro da fare, solo restituire il risultato. Senza TCO, ogni chiamata ricorsiva aggiunge un nuovo stack frame, e la profondità massima della ricorsione è limitata dalla memoria disponibile. Con TCO, l'engine riconosce che il frame corrente non servirà più dopo la tail call e lo riusa invece di crearne uno nuovo — la ricorsione può essere quindi arbitrariamente profonda con un singolo frame. La rilevanza è che alcuni algoritmi naturalmente ricorsivi (ricorsione su strutture ad albero, certi pattern funzionali) diventavano impraticabili in JS senza TCO. ES6 la richiede esplicitamente perché non è solo un'ottimizzazione di velocità, ma un'abilitazione di pattern altrimenti impossibili.

</details>

<details>
<summary>Come si riscrive una funzione ricorsiva per renderla TCO-friendly?</summary>

Il pattern generale è introdurre un **accumulatore** come parametro aggiuntivo, in modo che il valore parziale venga passato avanti invece di essere mantenuto sullo stack. La chiamata ricorsiva diventa così l'ultima operazione della funzione — nessuna operazione in sospeso che richiederebbe di mantenere il frame. Esempio con factorial: la versione classica `return n * factorial(n-1)` non è TCO perché dopo il ritorno di `factorial(n-1)` c'è ancora la moltiplicazione per `n`. La versione TCO-friendly `return fact(n-1, n * res)` calcola `n * res` prima di chiamare `fact`, passando il risultato parziale come argomento — dopo il ritorno non c'è nient'altro, il frame può essere riusato.

</details>
