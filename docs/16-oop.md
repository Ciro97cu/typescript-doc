# Programmazione a oggetti

La programmazione a oggetti (object-oriented programming) è un paradigma che organizza il codice attorno al concetto di oggetto, ossia un'entità che racchiude al proprio interno sia i dati (lo stato) sia il comportamento (i metodi che operano su quei dati). TypeScript estende le classi di JavaScript con un sistema di tipi statico e con costrutti dedicati, rendendo questo stile di programmazione più robusto e verificabile in fase di compilazione.

L'object-oriented programming si fonda su quattro principi cardine. L'incapsulamento consiste nel raggruppare stato e comportamento in un'unica unità e nel proteggere lo stato interno dall'accesso diretto, esponendo solo ciò che è necessario. L'ereditarietà permette a una classe di derivare proprietà e metodi da un'altra, favorendo il riuso e l'organizzazione gerarchica del codice. Il polimorfismo consente a entità diverse di rispondere in modo differente allo stesso messaggio, tipicamente sovrascrivendo i metodi ereditati. L'astrazione, infine, mette in evidenza le funzionalità essenziali nascondendo i dettagli implementativi, così che chi utilizza una classe debba conoscere soltanto cosa essa fa e non come lo fa.

## Ereditarietà

L'ereditarietà si esprime con la keyword `extends`, che crea una sottoclasse a partire da una superclasse. La sottoclasse riceve automaticamente le proprietà e i metodi della classe da cui deriva e può aggiungerne di nuovi oppure ridefinire quelli esistenti tramite override. Questo meccanismo modella le relazioni di tipo "è un": un dipendente è una persona, un quadrato è una forma, e così via.

Quando la sottoclasse definisce un proprio `constructor`, è obbligatorio invocare `super()` per eseguire il costruttore della superclasse. La chiamata a `super()` deve precedere qualsiasi uso di `this` all'interno del costruttore, perché lo stato ereditato non risulta inizializzato finché il costruttore della superclasse non è stato eseguito.

```ts
class Persona {
  constructor(
    public nome: string,
    public eta: number,
  ) {}

  presentati(): string {
    return `Mi chiamo ${this.nome} e ho ${this.eta} anni.`;
  }
}

class Dipendente extends Persona {
  constructor(
    nome: string,
    eta: number,
    public reparto: string,
  ) {
    // La chiamata a super() esegue il costruttore di Persona e va
    // effettuata prima di accedere a this.
    super(nome, eta);
  }

  // Override del metodo ereditato.
  presentati(): string {
    return `${super.presentati()} Lavoro nel reparto ${this.reparto}.`;
  }
}

const d = new Dipendente("Alessia", 34, "Ingegneria");
console.log(d.presentati());
// "Mi chiamo Alessia e ho 34 anni. Lavoro nel reparto Ingegneria."
```

All'interno dell'override è possibile richiamare l'implementazione della superclasse tramite `super.metodo()`, riutilizzandone il risultato senza duplicare la logica.

## protected

Tra gli access modifier, `protected` occupa una posizione intermedia tra `public` e `private`. Un membro `protected` è accessibile all'interno della classe che lo dichiara e all'interno delle sue sottoclassi, ma non dall'esterno. Questo lo rende adatto a esporre dettagli di stato alle classi derivate pur mantenendoli nascosti al codice client.

```ts
class ContoBancario {
  protected saldo: number;

  constructor(saldoIniziale: number) {
    this.saldo = saldoIniziale;
  }
}

class ContoRisparmio extends ContoBancario {
  applicaInteressi(tasso: number): void {
    // saldo è accessibile perché ContoRisparmio è una sottoclasse.
    this.saldo += this.saldo * tasso;
  }
}

const conto = new ContoRisparmio(1000);
conto.applicaInteressi(0.05);

// Errore: Property 'saldo' is protected and only accessible within
// class 'ContoBancario' and its subclasses.
console.log(conto.saldo);
```

## Getters e setters

Gli accessor `get` e `set` permettono di intercettare la lettura e la scrittura di una proprietà come se si trattasse di un campo ordinario, eseguendo però del codice arbitrario al loro interno. Un `getter` viene invocato quando si legge la proprietà e può restituire un valore calcolato; un `setter` viene invocato quando le si assegna un valore e rappresenta il punto naturale in cui inserire la validazione. Un caso tipico prevede una proprietà `private` con un nome convenzionale preceduto dal trattino basso, affiancata da una coppia di accessor pubblici che ne controllano l'accesso.

```ts
class Temperatura {
  private _celsius = 0;

  get celsius(): number {
    return this._celsius;
  }

  set celsius(valore: number) {
    // Validazione: si rifiutano valori sotto lo zero assoluto.
    if (valore < -273.15) {
      throw new Error("Temperatura inferiore allo zero assoluto.");
    }
    this._celsius = valore;
  }

  // getter calcolato di sola lettura: non esiste un set corrispondente.
  get fahrenheit(): number {
    return this._celsius * 1.8 + 32;
  }
}

const t = new Temperatura();
t.celsius = 25; // invoca il setter
console.log(t.celsius); // 25, invoca il getter
console.log(t.fahrenheit); // 77

// Errore: lancia un'eccezione perché il valore non è valido.
t.celsius = -300;
```

Definire un solo `get` senza il corrispondente `set` produce una proprietà di sola lettura: ogni tentativo di assegnazione viene segnalato come errore dal compilatore.

## Membri static

I membri dichiarati con `static` appartengono alla classe nel suo complesso e non alle singole istanze. Vi si accede direttamente tramite il nome della classe, senza bisogno di creare un oggetto con `new`. Risultano utili per costanti condivise, contatori globali o funzioni di utilità logicamente legate alla classe ma indipendenti dallo stato di una specifica istanza.

All'interno della classe, un membro statico va referenziato attraverso il nome della classe e non tramite `this`, poiché `this` denota l'istanza corrente. Questa regola vale anche nel corpo di un `get` o di un `set`.

```ts
class Cerchio {
  static readonly PI = 3.14159;
  static contatore = 0;

  constructor(public raggio: number) {
    // Si accede al membro statico tramite il nome della classe.
    Cerchio.contatore++;
  }

  get area(): number {
    // Anche dentro un getter il membro statico si referenzia
    // con il nome della classe, non con this.
    return Cerchio.PI * this.raggio ** 2;
  }
}

new Cerchio(2);
new Cerchio(5);

console.log(Cerchio.contatore); // 2
console.log(Cerchio.PI); // 3.14159
```

## Classi e metodi abstract

L'astrazione trova espressione concreta nelle classi astratte, dichiarate con la keyword `abstract`. Una classe astratta non può essere istanziata direttamente: serve esclusivamente come base da cui derivare altre classi. Al suo interno può contenere metodi concreti, completi di implementazione, e metodi astratti, privi di corpo e contraddistinti anch'essi dalla keyword `abstract`. Ogni sottoclasse non astratta è obbligata a fornire un'implementazione per tutti i metodi astratti ereditati, garantendo così un contratto comune pur lasciando libertà sui dettagli.

```ts
abstract class Forma {
  constructor(public nome: string) {}

  // Metodo astratto: nessuna implementazione, va definito nelle sottoclassi.
  abstract area(): number;

  // Metodo concreto: condiviso da tutte le sottoclassi.
  descrivi(): string {
    return `${this.nome} con area ${this.area().toFixed(2)}.`;
  }
}

class Rettangolo extends Forma {
  constructor(
    public base: number,
    public altezza: number,
  ) {
    super("Rettangolo");
  }

  area(): number {
    return this.base * this.altezza;
  }
}

class Cerchio2 extends Forma {
  constructor(public raggio: number) {
    super("Cerchio");
  }

  area(): number {
    return Math.PI * this.raggio ** 2;
  }
}

const forme: Forma[] = [new Rettangolo(3, 4), new Cerchio2(2)];
for (const f of forme) {
  console.log(f.descrivi());
}

// Errore: Cannot create an instance of an abstract class.
const x = new Forma("generica");
```

La possibilità di trattare oggetti di tipo `Rettangolo` e `Cerchio2` attraverso il tipo comune `Forma`, ottenendo per ciascuno il comportamento appropriato di `area()`, illustra direttamente il polimorfismo.

## Pattern Singleton

Il pattern Singleton garantisce che di una determinata classe esista una sola istanza in tutta l'applicazione, fornendo un punto di accesso globale a essa. Lo si realizza rendendo `private` il costruttore, così da impedire la creazione di istanze tramite `new` dall'esterno, e affiancandolo a un metodo statico, convenzionalmente chiamato `getIstanza`, che alla prima invocazione crea l'oggetto e lo memorizza in un campo statico, restituendo poi sempre quella stessa istanza nelle chiamate successive.

```ts
class Configurazione {
  private static istanza: Configurazione | null = null;
  private impostazioni = new Map<string, string>();

  // Il costruttore privato blocca la creazione di istanze dall'esterno.
  private constructor() {}

  static getIstanza(): Configurazione {
    if (Configurazione.istanza === null) {
      Configurazione.istanza = new Configurazione();
    }
    return Configurazione.istanza;
  }

  imposta(chiave: string, valore: string): void {
    this.impostazioni.set(chiave, valore);
  }

  leggi(chiave: string): string | undefined {
    return this.impostazioni.get(chiave);
  }
}

const a = Configurazione.getIstanza();
a.imposta("tema", "scuro");

const b = Configurazione.getIstanza();
console.log(b.leggi("tema")); // "scuro": a e b sono la stessa istanza
console.log(a === b); // true

// Errore: Constructor of class 'Configurazione' is private and only
// accessible within the class declaration.
const c = new Configurazione();
```

Poiché il campo `istanza` è esso stesso statico, persiste per l'intera durata del programma e funge da riferimento condiviso che lega insieme tutte le chiamate a `getIstanza`.

## Domande

<details>
<summary>Perché in un costruttore di una sottoclasse la chiamata a super() deve precedere ogni uso di this?</summary>

Perché lo stato ereditato dalla superclasse non risulta inizializzato finché il suo costruttore non è stato eseguito. Accedere a `this` prima di `super()` significherebbe operare su un oggetto non ancora completamente costruito; per questo motivo il compilatore segnala l'errore "'super' must be called before accessing 'this' in the constructor of a derived class."

</details>

<details>
<summary>Qual è la differenza tra i modificatori private e protected rispetto all'ereditarietà?</summary>

Un membro `private` è accessibile soltanto all'interno della classe che lo dichiara, comprese le sue istanze, ma non dalle sottoclassi. Un membro `protected`, invece, è accessibile sia nella classe che lo dichiara sia in tutte le sue sottoclassi. Nessuno dei due è accessibile dal codice esterno alla gerarchia.

</details>

<details>
<summary>Come si accede a una proprietà static dall'interno di un metodo o di un getter della stessa classe?</summary>

Tramite il nome della classe e non tramite `this`. Un membro statico appartiene alla classe e non all'istanza, mentre `this` denota l'istanza corrente. Scrivere `Cerchio.PI` è corretto, mentre `this.PI` non risolve il membro statico. La regola vale anche all'interno di `get` e `set`.

</details>

<details>
<summary>Cosa accade se si tenta di istanziare direttamente una classe abstract con new?</summary>

Il compilatore emette l'errore "Cannot create an instance of an abstract class." Una classe astratta funge unicamente da base per la derivazione: deve essere estesa da una sottoclasse concreta che implementi tutti i suoi metodi astratti, e solo quella sottoclasse può essere istanziata.

</details>

<details>
<summary>Quali due elementi rendono possibile il pattern Singleton e perché?</summary>

Un costruttore `private`, che impedisce al codice esterno di creare nuove istanze con `new`, e un metodo statico come `getIstanza` abbinato a un campo statico che conserva l'unica istanza. Il metodo crea l'oggetto solo alla prima invocazione e in seguito restituisce sempre il riferimento già memorizzato, assicurando che esista una sola istanza condivisa.

</details>
