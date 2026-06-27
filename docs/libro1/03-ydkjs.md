# Introduzione a YDKJS

Questo capitolo è una mappa della serie: per ogni titolo viene anticipato l'argomento centrale e il punto di vista con cui viene affrontato. Non è una sintesi — è un orientamento su cosa aspettarsi e perché vale la pena affrontarlo.

## Scope & Closures

Il punto di partenza è smontare un luogo comune: JavaScript non è un linguaggio interpretato nel senso tradizionale. Il JavaScript engine compila il codice immediatamente prima (e a volte durante) l'esecuzione. Questo non è un dettaglio tecnico irrilevante — capire come funziona il compiler è il prerequisito per capire come JavaScript trova e gestisce variabili e function declaration, e quindi per capire il **lexical scope** e l'**hoisting**.

La comprensione del lexical scope è, a sua volta, il prerequisito indispensabile per capire la **closure**. La closure non è una funzionalità opzionale o avanzata: è forse il concetto più importante dell'intero linguaggio. Tentare di studiarla senza aver prima consolidato lo scope significa quasi certamente non afferrarla davvero.

Un'applicazione diretta e molto diffusa della closure è il **module pattern**, introdotto brevemente nel capitolo precedente. È probabilmente il pattern di organizzazione del codice più usato in JavaScript, e comprenderne le fondamenta in profondità dovrebbe essere una priorità.

## this & Object Prototypes

Il fraintendimento più radicato e persistente su JavaScript riguarda il keyword `this`: in molti credono che si riferisca alla funzione in cui appare. Non è così.

`this` viene determinato dinamicamente in base a **come** la funzione viene chiamata, non a dove è definita. Esistono quattro regole precise che permettono di determinarlo sempre, in qualsiasi situazione.

Strettamente legato a `this` c'è il meccanismo del **prototype** — una catena di ricerca per le proprietà degli oggetti, analoga al lexical scope per le variabili. Il principale equivoco legato al prototype è il tentativo di usarlo per emulare classi ed ereditarietà di altri linguaggi. La sintassi sembra suggerire che esistano classi in JavaScript, ma il meccanismo del prototype è fondamentalmente diverso — e opposto per comportamento — all'ereditarietà classica.

L'alternativa proposta dalla serie non è solo una preferenza sintattica: il **behavior delegation** è un pattern di design radicalmente diverso, più potente, che non richiede di simulare classi o ereditarietà. Questa posizione va controcorrente rispetto a quasi tutta la letteratura JavaScript esistente, ma parte da una comprensione del linguaggio così com'è, non come vorremmo che fosse.

## Types & Grammar

Il terzo titolo della serie affronta un argomento che genera più frustrazione di quasi tutti gli altri: la coercizione di tipo. L'opinione dominante è che la coercizione implicita sia una caratteristica difettosa del linguaggio da evitare sempre. Esistono persino strumenti il cui unico scopo è segnalare qualsiasi uso della coercizione nel codice.

La tesi del libro è l'opposta: la coercizione non è un difetto e non è imprevedibile — è un meccanismo con regole precise che, una volta apprese, risulta non solo comprensibile ma utile. Dopo aver compreso come funzionano davvero i tipi e i valori, emerge che la coercizione implicita — usata correttamente — può rendere il codice più leggibile, non meno.

L'obiettivo non è convincere a usare ciecamente la coercizione ovunque, ma a non escluderla per abitudine o per conformarsi a convenzioni non ragionate.

## Async & Performance

I primi tre titoli della serie trattano la meccanica del linguaggio. Il quarto si sposta verso i pattern che governano la **programmazione asincrona** — un aspetto sempre più centrale non solo per le prestazioni, ma per la leggibilità e manutenibilità del codice.

Il libro comincia chiarendo la terminologia: "async", "parallelo" e "concorrente" non sono sinonimi e si applicano in modo diverso a JavaScript.

I **callback** sono il meccanismo principale di asincronia in JavaScript, ma da soli si rivelano insufficienti per le esigenze del codice moderno. I due problemi fondamentali sono:

- **Inversion of Control (IoC)** — quando si passa un callback a una funzione di terze parti, si perde il controllo su come e quando quel callback verrà chiamato.
- **Non-linearità** — il flusso del codice con callback annidati non si legge in modo sequenziale, rendendo difficile seguire la logica del programma.

ES6 introduce due meccanismi per risolvere questi problemi: le **promise** e i **generator**.

Le promise sono un wrapper intorno a un "valore futuro" che permette di ragionare sul risultato di un'operazione asincrona indipendentemente dal fatto che sia già disponibile o meno. Risolvono il problema dell'IoC instradando i callback attraverso un meccanismo affidabile e componibile.

I generator introducono una nuova modalità di esecuzione: una funzione può essere **sospesa** su punti di `yield` e ripresa in seguito. Questo permette di scrivere codice asincrono con struttura sincrona e sequenziale, eliminando la non-linearità tipica dei callback.

La combinazione di promise e generator rappresenta, al momento della serie, il pattern più efficace per la gestione dell'asincronia in JavaScript — e costituisce le fondamenta su cui sono state costruite funzionalità successive come `async/await`.

Il libro copre anche ottimizzazione delle prestazioni: parallelismo con Web Workers, parallelismo sui dati con SIMD, tecniche di benchmarking corrette e le differenze tra microperformance e prestazioni reali dell'applicazione.

## ES6 & Beyond

JavaScript non smette di evolversi, e il ritmo di evoluzione sta accelerando. Questo titolo è dedicato alle funzionalità introdotte con ES6 e a quelle prevedibili nelle versioni successive — non solo come catalogo, ma con l'attenzione al "perché" di ogni aggiunta.

Le aree principali coperte da ES6 sono:

- **Nuova sintassi** — destructuring, default parameter values, template literal, arrow function, `let`/`const`, for..of
- **Nuove strutture dati** — Map, Set, WeakMap, WeakSet, TypedArray
- **Organizzazione del codice** — moduli nativi, classi, generator, iteratori
- **Nuove API** — aggiunte a `Array`, `Object`, `Math`, `Number`, `String`
- **Metaprogrammazione** — Proxy, Reflect, Symbol

La prospettiva del libro è che conoscere le funzionalità ES6 non basta: è necessario capire cosa risolvono e come si integrano con i meccanismi profondi del linguaggio già trattati nei titoli precedenti.

---

## ⚡ Ripasso veloce

**Scope & Closures**: JS non è interpretato — il codice viene compilato JIT. Il lexical scope è il prerequisito per capire la closure. La closure è il concetto più importante del linguaggio. Il module pattern è la sua applicazione più diffusa.

**this & Object Prototypes**: `this` dipende da come la funzione viene chiamata, non da dove è definita. Il prototype è una catena di ricerca per le proprietà — non un meccanismo di ereditarietà. Il behavior delegation è l'alternativa naturale alle classi simulate.

**Types & Grammar**: la coercizione implicita ha regole precise e può essere uno strumento utile, non un difetto da evitare sempre.

**Async & Performance**: i callback da soli hanno due limiti strutturali (IoC e non-linearità). Promise e generator li risolvono. La loro combinazione è la base di `async/await`.

**ES6 & Beyond**: nuova sintassi, nuove strutture dati, moduli nativi e metaprogrammazione. Le funzionalità ES6 si capiscono meglio conoscendo i fondamentali del linguaggio.

---

## Domande

<details>
<summary>Perché la serie si chiama "You Don't Know JS" e qual è il suo scopo?</summary>

Il titolo non è una critica ma una constatazione: JavaScript è un linguaggio che si può usare senza capirlo davvero, e spesso è proprio questo il percorso seguito dalla maggior parte degli sviluppatori. La serie nasce per affrontare questa tendenza — non accontentarsi di un sottoinsieme "sicuro" del linguaggio, ma comprenderne a fondo i meccanismi, incluse le parti più difficili o controverse. L'obiettivo è una comprensione solida, non solo operativa.

</details>

<details>
<summary>Quali sono i due problemi strutturali dei callback che promise e generator risolvono?</summary>

Il primo è l'**Inversion of Control (IoC)**: quando si passa un callback a codice di terze parti, si cede il controllo su quando, quante volte e con quale argomento quel callback verrà invocato — non c'è garanzia di comportamento corretto. Il secondo è la **non-linearità**: il codice con callback annidati non si legge in sequenza, rendendo difficile seguire il flusso logico e ragionare sugli errori. Le promise gestiscono l'IoC attraverso un contratto affidabile; i generator permettono di scrivere codice asincrono con struttura sequenziale.

</details>

<details>
<summary>Perché il behavior delegation è considerato superiore all'emulazione delle classi tramite prototype?</summary>

Il prototype di JavaScript non è un meccanismo di ereditarietà nel senso classico: gli oggetti non copiano comportamento da un prototype, ma delegano la ricerca delle proprietà a un altro oggetto nella catena. Usare il prototype per simulare classi significa combattere contro la natura del linguaggio, introducendo complessità e ambiguità non necessarie. Il behavior delegation abbraccia invece il modello reale: oggetti che delegano esplicitamente responsabilità ad altri oggetti, senza finzione di classi o gerarchie di ereditarietà.

</details>

<details>
<summary>In che senso JavaScript è un linguaggio "compilato" se viene spesso descritto come interpretato?</summary>

La distinzione tradizionale tra linguaggi compilati (traduzione anticipata in codice macchina) e interpretati (esecuzione riga per riga) non descrive accuratamente JavaScript. Il JS engine analizza e compila il codice sorgente in bytecode ottimizzato immediatamente prima di eseguirlo — processo noto come just-in-time compilation (JIT). Questa compilazione è reale, non una metafora: implica parsing, analisi sintattica, ottimizzazioni e generazione di codice. Capirlo cambia come si pensa all'hoisting, allo scope e al comportamento dell'engine in fase di esecuzione.

</details>
