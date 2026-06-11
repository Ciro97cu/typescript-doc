# Literal types

Un literal type rappresenta un singolo valore concreto trattato come tipo a sé stante. Anziché ammettere qualunque valore appartenente a un tipo primitivo, un literal type restringe ciò che è assegnabile a un'unica costante specifica: una determinata stringa, un determinato numero o un determinato valore booleano. In questo modo il type system non descrive più semplicemente la "forma" del dato, ma vincola il dato a far parte di un insieme chiuso e predeterminato di valori leciti.

I literal type esistono in tre varianti, allineate ai primitivi che li generano. I string literal type, come `"submit"`, ammettono soltanto quell'esatta sequenza di caratteri. I numeric literal type, come `200` o `404`, ammettono soltanto quel valore numerico. I boolean literal type, `true` e `false`, restringono il valore a una sola delle due possibilità del tipo `boolean`.

```ts
// Ogni literal type ammette un solo valore concreto
let metodo: "submit";
metodo = "submit"; // ok

metodo = "reset";
// Errore: Type '"reset"' is not assignable to type '"submit"'.
```

## Restringere i valori con i literal type

Un literal type isolato è raramente utile, perché un valore vincolato a una sola costante non offre alcuna libertà. La loro potenza emerge quando vengono combinati in un union type: si ottiene così un insieme finito ed esplicito di valori ammessi, che il compilatore verifica in fase di sviluppo. Questo pattern permette di modellare con precisione concetti che nella realtà assumono solo poche varianti, come lo stato di una richiesta, il tipo di un pulsante HTML o la direzione di un movimento.

```ts
// Union di string literal: solo questi tre valori sono leciti
type ButtonType = "submit" | "reset" | "button";

function creaPulsante(tipo: ButtonType): void {
  // ...
}

creaPulsante("submit"); // ok
creaPulsante("reset"); // ok

creaPulsante("invia");
// Errore: Argument of type '"invia"' is not assignable to parameter of type 'ButtonType'.
```

Anche i numeric literal type si prestano a questo utilizzo, ad esempio per rappresentare un insieme chiuso di codici di stato o un valore scelto fra alternative predefinite. Il vantaggio è duplice: il compilatore segnala immediatamente l'uso di un valore non previsto e, allo stesso tempo, gli strumenti di sviluppo offrono il completamento automatico delle sole alternative valide.

```ts
// Union di numeric literal per i codici di stato gestiti
type CodiceStato = 200 | 404 | 500;

function gestisci(codice: CodiceStato): string {
  switch (codice) {
    case 200:
      return "OK";
    case 404:
      return "Non trovato";
    case 500:
      return "Errore interno";
  }
}

gestisci(200); // ok

gestisci(301);
// Errore: Argument of type '301' is not assignable to parameter of type 'CodiceStato'.
```

I literal type collaborano in modo naturale con il narrowing. All'interno di un blocco controllato da un confronto di uguaglianza o da uno `switch`, il compilatore riduce automaticamente il tipo al singolo literal corrispondente, rendendo sicuro l'accesso al ramo di codice pertinente. Questa combinazione è alla base delle discriminated union, in cui una proprietà tipizzata con un literal type funge da discriminante fra le varianti di un oggetto.

## Widening dei literal type

Quando il compilatore inferisce il tipo di una variabile a partire dal valore con cui viene inizializzata, il comportamento dipende dalla modalità di dichiarazione. Con `let` e `var` la variabile è riassegnabile, perciò mantenere il literal type sarebbe inutilmente restrittivo: il compilatore applica il widening, ovvero allarga il tipo letterale al tipo primitivo generale a cui appartiene. Una stringa diventa `string`, un numero diventa `number`, un booleano diventa `boolean`.

```ts
// Con let il literal viene allargato (widening) al tipo generale
let saluto = "ciao";
// Tipo inferito: string

saluto = "buongiorno"; // ok: qualsiasi string è ammessa
```

Con `const`, invece, la variabile non può essere riassegnata: il valore resta immutabile per tutta la sua vita. Per questo motivo il compilatore non applica il widening e conserva il literal type più specifico, deducendo come tipo l'esatto valore assegnato.

```ts
// Con const il literal type specifico viene mantenuto
const saluto = "ciao";
// Tipo inferito: "ciao"

const codice = 200;
// Tipo inferito: 200

const attivo = true;
// Tipo inferito: true
```

Questa differenza ha conseguenze pratiche quando un valore inizializzato con `let` viene passato dove è atteso un literal type, oppure usato per inizializzare una proprietà di un oggetto. Poiché il tipo è già stato allargato, può non essere più compatibile con l'insieme ristretto richiesto.

```ts
type Direzione = "su" | "giu";

let valore = "su"; // tipo inferito: string
const valoreCostante = "su"; // tipo inferito: "su"

function muovi(direzione: Direzione): void {
  // ...
}

muovi(valoreCostante); // ok: il literal type "su" è preservato

muovi(valore);
// Errore: Argument of type 'string' is not assignable to parameter of type 'Direzione'.
```

Anche le proprietà degli oggetti sono soggette al widening: quando un oggetto viene creato con un literal `let` o assegnato a una variabile mutabile, le sue proprietà assumono i tipi primitivi generali, perché potrebbero essere riassegnate. Per preservare i literal type in profondità su un'intera struttura si ricorre a `as const`, che marca l'oggetto (o l'array) come readonly e impedisce il widening di tutte le sue proprietà.

```ts
// Senza as const: le proprietà subiscono il widening
const configMutabile = {
  metodo: "GET",
};
// Tipo di metodo: string

// Con as const: i literal type vengono preservati e le proprietà sono readonly
const config = {
  metodo: "GET",
} as const;
// Tipo di metodo: "GET" (readonly)

config.metodo = "POST";
// Errore: Cannot assign to 'metodo' because it is a read-only property.
```

In alternativa, è possibile forzare il mantenimento di un literal type tramite una type assertion mirata o annotando esplicitamente la variabile con il literal type desiderato, evitando così di affidarsi all'inferenza.

```ts
// Annotazione esplicita: il literal type viene imposto, niente widening
let metodo: "GET" = "GET";
// Tipo: "GET"

// Type assertion puntuale sul singolo valore
let altro = "POST" as const;
// Tipo: "POST"
```

## Domande

<details>
<summary>Qual è la differenza tra il tipo inferito di una variabile dichiarata con `let` e una dichiarata con `const`, quando entrambe sono inizializzate con la stringa `"ciao"`?</summary>

Con `let saluto = "ciao"` il compilatore applica il widening e inferisce il tipo `string`, perché la variabile è riassegnabile e mantenere il literal sarebbe inutilmente restrittivo. Con `const saluto = "ciao"`, essendo la variabile immutabile, non c'è widening e il tipo inferito è il literal type specifico `"ciao"`.
</details>

<details>
<summary>Perché un literal type isolato come `type T = "submit"` è poco utile, e come se ne sfrutta realmente la potenza?</summary>

Un literal type isolato vincola la variabile a un unico valore, lasciando di fatto una sola possibilità e quindi nessuna libertà. La loro utilità emerge combinandoli in un union type, ad esempio `type ButtonType = "submit" | "reset" | "button"`: si ottiene un insieme finito ed esplicito di valori ammessi, verificato dal compilatore in fase di sviluppo e suggerito dal completamento automatico.
</details>

<details>
<summary>Dato `let valore = "su"` con `type Direzione = "su" | "giu"`, perché `muovi(valore)` produce un errore mentre `muovi("su")` no?</summary>

Per via del widening, `let valore = "su"` ha tipo inferito `string`, non `"su"`. Il tipo `string` non è assegnabile a `Direzione`, perciò il compilatore segnala l'errore. Passando direttamente il literal `"su"`, invece, il valore corrisponde a un membro dell'union e l'assegnazione è valida. L'errore tipico è: `Argument of type 'string' is not assignable to parameter of type 'Direzione'`.
</details>

<details>
<summary>Che effetto ha `as const` su un oggetto rispetto ai literal type delle sue proprietà?</summary>

Senza `as const`, le proprietà di un oggetto subiscono il widening verso i tipi primitivi generali, perché potrebbero essere riassegnate. Applicando `as const`, il compilatore preserva i literal type specifici di ogni proprietà e marca l'oggetto come readonly. Così `{ metodo: "GET" } as const` dà a `metodo` il tipo `"GET"` e qualsiasi tentativo di riassegnazione viene rifiutato in fase di compilazione.
</details>

<details>
<summary>In quali contesti i literal type collaborano con il narrowing del compilatore?</summary>

All'interno di un confronto di uguaglianza o di uno `switch`, il compilatore restringe automaticamente il tipo al singolo literal corrispondente nel ramo pertinente, rendendo sicuro l'accesso ai valori. Questa collaborazione è il fondamento delle discriminated union, dove una proprietà tipizzata con un literal type funge da discriminante per distinguere in modo type-safe le varianti di un oggetto.
</details>
