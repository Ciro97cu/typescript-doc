# any, unknown e never

Oltre ai tipi che descrivono valori concreti, il type system di TypeScript mette a disposizione alcuni tipi speciali che ne governano i confini. Tra questi, `any`, `unknown` e `never` ricoprono ruoli particolari: il primo disattiva i controlli, il secondo rappresenta in modo sicuro un valore di cui ancora non si conosce la forma, il terzo descrive ciò che non può mai esistere. Comprenderne le differenze è fondamentale per scrivere codice robusto e per sfruttare al meglio il type checking.

## any

Il tipo `any` rappresenta qualsiasi valore e di fatto disattiva il type system per la variabile a cui viene applicato. A un valore di tipo `any` si può assegnare qualunque cosa, e da esso si può leggere qualunque proprietà o invocare qualunque metodo senza che il compilatore sollevi obiezioni. Questa libertà ha però un costo elevato: si perdono il type checking, l'autocompletamento e la documentazione implicita che i tipi forniscono. Gli errori che il compilatore avrebbe normalmente intercettato vengono rimandati al runtime, dove diventano molto più difficili da individuare.

```ts
let valore: any = "ciao";

valore = 42; // ammesso: any accetta qualsiasi tipo
valore = { nome: "Ada" }; // ammesso

// Nessuno di questi accessi viene segnalato dal compilatore,
// ma il secondo provoca un errore a runtime
console.log(valore.nome); // "Ada"
console.log(valore.toUpperCase()); // TypeError a runtime: valore.toUpperCase is not a function
```

Il problema più insidioso di `any` è la sua natura "contagiosa": un valore `any` può essere assegnato a una variabile di qualsiasi altro tipo, propagando l'assenza di controlli ben oltre il punto in cui è stato introdotto.

```ts
function caricaDati(): any {
  return JSON.parse('{"eta": "trenta"}');
}

const eta: number = caricaDati().eta; // accettato, ma a runtime eta è una stringa
const doppio = eta * 2; // NaN, nessun errore di compilazione
```

In TypeScript 6.0 l'opzione `strict` è abilitata in modo predefinito, e con essa `noImplicitAny`: un parametro o una variabile privi di annotazione il cui tipo non possa essere dedotto generano un errore di compilazione, invece di assumere silenziosamente il tipo `any`. Questo aiuta a evitare che `any` si insinui nel codice senza che lo sviluppatore se ne accorga.

```ts
// Con noImplicitAny attivo (default in 6.0)
function saluta(nome) {
  // Errore: Parameter 'nome' implicitly has an 'any' type.
  return "Ciao " + nome;
}

// Corretto: il tipo è esplicito
function salutaTipizzato(nome: string) {
  return "Ciao " + nome;
}
```

Il tipo `any` resta utile in casi circoscritti, come la migrazione progressiva di una base di codice JavaScript verso TypeScript, ma il suo impiego va considerato un'eccezione temporanea e non una scelta di progettazione. Quando si ha a che fare con un valore di tipo realmente sconosciuto, la risposta corretta è `unknown`.

## unknown

Il tipo `unknown` è la controparte type-safe di `any`. Anch'esso può accogliere un valore di qualsiasi tipo, ma a differenza di `any` non consente di farne nulla finché non se ne è verificato il tipo effettivo. In altre parole, su un valore `unknown` non si possono leggere proprietà, invocare metodi o eseguire operazioni: il compilatore obbliga prima a restringere il tipo tramite narrowing.

```ts
let valore: unknown = "ciao";

valore = 42; // ammesso: unknown accetta qualsiasi tipo, come any
valore = { nome: "Ada" }; // ammesso

// Ma ogni accesso diretto viene bloccato dal compilatore:
console.log(valore.nome); // Errore: 'valore' is of type 'unknown'.
console.log(valore.toUpperCase()); // Errore: 'valore' is of type 'unknown'.
```

Per operare su un valore `unknown` occorre prima dimostrare al compilatore di che tipo si tratta, ricorrendo a una type guard come `typeof`, `instanceof` oppure `in`. All'interno del ramo in cui il controllo ha avuto successo, il tipo viene ristretto e l'accesso diventa lecito.

```ts
function stampaLunghezza(valore: unknown): void {
  if (typeof valore === "string") {
    // Qui valore è ristretto a string: l'accesso è sicuro
    console.log(valore.length);
  } else if (Array.isArray(valore)) {
    // Qui valore è ristretto a un array (any[])
    console.log(valore.length);
  } else {
    console.log("Valore non misurabile");
  }
}

stampaLunghezza("TypeScript"); // 10
stampaLunghezza([1, 2, 3]); // 3
stampaLunghezza(42); // "Valore non misurabile"
```

Una differenza importante riguarda l'assegnabilità: mentre `any` può essere assegnato a qualsiasi tipo, un valore `unknown` può essere assegnato soltanto a `unknown` stesso o ad `any`. Questo è ciò che impedisce all'incertezza di propagarsi inosservata nel resto del programma.

```ts
let sconosciuto: unknown = "testo";

let comeAny: any = sconosciuto; // ammesso
let comeUnknown: unknown = sconosciuto; // ammesso
let comeString: string = sconosciuto; // Errore: Type 'unknown' is not assignable to type 'string'.
```

Lo scenario tipico in cui `unknown` dà il meglio di sé è il consumo di dati provenienti dall'esterno, la cui forma non è garantita: il valore restituito dal parsing di una risposta JSON, l'argomento di un blocco `catch`, o l'input di una funzione generica. In questi casi `unknown` impone di validare i dati prima di usarli, trasformando un possibile errore a runtime in un controllo esplicito a compile time.

```ts
function leggiNome(dati: unknown): string {
  if (
    typeof dati === "object" &&
    dati !== null &&
    "nome" in dati &&
    typeof dati.nome === "string"
  ) {
    // Dopo i controlli, dati.nome è ristretto a string
    return dati.nome;
  }
  throw new Error("Formato dati non valido");
}

const risposta: unknown = JSON.parse('{"nome": "Ada"}');
console.log(leggiNome(risposta)); // "Ada"
```

Anche nella gestione delle eccezioni `unknown` è la scelta corretta: la variabile catturata in un blocco `catch` ha tipo `unknown`, perché in JavaScript si può lanciare qualsiasi valore, non solo un'istanza di `Error`.

```ts
try {
  throw new Error("Qualcosa è andato storto");
} catch (errore: unknown) {
  if (errore instanceof Error) {
    // Qui errore è ristretto a Error
    console.log(errore.message);
  } else {
    console.log("Errore non standard:", errore);
  }
}
```

Il motivo per cui `unknown` è preferibile ad `any` si riassume così: entrambi accettano qualsiasi valore in ingresso, ma `unknown` mantiene attivo il type system imponendo una verifica prima dell'uso, mentre `any` lo disattiva completamente. Con `unknown` la sicurezza dei tipi viene preservata e gli errori emergono durante la compilazione; con `any` vengono semplicemente rimandati al runtime. Quando un valore è davvero sconosciuto, dunque, la scelta corretta è `unknown`.

## never

Il tipo `never` rappresenta un valore che non può mai esistere. È il tipo di ritorno delle funzioni che non terminano mai in modo normale, ovvero quelle che lanciano sempre un'eccezione oppure entrano in un ciclo infinito. A differenza di `void`, che indica una funzione che termina senza restituire un valore utile, `never` indica una funzione il cui flusso di esecuzione non raggiunge mai il punto di ritorno.

```ts
// Funzione che lancia sempre un'eccezione: il tipo di ritorno inferito è never
function lanciaErrore(messaggio: string): never {
  throw new Error(messaggio);
}

// Funzione con ciclo infinito: non termina mai, quindi never
function ciclaPerSempre(): never {
  while (true) {
    // ...elaborazione che non si interrompe mai
  }
}
```

Il tipo `never` è anche il tipo dell'insieme vuoto: nessun valore gli è assegnabile (a parte un altro `never`), mentre `never` è assegnabile a qualunque altro tipo. Questa proprietà lo rende lo strumento naturale per i controlli di esaustività su una discriminated union. Assegnando il valore residuo a una variabile di tipo `never` nel ramo `default`, il compilatore segnala un errore se qualche variante non è stata gestita, trasformando un'omissione in un errore di compilazione.

```ts
type Forma =
  | { tipo: "cerchio"; raggio: number }
  | { tipo: "quadrato"; lato: number };

function area(forma: Forma): number {
  switch (forma.tipo) {
    case "cerchio":
      return Math.PI * forma.raggio ** 2;
    case "quadrato":
      return forma.lato ** 2;
    default:
      // Tutte le varianti sono gestite: qui forma ha tipo never
      const esaustivo: never = forma;
      return esaustivo;
  }
}
```

Se in seguito si aggiunge una nuova variante alla union senza gestirla nello `switch`, l'assegnazione a `never` smette di compilare, segnalando immediatamente il caso dimenticato.

```ts
type Forma =
  | { tipo: "cerchio"; raggio: number }
  | { tipo: "quadrato"; lato: number }
  | { tipo: "triangolo"; base: number; altezza: number }; // nuova variante

function area(forma: Forma): number {
  switch (forma.tipo) {
    case "cerchio":
      return Math.PI * forma.raggio ** 2;
    case "quadrato":
      return forma.lato ** 2;
    default:
      // Errore: Type '{ tipo: "triangolo"; ... }' is not assignable to type 'never'.
      const esaustivo: never = forma;
      return esaustivo;
  }
}
```

Il tipo `never` compare inoltre come risultato di operazioni sui tipi che non lasciano alcun valore possibile, come l'intersezione di tipi incompatibili o la sottrazione completa di un union tramite `Exclude`.

```ts
type Impossibile = string & number; // never: nessun valore è insieme string e number

type Vuoto = Exclude<"a" | "b", "a" | "b">; // never: l'union viene svuotata
```

## Domande

<details>
<summary>Qual è la differenza fondamentale tra any e unknown, dato che entrambi accettano qualsiasi valore?</summary>

Entrambi possono accogliere un valore di qualsiasi tipo, ma si comportano in modo opposto rispetto al type system. Con `any` il type checking viene disattivato: si può leggere qualsiasi proprietà o invocare qualsiasi metodo senza controlli, e il valore è assegnabile a qualunque altro tipo, propagando l'assenza di sicurezza. Con `unknown` il type system resta attivo: non è consentita alcuna operazione sul valore finché non se ne è ristretto il tipo tramite narrowing, e il valore è assegnabile soltanto a `unknown` o ad `any`. Per questo `unknown` è preferibile quando un valore è realmente sconosciuto.

</details>

<details>
<summary>Perché la variabile di un blocco catch ha tipo unknown e come la si gestisce correttamente?</summary>

In JavaScript è possibile lanciare qualsiasi valore, non solo istanze di `Error`, quindi il compilatore non può presumere il tipo di ciò che viene catturato e lo tipizza come `unknown`. Per usarlo in sicurezza occorre restringerne il tipo, tipicamente con `instanceof`:

```ts
try {
  // ...
} catch (errore: unknown) {
  if (errore instanceof Error) {
    console.log(errore.message);
  }
}
```

</details>

<details>
<summary>Qual è la differenza tra il tipo void e il tipo never come valore di ritorno di una funzione?</summary>

`void` indica una funzione che termina la propria esecuzione senza restituire un valore significativo (in JavaScript restituisce `undefined`). `never` indica una funzione che non termina mai in modo normale: o lancia sempre un'eccezione, o entra in un ciclo infinito, e perciò non raggiunge mai il punto di ritorno. Una funzione `void` restituisce il controllo al chiamante, una funzione `never` no.

</details>

<details>
<summary>Come si sfrutta never per garantire la gestione esaustiva di una discriminated union?</summary>

Poiché nessun valore è assegnabile a `never`, nel ramo `default` di uno `switch` sulla proprietà discriminante si assegna il valore residuo a una variabile di tipo `never`. Se tutte le varianti sono gestite, in quel ramo il valore ha già tipo `never` e l'assegnazione compila. Se in futuro si aggiunge una variante non gestita, il valore residuo non sarà più `never` e l'assegnazione produrrà un errore di compilazione, segnalando il caso dimenticato.

```ts
default:
  const esaustivo: never = forma;
  return esaustivo;
```

</details>

<details>
<summary>Cosa accade in TypeScript 6.0 a un parametro di funzione privo di annotazione di tipo?</summary>

Con i valori predefiniti di TypeScript 6.0, `strict` è attivo e include `noImplicitAny`. Un parametro privo di annotazione, il cui tipo non possa essere dedotto dal contesto, genera l'errore "Parameter implicitly has an 'any' type" invece di assumere silenziosamente il tipo `any`. Per risolverlo occorre annotare esplicitamente il parametro con il tipo corretto.

</details>
