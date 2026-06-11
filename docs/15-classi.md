# Classi

Le classi sono il costrutto che TypeScript mette a disposizione per definire strutture dati complesse insieme al comportamento che opera su di esse. Una classe funge da modello (blueprint) a partire dal quale si creano oggetti, chiamati istanze, tramite l'operatore `new`. Pur trattandosi di un costrutto già presente in JavaScript moderno, TypeScript lo arricchisce con il controllo statico dei tipi, con gli access modifier e con una serie di scorciatoie sintattiche che riducono il codice ripetitivo.

Una classe si definisce con la keyword `class` seguita dal nome, per convenzione in PascalCase, e da un blocco che ne racchiude i membri.

```ts
class Persona {}

const p = new Persona();
console.log(p instanceof Persona); // true
```

## Proprietà, metodi e constructor

Lo stato di un'istanza è descritto dalle sue proprietà, ovvero variabili associate all'oggetto e dichiarate all'interno del corpo della classe con il proprio tipo. Il comportamento è invece definito dai metodi, funzioni che operano su quello stato. Esiste poi un metodo speciale, il `constructor`, che viene invocato automaticamente al momento della creazione dell'istanza con `new` e che serve a inizializzare le proprietà a partire dagli argomenti ricevuti.

Va tenuto presente che, con le impostazioni predefinite di TypeScript 6.0 (dove `strict` vale `true`), è attivo anche `strictPropertyInitialization`: ogni proprietà dichiarata deve essere inizializzata, nel constructor oppure tramite un valore di default, altrimenti il compilatore segnala un errore.

```ts
class Persona {
  nome: string;
  eta: number;

  constructor(nome: string, eta: number) {
    this.nome = nome;
    this.eta = eta;
  }

  saluta(): string {
    return `Ciao, mi chiamo ${this.nome} e ho ${this.eta} anni.`;
  }
}

const mario = new Persona("Mario", 30);
console.log(mario.saluta()); // Ciao, mi chiamo Mario e ho 30 anni.
```

Se una proprietà venisse dichiarata senza essere mai inizializzata, il compilatore reagirebbe così:

```ts
class Conto {
  saldo: number;
  // Errore: Property 'saldo' has no initializer and is not definitely assigned in the constructor.
}
```

## this e il problema dei callback

All'interno di un metodo, la keyword `this` fa riferimento all'istanza su cui il metodo è stato invocato. Il punto delicato è che, in JavaScript, il valore di `this` non è legato al punto in cui il metodo viene definito, bensì al modo in cui viene chiamato. Quando un metodo viene estratto dall'oggetto e passato altrove come callback, ad esempio a `setTimeout` o a un gestore di eventi, perde il proprio collegamento con l'istanza: in modalità strict, che con TypeScript 6.0 è quella predefinita, `this` diventa `undefined`, con il conseguente errore a runtime.

```ts
class Contatore {
  conteggio = 0;

  incrementa() {
    this.conteggio++;
    console.log(this.conteggio);
  }
}

const c = new Contatore();

// Passato come callback, il metodo perde il riferimento all'istanza:
setTimeout(c.incrementa, 100);
// TypeError a runtime: Cannot read properties of undefined (reading 'conteggio')
```

Per anticipare il problema già in fase di build, è possibile dichiarare esplicitamente il tipo atteso di `this` tramite un parametro fittizio chiamato proprio `this`, posto per primo nella lista dei parametri: si tratta di un parametro puramente di tipo, che non occupa una posizione effettiva fra gli argomenti e scompare nel JavaScript generato. In questo modo il compilatore segnala ogni invocazione in cui `this` non corrisponde al tipo dichiarato. L'opzione `noImplicitThis`, inclusa in `strict`, contribuisce inoltre segnalando le funzioni in cui `this` assumerebbe implicitamente il tipo `any`.

```ts
class Contatore {
  conteggio = 0;

  incrementa(this: Contatore) {
    this.conteggio++;
  }
}

const c = new Contatore();
const staccato = c.incrementa;
staccato();
// Errore: The 'this' context of type 'void' is not assignable to method's 'this' of type 'Contatore'.
```

Esistono due rimedi consolidati. Il primo consiste nel legare esplicitamente il contesto con `bind`, che restituisce una nuova funzione il cui `this` è fissato all'istanza indicata.

```ts
setTimeout(c.incrementa.bind(c), 100); // stampa 1, this è garantito
```

Il secondo, più ergonomico, consiste nel definire il metodo come arrow function assegnata a un campo: le arrow function non hanno un proprio `this`, ma lo ereditano dal contesto in cui vengono create, ovvero l'istanza. In questo modo il legame è preservato anche quando il metodo viene passato in giro.

```ts
class Contatore {
  conteggio = 0;

  // Arrow function come campo: this resta sempre legato all'istanza
  incrementa = () => {
    this.conteggio++;
    console.log(this.conteggio);
  };
}

const c = new Contatore();
setTimeout(c.incrementa, 100); // stampa 1 correttamente
```

## L'operatore di assegnazione definitiva !

In alcuni scenari una proprietà non viene inizializzata nel constructor, bensì da un meccanismo esterno che il compilatore non è in grado di analizzare, come un metodo di setup, un framework di dependency injection o una query al DOM eseguita poco dopo la costruzione. In questi casi `strictPropertyInitialization` segnalerebbe un errore, pur sapendo lo sviluppatore che il valore sarà comunque presente prima di qualsiasi lettura.

La definite assignment assertion, ovvero il punto esclamativo `!` posto dopo il nome della proprietà, comunica al compilatore che quella proprietà verrà sicuramente assegnata prima del suo utilizzo e che pertanto non va considerata potenzialmente `undefined`.

```ts
class FormUtente {
  campoNome!: HTMLInputElement;

  inizializza() {
    this.campoNome = document.querySelector("#nome")!;
  }
}
```

Si tratta di uno strumento da usare con cautela: l'asserzione disattiva un controllo di sicurezza ma non garantisce alcunché a runtime. Se la proprietà non viene effettivamente assegnata, l'accesso restituirà `undefined` e l'errore emergerà soltanto durante l'esecuzione. Dove possibile, è preferibile inizializzare la proprietà nel constructor o dichiararne il tipo come opzionale.

## Access modifier: public e private

TypeScript permette di regolare la visibilità dei membri di una classe tramite gli access modifier. Un membro `public` è accessibile ovunque, sia all'interno della classe sia dall'esterno, ed è il comportamento predefinito quando non si specifica alcun modificatore. Un membro `private` è invece accessibile soltanto dall'interno della classe che lo dichiara: ogni tentativo di accedervi dall'esterno viene bloccato dal compilatore. Esiste anche `protected`, che estende la visibilità alle sottoclassi.

```ts
class ContoBancario {
  public titolare: string;
  private saldo: number;

  constructor(titolare: string, saldoIniziale: number) {
    this.titolare = titolare;
    this.saldo = saldoIniziale;
  }

  deposita(importo: number): void {
    this.saldo += importo;
  }

  consultaSaldo(): number {
    return this.saldo;
  }
}

const conto = new ContoBancario("Anna", 100);
conto.deposita(50);
console.log(conto.titolare);       // Anna
console.log(conto.consultaSaldo()); // 150
console.log(conto.saldo);
// Errore: Property 'saldo' is private and only accessible within class 'ContoBancario'.
```

Va segnalato che gli access modifier di TypeScript impongono i loro vincoli soltanto in fase di compilazione e vengono rimossi dal JavaScript generato. Quando serve un incapsulamento effettivo anche a runtime, JavaScript moderno offre i campi privati nativi con il prefisso `#`, riconosciuti pienamente da TypeScript.

```ts
class Segreto {
  #chiave: string;

  constructor(chiave: string) {
    this.#chiave = chiave;
  }

  verifica(tentativo: string): boolean {
    return tentativo === this.#chiave;
  }
}
```

## Shorthand initialization

Dichiarare una proprietà, riceverla come parametro del constructor e poi assegnarla a `this` è un pattern così frequente da risultare verboso. TypeScript offre una scorciatoia, nota come shorthand initialization (o parameter properties): anteponendo un access modifier (o `readonly`) a un parametro del constructor, il compilatore dichiara automaticamente una proprietà con lo stesso nome e la inizializza con il valore ricevuto, senza bisogno di scrivere l'assegnazione esplicita.

```ts
class Prodotto {
  constructor(
    public nome: string,
    private prezzo: number,
    readonly codice: string,
  ) {}

  descrizione(): string {
    return `${this.nome} (${this.codice}): ${this.prezzo} euro`;
  }
}

const libro = new Prodotto("Il nome della rosa", 18, "ABC-001");
console.log(libro.nome);  // Il nome della rosa
console.log(libro.codice); // ABC-001
```

Il punto fondamentale è che il modificatore non può essere omesso: senza `public`, `private`, `protected` o `readonly` davanti al parametro, quest'ultimo resta un semplice argomento del constructor e non viene promosso a proprietà dell'istanza.

```ts
class Esempio {
  constructor(valore: number) {} // 'valore' è solo un parametro locale
  // this.valore NON esiste
}
```

## readonly

Il modificatore `readonly` rende una proprietà di sola lettura dopo l'inizializzazione: è consentito assegnarle un valore al momento della dichiarazione oppure all'interno del constructor, ma ogni successiva riassegnazione viene rifiutata dal compilatore. Risulta particolarmente adatto per valori che devono restare costanti per tutta la vita dell'istanza, come identificatori o codici.

```ts
class Utente {
  readonly id: number;
  nome: string;

  constructor(id: number, nome: string) {
    this.id = id;     // consentito: assegnazione nel constructor
    this.nome = nome;
  }
}

const u = new Utente(1, "Luca");
u.nome = "Marco"; // consentito
u.id = 2;
// Errore: Cannot assign to 'id' because it is a read-only property.
```

Anche `readonly`, come gli access modifier, è un vincolo di livello statico: viene verificato dal compilatore e non lascia traccia nel JavaScript prodotto. La sua protezione, inoltre, è superficiale: impedisce di riassegnare la proprietà, ma se questa contiene un oggetto o un array, il contenuto di tale oggetto rimane modificabile.

```ts
class Configurazione {
  readonly opzioni: string[];

  constructor(opzioni: string[]) {
    this.opzioni = opzioni;
  }
}

const config = new Configurazione(["a", "b"]);
config.opzioni.push("c"); // consentito: l'array interno non è readonly
config.opzioni = [];
// Errore: Cannot assign to 'opzioni' because it is a read-only property.
```

`readonly` si combina volentieri con la shorthand initialization, dando luogo a proprietà immutabili dichiarate in modo estremamente conciso.

```ts
class Punto {
  constructor(
    public readonly x: number,
    public readonly y: number,
  ) {}
}

const origine = new Punto(0, 0);
console.log(`${origine.x}, ${origine.y}`); // 0, 0
```

## Domande

<details>
<summary>Con le impostazioni predefinite di TypeScript 6.0, perché dichiarare una proprietà senza inizializzarla produce un errore?</summary>

Perché in TypeScript 6.0 l'opzione `strict` è attiva per default e con essa `strictPropertyInitialization`. Questo controllo richiede che ogni proprietà non opzionale sia inizializzata, nel constructor o con un valore di default; in caso contrario il compilatore segnala `Property '...' has no initializer and is not definitely assigned in the constructor`. Si può aggirare il controllo, quando appropriato, con la definite assignment assertion `!` o rendendo la proprietà opzionale.
</details>

<details>
<summary>Perché passare un metodo come callback (ad esempio a setTimeout) può far perdere il riferimento a this, e quali soluzioni esistono?</summary>

Perché in JavaScript il valore di `this` dipende da come la funzione viene chiamata, non da dove viene definita. Estraendo il metodo dall'oggetto e passandolo come callback, il legame con l'istanza si perde e, in modalità strict (default in TypeScript 6.0), `this` diventa `undefined`. Le due soluzioni consolidate sono legare esplicitamente il contesto con `metodo.bind(istanza)`, oppure definire il metodo come arrow function assegnata a un campo, poiché le arrow function ereditano `this` dal contesto di creazione, ovvero l'istanza.
</details>

<details>
<summary>Che differenza c'è tra la garanzia offerta da ! (definite assignment assertion) e quella di readonly o degli access modifier a runtime?</summary>

Tutti e tre operano esclusivamente in fase di compilazione e non lasciano traccia nel JavaScript generato. L'operatore `!` non garantisce nulla a runtime: si limita a silenziare il controllo di inizializzazione, perciò se la proprietà non viene davvero assegnata l'accesso restituirà `undefined`. Allo stesso modo `readonly` e `private`/`protected` sono vincoli verificati solo dal compilatore. Per un incapsulamento effettivo a runtime occorre ricorrere ai campi privati nativi con prefisso `#`.
</details>

<details>
<summary>Nella shorthand initialization del constructor, cosa accade se si dimentica l'access modifier davanti al parametro?</summary>

Senza un modificatore (`public`, `private`, `protected` o `readonly`) davanti al parametro, non viene creata alcuna proprietà dell'istanza: il parametro resta un semplice argomento locale del constructor, visibile solo al suo interno. La promozione automatica a proprietà avviene unicamente in presenza di un modificatore, che quindi non può essere omesso.

```ts
class Esempio {
  constructor(public a: number, b: number) {}
  // this.a esiste, this.b NO
}
```
</details>

<details>
<summary>Perché una proprietà readonly che contiene un array permette comunque di modificarne il contenuto con push?</summary>

Perché `readonly` impedisce la riassegnazione della proprietà, non rende immutabile il valore che essa contiene. La protezione è superficiale (shallow): vieta `config.opzioni = [...]`, ma non `config.opzioni.push(...)`, dato che l'array interno non è dichiarato come `readonly`. Per impedire anche le mutazioni del contenuto si possono usare tipi come `readonly string[]` o `ReadonlyArray<string>`, ricordando che si tratta comunque di garanzie a livello di tipo e non di immutabilità a runtime (per quella serve `Object.freeze`).
</details>
