# Behavior Delegation

Il capitolo 5 ha dimostrato che `[[Prototype]]` non è un sistema di classi. Questo capitolo propone la risposta pratica: smettere di simulare le classi e usare il meccanismo così com'è — come **delegazione tra oggetti**.

## Toward Delegation-Oriented Design

Il meccanismo `[[Prototype]]` è, nella sua essenza, una catena di link tra oggetti. Quando una proprietà non viene trovata su un oggetto, l'engine segue il link al successivo, e così via. Questa è la delegazione.

### Class Theory vs Delegation Theory

Nel design classico si definisce una classe padre `Task` con comportamento condiviso, poi si creano classi figlie `XYZ` e `ABC` che ereditano da `Task` e specializzano il comportamento tramite override e polimorfismo.

Nel design a delegazione, invece, si definisce un **oggetto** `Task` con utility condivise, poi si creano oggetti `XYZ` e `ABC` collegati a `Task` via `[[Prototype]]`. Quando `XYZ` ha bisogno di un comportamento di `Task`, lo **delega** — non lo sovrascrive.

```js
var Task = {
    setID: function(ID) { this.id = ID; },
    outputID: function() { console.log(this.id); }
};

/* XYZ delega a Task */
var XYZ = Object.create(Task);

XYZ.prepareTask = function(ID, Label) {
    this.setID(ID);      /* delegato a Task */
    this.label = Label;
};
XYZ.outputTaskDetails = function() {
    this.outputID();     /* delegato a Task */
    console.log(this.label);
};
```

Questo stile si chiama **OLOO** — Objects Linked to Other Objects. Non ci sono classi, costruttori, o `.prototype`: solo oggetti collegati.

### Differenze chiave rispetto al pattern a classi

**Stato sui delegatori**: i dati (`id`, `label`) vivono su `XYZ`, non su `Task`. Il delegate (l'oggetto superiore) contiene utility; il delegator (l'oggetto inferiore) contiene lo stato specifico.

**Nomi distinti**: il design a classi promuove l'override con metodi dello stesso nome (`outputTask` su `Task` e su `XYZ`). OLOO fa il contrario — nomi diversi e descrittivi (`prepareTask`, `outputTaskDetails`) evitano conflitti e rendono il codice autoesplicativo.

**`this` corretto automaticamente**: quando `XYZ.prepareTask()` chiama `this.setID()`, l'engine trova `setID` su `Task` tramite la chain, ma il `this` binding rimane `XYZ` per le regole di call-site (Cap 2). La delegazione è trasparente.

### Delegazione mutua — non permessa

Non è possibile creare un ciclo bidirezionale di delegazione (A → B → A): l'engine lancia un errore alla creazione del link. La chain deve essere aciclica.

---

## Modelli mentali a confronto

Il codice class-oriented classico:

```js
function Foo(who) { this.me = who; }
Foo.prototype.identify = function() { return "I am " + this.me; };

function Bar(who) { Foo.call(this, who); }
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.speak = function() { alert("Hello, " + this.identify() + "."); };

var b1 = new Bar("b1");
var b2 = new Bar("b2");
b1.speak();
b2.speak();
```

Il codice OLOO equivalente:

```js
var Foo = {
    init: function(who) { this.me = who; },
    identify: function() { return "I am " + this.me; }
};

var Bar = Object.create(Foo);
Bar.speak = function() { alert("Hello, " + this.identify() + "."); };

var b1 = Object.create(Bar);
b1.init("b1");
var b2 = Object.create(Bar);
b2.init("b2");
b1.speak();
b2.speak();
```

La struttura `[[Prototype]]` è identica — `b1 → Bar → Foo`. Ma il codice OLOO elimina costruttori, `.prototype`, e `new`. Il modello mentale è molto più semplice: tre oggetti collegati, niente altro.

---

## Classi vs Oggetti — esempio pratico

### Widget/Button con il pattern a classi

```js
/* approccio classico */
function Widget(width, height) {
    this.width = width || 50;
    this.height = height || 50;
    this.$elem = null;
}
Widget.prototype.render = function($where) {
    if (this.$elem) {
        this.$elem.css({ width: this.width + "px", height: this.height + "px" })
                  .appendTo($where);
    }
};

function Button(width, height, label) {
    Widget.call(this, width, height); /* "super" call */
    this.label = label || "Default";
    this.$elem = $("<button>").text(this.label);
}
Button.prototype = Object.create(Widget.prototype);

Button.prototype.render = function($where) {
    Widget.prototype.render.call(this, $where); /* explicit pseudopolymorphism */
    this.$elem.click(this.onClick.bind(this));
};
Button.prototype.onClick = function(evt) {
    console.log("Button '" + this.label + "' clicked!");
};
```

Stesso esempio con ES6 `class`:

```js
class Widget {
    constructor(width, height) {
        this.width = width || 50;
        this.height = height || 50;
        this.$elem = null;
    }
    render($where) {
        if (this.$elem) {
            this.$elem.css({ width: this.width + "px", height: this.height + "px" })
                      .appendTo($where);
        }
    }
}

class Button extends Widget {
    constructor(width, height, label) {
        super(width, height);
        this.label = label || "Default";
        this.$elem = $("<button>").text(this.label);
    }
    render($where) {
        super($where);
        this.$elem.click(this.onClick.bind(this));
    }
    onClick(evt) {
        console.log("Button '" + this.label + "' clicked!");
    }
}
```

La sintassi è più pulita, ma il meccanismo sottostante è lo stesso `[[Prototype]]`. I problemi concettuali descritti nei capitoli 4 e 5 rimangono.

### Widget/Button con OLOO

```js
var Widget = {
    init: function(width, height) {
        this.width = width || 50;
        this.height = height || 50;
        this.$elem = null;
    },
    insert: function($where) {
        if (this.$elem) {
            this.$elem.css({ width: this.width + "px", height: this.height + "px" })
                      .appendTo($where);
        }
    }
};

var Button = Object.create(Widget);
Button.setup = function(width, height, label) {
    this.init(width, height); /* delegato a Widget */
    this.label = label || "Default";
    this.$elem = $("<button>").text(this.label);
};
Button.build = function($where) {
    this.insert($where);     /* delegato a Widget */
    this.$elem.click(this.onClick.bind(this));
};
Button.onClick = function(evt) {
    console.log("Button '" + this.label + "' clicked!");
};

/* utilizzo */
var btn1 = Object.create(Button);
btn1.setup(125, 30, "Hello");
btn1.build($body);
```

I metodi hanno nomi distinti (`init`/`setup`, `insert`/`build`) — non c'è mai bisogno di explicit pseudopolymorphism. La separazione tra creazione (`Object.create`) e inizializzazione (`setup`) è esplicita e permette maggiore flessibilità (es. pool di oggetti pre-creati, inizializzati in un secondo momento).

---

## Design semplificato — esempio Controller/Auth

Il pattern a classi per un sistema login/auth richiede tre entità: `Controller` (base), `LoginController` e `AuthController` (figli), con override di `success()` e `failure()` in entrambi i figli, e composizione aggiuntiva perché `AuthController` ha bisogno di un'istanza di `LoginController`.

Con OLOO, bastano **due oggetti** — `LoginController` e `AuthController` — dove il secondo delega al primo:

```js
var LoginController = {
    errors: [],
    getUser() { return document.getElementById("login_username").value; },
    getPassword() { return document.getElementById("login_password").value; },
    validateEntry(user, pw) {
        user = user || this.getUser();
        pw   = pw   || this.getPassword();
        if (!(user && pw)) { return this.failure("Inserire username e password!"); }
        if (user.length < 5) { return this.failure("Password di almeno 5 caratteri!"); }
        return true;
    },
    showDialog(title, msg) { /* mostra dialog */ },
    failure(err) {
        this.errors.push(err);
        this.showDialog("Error", "Login non valido: " + err);
    }
};

/* AuthController delega a LoginController */
var AuthController = Object.create(LoginController);
AuthController.errors = [];
AuthController.checkAuth = function() {
    var user = this.getUser();
    var pw   = this.getPassword();
    if (this.validateEntry(user, pw)) {
        this.server("/check-auth", { user, pw })
            .then(this.accepted.bind(this))
            .fail(this.rejected.bind(this));
    }
};
AuthController.server = function(url, data) { return $.ajax({ url, data }); };
AuthController.accepted = function() { this.showDialog("Success", "Autenticato!"); };
AuthController.rejected = function(err) { this.failure("Auth fallita: " + err); };

/* utilizzo — nessuna istanziazione necessaria */
AuthController.checkAuth();
```

Nessuna classe base condivisa, nessuna composizione artificiale, nessun override di metodi con lo stesso nome. I metodi su `AuthController` si chiamano `accepted`/`rejected` — nomi descrittivi del compito specifico — invece di `success`/`failure` come sul livello superiore.

---

## Sintassi ES6 concisa con OLOO

ES6 permette la sintassi shorthand per i metodi anche negli object literal, rendendo OLOO ancora più compatto:

```js
var LoginController = {
    errors: [],
    getUser()     { /* ... */ },
    getPassword() { /* ... */ },
    validateEntry(user, pw) { /* ... */ },
    failure(err)  { /* ... */ }
};

/* ES6: object literal + Object.setPrototypeOf */
var AuthController = {
    errors: [],
    checkAuth() { /* ... */ },
    server(url, data) { /* ... */ },
    accepted() { /* ... */ },
    rejected(err) { /* ... */ }
};
Object.setPrototypeOf(AuthController, LoginController);
```

**Attenzione — metodi concisi e auto-riferimento**: la sintassi shorthand `bar() {}` genera una function expression anonima. Se il metodo ha bisogno di riferirsi a se stesso (ricorsione, event unbinding), non esiste un identificatore lessicale. In quel caso va usata la forma esplicita con nome: `baz: function baz() { baz(x*2); }`.

---

## Introspezione — OLOO vs classi

Con il pattern class-oriented l'introspezione usa `instanceof` e `.prototype`, producendo codice verboso e concettualmente fuorviante:

```js
/* class-style — verboso e indiretto */
Bar.prototype instanceof Foo;               // true
Foo.prototype.isPrototypeOf(Bar.prototype); // true
b1 instanceof Foo;                          // true
b1 instanceof Bar;                          // true
```

Con OLOO, dove gli oggetti sono piani e collegati direttamente, l'introspezione diventa diretta:

```js
/* OLOO-style — diretto */
Foo.isPrototypeOf(Bar); // true
Bar.isPrototypeOf(b1);  // true
Foo.isPrototypeOf(b1);  // true
Object.getPrototypeOf(b1) === Bar; // true
```

Non serve più `instanceof`, che presuppone una relazione class-like. Si usa direttamente `isPrototypeOf()` tra oggetti — la domanda è semplicemente "sei nella mia chain?".

**Duck typing**: un'alternativa pratica all'introspezione è controllare l'esistenza del comportamento invece del tipo: `if (a1.something) { a1.something(); }`. Funziona bene in casi semplici, ma è fragile quando si fanno assunzioni implicite sulle altre capacità dell'oggetto. Un esempio noto: ES6 Promises considerano qualsiasi oggetto con un metodo `.then()` come "thenable" — se un oggetto non-Promise ha un `.then()` per altri motivi, può causare comportamenti inattesi.

---

## ⚡ Ripasso veloce

**OLOO** (Objects Linked to Other Objects): stile di codice in cui gli oggetti sono collegati direttamente tramite `Object.create()` senza costruttori, `.prototype` o `new`.

**Delegazione vs ereditarietà**: la chain `[[Prototype]]` di JS è, per natura, un meccanismo di delegazione — non di copia. Usarla come tale, invece di simulare classi, produce codice più semplice e coerente con il linguaggio.

**Nomi distinti per livelli diversi**: in OLOO si evitano metodi con lo stesso nome a livelli diversi della chain (shadowing). Nomi descrittivi e specifici sostituiscono nomi generici da sovrascrivere.

**Separazione creazione/inizializzazione**: `Object.create(Obj)` crea il link; un metodo separato (`setup`, `init`) inizializza lo stato. Questo disaccoppiamento è impossibile con i constructor classici.

```js
/* OLOO in sintesi */
var Task = { doWork() { /* utility */ } };
var Job  = Object.create(Task);
Job.run = function(data) {
    this.doWork(); /* delegato a Task */
    /* logica specifica di Job */
};

/* introspezione pulita */
Task.isPrototypeOf(Job); // true
```

| Aspetto | Class-style | OLOO |
|---------|-------------|------|
| Entità | Classe padre + figli | Oggetti pari collegati |
| Nomi metodi | Stessi (override) | Distinti (no shadowing) |
| Creazione | `new Constructor()` | `Object.create(obj)` |
| Introspezione | `instanceof` (indiretto) | `isPrototypeOf()` (diretto) |
| Pseudopolimorfismo | `Parent.method.call(this)` | Non necessario |

---

## Domande

<details>
<summary>Cosa significa OLOO e come si differenzia dal pattern a classi?</summary>

OLOO significa "Objects Linked to Other Objects" — oggetti collegati ad altri oggetti. Nel pattern a classi si definiscono gerarchie verticali: classe padre con comportamento condiviso, classi figlie che ereditano e sovrascrivono. In OLOO si creano oggetti "pari" (peer) collegati direttamente tramite `[[Prototype]]`: non ci sono classi, costruttori, né `new`. Quando un oggetto ha bisogno di un comportamento che non possiede, lo delega all'oggetto collegato superiore. La differenza concettuale fondamentale è che le classi implicano copia (del comportamento), OLOO implica collegamento (e delegazione dinamica).

</details>

<details>
<summary>Perché in OLOO si usano nomi di metodo diversi ai diversi livelli della chain?</summary>

Nel design a classi, i nomi uguali a livelli diversi sono intenzionali: permettono il polimorfismo (override). In OLOO il shadowing — avere lo stesso nome a livelli diversi — è problematico perché crea ambiguità e richiede explicit pseudopolymorphism (`SuperObj.method.call(this)`) per disambiguare. Usare nomi descrittivi e distinti (`setup` invece di `init` su Button, `insert` invece di `render` su Widget) rende l'API autoesplicativa, elimina la necessità di disambiguazione, e rende il codice più facile da capire e mantenere.

</details>

<details>
<summary>Perché OLOO separa la creazione dall'inizializzazione, e qual è il vantaggio?</summary>

Con `new Constructor()`, creazione e inizializzazione avvengono in un unico passo atomico. Con OLOO, la creazione del link (`Object.create(Obj)`) e l'inizializzazione (`obj.setup(...)`) sono operazioni separate. Questo permette scenari che i constructor classici non supportano agevolmente: si possono pre-creare oggetti in un pool all'avvio del programma e inizializzarli solo al momento dell'uso, oppure inizializzare lo stesso oggetto più volte con parametri diversi. È un'applicazione del principio di separazione delle responsabilità — la struttura (link) e lo stato iniziale (dati) sono concetti distinti.

</details>

<details>
<summary>Perché `instanceof` è meno adatto all'introspezione in OLOO rispetto a `isPrototypeOf()`?</summary>

`instanceof` risponde alla domanda "Il `[[Prototype]]` di `a` contiene `Foo.prototype`?" — richiede un riferimento a una funzione, non direttamente a un oggetto. Presuppone il pattern class-oriented (funzioni costruttore con `.prototype`). Con OLOO, dove non esistono funzioni costruttore, `instanceof` non può essere usato nel modo naturale. `isPrototypeOf()` risponde direttamente: "Sei nella mia chain?" — opera su due oggetti qualsiasi senza bisogno di funzioni intermedie o `.prototype`. È più diretto, più leggibile, e coerente con la natura di OLOO.

</details>

<details>
<summary>Qual è il rischio del "duck typing" e quando è accettabile usarlo?</summary>

Il "duck typing" verifica la presenza di un comportamento invece del tipo: `if (a1.something) { a1.something(); }`. Il rischio è l'inferenza implicita di ulteriori capacità non testate: trovare `.then()` su un oggetto e concludere che è una Promise, per esempio, è una assunzione non verificata che può causare comportamenti inattesi se l'oggetto non è una Promise ma ha un `.then()` per ragioni proprie (ES6 Promises fanno esattamente questa assunzione su qualsiasi oggetto "thenable"). È accettabile in contesti controllati dove le assunzioni sono documentate e verificate, ma diventa fragile quando si estende oltre la sola proprietà testata.

</details>
