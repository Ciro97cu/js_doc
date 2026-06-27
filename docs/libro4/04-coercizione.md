# Coercizione

La **coercion** (coercizione) è la conversione di un valore da un tipo a un altro. In JavaScript avviene in modo pervasivo, ed è uno dei meccanismi più fraintesi del linguaggio. Non è un difetto: è un sistema con regole precise che, capite a fondo, risulta utile e prevedibile.

Si distinguono due forme: **coercion esplicita**, quando la conversione è intenzionale e visibile nel codice, e **coercion implicita**, quando avviene come effetto collaterale di un'altra operazione.

---

## Operazioni astratte di conversione

Prima di analizzare le forme esplicite e implicite, è utile conoscere le operazioni interne che la specifica usa per convertire valori: `ToString`, `ToNumber`, `ToBoolean`.

### `ToString`

Converte qualsiasi valore alla sua rappresentazione stringa. Per i primitivi il risultato è intuitivo: `null` → `"null"`, `undefined` → `"undefined"`, `true` → `"true"`, `42` → `"42"`. I numeri molto grandi o molto piccoli usano la notazione esponenziale: `1.07e21`.

Per gli oggetti, `ToString` invoca il metodo `.toString()`. Gli array usano un algoritmo custom: uniscono gli elementi con virgola e stringificano i valori `null`/`undefined` come stringa vuota.

```js
String([1, 2, 3]);    // "1,2,3"
String([null, undefined]); // ","  ← entrambi diventano ""
```

**`JSON.stringify()`** è correlato ma non identico a `ToString`. Serializza valori a JSON, ma gestisce diversamente alcuni casi:

```js
JSON.stringify(undefined);              // undefined (non "undefined")
JSON.stringify(function(){});           // undefined
JSON.stringify({ a: 42, b: undefined }); // '{"a":42}' — undefined escluso
JSON.stringify([1, undefined, 3]);       // '[1,null,3]' — undefined→null negli array
```

Se un oggetto ha un metodo `.toJSON()`, viene invocato prima della serializzazione per ottenere il valore da stringificare. Il secondo parametro di `JSON.stringify()` è un **replacer** (array di chiavi da includere, o funzione di filtraggio); il terzo è l'indentazione.

### `ToNumber`

Converte qualsiasi valore a un numero:

- `true` → `1`, `false` → `0`
- `null` → `0`, `undefined` → `NaN`
- `""` → `0`, `"3"` → `3`, `"0x1f"` → `31` (hex)
- stringhe con whitespace ai bordi vengono trimmati prima della conversione
- stringhe non parsabili → `NaN`

Per oggetti e array, `ToNumber` chiama prima `ToPrimitive`, che tenta `valueOf()` e poi `toString()`. Se il risultato è un primitivo, viene applicata `ToNumber` su quello.

```js
Number("");       // 0
Number("  42  "); // 42
Number("0x1f");   // 31
Number(null);     // 0
Number(undefined); // NaN
Number([]);       // 0   ← [] → "" → 0
Number([3]);      // 3   ← [3] → "3" → 3
Number(["a"]);    // NaN
```

### `ToBoolean`

Il funzionamento di `ToBoolean` è semplice: **esiste una lista fissa di valori falsy**; tutto il resto è truthy.

**Valori falsy** (gli unici che producono `false`):
- `undefined`
- `null`
- `false`
- `+0`, `-0`, `NaN`
- `""` (stringa vuota)

Qualsiasi altro valore — inclusi `"0"`, `"false"`, `[]`, `{}`, funzioni — è **truthy**.

```js
Boolean("0");       // true — stringa non vuota
Boolean([]);        // true — oggetto (sempre truthy)
Boolean(new Boolean(false)); // true — il wrapper è un oggetto
```

**Falsy objects**: esistono valori che si comportano come falsy pur essendo oggetti — tipicamente un'eredità dei browser (come `document.all` in ambienti legacy). Non fanno parte dello standard JS ma della specifica host. Da non confondere con la regola generale.

---

## Coercion esplicita

### Da valore a stringa

```js
var a = 42;
String(a);   // "42" — invoca ToString, nessun new
a.toString(); // "42" — metodo sul valore (autoboxing)
```

`String()` e `.toString()` sono entrambi espliciti. `.toString()` accetta un parametro opzionale per la base numerica:

```js
(255).toString(16); // "ff"
(255).toString(2);  // "11111111"
```

### Da valore a numero

```js
Number("3.14");   // 3.14
Number(true);     // 1
Number(false);    // 0
Number(null);     // 0
Number(undefined); // NaN

/* operatore unario + — equivalente ma implicito nel senso della visibilità */
+"3.14"; // 3.14
+true;   // 1
+"";     // 0
```

L'operatore unario `+` converte il valore a numero. È comunemente usato per convertire oggetti `Date` in timestamp numerico:

```js
var d = new Date();
+d; // timestamp numerico — equivalente a d.getTime()
```

### Da data a numero

```js
var timestamp = +new Date(); // forma concisa
var timestamp = new Date().getTime(); // forma verbosa ma più chiara
var timestamp = Date.now(); // ES5 — la preferita
```

### `~` — tilde e il pattern indexOf

L'operatore `~` (bitwise NOT) converte prima il suo operando a intero a 32 bit (via `ToInt32`) e poi inverte tutti i bit. L'effetto pratico: `~n` equivale a `-(n+1)`. Il caso speciale è `-1`: `~(-1) === 0`.

Questo è utile con `indexOf()`, che restituisce `-1` come sentinella di "non trovato":

```js
var a = "Hello World";

/* idioma tradizionale */
if (a.indexOf("lo") >= 0) { /* trovato */ }
if (a.indexOf("lo") === -1) { /* non trovato */ }

/* con ~ : -1 diventa 0 (falsy), qualsiasi altro indice diventa truthy */
if (~a.indexOf("lo")) { /* trovato */ }
if (!~a.indexOf("lo")) { /* non trovato */ }
```

`~~` (doppio tilde) è usato talvolta come alternativa a `Math.trunc()` per troncare i decimali (ma funziona solo su numeri nel range dei 32 bit):

```js
~~3.14; // 3
~~-3.14; // -3
Math.floor(-3.14); // -4 — diverso! floor arrotonda verso -∞
```

### `parseInt()` vs coercion numerica

`parseInt()` è tollerante: estrae il numero da una stringa che inizia con caratteri numerici, ignorando il resto. `Number()` è stretto: se la stringa contiene qualsiasi carattere non numerico, restituisce `NaN`.

```js
parseInt("42px");  // 42 — tollerante
Number("42px");    // NaN — stretto

parseInt("08");    /* in pre-ES5 poteva interpretare come ottale → 0! */
parseInt("08", 10); // 42 sempre specificare la base
```

`parseInt()` opera su stringhe: se il primo argomento non è una stringa, viene convertito a stringa prima. Questo produce curiosità come `parseInt(0.000008)` → `parseInt("8e-7")` → `8`.

### Da valore a booleano

```js
Boolean(0);   // false
Boolean("");  // false
Boolean(null); // false
Boolean({}); // true

/* ! e !! — forma compatta */
!0;    // true
!!0;   // false — doppia negazione → boolean esplicito
!!""; // false
!!"0"; // true
```

`!!` è l'idioma standard per coercere a booleano in modo esplicito e leggibile.

---

## Coercion implicita

La coercion implicita avviene quando il linguaggio converte un valore automaticamente come parte di un'operazione. È "implicita" perché non è il principale intento dell'espressione.

### `+` con stringhe

L'operatore `+` ha doppia funzione: addizione numerica e concatenazione di stringhe. Se **almeno uno** degli operandi è una stringa (o viene convertito a stringa via `ToPrimitive`), l'operazione diventa concatenazione:

```js
42 + "0";   // "420" — concatenazione
42 + 0;     // 42    — addizione
"1" + "2";  // "12"
[] + {};    // "[object Object]"
{} + [];    // 0 — {} interpretato come blocco vuoto, + [] → 0
```

Pattern comune per convertire un numero a stringa: `n + ""`. Equivalente a `String(n)` ma con una differenza: `n + ""` chiama `valueOf()` sull'oggetto e poi stringifica il primitivo risultante; `String(n)` chiama `.toString()` direttamente.

```js
var a = { valueOf: () => 42, toString: () => "4" };
a + "";       // "42" — usa valueOf
String(a);    // "4"  — usa toString
```

### `-`, `*`, `/` — conversione a numero

Gli operatori aritmetici diversi da `+` forzano sempre la conversione a numero:

```js
"3" - 1;    // 2
"6" / "2";  // 3
"5" * "3";  // 15
[3] - [1];  // 2 ← [3]→"3"→3, [1]→"1"→1
```

Pattern per convertire stringa a numero: `a - 0` (o `a * 1`).

### `||` e `&&` come selettori di operandi

In JavaScript, `||` e `&&` **non restituiscono un booleano** — restituiscono uno dei due operandi. Sono "selettori":

- `a || b`: se `a` è truthy, restituisce `a`; altrimenti restituisce `b`
- `a && b`: se `a` è falsy, restituisce `a`; altrimenti restituisce `b`

```js
42 || "abc";  // 42
null || "abc"; // "abc"
42 && "abc";  // "abc"
null && "abc"; // null
```

`||` è idiomaticamente usato per valori di default:

```js
function foo(a, b) {
    a = a || "default_a"; // ATTENZIONE: falsy-value gotcha
    b = b || 42;
}
```

Il gotcha: se `a` è `0`, `""` o `false` — valori falsy ma intenzionali — verranno rimpiazzati dal default. In ES6, i parametri di default (`function foo(a = "default_a")`) risolvono questo problema.

`&&` è spesso usato per il **guard pattern**: eseguire qualcosa solo se il primo operando è truthy.

```js
if (foo) foo.bar(); /* idioma classico */
foo && foo.bar();   /* idioma &&-guard — equivalente */
```

### Coercion booleana implicita

Si attiva nei contesti che richiedono un booleano: `if`, il test del `while`/`for`, la condizione del ternario `? :`, e l'operando sinistro di `||`/`&&`. In tutti questi contesti, il valore viene automaticamente convertito con `ToBoolean`.

### `Symbol` e coercion

I symbol hanno regole speciali: la coercion esplicita a stringa è permessa, ma quella implicita lancia `TypeError`. La coercion a numero non è mai permessa.

```js
var s = Symbol("ok");
String(s);   // "Symbol(ok)" — esplicito ok
s + "";      // TypeError: can't convert symbol to string
+s;          // TypeError
```

---

## `==` vs `===`

La distinzione corretta non è "== controlla solo il valore, === controlla anche il tipo". La distinzione è: **== permette la coercion, === non la permette**.

Se i tipi dei due operandi sono già uguali, `==` e `===` si comportano in modo identico (a parte le eccezioni di `NaN` e `-0`). La coercion entra in gioco solo quando i tipi sono diversi.

### Regole dell'Abstract Equality Comparison

La specifica definisce l'algoritmo per `==` con regole precise:

**Stringa == Numero**: la stringa viene convertita a numero.

```js
42 == "42"; // true ← "42" diventa 42
```

**Booleano == qualsiasi cosa**: il booleano viene convertito a numero **per primo**, prima di qualsiasi altro confronto.

```js
true == 1;  // true ← true diventa 1
false == 0; // true ← false diventa 0
true == "1"; // true ← true→1, "1"→1
true == "true"; /* false ← true→1, "true"→NaN */
```

Conseguenza: **non usare mai `== true` o `== false`**. Il confronto con `===` o un test booleano diretto è sempre preferibile.

**`null` == `undefined`**: questi due valori sono uguali tra loro con `==`, e non uguali a nessun altro valore. Questo rende il check `== null` un idioma sicuro e compatto:

```js
null == undefined; // true
null == false;     // false
null == 0;         // false
undefined == "";   // false

/* check "è null o undefined" */
var a = null;
if (a == null) { /* vero anche se a è undefined */ }
```

**Oggetto == primitivo**: l'oggetto viene convertito con `ToPrimitive` (chiama `valueOf()`, poi `toString()`) e il risultato viene confrontato con il primitivo.

```js
42 == [42];    // true ← [42]→"42"→42
"abc" == Object("abc"); // true ← Object("abc") unboxed → "abc"
```

Eccezioni: `null` e `undefined` non possono essere wrapped, quindi `Object(null) == null` è `false`.

### I 7 casi problematici

Confrontando tutti i possibili confronti tra valori falsy, emergono 7 "gotcha" reali:

```js
"0" == false;  // true — UH OH
false == 0;    // true — UH OH
false == "";   // true — UH OH
false == [];   // true — UH OH
"" == 0;       // true — UH OH
"" == [];      // true — UH OH
0 == [];       // true — UH OH
```

Quattro dei sette coinvolgono `== false` o `== true` — già esclusi dalla regola "non usare mai == con booleani". Rimangono tre casi problematici reali (`"" == 0`, `"" == []`, `0 == []`).

Altri casi apparentemente "pazzi" si spiegano facilmente:

```js
[] == ![];  // true — ![] diventa false, poi [] == false → 0 == 0
2 == [2];   // true — [2]→"2"→2
"" == [null]; // true — [null]→""
0 == "\n";  // true — "\n"→0 (whitespace trimmed→"")
```

### Regole pratiche per usare `==` in sicurezza

1. **Non usare mai `==` con `true` o `false`** — usare il test booleano diretto o `===`.
2. **Fare attenzione con `[]`, `""`, `0` su entrambi i lati** — valutare se `===` è più chiaro.
3. **`== null` è sicuro e idiomatico** — controlla sia `null` sia `undefined`.
4. **`typeof x == "function"`** è 100% equivalente a `=== "function"` — `typeof` restituisce sempre una stringa non vuota tra 7 possibilità.

```js
/* check null/undefined in un colpo — idioma corretto */
if (a == null) { /* a è null o undefined */ }

/* evitare */
if (a == false) { /* comportamento sorprendente */ }
if (a == true)  { /* idem */ }
```

La scelta tra `==` e `===` si riduce a: la coercion è utile e prevedibile in questo contesto? Se sì, usare `==`. Se no — o se ci sono dubbi — usare `===`.

---

## Confronto relazionale astratto

L'operatore `<` (e per estensione `>`, `<=`, `>=`) applica la propria forma di coercion implicita. `a > b` è trattato come `b < a`.

L'algoritmo chiama prima `ToPrimitive` su entrambi gli operandi. Se **entrambi** i risultati sono stringhe, si esegue un confronto lessicografico carattere per carattere. Altrimenti, entrambi vengono convertiti a numero.

```js
/* numerico */
var a = [42];
var b = ["43"];
a < b;  // true ← 42 < 43

/* lessicografico — entrambi array di stringhe */
var a = ["42"];
var b = ["043"];
a < b;  // false ← "4" > "0" lessicograficamente

/* oggetti — confronto impossibile */
var a = { b: 42 };
var b = { b: 43 };
a < b;  // false ← "[object Object]" == "[object Object]"
a > b;  // false
a == b; // false (riferimenti diversi)
a <= b; // true ← spec: !(b < a) → !(false) → true
a >= b; // true ← spec: !(a < b) → !(false) → true
```

Il comportamento di `<=` e `>=` sorprende: la specifica li definisce come **negazione del contrario** (`a <= b` è `!(b < a)`), non come "minore o uguale". Quando due oggetti producono la stessa stringa, `a < b` è `false`, `b < a` è `false`, quindi `a <= b` e `a >= b` sono entrambi `true`.

Non esiste un "confronto relazionale stretto" come `===` per `==`. Per essere sicuri, si coercono esplicitamente i valori prima del confronto:

```js
var a = [42];
var b = "043";
a < b;                      // false — confronto stringa!
Number(a) < Number(b);      // true  — confronto numerico esplicito
```

---

## ⚡ Ripasso veloce

**ToString**: oggetti usano `.toString()`, array uniscono con virgola. `JSON.stringify` esclude `undefined` e funzioni.

**ToNumber**: `""` → `0`, `null` → `0`, `undefined` → `NaN`. Array via ToPrimitive → stringa → numero.

**ToBoolean**: lista fissa di falsy: `undefined`, `null`, `false`, `+0/-0/NaN`, `""`. Tutto il resto è truthy.

**Coercion esplicita**: `String()`, `Number()`, `Boolean()`, `!!`, `+n`, `n - 0`.

**Coercion implicita**: `+` con una stringa → concatenazione; `-`/`*`/`/` → sempre numero; `||`/`&&` restituiscono un operando (non un booleano); contesti booleani attivano `ToBoolean`.

**`==` vs `===`**: la differenza è la coercion, non la verifica del tipo. Regole chiave:
- Stringa == Numero → stringa coerced a numero
- Booleano == qualsiasi → booleano coerced a numero PRIMA
- Oggetto == primitivo → `ToPrimitive(oggetto)`
- `null == undefined` → `true` (solo tra loro)

**I 7 gotcha** di `==` si evitano con due regole: mai `==` con booleani; attenzione a `[]`, `""`, `0`.

**Confronto relazionale**: entrambe stringhe → lessicografico; altrimenti → numerico. `<=` e `>=` sono negazione del contrario, non "minore/maggiore o uguale".

```js
/* ToNumber curiosità */
Number("");       // 0
Number(null);     // 0
Number(undefined); // NaN
Number([]);       // 0
Number([3]);      // 3

/* || e && come selettori */
var a = null;
var b = a || "default"; // "default"
var c = a && a.foo;     // null — guard pattern

/* == null — idioma sicuro */
if (a == null) { /* a è null o undefined */ }

/* confronto relazionale esplicito */
Number([42]) < Number("043"); // true — numerico
```

---

## Domande

<details>
<summary>Perché `[] == ![]` è `true` in JavaScript?</summary>

L'espressione si valuta in due fasi. Prima, l'operatore `!` trasforma `[]` in `false` (perché `[]` è truthy, e `!truthy` → `false`), quindi l'espressione diventa `[] == false`. Poi si applicano le regole di abstract equality: `false` viene convertito a numero → `0`; l'array `[]` viene convertito via `ToPrimitive` → `""` → `0`. Infine `0 == 0` è `true`. Il risultato non è "magia": segue regole precise, ma la catena di conversioni è lunga e non intuitiva — per questo è tra i casi da evitare con `==`.

</details>

<details>
<summary>Qual è la differenza tra `a + ""` e `String(a)` per convertire a stringa?</summary>

Entrambi producono una stringa, ma seguono percorsi diversi quando l'operando è un oggetto con metodi `valueOf` e `toString` personalizzati. `a + ""` invoca `ToPrimitive(a)`, che chiama prima `valueOf()`: se restituisce un primitivo, quello viene stringificato. `String(a)` invoca direttamente `ToString(a)`, che chiama `toString()`. Per i tipi built-in il risultato è identico, ma per oggetti custom possono divergere:

```js
var a = { valueOf: () => 42, toString: () => "4" };
a + "";       // "42" — usa valueOf
String(a);    // "4"  — usa toString
```

</details>

<details>
<summary>Perché non si dovrebbe mai usare `== true` o `== false`?</summary>

Perché la specifica converte prima il booleano a numero (`true` → `1`, `false` → `0`) prima di confrontarlo con l'altro operando. Il risultato è quasi sempre controintuitivo: `"true" == true` è `false` (perché `"true"` → `NaN` e `1 != NaN`); `"0" == false` è `true` (perché `"0"` → `0` e `false` → `0`). L'intento di chi scrive `if (x == true)` è quasi sempre verificare se `x` è truthy — il che si fa semplicemente con `if (x)`. Se si vuole verificare uguaglianza stretta con il booleano, si usa `=== true`.

</details>

<details>
<summary>In cosa `null == undefined` è diverso da `null === undefined`, e perché il primo è utile?</summary>

`null === undefined` è `false` — tipi diversi, nessuna coercion, `===` li distingue. `null == undefined` è `true` — è una delle poche regole speciali della abstract equality: questi due valori sono considerati "ugualmente vuoti" da `==`. La conseguenza pratica è un idioma compatto: `if (a == null)` è vero sia se `a` è `null` sia se è `undefined`, e falso per qualsiasi altro valore (inclusi `0`, `""`, `false`). Questo è uno dei rari casi in cui `==` è preferibile a `===` per la sua concisione senza ambiguità.

</details>

<details>
<summary>Perché `a <= b` può essere `true` anche quando `a < b`, `a == b` e `a > b` sono tutti `false`?</summary>

La specifica definisce `a <= b` non come "minore o uguale" ma come `!(b < a)` — la negazione del "b è strettamente minore di a". Quando due oggetti producono la stessa stringa da `ToPrimitive` (per esempio entrambi `"[object Object]"`), `a < b` è `false` e `b < a` è anche `false`. Di conseguenza `!(b < a)` è `true`. Lo stesso ragionamento porta `a >= b` ad essere `true`. Il confronto `a == b` è `false` perché per gli oggetti `==` confronta i riferimenti (non i valori stringa), e i due oggetti sono istanze distinte. È un comportamento sorprendente ma logicamente consistente con le definizioni della specifica.

</details>
