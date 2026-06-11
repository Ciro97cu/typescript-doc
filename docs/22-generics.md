# Generics

I generics permettono di scrivere componenti riutilizzabili che lavorano con tipi diversi senza rinunciare alla type safety. L'idea di fondo è parametrizzare un costrutto (funzione, classe, interfaccia o type alias) rispetto a uno o più tipi, lasciati indeterminati al momento della definizione e fissati solo quando il costrutto viene effettivamente utilizzato. In questo modo si evita sia la duplicazione del codice, sia la rinuncia all'informazione di tipo che si avrebbe ricorrendo ad `any`.

Il vantaggio rispetto ad `any` è sostanziale: con `any` il compilatore smette di verificare qualsiasi cosa, mentre con un parametro di tipo la relazione fra ingresso e uscita resta tracciata. TypeScript è in grado di inferire automaticamente gli argomenti di tipo nella maggior parte dei casi, riducendo la verbosità senza perdere il controllo statico.

## Funzioni generiche

Una funzione generica dichiara uno o più parametri di tipo fra parentesi angolari subito dopo il nome. Per convenzione il primo parametro si chiama `T` (da *type*), ma è possibile assegnargli un nome più descrittivo. Il parametro di tipo si comporta come una variabile che rappresenta un tipo concreto, determinato a ogni chiamata.

L'esempio canonico è la funzione identità, che restituisce esattamente ciò che riceve mantenendo il tipo dell'argomento:

```ts
function identita<T>(arg: T): T {
  return arg;
}

// Argomento di tipo esplicito
const testo = identita<string>("ciao"); // testo: string

// Argomento di tipo inferito dall'argomento
const numero = identita(42); // numero: number
```

La differenza rispetto a una firma con `any` è che il tipo viene propagato: il valore di ritorno conserva il tipo dell'argomento, quindi l'autocompletamento e il controllo dei tipi continuano a funzionare a valle della chiamata. Una funzione che accettasse e restituisse `any` perderebbe invece ogni informazione.

Più parametri di tipo consentono di mettere in relazione argomenti distinti. Una funzione che fonde due oggetti, per esempio, può dichiarare un parametro per ciascuno e restituire l'intersezione dei due:

```ts
function unisci<T, U>(primo: T, secondo: U): T & U {
  return { ...primo, ...secondo };
}

const persona = unisci({ nome: "Anna" }, { eta: 30 });
// persona: { nome: string } & { eta: number }
console.log(persona.nome, persona.eta);
```

## Built-in generics

Gran parte della libreria standard è già definita in termini di generics, spesso senza che ciò risulti evidente. Il tipo `Array<T>` descrive un array i cui elementi sono tutti di tipo `T`; la notazione `T[]` è semplicemente una forma abbreviata ed equivalente. Allo stesso modo, `Promise<T>` rappresenta un'operazione asincrona che, una volta completata, si risolve con un valore di tipo `T`.

```ts
// Le due forme sono equivalenti
const numeri: number[] = [1, 2, 3];
const altriNumeri: Array<number> = [4, 5, 6];

// Promise<T>: il valore risolto ha tipo T
function caricaUtente(id: number): Promise<{ id: number; nome: string }> {
  return Promise.resolve({ id, nome: "Mario" });
}

async function main(): Promise<void> {
  const utente = await caricaUtente(1);
  // utente: { id: number; nome: string }
  console.log(utente.nome);
}
```

Il parametro di tipo di `Promise<T>` viene tipicamente dedotto dal tipo di ritorno della funzione o dal valore con cui la promise viene risolta. Quando si attende una `Promise<T>` con `await`, il risultato ha proprio tipo `T`, ed è quindi pienamente tipizzato anche negli accessi successivi.

## Constraints con `extends`

Un parametro di tipo non vincolato può rappresentare qualunque tipo, e questo limita ciò che è lecito farne all'interno del corpo della funzione: non è possibile, per esempio, accedere a una proprietà che non tutti i tipi possiedono. I constraints risolvono il problema imponendo che il parametro di tipo soddisfi una determinata forma, tramite la parola chiave `extends`.

Vincolando `T` a un'interfaccia che dichiara una certa proprietà, il compilatore garantisce che tale proprietà sia disponibile e ne consente l'uso in modo sicuro:

```ts
interface ConLunghezza {
  length: number;
}

function lunghezza<T extends ConLunghezza>(elemento: T): number {
  // L'accesso a .length è sicuro: il constraint lo garantisce
  return elemento.length;
}

lunghezza("una stringa"); // ok: string ha length
lunghezza([1, 2, 3]); // ok: gli array hanno length
lunghezza({ length: 10, valore: 3 }); // ok

// Errore: Argument of type 'number' is not assignable to
// parameter of type 'ConLunghezza'.
lunghezza(42);
```

Il constraint restringe i tipi ammessi ma non sostituisce il parametro di tipo: `T` continua a rappresentare il tipo concreto passato alla chiamata, conservandone tutte le proprietà aggiuntive. È un comportamento diverso dall'accettare direttamente `ConLunghezza` come tipo del parametro, caso in cui l'informazione sulle proprietà extra andrebbe persa.

## `keyof` e accesso type-safe alle proprietà

L'operatore `keyof`, applicato a un tipo oggetto, produce l'union dei nomi delle sue proprietà sotto forma di literal type. Dato un tipo con proprietà `nome` ed `eta`, `keyof` restituisce l'union `"nome" | "eta"`.

```ts
interface Persona {
  nome: string;
  eta: number;
}

type ChiaviPersona = keyof Persona; // "nome" | "eta"
```

La vera utilità emerge combinando `keyof` con i generics. Una funzione che estrae il valore di una proprietà da un oggetto deve garantire due cose: che la chiave richiesta esista davvero e che il tipo restituito corrisponda a quello della proprietà. Entrambi gli obiettivi si raggiungono con il pattern `K extends keyof T`, abbinato all'indexed access type `T[K]`:

```ts
function getProprieta<T, K extends keyof T>(oggetto: T, chiave: K): T[K] {
  return oggetto[chiave];
}

const persona: Persona = { nome: "Giulia", eta: 28 };

const n = getProprieta(persona, "nome"); // n: string
const e = getProprieta(persona, "eta"); // e: number

// Errore: Argument of type '"indirizzo"' is not assignable to
// parameter of type 'keyof Persona'.
getProprieta(persona, "indirizzo");
```

Il vincolo `K extends keyof T` impedisce di passare un nome di proprietà inesistente, mentre `T[K]` fa sì che il tipo di ritorno dipenda dalla chiave specifica richiesta: passando `"nome"` si ottiene un `string`, passando `"eta"` un `number`. Si tratta di un accesso alle proprietà completamente type-safe, in cui un errore di battitura nella chiave viene intercettato in fase di compilazione anziché a runtime.

## Classi generiche

Anche le classi possono dichiarare parametri di tipo, indicati fra parentesi angolari dopo il nome della classe. Il parametro è visibile in tutto il corpo della classe, quindi proprietà e metodi possono farvi riferimento, garantendo che lavorino sempre con il tipo corretto. Un esempio tipico è un contenitore di dati che memorizza elementi di un tipo omogeneo:

```ts
class DataStorage<T> {
  private dati: T[] = [];

  aggiungi(elemento: T): void {
    this.dati.push(elemento);
  }

  rimuovi(elemento: T): void {
    const indice = this.dati.indexOf(elemento);
    if (indice > -1) {
      this.dati.splice(indice, 1);
    }
  }

  getElementi(): T[] {
    return [...this.dati];
  }
}

const archivioTesti = new DataStorage<string>();
archivioTesti.aggiungi("alfa");
archivioTesti.aggiungi("beta");
// Errore: Argument of type 'number' is not assignable to
// parameter of type 'string'.
archivioTesti.aggiungi(10);

const archivioNumeri = new DataStorage<number>();
archivioNumeri.aggiungi(1);
archivioNumeri.aggiungi(2);
const elementi = archivioNumeri.getElementi(); // elementi: number[]
```

Specificando `DataStorage<string>` si ottiene un contenitore che accetta e restituisce esclusivamente stringhe, mentre `DataStorage<number>` opera unicamente su numeri: la stessa definizione di classe serve entrambi i casi senza alcuna duplicazione. L'argomento di tipo viene fissato alla costruzione e propagato a ogni metodo, così che il compilatore segnali immediatamente qualsiasi uso incoerente.

Quando serve, anche le classi generiche possono dichiarare constraints sui propri parametri di tipo, con la medesima sintassi `extends` vista per le funzioni:

```ts
interface ConId {
  id: number;
}

class Repository<T extends ConId> {
  private elementi: T[] = [];

  salva(elemento: T): void {
    this.elementi.push(elemento);
  }

  trovaPerId(id: number): T | undefined {
    return this.elementi.find((e) => e.id === id);
  }
}

const repo = new Repository<{ id: number; titolo: string }>();
repo.salva({ id: 1, titolo: "Introduzione" });
const trovato = repo.trovaPerId(1); // trovato: { id: number; titolo: string } | undefined
```

In questo caso il constraint assicura che ogni tipo gestito dal `Repository` disponga di una proprietà `id`, indispensabile al metodo `trovaPerId` per effettuare la ricerca.

## Domande

<details>
<summary>Qual è la differenza pratica fra una funzione generica e una funzione che usa <code>any</code> per parametro e tipo di ritorno?</summary>

Con `any` il compilatore smette di verificare i tipi e l'informazione viene persa: il valore di ritorno è di tipo `any` e a valle della chiamata non si hanno né controllo dei tipi né autocompletamento. Con un parametro di tipo generico, invece, la relazione fra ingresso e uscita resta tracciata. In `function identita<T>(arg: T): T`, il tipo di ritorno coincide con quello dell'argomento, quindi il risultato conserva il tipo concreto passato alla chiamata e tutti i controlli statici continuano a operare.
</details>

<details>
<summary>Che cosa cambia, per il corpo di una funzione, vincolare un parametro di tipo con <code>T extends ConLunghezza</code> rispetto a non vincolarlo?</summary>

Un parametro di tipo non vincolato può rappresentare qualunque tipo, perciò non è lecito accedere a proprietà specifiche al suo interno, perché non è garantito che esistano. Il constraint `T extends ConLunghezza` impone che il tipo soddisfi quella forma, cosicché il compilatore consente di accedere in modo sicuro alle proprietà dichiarate da `ConLunghezza` (per esempio `length`). Il vincolo restringe i tipi ammessi senza sostituire `T`: il parametro continua a rappresentare il tipo concreto, comprese le sue eventuali proprietà aggiuntive.
</details>

<details>
<summary>Nel pattern <code>function getProprieta&lt;T, K extends keyof T&gt;(oggetto: T, chiave: K): T[K]</code>, quale ruolo svolgono rispettivamente <code>K extends keyof T</code> e <code>T[K]</code>?</summary>

`keyof T` produce l'union dei nomi delle proprietà di `T`, quindi `K extends keyof T` vincola la chiave a essere uno di quei nomi: passare una chiave inesistente genera un errore di compilazione. `T[K]` è un indexed access type e indica il tipo della proprietà corrispondente alla chiave specifica richiesta, facendo dipendere il tipo di ritorno dalla chiave. Passando `"nome"` si ottiene un `string`, passando `"eta"` un `number`. L'accesso alle proprietà risulta così completamente type-safe.
</details>

<details>
<summary>Le notazioni <code>number[]</code> e <code>Array&lt;number&gt;</code> sono equivalenti?</summary>

Sì, sono due forme della stessa cosa. `Array<T>` è un built-in generic della libreria standard e `T[]` ne è la sintassi abbreviata. `number[]` e `Array<number>` descrivono entrambe un array i cui elementi sono tutti di tipo `number`, e sono pienamente interscambiabili.
</details>

<details>
<summary>Indicando <code>new DataStorage&lt;string&gt;()</code>, in che momento viene fissato l'argomento di tipo e quale effetto produce sui metodi della classe?</summary>

L'argomento di tipo viene fissato al momento della costruzione dell'istanza e poi propagato a tutto il corpo della classe. In `DataStorage<string>`, il parametro `T` diventa `string`, quindi `aggiungi` accetta soltanto stringhe e `getElementi` restituisce un `string[]`. Una chiamata come `archivio.aggiungi(10)` su un'istanza `DataStorage<string>` viene segnalata come errore dal compilatore. La stessa definizione di classe serve qualunque tipo concreto, per esempio `DataStorage<number>`, senza duplicazione di codice.
</details>
