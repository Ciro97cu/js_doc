# Oggetti

Gli oggetti sono il meccanismo centrale di JavaScript: quasi tutto in JS ruota attorno a essi. Questo capitolo esplora la loro struttura interna, le proprietà, i descriptor, e i modi per iterare sui valori.

## Sintassi

Gli oggetti esistono in due forme equivalenti:

```js
/* literal form — quasi sempre preferita */
var myObj = { key: value };

/* constructed form — usata raramente */
var myObj = new Object();
myObj.key = value;
```

Le due forme producono oggetti identici. La differenza è che nella forma letterale si possono dichiarare più coppie chiave/valore in una volta sola; nella forma costruita le proprietà si aggiungono una per una.

---

## Tipi

JavaScript ha sei tipi primitivi: `string`, `number`, `boolean`, `null`, `undefined` e `object`. I primi cinque **non sono oggetti** — sono primitivi immutabili. `null` restituisce `"object"` da `typeof` per un bug storico del linguaggio, ma è un primitivo a sé stante.

L'affermazione "tutto in JavaScript è un oggetto" è falsa.

Esistono però dei **built-in objects** — funzioni costruttore che producono sottotipi di oggetto:

`String`, `Number`, `Boolean`, `Object`, `Function`, `Array`, `Date`, `RegExp`, `Error`

Il punto chiave è l'**autoboxing**: l'engine converte automaticamente un primitivo nel suo wrapper oggetto quando è necessario accedere a metodi o proprietà. Non serve mai creare esplicitamente `new String("...")`:

```js
var s = "sono una stringa";
console.log(s.length);    // 16 — autoboxing a String object
console.log(s.charAt(3)); // "o"
```

`null` e `undefined` non hanno wrapper. `Date` esiste solo nella forma costruita (non ha literal). `Object`, `Array`, `Function` e `RegExp` sono oggetti sia in forma literal che costruita.

---

## Contenuti e accesso alle proprietà

Un oggetto è una collezione di coppie nome/valore. Le proprietà si accedono con due sintassi:

```js
var obj = { a: 2 };
obj.a;   // 2 — property access (dot notation)
obj["a"]; // 2 — key access (bracket notation)
```

La differenza pratica: la dot notation richiede un identificatore valido; la bracket notation accetta qualsiasi stringa UTF-8 e permette l'accesso dinamico:

```js
var prefix = "foo";
var obj = { foobar: "hello" };
obj[prefix + "bar"]; // "hello" — computed key
```

I nomi di proprietà sono sempre stringhe. Qualsiasi altro valore usato come chiave viene prima convertito in stringa (`true` → `"true"`, `3` → `"3"`, un oggetto → `"[object Object]"`).

**ES6 computed property names**: è possibile usare espressioni come chiave nella dichiarazione letterale:

```js
var prefix = "foo";
var obj = {
    [prefix + "bar"]: "hello",
    [prefix + "baz"]: "world"
};
obj["foobar"]; // "hello"
```

### Property vs Method

In JavaScript non esiste una distinzione vera tra "proprietà" e "metodo". Una funzione non "appartiene" a un oggetto: è solo un valore memorizzato in una proprietà. Il binding di `this` avviene a runtime in base al call-site, non perché la funzione "sia" dell'oggetto.

### Array

Gli array sono oggetti con indici numerici. Si possono aggiungere proprietà con nome senza cambiarne `length`, ma un nome che sembra un numero viene interpretato come indice:

```js
var arr = ["foo", 42, "bar"];
arr.baz = "baz";
arr.length; // 3 — non cambia
arr["3"] = "qux";
arr.length; // 4 — interpretato come indice!
```

---

## Duplicare oggetti

Non esiste un `copy()` built-in perché la semantica non è univoca: shallow copy o deep copy? Come gestire i riferimenti circolari?

**Shallow copy JSON-safe** (solo per oggetti serializzabili):

```js
var newObj = JSON.parse(JSON.stringify(someObj));
```

**Shallow copy con `Object.assign()`** (ES6) — copia le proprietà enumerabili proprie delle sorgenti nel target:

```js
var newObj = Object.assign({}, myObj);
newObj.a === myObj.a; // true per i primitivi (copia di valore)
newObj.b === myObj.b; // true per gli oggetti (copia di riferimento)
```

`Object.assign` usa assegnamento `=` semplice: i descriptor delle proprietà sorgente non vengono preservati nel target.

---

## Property descriptor

Ogni proprietà ha un **descriptor** — un oggetto che ne descrive le caratteristiche oltre al valore:

```js
Object.getOwnPropertyDescriptor({ a: 2 }, "a");
// { value: 2, writable: true, enumerable: true, configurable: true }
```

Con `Object.defineProperty()` si può creare o modificare una proprietà con caratteristiche esplicite:

```js
var obj = {};
Object.defineProperty(obj, "a", {
    value: 2,
    writable: false,    // non modificabile
    configurable: false, // non ridefinibile né eliminabile
    enumerable: true
});
```

**`writable: false`** — il tentativo di assegnamento fallisce silenziosamente in non-strict mode; in strict mode lancia `TypeError`.

**`configurable: false`** — impedisce di modificare il descriptor con `defineProperty()` e impedisce il `delete` della proprietà. È un'operazione **irreversibile**: non si può tornare a `true`. Eccezione: `writable` può essere cambiato da `true` a `false` anche dopo che `configurable` è `false`, ma non il contrario.

**`enumerable: false`** — la proprietà esiste e si può accedere, ma non appare nei loop `for..in` e non viene restituita da `Object.keys()`.

### Livelli di immutabilità

| Metodo | Effetto |
|--------|---------|
| `Object.preventExtensions(obj)` | Impedisce l'aggiunta di nuove proprietà |
| `Object.seal(obj)` | `preventExtensions` + `configurable: false` su tutte le proprietà |
| `Object.freeze(obj)` | `seal` + `writable: false` su tutte le proprietà data |

Tutti e tre creano **shallow immutability**: i valori oggetto referenziati dalle proprietà rimangono modificabili.

---

## `[[Get]]` e `[[Put]]`

Accedere a `obj.a` non è un semplice lookup diretto: l'engine invoca l'operazione interna `[[Get]]`, che prima cerca la proprietà sull'oggetto e poi percorre la prototype chain (Cap 5). Se la proprietà non viene trovata da nessuna parte, restituisce `undefined` — non lancia `ReferenceError` come accade con le variabili non dichiarate.

```js
var obj = { a: undefined };
obj.a; // undefined — proprietà esiste
obj.b; // undefined — proprietà non esiste: [[Get]] fallisce silenziosamente
```

I due casi sono indistinguibili dal valore; si usano `in` o `hasOwnProperty` per distinguerli (vedi sotto).

`[[Put]]` gestisce l'assegnamento: verifica se la proprietà è un accessor descriptor (chiama il setter), se è `writable: false` (fallisce o lancia TypeError), altrimenti assegna normalmente.

### Getter e Setter

Le **accessor properties** (proprietà accessore) sostituiscono `[[Get]]` e `[[Put]]` con funzioni personalizzate. Un getter o setter viene definito nel descriptor invece dei campi `value`/`writable`:

```js
var obj = {
    get a() {
        return this._a_;
    },
    set a(val) {
        this._a_ = val * 2;
    }
};

obj.a = 2;
obj.a; // 4 — il setter ha moltiplicato per 2
```

Se si definisce solo il getter senza setter, qualsiasi assegnamento viene silenziosamente ignorato.

---

## Esistenza di una proprietà

`in` verifica se una proprietà esiste sull'oggetto **o sulla prototype chain**. `hasOwnProperty()` verifica solo sull'oggetto diretto:

```js
var obj = { a: 2 };
"a" in obj;                  // true
"toString" in obj;           // true — trovata nella prototype chain
obj.hasOwnProperty("a");     // true
obj.hasOwnProperty("toString"); // false — non è propria
```

`in` **non** verifica valori in un array: `4 in [2, 4, 6]` controlla se esiste l'indice `4`, non se `4` è un elemento.

Per oggetti creati con `Object.create(null)` (senza prototype) `hasOwnProperty` non è disponibile — si usa `Object.prototype.hasOwnProperty.call(obj, "a")`.

---

## Enumerazione

`for..in` itera sulle proprietà **enumerabili** dell'oggetto e della sua prototype chain. Per solo l'oggetto diretto:

- `Object.keys(obj)` → array delle proprietà enumerabili proprie
- `Object.getOwnPropertyNames(obj)` → array di tutte le proprietà proprie, enumerabili o meno

```js
var obj = {};
Object.defineProperty(obj, "a", { enumerable: true,  value: 2 });
Object.defineProperty(obj, "b", { enumerable: false, value: 3 });

Object.keys(obj);                  // ["a"]
Object.getOwnPropertyNames(obj);   // ["a", "b"]
obj.propertyIsEnumerable("b");     // false
```

---

## Iterazione

`for..in` itera sui nomi delle proprietà (non sui valori). Per iterare direttamente sui valori:

**Array** — `for..of` (ES6) usa il **Symbol.iterator** built-in:

```js
var arr = [1, 2, 3];
for (var v of arr) {
    console.log(v); // 1, 2, 3
}
```

Internamente, `for..of` chiama ripetutamente `next()` sull'oggetto iteratore fino a `{ done: true }`:

```js
var it = arr[Symbol.iterator]();
it.next(); // { value: 1, done: false }
it.next(); // { value: 2, done: false }
it.next(); // { value: 3, done: false }
it.next(); // { value: undefined, done: true }
```

**Oggetti** — non hanno un `@@iterator` built-in. È possibile definirne uno custom:

```js
var obj = { a: 2, b: 3 };
Object.defineProperty(obj, Symbol.iterator, {
    enumerable: false,
    value: function() {
        var o = this, idx = 0, ks = Object.keys(o);
        return {
            next: function() {
                return { value: o[ks[idx++]], done: idx > ks.length };
            }
        };
    }
});

for (var v of obj) { console.log(v); } // 2, 3
```

---

## ⚡ Ripasso veloce

**Tipi**: 6 primitivi in JS, `object` è uno di essi — non "tutto è un oggetto". I primitivi vengono autoboxati quando si accede ai loro metodi.

**Property access**: `.prop` (identificatore valido) vs `["prop"]` (qualsiasi stringa, computed). I nomi di proprietà sono sempre stringhe.

**Descriptor**: ogni proprietà ha `writable`, `configurable`, `enumerable`. `Object.defineProperty()` li controlla esplicitamente.

**Immutabilità**: `preventExtensions` < `seal` < `freeze`, tutti shallow.

**`[[Get]]`**: accesso a proprietà inesistente → `undefined` (non `ReferenceError`). Usare `"a" in obj` o `hasOwnProperty("a")` per verificare l'esistenza.

**`for..of`** vs **`for..in`**: `for..of` itera sui valori tramite `Symbol.iterator`; `for..in` itera sui nomi delle proprietà enumerabili.

---

## Domande

<details>
<summary>Perché "tutto in JavaScript è un oggetto" è un'affermazione falsa?</summary>

Perché JavaScript ha sei tipi primitivi — `string`, `number`, `boolean`, `null`, `undefined` e `object` — e solo l'ultimo è un oggetto. I primitivi `string`, `number`, `boolean` hanno wrapper oggetto (`String`, `Number`, `Boolean`) e vengono automaticamente convertiti (autoboxing) quando si tenta di accedere a metodi o proprietà su di essi, ma i valori primitivi stessi non sono oggetti. `null` restituisce `"object"` da `typeof` a causa di un bug storico del linguaggio, ma è un primitivo a sé stante.

</details>

<details>
<summary>Qual è la differenza tra `in` e `hasOwnProperty()` per verificare l'esistenza di una proprietà?</summary>

`in` verifica se una proprietà esiste sull'oggetto o in qualsiasi punto della sua prototype chain — quindi troverà anche proprietà ereditate come `toString`. `hasOwnProperty()` verifica solo sull'oggetto diretto, senza risalire la prototype chain. Per la maggior parte dei casi pratici, quando si vuole sapere se una proprietà appartiene specificamente all'oggetto e non alla sua catena ereditaria, si usa `hasOwnProperty()`. È anche importante non usare `in` per verificare se un valore è presente in un array: `in` controlla gli indici (le chiavi), non i valori.

</details>

<details>
<summary>Cosa significa `configurable: false` e perché è irreversibile?</summary>

`configurable: false` impedisce di modificare il descriptor della proprietà tramite `Object.defineProperty()` e impedisce di eliminarla con `delete`. Una volta impostato a `false`, non è possibile tornare a `true` — qualsiasi tentativo di ridefinire il descriptor lancia `TypeError`. L'unica eccezione è che `writable` può essere cambiato da `true` a `false` anche con `configurable: false`, ma non il contrario. L'irreversibilità è intenzionale: serve per creare proprietà permanentemente "sigillate" in librerie e API.

</details>

<details>
<summary>Qual è la differenza tra `for..in` e `for..of`?</summary>

`for..in` itera sui **nomi** (chiavi) delle proprietà enumerabili di un oggetto, risalendo la prototype chain. Non itera direttamente sui valori. `for..of` itera sui **valori** attraverso il protocollo dell'iteratore (`Symbol.iterator` + metodo `next()`). Gli array hanno un iteratore built-in; gli oggetti semplici no — bisogna definirne uno custom con `Object.defineProperty()` e `Symbol.iterator`. `for..of` è la scelta corretta per iterare sugli elementi di un array senza preoccuparsi degli indici o delle proprietà non numeriche aggiunte all'array.

</details>

<details>
<summary>Perché accedere a una proprietà inesistente restituisce `undefined` invece di lanciare un errore?</summary>

Perché l'operazione `[[Get]]` — invocata internamente a ogni accesso di proprietà — percorre l'oggetto e la sua prototype chain cercando il nome richiesto. Se non lo trova da nessuna parte, restituisce `undefined` come valore di default, senza lanciare eccezioni. Questo comportamento differisce da quello delle variabili non dichiarate, per cui l'engine lancia un `ReferenceError`. La conseguenza pratica è che `obj.propInesistente` e `obj.propEsplicitamenteUndefined` producono lo stesso valore `undefined` — per distinguerli si deve usare `"prop" in obj` oppure `obj.hasOwnProperty("prop")`.

</details>
