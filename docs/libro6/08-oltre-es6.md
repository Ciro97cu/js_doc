# Oltre ES6

ES6 ha introdotto una quantità considerevole di novità, ma il TC39 non si è fermato lì: nel momento in cui ES6 veniva finalizzato, erano già in lavorazione numerose proposte per le versioni successive. Questo capitolo ne esamina alcune tra le più significative, alcune delle quali erano già supportate da transpiler come Babel o Traceur al momento della scrittura del libro.

> **Nota:** Le funzionalità descritte qui erano in vari stadi di proposta al momento della pubblicazione (2015). La maggior parte ha poi raggiunto la specifica ufficiale — con la notevole eccezione di `Object.observe()`, ritirata dal processo TC39 nel dicembre 2015.

---

## async functions

Il pattern generator + Promise + runner visto nel capitolo 4 è potente, ma verboso. La proposta `async function` lo eleva a sintassi nativa, eliminando la necessità di un'utility esterna come `run()`.

Si consideri il codice con generator:

```js
run(function* main() {
    var ret = yield step1();
    try {
        ret = yield step2(ret);
    }
    catch (err) {
        ret = yield step2Failed(err);
    }
    ret = yield Promise.all([
        step3a(ret),
        step3b(ret),
        step3c(ret)
    ]);
    yield step4(ret);
})
.then(fulfilled, rejected);
```

Con la sintassi `async function` lo stesso flusso si esprime così:

```js
async function main() {
    var ret = await step1();
    try {
        ret = await step2(ret);
    }
    catch (err) {
        ret = await step2Failed(err);
    }
    ret = await Promise.all([
        step3a(ret),
        step3b(ret),
        step3c(ret)
    ]);
    await step4(ret);
}

main()
.then(fulfilled, rejected);
```

La simmetria con i generator è precisa: `function*` diventa `async function`, `yield` diventa `await`. L'engine sa autonomamente come attendere la risoluzione di una Promise prima di procedere. La chiamata a `main()` restituisce direttamente una Promise osservabile — equivalente a quella che `run(main)` avrebbe restituito.

`async function` è zucchero sintattico (sugar, cioè una forma abbreviata che il compilatore espande nel pattern sottostante) per generators + promises + run(). Non aggiunge nuova semantica, ma rende il codice molto più leggibile.

### Caveat: nessuna cancellazione

Il limite principale di `async function` è che, restituendo una Promise ordinaria, non esiste un meccanismo nativo per cancellare l'esecuzione dall'esterno. Se un'operazione async è costosa in termini di risorse e il risultato non serve più, non c'è modo diretto di fermarla.

```js
async function request(url) {
    var resp = await new Promise(function(resolve, reject) {
        var xhr = new XMLHttpRequest();
        xhr.open('GET', url);
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4) {
                xhr.status == 200 ? resolve(xhr) : reject(xhr.statusText);
            }
        };
        xhr.send();
    });
    return resp.responseText;
}

var pr = request('http://some.url.1');
// pr è una Promise — non esiste pr.cancel()
```

Le soluzioni discusse al momento includevano: token di cancellazione passati alla funzione, tipi di ritorno alternativi (osservabili, control token), oppure semplicemente accettare lo status quo. Al momento della pubblicazione il comportamento era quello di restituire Promise ordinarie.

---

## Object.observe() *(proposta ritirata)*

> **Importante:** `Object.observe()` è stata ritirata dalla specifica TC39 nel dicembre 2015, poco dopo la pubblicazione di questo libro. Non è implementata nei browser moderni e non dovrebbe essere usata. La sezione è mantenuta per completezza storica.

La proposta consisteva nell'osservare (observe, cioè monitorare) le modifiche a un oggetto tramite una callback, senza dover scrivere wrapper o usare framework. Esistevano sei tipi di evento predefiniti: `add`, `update`, `delete`, `reconfigure`, `setPrototype`, `preventExtensions`.

```js
var obj = { a: 1, b: 2 };

Object.observe(
    obj,
    function(changes) {
        for (var change of changes) {
            console.log(change);
        }
    },
    ['add', 'update', 'delete']  // filtro: solo questi tipi
);

obj.c = 3;    // { name: 'c', object: obj, type: 'add' }
obj.a = 42;   // { name: 'a', object: obj, type: 'update', oldValue: 1 }
delete obj.b; // { name: 'b', object: obj, type: 'delete', oldValue: 2 }
```

A differenza dei Proxy (capitolo 7), che intercettano le operazioni *prima* che avvengano, `Object.observe()` notificava *dopo* la modifica — alla fine del ciclo evento corrente.

### Custom change events

Tramite `Object.getNotifier()` era possibile emettere eventi personalizzati:

```js
function observer(changes) {
    for (var change of changes) {
        if (change.type == 'recalc') {
            change.object.c = change.object.oldValue + change.object.a + change.object.b;
        }
    }
}

var obj = { a: 1, b: 2, c: 3 };
Object.observe(obj, observer, ['recalc']);

var notifier = Object.getNotifier(obj);
obj.a = 6;
obj.b = 33;
notifier.notify({ type: 'recalc', name: 'c', oldValue: obj.c });

/* dopo la consegna degli eventi: obj.c === 42 */
```

Per interrompere l'osservazione si usava `Object.unobserve(obj, observer)`.

---

## Operatore di esponenziazione

ES2016 ha aggiunto l'operatore `**` come abbreviazione di `Math.pow()`:

```js
var a = 2;

a ** 4;  // equivale a Math.pow(a, 4) → 16

a **= 3; // a = Math.pow(a, 3)
a;       // 8
```

La sintassi è coerente con Python, Ruby e Perl. Supporta anche la forma di assegnazione composta `**=`.

---

## Spread/rest su oggetti

L'operatore `...` era già definito per array in ES6. La proposta post-ES6 lo estende agli oggetti.

**Spread** — unire proprietà di più oggetti:

```js
var o1 = { a: 1, b: 2 },
    o2 = { c: 3 },
    o3 = { ...o1, ...o2, d: 4 };

console.log(o3.a, o3.b, o3.c, o3.d); // 1 2 3 4
```

**Rest** — raccogliere le proprietà rimanenti dopo una destructuring:

```js
var o1 = { b: 2, c: 3, d: 4 };
var { b, ...o2 } = o1;

console.log(b, o2.c, o2.d); // 2 3 4
// o2 contiene { c: 3, d: 4 } — tutto tranne b
```

Entrambe le forme sono diventate standard con ES2018 (object spread/rest).

---

## Array#includes()

Prima di `includes()`, verificare la presenza di un valore in un array richiedeva di interpretare il risultato numerico di `indexOf()`:

```js
var vals = ['foo', 'bar', 42, 'baz'];

if (vals.indexOf(42) >= 0) { /* trovato */ }

// oppure con il trick della bitwise NOT:
if (~vals.indexOf(42)) { /* trovato */ }
```

Entrambe le forme sono indirette: `indexOf()` restituisce un indice, non un booleano. Peggio, non trova i valori `NaN` nell'array.

`Array#includes()` risolve entrambi i problemi:

```js
var vals = ['foo', 'bar', 42, 'baz'];
if (vals.includes(42)) { /* trovato */ }

// trova anche NaN:
[1, NaN, 3].includes(NaN); // true
[1, NaN, 3].indexOf(NaN);  // -1 (non trovato — limite di indexOf)
```

> `includes()` non distingue tra `+0` e `-0` (li considera uguali). Se questa distinzione è rilevante, si può usare `Object.is()` (capitolo 6) per una ricerca manuale.

---

## SIMD

SIMD (Single Instruction, Multiple Data — un'istruzione su più dati contemporaneamente) espone istruzioni CPU di basso livello che operano su vettori di numeri in parallelo, anziché su un valore alla volta.

```js
var v1 = SIMD.float32x4(3.14159, 21.0, 32.3, 55.55);
var v2 = SIMD.float32x4(2.1, 3.2, 4.3, 5.4);

SIMD.float32x4.mul(v1, v2);
// [ 6.597..., 67.2, 138.89, 299.97 ]
// — moltiplicazione di tutti e 4 gli elementi in una sola operazione
```

Oltre a `mul()`, l'API includeva `sub()`, `div()`, `abs()`, `neg()`, `sqrt()` e molte altre. Il parallelismo sui dati che ne deriva è critico per applicazioni JS ad alte prestazioni come simulazioni fisiche, grafica 3D o DSP.

> SIMD.js è stata successivamente ritirata dal processo TC39 (2017) a favore di WebAssembly, che offre capacità equivalenti con approccio più flessibile.

---

## WebAssembly (WASM)

WebAssembly rappresenta forse la novità più significativa dell'era post-ES6: un formato binario che i motori JavaScript possono eseguire *senza* doverlo prima passare attraverso il parser JS.

### Il problema: tutti i percorsi passavano per JS

Fino all'annuncio di WASM, qualunque linguaggio volesse girare nel browser doveva essere compilato in JavaScript (o un suo subset come asm.js). Mozilla aveva introdotto asm.js come sottoinsieme di JS altamente ottimizzabile, capace di raggiungere prestazioni vicine al C nativo. Ma asm.js restava comunque testo JS da parsare.

### Come funziona WASM

WASM propone un formato di rappresentazione binaria di un AST (Abstract Syntax Tree — la struttura ad albero che descrive il codice) altamente compresso. L'engine riceve questo binario e lo esegue direttamente, saltando il parsing JS. Linguaggi come C, C++ o Rust possono essere compilati direttamente in WASM, guadagnando un ulteriore vantaggio in velocità rispetto alla via asm.js.

```
C/C++ → compilatore → .wasm (binario) → JS engine (esecuzione diretta)
C/C++ → compilatore → asm.js (testo)  → JS engine (parsing + esecuzione)
```

### WASM non sostituisce JS

Il punto fondamentale è che WASM non è un sostituto di JavaScript. I due coesistono e si integrano:

- Il codice JS e il codice WASM possono chiamarsi reciprocamente come normali moduli.
- JS rimarrà il linguaggio del web per tutto ciò che è già scritto in JS.
- WASM diventerà il target preferito per le parti dell'applicazione che richiedono massime prestazioni, scritte in linguaggi a tipizzazione forte come C++ o Rust.

L'effetto netto è che WASM allontana dalla specifica JS la pressione a introdurre funzionalità radicali (come i thread), che potranno invece trovare spazio come estensioni WASM senza destabilizzare l'ecosistema JS esistente.

---

## ⚡ Ripasso veloce

**`async function` / `await`:** sugar sintattico per generators + promises + run(). Sostituisce `function*` e `yield`. Restituisce una Promise. Limite: nessuna cancellazione nativa.

```js
async function main() {
    const result = await fetch('/api/data');
    return result.json();
}
main().then(console.log);
```

**Esponenziazione:** `a ** n` = `Math.pow(a, n)`. Forma composta: `a **= 3`.

**Object spread/rest:** `{ ...o1, ...o2, extra: 1 }` per unire; `var { a, ...rest } = obj` per raccogliere il resto.

**`Array#includes()`:** restituisce un booleano, trova `NaN` (a differenza di `indexOf`), non distingue `+0` / `-0`.

```js
[1, NaN, 3].includes(NaN); // true
[1, NaN, 3].indexOf(NaN);  // -1
```

**SIMD:** operazioni vettoriali su più numeri in parallelo (`float32x4`, `int32x4`). Proposta poi ritirata a favore di WASM.

**WebAssembly:** binario eseguito dall'engine JS senza parsing. Complemento a JS per codice ad alte prestazioni scritto in altri linguaggi. Non rimpiazza JS.

**`Object.observe()`:** proposta ritirata. Usare invece Proxy (capitolo 7) o librerie di state management.

---

## Domande

<details>
<summary>Qual è la relazione tra <code>async function</code>/<code>await</code> e il pattern generators + promises?</summary>

`async function` è zucchero sintattico che il runtime espande nel pattern generator + Promise + runner. `async function` corrisponde a `function*`, `await` corrisponde a `yield` di una Promise. La differenza principale è che non occorre un'utility esterna come `run()`: l'engine gestisce automaticamente l'attesa della Promise prima di riprendere l'esecuzione. La funzione restituisce direttamente una Promise osservabile con `.then()`.

</details>

<details>
<summary>Perché <code>Array#includes()</code> è preferibile a <code>indexOf()</code> per verificare la presenza di un valore?</summary>

`indexOf()` restituisce un indice numerico che richiede un confronto esplicito (`>= 0` oppure `~`) per ottenere un booleano, ed è semanticamente indiretto. Inoltre, `indexOf()` non è in grado di trovare `NaN` nell'array perché usa l'uguaglianza stretta (`===`), e `NaN !== NaN`. `includes()` invece restituisce direttamente `true`/`false` e usa un algoritmo di confronto che rileva correttamente `NaN`. L'unico caso in cui `includes()` diverge dall'intuizione è che non distingue `+0` da `-0`.

</details>

<details>
<summary>Cosa differenzia WebAssembly da asm.js, e quale ruolo ha rispetto a JavaScript?</summary>

asm.js è un sottoinsieme testuale di JavaScript ottimizzabile, ma richiede comunque che l'engine effettui il parsing del testo JS. WebAssembly è un formato binario compresso che rappresenta l'AST del codice e viene passato direttamente all'engine senza parsing, guadagnando ulteriore velocità. Entrambi permettono a linguaggi come C/C++ di girare nel browser con prestazioni vicine al nativo. WASM non sostituisce JavaScript: i due coesistono e si chiamano reciprocamente. WASM diventa il target preferito per parti ad alte prestazioni di applicazioni, mentre JS rimane il linguaggio principale per tutto ciò che è già scritto in JS.

</details>

<details>
<summary>Qual è il caveat principale delle <code>async function</code> rispetto alle Promise ordinarie?</summary>

Un'`async function` restituisce una Promise ordinaria. Dal momento che le Promise non hanno un meccanismo di cancellazione, non è possibile interrompere l'esecuzione di una funzione async dall'esterno una volta avviata. Se l'operazione è costosa in termini di risorse e il risultato non serve più (ad esempio, una richiesta XHR di cui non si vuole il risultato), non esiste un `pr.cancel()` nativo. Le soluzioni proposte includevano token di cancellazione passati come argomento o tipi di ritorno alternativi, ma nessuna era definitiva al momento della scrittura del libro.

</details>

<details>
<summary>Come si usano spread e rest con gli oggetti, e in cosa differiscono dall'uso con gli array?</summary>

Con gli array, `...` in spread copia gli elementi in una nuova struttura, mentre in rest raccoglie gli elementi rimanenti in un array. Con gli oggetti la semantica è analoga ma opera sulle proprietà enumerabili: `{ ...o1, ...o2, extra: 1 }` crea un nuovo oggetto con tutte le proprietà di `o1` e `o2` più `extra`. In rest, `var { a, ...resto } = obj` estrae `a` e raccoglie le proprietà rimanenti in `resto`. La differenza chiave rispetto agli array è che le proprietà duplicate vengono risolte con l'ultima che sovrascrive le precedenti, e l'ordine di enumerazione segue le regole degli oggetti.

</details>
