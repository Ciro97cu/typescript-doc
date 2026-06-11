# Utility types

Gli utility type sono generics predefiniti nella libreria standard di TypeScript che operano su un tipo esistente e ne producono uno nuovo come trasformazione. Il vantaggio principale risiede nel fatto che la struttura originale non viene riscritta né duplicata: a partire da un'unica definizione di riferimento si derivano varianti adattate a contesti diversi, e ogni modifica al tipo di partenza si propaga automaticamente a tutti i tipi derivati. Si tratta quindi di strumenti che rafforzano la coerenza del codice e riducono la ridondanza.

La maggior parte degli utility type è costruita internamente con mapped type e conditional type, ma per il loro utilizzo non è necessario conoscerne l'implementazione: è sufficiente comprendere quale trasformazione applicano. Vengono descritti di seguito quelli di uso più frequente.

## Partial<T>

`Partial<T>` rende opzionali tutte le proprietà di `T`. Il caso d'uso tipico è una funzione di aggiornamento che accetta solo un sottoinsieme dei campi di un oggetto: invece di pretendere l'intera struttura, si consente di passare unicamente le proprietà da modificare.

```ts
interface Todo {
  titolo: string;
  descrizione: string;
  completato: boolean;
}

function aggiornaTodo(todo: Todo, campiDaAggiornare: Partial<Todo>): Todo {
  // Si combina il todo esistente con i campi forniti
  return { ...todo, ...campiDaAggiornare };
}

const todo: Todo = {
  titolo: "Studiare TypeScript",
  descrizione: "Capitolo sugli utility type",
  completato: false,
};

// Si aggiorna solo una proprietà; le altre restano invariate
const aggiornato = aggiornaTodo(todo, { completato: true });
```

Poiché tutte le proprietà diventano opzionali, un oggetto vuoto `{}` è un valore valido per `Partial<Todo>`. Va quindi tenuto presente che `Partial` non garantisce la presenza di alcun campo.

## Required<T>

`Required<T>` è l'opposto di `Partial<T>`: rende obbligatorie tutte le proprietà di `T`, rimuovendo eventuali marcatori `?`. È utile quando un tipo ammette proprietà opzionali in fase di configurazione, ma in un determinato punto del programma si vuole avere la certezza che siano tutte valorizzate.

```ts
interface Configurazione {
  host?: string;
  porta?: number;
  timeout?: number;
}

function avviaServer(config: Required<Configurazione>): void {
  // Qui ogni proprietà è garantita: nessun accesso a undefined
  console.log(`${config.host}:${config.porta} (timeout ${config.timeout}ms)`);
}

const completa: Required<Configurazione> = {
  host: "localhost",
  porta: 8080,
  timeout: 5000,
};

avviaServer(completa);

// Errore: Property 'timeout' is missing in type
// '{ host: string; porta: number; }' but required in type 'Required<Configurazione>'.
avviaServer({ host: "localhost", porta: 8080 });
```

## Readonly<T>

`Readonly<T>` rende tutte le proprietà di `T` di sola lettura, introducendo un vincolo di immutabilità a livello di tipo. Dopo l'inizializzazione, qualsiasi tentativo di riassegnazione di una proprietà viene segnalato dal compilatore.

```ts
interface Punto {
  x: number;
  y: number;
}

const origine: Readonly<Punto> = { x: 0, y: 0 };

// Errore: Cannot assign to 'x' because it is a read-only property.
origine.x = 10;
```

Occorre precisare che `Readonly<T>` agisce solo al primo livello e solo a livello di tipo: non rende immutabili gli oggetti annidati né impedisce la mutazione a runtime. Per un'immutabilità effettiva durante l'esecuzione si ricorre a `Object.freeze`.

## Record<K, T>

`Record<K, T>` costruisce un tipo oggetto le cui chiavi appartengono all'union `K` e i cui valori sono di tipo `T`. È il modo idiomatico per descrivere una mappa o un dizionario in cui sia l'insieme delle chiavi sia il tipo dei valori sono noti.

```ts
type Ruolo = "admin" | "editor" | "lettore";

interface Permessi {
  lettura: boolean;
  scrittura: boolean;
}

const permessiPerRuolo: Record<Ruolo, Permessi> = {
  admin: { lettura: true, scrittura: true },
  editor: { lettura: true, scrittura: true },
  lettore: { lettura: true, scrittura: false },
};

// Quando K è un'union di literal, ogni chiave deve essere presente.
// Errore: Property 'lettore' is missing in type
// '{ admin: Permessi; editor: Permessi; }' but required in type 'Record<Ruolo, Permessi>'.
const incompleto: Record<Ruolo, Permessi> = {
  admin: { lettura: true, scrittura: true },
  editor: { lettura: true, scrittura: true },
};
```

Il primo parametro `K` deve essere un sottotipo di `string | number | symbol`. Se si utilizza `string` come chiave, l'oggetto risultante accetta qualunque chiave testuale, in modo analogo a una index signature.

## Pick<T, K>

`Pick<T, K>` costruisce un nuovo tipo selezionando da `T` soltanto le proprietà elencate nell'union di chiavi `K`. Risulta comodo quando da una struttura ricca serve esporne una versione ridotta, per esempio in un'anteprima o in un riepilogo.

```ts
interface Utente {
  id: number;
  nome: string;
  email: string;
  passwordHash: string;
}

// Si seleziona solo ciò che è sicuro mostrare
type UtentePubblico = Pick<Utente, "id" | "nome">;

const profilo: UtentePubblico = {
  id: 1,
  nome: "Giulia",
};

// Errore: Object literal may only specify known properties,
// and 'email' does not exist in type 'UtentePubblico'.
const errato: UtentePubblico = { id: 2, nome: "Marco", email: "x@y.it" };
```

## Omit<T, K>

`Omit<T, K>` è il complemento di `Pick`: produce un tipo con tutte le proprietà di `T` tranne quelle indicate in `K`. Si rivela pratico quando le proprietà da escludere sono meno numerose di quelle da mantenere.

```ts
interface Utente {
  id: number;
  nome: string;
  email: string;
  passwordHash: string;
}

// Tutte le proprietà tranne passwordHash
type UtenteSicuro = Omit<Utente, "passwordHash">;

const dati: UtenteSicuro = {
  id: 1,
  nome: "Giulia",
  email: "giulia@example.com",
};
```

A differenza di `Pick`, le chiavi indicate in `Omit` non sono vincolate a essere effettivamente presenti in `T`: il compilatore non segnala l'omissione di una chiave inesistente. Per questo `Pick` offre un controllo leggermente più rigoroso sulle chiavi specificate.

## Exclude<T, U>

`Exclude<T, U>` opera sulle union: rimuove da `T` tutti i membri assegnabili a `U`. Il risultato è quindi l'union dei tipi di `T` che non compaiono in `U`.

```ts
type Colore = "rosso" | "verde" | "blu" | "trasparente";

// Si escludono i valori non desiderati
type ColorePieno = Exclude<Colore, "trasparente">;
// ColorePieno = "rosso" | "verde" | "blu"

type Misti = string | number | boolean;
type SoloNonBooleani = Exclude<Misti, boolean>;
// SoloNonBooleani = string | number
```

## Extract<T, U>

`Extract<T, U>` è il duale di `Exclude`: dall'union `T` conserva soltanto i membri assegnabili a `U`. Permette di isolare un sottoinsieme di varianti che soddisfano un certo criterio.

```ts
type Evento =
  | { tipo: "click"; x: number; y: number }
  | { tipo: "tasto"; codice: string }
  | { tipo: "scroll"; delta: number };

// Si estrae solo la variante relativa alla tastiera
type EventoTasto = Extract<Evento, { tipo: "tasto" }>;
// EventoTasto = { tipo: "tasto"; codice: string }

type Misti = string | number | boolean;
type SoloStringhe = Extract<Misti, string>;
// SoloStringhe = string
```

## NonNullable<T>

`NonNullable<T>` rimuove `null` e `undefined` da `T`. Equivale concettualmente a `Exclude<T, null | undefined>` ed è utile quando, dopo un controllo, si vuole esprimere a livello di tipo che un valore non è più nullish.

```ts
type ValoreOpzionale = string | null | undefined;

type ValorePresente = NonNullable<ValoreOpzionale>;
// ValorePresente = string

function lunghezza(valore: ValoreOpzionale): number {
  const sicuro: NonNullable<ValoreOpzionale> = valore ?? "";
  return sicuro.length;
}
```

## Parameters<T>

`Parameters<T>` ricava da un tipo di funzione `T` la tupla dei tipi dei suoi parametri, nell'ordine in cui sono dichiarati. È particolarmente utile per riutilizzare la firma di una funzione esistente senza ripeterla manualmente. Il parametro `T` deve essere vincolato a un tipo di funzione, e per riferirsi al tipo di una funzione concreta si usa l'operatore `typeof`.

```ts
function creaUtente(nome: string, eta: number, attivo: boolean): void {
  console.log(nome, eta, attivo);
}

type ParametriCreaUtente = Parameters<typeof creaUtente>;
// ParametriCreaUtente = [nome: string, eta: number, attivo: boolean]

// Si riusa la tupla per inoltrare gli argomenti
function logEInoltra(...args: ParametriCreaUtente): void {
  console.log("Chiamata con:", args);
  creaUtente(...args);
}

logEInoltra("Anna", 30, true);
```

## ReturnType<T>

`ReturnType<T>` estrae il tipo di ritorno di un tipo di funzione `T`. Consente di mantenere allineato un tipo a ciò che una funzione effettivamente restituisce, anche quando tale tipo è inferito e non scritto esplicitamente.

```ts
function creaConfigurazione() {
  return {
    host: "localhost",
    porta: 8080,
    sicuro: false,
  };
}

// Il tipo segue automaticamente il valore restituito dalla funzione
type Configurazione = ReturnType<typeof creaConfigurazione>;
// Configurazione = { host: string; porta: number; sicuro: boolean }

const config: Configurazione = creaConfigurazione();
```

## Awaited<T>

`Awaited<T>` modella il risultato dell'operatore `await`: data una `Promise`, ne ricava il tipo con cui si risolve. La sua particolarità è la natura ricorsiva, che gli consente di gestire correttamente anche le promise annidate, restituendo sempre il tipo finale dopo aver svolto tutti i livelli di attesa.

```ts
type A = Awaited<Promise<string>>;
// A = string

type B = Awaited<Promise<Promise<number>>>;
// B = number (le promise annidate vengono risolte ricorsivamente)

async function caricaUtente(): Promise<{ id: number; nome: string }> {
  return { id: 1, nome: "Giulia" };
}

// Si ottiene il tipo del valore risolto combinando ReturnType e Awaited
type Utente = Awaited<ReturnType<typeof caricaUtente>>;
// Utente = { id: number; nome: string }
```

## Domande

<details>
<summary>Qual è la differenza fra Partial&lt;T&gt; e Required&lt;T&gt;?</summary>

`Partial<T>` rende opzionali tutte le proprietà di `T`, aggiungendo a ciascuna il marcatore `?`; di conseguenza anche un oggetto vuoto soddisfa il tipo. `Required<T>` esegue la trasformazione inversa, rimuovendo i marcatori `?` e rendendo obbligatorie tutte le proprietà. Sono quindi operazioni complementari: applicando l'uno e poi l'altro si ottiene un tipo con tutte le proprietà richieste.
</details>

<details>
<summary>Perché Pick&lt;T, K&gt; offre un controllo più rigoroso sulle chiavi rispetto a Omit&lt;T, K&gt;?</summary>

In `Pick<T, K>` il parametro `K` è vincolato a `keyof T`, quindi indicare una chiave che non appartiene a `T` provoca un errore di compilazione. In `Omit<T, K>` invece `K` è vincolato a `keyof any` (ossia `string | number | symbol`), perciò è possibile elencare chiavi inesistenti in `T` senza che il compilatore lo segnali. `Pick` valida quindi le chiavi specificate, mentre `Omit` è più permissivo.
</details>

<details>
<summary>Quale relazione lega Exclude, Extract e NonNullable?</summary>

Tutti e tre operano sulle union. `Exclude<T, U>` rimuove da `T` i membri assegnabili a `U`, mentre `Extract<T, U>` ne conserva solo quelli assegnabili a `U`: sono operazioni duali. `NonNullable<T>` è un caso particolare di esclusione, equivalente a `Exclude<T, null | undefined>`, e serve a eliminare i valori nullish.
</details>

<details>
<summary>Perché con Parameters e ReturnType si usa spesso typeof davanti al nome della funzione?</summary>

`Parameters<T>` e `ReturnType<T>` richiedono come argomento un tipo di funzione, non un valore. Il nome di una funzione dichiarata è un valore, non un tipo; l'operatore `typeof` applicato a quel nome ne ricava il tipo della firma, che può poi essere passato all'utility. Si scrive perciò `ReturnType<typeof miaFunzione>` e non `ReturnType<miaFunzione>`.
</details>

<details>
<summary>Cosa rende Awaited&lt;T&gt; adatto a gestire le promise annidate?</summary>

`Awaited<T>` è definito in modo ricorsivo: se il tipo che risolve è a sua volta una `Promise`, l'utility continua a svolgere i livelli finché non raggiunge un tipo che non è più una promise. Per questo `Awaited<Promise<Promise<number>>>` produce `number` e non `Promise<number>`, riproducendo fedelmente il comportamento di `await` a runtime.
</details>
