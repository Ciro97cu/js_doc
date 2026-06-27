# ES6: ora e futuro

Questo capitolo introduce il contesto in cui si colloca ES6 (ECMAScript 2015): perché è un salto radicale rispetto a ES5, come cambia il modo di intendere l'evoluzione del linguaggio, e quali strumenti permettono di usare le nuove funzionalità oggi senza aspettare che tutti i browser le supportino.

---

## Versioning

Lo standard JavaScript si chiama ufficialmente **ECMAScript** (abbreviato ES). La versione ES3 è stata il primo baseline diffuso (IE6–8, Android 2.x). ES4 non è mai arrivato per ragioni politiche. ES5 (2009) è stato la base del JS moderno per anni.

ES6 — tecnicamente ES2015 — è un'evoluzione radicale, non un insieme incrementale di piccole aggiunte come ES5. Introduce nuove forme sintattiche, nuovi pattern organizzativi, nuove API per i tipi di dati esistenti.

A partire da ES6 il comitato TC39 ha adottato un modello di versioning annuale (ES2016, ES2017, …). Più importante ancora, il ritmo di evoluzione è diventato così rapido che i browser implementano le funzionalità man mano che si stabilizzano, prima ancora che la specifica sia formalmente approvata. Il risultato pratico è che **JS si avvicina sempre più a uno standard "evergreen"**: invece di aspettare che una versione sia completa, vale la pena ragionare funzionalità per funzionalità. L'etichetta di versione conta sempre meno; conta se una specifica feature è supportata nell'ambiente target.

---

## Transpiling

Il problema: si vuole usare ES6 oggi, ma molti ambienti supportano solo ES5. La soluzione non è aspettare — aspettare significa restare anni indietro sulla curva.

La soluzione è il **transpiling** (contrazione di *transformation* + *compiling*): uno strumento che trasforma il codice ES6 in codice ES5 equivalente o molto vicino.

Esempio — shorthand property in un object literal:

```js
/* ES6 */
var foo = [1, 2, 3];
var obj = { foo };      // shorthand: equivale a { foo: foo }

/* Transpilato in ES5 */
var foo = [1, 2, 3];
var obj = { foo: foo };
```

Il transpiler (es. Babel) esegue queste trasformazioni automaticamente come step del build workflow, accanto a linting e minificazione. Il codice sorgente rimane moderno ES6+; l'output distribuito è ES5 compatibile con ambienti più vecchi.

Il transpiling copre **nuova sintassi** (arrow function, destructuring, template literals, class, generator, ecc.) che non può essere emulata aggiungendo semplicemente del codice runtime.

---

## Shims e polyfill

Non tutto ciò che è nuovo richiede un transpiler. Le **nuove API** — funzioni, metodi, classi — spesso possono essere **polyfillate** (o "shimmate"): si definisce in ES5 un comportamento equivalente a quello che un engine ES6 offre nativamente.

Esempio — `Object.is()`, una utility che controlla l'uguaglianza stretta senza le eccezioni di `===` per `NaN` e `-0`:

```js
if (!Object.is) {
    Object.is = function(v1, v2) {
        /* test per -0: 1/0 === Infinity, 1/-0 === -Infinity */
        if (v1 === 0 && v2 === 0) {
            return 1 / v1 === 1 / v2;
        }
        /* test per NaN: unico valore non uguale a sé stesso */
        if (v1 !== v1) {
            return v2 !== v2;
        }
        return v1 === v2;
    };
}
```

Il guard `if (!Object.is)` è essenziale: il polyfill non sovrascrive mai un'implementazione nativa già presente. Se il browser supporta `Object.is()`, si usa quella.

La distinzione fondamentale:

| | Transpiler | Polyfill |
|---|---|---|
| Cosa copre | Nuova **sintassi** | Nuove **API** |
| Quando serve | Build time | Runtime |
| Esempi | arrow fn, class, destructuring | `Promise`, `Array.from`, `Object.is` |

La strategia corretta è usare entrambi: un transpiler nel workflow di build per la sintassi, e una libreria di polyfill (come ES6 Shim) inclusa nel runtime per le API.

---

## ⚡ Ripasso veloce

- **ES6 / ES2015** — salto radicale, non incrementale; introduce sintassi nuova, non solo API
- **Versioning evergreen** — il ritmo di evoluzione è per-feature, non per-versione; ragionare funzionalità per funzionalità
- **Transpiler** (es. Babel) — trasforma sintassi ES6+ in ES5 equivalente a build time
- **Polyfill** — implementa in ES5 nuove API che i vecchi ambienti non hanno; sempre con guard `if (!API)`
- Aspettare che tutti i browser supportino ES6 = restare anni indietro

```js
/* polyfill pattern: controlla prima, definisci solo se mancante */
if (!Object.is) {
    Object.is = function(v1, v2) {
        if (v1 === 0 && v2 === 0) return 1/v1 === 1/v2;
        if (v1 !== v1) return v2 !== v2;
        return v1 === v2;
    };
}

/* Object.is gestisce NaN e -0 correttamente */
Object.is(NaN, NaN);    // true  (=== dà false)
Object.is(0, -0);       // false (=== dà true)
Object.is(1, 1);        // true
```

---

## Domande

<details>
<summary>Qual è la differenza tra transpiling e polyfilling, e quando serve ciascuno?</summary>

Il transpiling agisce sulla **sintassi**: trasforma costrutti ES6+ (arrow function, destructuring, class, generator) in codice ES5 equivalente a build time. La nuova sintassi non può essere emulata a runtime perché il parser JS resterebbe bloccato su token sconosciuti prima ancora di eseguire qualcosa. Il polyfilling agisce sulle **API**: aggiunge a runtime funzioni, metodi o classi che il vecchio ambiente non ha (`Promise`, `Array.from`, `Object.assign`, ecc.). Le API sono normali valori JS — si possono definire con codice ES5. La strategia corretta è usarli entrambi: Babel (o simili) nel build per la sintassi, una libreria di shim come ES6 Shim inclusa nel bundle per le API.

</details>

<details>
<summary>Perché il guard <code>if (!Object.is)</code> è importante in un polyfill?</summary>

Il polyfill definisce un comportamento di fallback per ambienti che non hanno ancora l'API. Se il browser la supporta già nativamente, l'implementazione nativa è sempre preferibile: è ottimizzata dall'engine, aggiornata con la specifica, e garantita corretta. Senza il guard, il polyfill sovrascrive silenziosamente l'implementazione nativa, anche in browser moderni che già la supportano correttamente — potenzialmente introducendo regressioni o prestazioni peggiori. Il pattern corretto è quindi sempre: controlla se l'API esiste, definiscila solo se è assente.

</details>

<details>
<summary>Perché aspettare che tutti i browser supportino ES6 è considerato un approccio sbagliato?</summary>

Perché storicamente ha causato un ritardo di anni nell'adozione di funzionalità già stabili. ES5 è stato finalizzato nel 2009 ma molti codebase hanno iniziato ad adottare `strict mode` e altri costrutti ES5 solo intorno al 2015 — sei anni dopo. Con il ritmo attuale di evoluzione di JS, attendere significherebbe sempre essere in ritardo di una o più generazioni di funzionalità. I transpiler (per la sintassi) e i polyfill (per le API) risolvono esattamente questo problema: permettono di scrivere codice moderno oggi, distribuendo output compatibile con ambienti più vecchi. La curva di adozione non deve più dipendere dal browser più lento — dipende dalla toolchain.

</details>

<details>
<summary>Cosa significa che JavaScript si sta evolvendo verso uno standard "evergreen"?</summary>

Significa che il modello di versioning monolitico (ES3 → ES5 → ES6 con anni tra una versione e l'altra) sta cedendo il posto a un'evoluzione continua per feature. Le proposte TC39 passano attraverso stage (0→4) e i browser iniziano a implementare le feature già agli stage 3-4, molto prima che la specifica annuale sia formalmente approvata. In pratica, quando una feature raggiunge la stabilità nella specifica, è già disponibile nelle versioni recenti dei principali browser. Questo significa che l'etichetta "ES2020" o "ES2022" diventa meno rilevante dell'elenco delle feature specifiche supportate nell'ambiente target — e gli strumenti come Babel e `@babel/preset-env` già operano esattamente in questo modo, configurabili per target browser, non per versione ES.

</details>
