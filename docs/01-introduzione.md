# Introduzione a TypeScript

TypeScript è un linguaggio di programmazione open source sviluppato da Microsoft. La sua caratteristica fondamentale è quella di essere un superset di JavaScript: ogni programma JavaScript valido è anche un programma TypeScript valido. Su questa base, TypeScript aggiunge un sistema di tipi statici che il compilatore analizza prima dell'esecuzione, consentendo di scrivere codice più chiaro e manutenibile e di individuare un'ampia categoria di errori durante lo sviluppo, anziché a runtime.

Poiché TypeScript estende JavaScript senza modificarne la semantica di esecuzione, il codice scritto viene tradotto (transpiled) in JavaScript standard, eseguibile in qualsiasi browser o in Node.js. Le annotazioni di tipo esistono soltanto in fase di sviluppo e di compilazione: vengono rimosse durante la generazione del JavaScript e non hanno alcun effetto sul comportamento a runtime.

Questa guida è validata su **TypeScript 6.0**, la release stabile più recente, e tutti gli esempi riflettono le impostazioni predefinite e le funzionalità di tale versione.

## Superset di JavaScript

Il fatto che TypeScript sia un superset di JavaScript ha una conseguenza pratica immediata: l'adozione può essere graduale. È possibile partire da un file JavaScript esistente, rinominarlo con estensione `.ts` e iniziare ad aggiungere annotazioni di tipo dove servono, senza dover riscrivere tutto.

Il codice seguente è valido sia come JavaScript sia come TypeScript:

```ts
function saluta(nome) {
  return "Ciao, " + nome;
}

console.log(saluta("Mario"));
```

Aggiungendo le annotazioni di tipo, lo stesso codice acquista la verifica statica che caratterizza TypeScript:

```ts
function saluta(nome: string): string {
  return "Ciao, " + nome;
}

console.log(saluta("Mario"));
// Errore: Argument of type 'number' is not assignable to parameter of type 'string'.
console.log(saluta(42));
```

## Tipi statici

JavaScript è a tipizzazione dinamica: il tipo di un valore viene determinato a runtime e una stessa variabile può cambiare tipo durante l'esecuzione. Questa flessibilità rende possibili errori che si manifestano soltanto quando il programma è già in esecuzione, spesso in situazioni difficili da riprodurre.

```ts
// Comportamento ammesso in JavaScript: la variabile cambia tipo a runtime
let valore = "10";
valore = valore * 2; // a runtime produce NaN, senza errori evidenti
```

TypeScript è invece a tipizzazione statica: il tipo viene fissato in fase di dichiarazione e ogni operazione successiva viene controllata dal compilatore. L'errore equivalente viene quindi segnalato prima dell'esecuzione:

```ts
let valore: string = "10";
// Errore: The left-hand side of an arithmetic operation must be of type
// 'any', 'number', 'bigint' or an enum type.
valore = valore * 2;
```

Non sempre è necessario annotare esplicitamente i tipi. Grazie alla type inference, il compilatore deduce automaticamente il tipo a partire dal valore assegnato. L'annotazione esplicita resta utile quando l'inferenza non è sufficiente, come tipicamente accade per i parametri di funzione.

```ts
// Il tipo viene inferito come number, l'annotazione è superflua
let contatore = 0;

// Senza annotazione il parametro sarebbe implicitamente 'any':
// con strict attivo il compilatore lo segnala
function raddoppia(n: number): number {
  return n * 2;
}
```

## Vantaggi

L'impiego di tipi statici previene molti errori comuni, come l'accesso a proprietà inesistenti, il passaggio di argomenti del tipo sbagliato o la gestione incompleta dei valori `null` e `undefined`. Oltre alla sicurezza, TypeScript offre un tooling avanzato: gli editor che ne sfruttano le informazioni di tipo forniscono autocompletamento intelligente, navigazione precisa nel codice, refactoring affidabili e documentazione contestuale direttamente durante la scrittura.

Un ulteriore vantaggio è l'allineamento con JavaScript moderno: TypeScript supporta le funzionalità più recenti del linguaggio, come classi, arrow function e moduli ES, e le arricchisce con costrutti propri quali interfacce, generics e tipi avanzati. Nelle configurazioni predefinite di TypeScript 6.0 il controllo rigoroso (`strict`) è attivo: in questo modo i comportamenti più insidiosi vengono intercettati fin da subito.

```ts
interface Persona {
  nome: string;
  eta: number;
}

function descrivi(persona: Persona): string {
  // L'editor suggerisce 'nome' ed 'eta' tramite autocompletamento
  return `${persona.nome} ha ${persona.eta} anni`;
}

// Errore: Property 'eta' is missing in type '{ nome: string; }'
//         but required in type 'Persona'.
descrivi({ nome: "Anna" });
```

## Principali innovazioni

Rispetto a JavaScript, TypeScript introduce un insieme di costrutti che ampliano in modo significativo le possibilità espressive del codice. Di seguito vengono presentate le innovazioni più rilevanti, ciascuna con un esempio essenziale; ognuna di esse è approfondita nelle pagine dedicate della guida.

### Tipi statici

Le annotazioni di tipo permettono di descrivere la forma dei dati e le firme delle funzioni, rendendo esplicite le aspettative del codice e delegando al compilatore la verifica della loro coerenza.

```ts
let titolo: string = "Introduzione";
let pagine: number = 320;
let disponibile: boolean = true;
let etichette: string[] = ["guida", "typescript"];
```

### Interfacce

Un'interfaccia definisce un contratto per la forma di un oggetto, elencando le proprietà e i metodi attesi. Le interfacce sono estendibili e possono essere implementate dalle classi.

```ts
interface Veicolo {
  marca: string;
  avvia(): void;
}

class Auto implements Veicolo {
  constructor(public marca: string) {}

  avvia(): void {
    console.log(`${this.marca} avviata`);
  }
}
```

### Generics

I generics consentono di scrivere componenti riutilizzabili che operano su tipi diversi mantenendo la type safety. Un parametro di tipo, indicato per convenzione con una lettera maiuscola come `T`, viene risolto al momento dell'uso.

```ts
function primo<T>(elementi: T[]): T | undefined {
  return elementi[0];
}

const n = primo([1, 2, 3]); // tipo inferito: number | undefined
const s = primo(["a", "b"]); // tipo inferito: string | undefined
```

### Moduli

TypeScript adotta i moduli ES come sistema standard per organizzare il codice in file separati, esponendo e importando funzionalità tramite `export` e `import`. Nelle impostazioni predefinite di TypeScript 6.0 il valore di `module` è `esnext`.

```ts
// file: matematica.ts
export function somma(a: number, b: number): number {
  return a + b;
}

// file: main.ts
import { somma } from "./matematica.js";

console.log(somma(2, 3));
```

### Decorators

Un decorator è una dichiarazione speciale, nella forma `@espressione`, che si applica a una classe o a un suo membro (metodo, accessor o campo) per aggiungervi comportamento o annotazioni. TypeScript 6.0 supporta i decorator standard di ECMAScript (Stage 3), abilitati di default e senza alcuna opzione aggiuntiva. Un decorator riceve due argomenti, il valore decorato e un oggetto `context` che ne descrive il contesto.

```ts
function log(
  metodoOriginale: Function,
  context: ClassMethodDecoratorContext
) {
  return function (this: unknown, ...args: unknown[]) {
    console.log(`Chiamata a ${String(context.name)}`);
    return metodoOriginale.apply(this, args);
  };
}

class Calcolatrice {
  @log
  somma(a: number, b: number): number {
    return a + b;
  }
}
```

### Tipi avanzati

Oltre ai tipi di base, TypeScript offre costrutti per comporre e trasformare i tipi: gli union type descrivono valori che possono assumere una tra più forme, gli intersection type combinano più tipi, i literal type restringono i valori ammessi a costanti specifiche, mentre mapped type e conditional type permettono di derivare nuovi tipi da quelli esistenti.

```ts
// Union type
type Esito = "ok" | "errore" | "in_attesa";

// Conditional type
type Elemento<T> = T extends (infer U)[] ? U : T;

type A = Elemento<number[]>; // number
type B = Elemento<string>; // string
```

### Tooling

Le informazioni di tipo non servono soltanto al compilatore: alimentano anche gli strumenti di sviluppo. Il compilatore ufficiale `tsc` verifica e traduce il codice, mentre l'integrazione con gli editor offre diagnostica in tempo reale, completamento e refactoring. Per eseguire rapidamente un file TypeScript durante lo sviluppo si può ricorrere a strumenti come `tsx`, mentre la compilazione continua resta disponibile tramite la modalità watch.

```bash
# Installazione di TypeScript nel progetto
npm install --save-dev typescript

# Compilazione di tutti i file secondo tsconfig.json, in modalità watch
npx tsc --watch

# Esecuzione diretta di un file TypeScript senza compilazione manuale
npx tsx main.ts
```

Una configurazione minima di partenza, coerente con le impostazioni predefinite di TypeScript 6.0, può limitarsi a poche opzioni in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "es2025",
    "module": "esnext",
    "moduleResolution": "nodenext",
    "strict": true
  }
}
```

## Domande

<details>
<summary>Cosa significa che TypeScript è un superset di JavaScript?</summary>

Significa che la sintassi di JavaScript è interamente compresa in quella di TypeScript: ogni programma JavaScript valido è anche un programma TypeScript valido. TypeScript estende JavaScript aggiungendo un sistema di tipi statici e alcuni costrutti propri, ma non ne modifica la semantica di esecuzione. Il codice TypeScript viene tradotto in JavaScript standard, e le annotazioni di tipo vengono rimosse durante la compilazione.
</details>

<details>
<summary>Qual è la differenza pratica tra tipizzazione statica e tipizzazione dinamica rispetto agli errori?</summary>

Con la tipizzazione dinamica di JavaScript il tipo di un valore è determinato a runtime, quindi gli errori di tipo si manifestano soltanto durante l'esecuzione. Con la tipizzazione statica di TypeScript il tipo è fissato in fase di dichiarazione e viene controllato dal compilatore: gli errori di tipo emergono già durante lo sviluppo, prima che il programma venga eseguito.
</details>

<details>
<summary>Le annotazioni di tipo influiscono sul comportamento del programma a runtime?</summary>

No. Le annotazioni di tipo esistono solo in fase di sviluppo e di compilazione. Durante la traduzione in JavaScript vengono rimosse, quindi non producono alcun codice e non alterano il comportamento del programma in esecuzione. Il loro scopo è la verifica statica e il supporto degli strumenti di sviluppo.
</details>

<details>
<summary>Quale tipo di decorator supporta TypeScript 6.0 e con quale firma?</summary>

TypeScript 6.0 supporta i decorator standard di ECMAScript (Stage 3), abilitati di default senza alcuna opzione di configurazione aggiuntiva. La firma prevede due argomenti: il valore decorato e un oggetto `context` (ad esempio `ClassMethodDecoratorContext` per un metodo) che espone informazioni come `kind`, `name`, `static` e `addInitializer`.

```ts
function decoratore(
  valore: Function,
  context: ClassMethodDecoratorContext
) {
  // ...
}
```

</details>

<details>
<summary>A cosa servono i generics e quale vantaggio offrono rispetto all'uso di un tipo generico come 'any'?</summary>

I generics permettono di scrivere componenti riutilizzabili che operano su tipi diversi senza rinunciare alla type safety. A differenza di `any`, che disattiva del tutto il controllo dei tipi, un parametro di tipo come `T` mantiene la relazione tra input e output: il tipo concreto viene risolto al momento dell'uso, e il compilatore continua a verificare la coerenza e a fornire autocompletamento sul risultato.
</details>
