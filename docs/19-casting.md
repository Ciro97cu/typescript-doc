# Type casting e const assertions

In certe situazioni il programmatore conosce il tipo effettivo di un valore meglio del compilatore. Ciò accade tipicamente quando un'API restituisce un tipo volutamente ampio, oppure quando un dato proviene dall'esterno (una risposta di rete, l'input dell'utente, il DOM) e viene descritto in modo generico. Per comunicare al compilatore questa informazione aggiuntiva si utilizza una type assertion, comunemente chiamata anche type casting.

## Type assertion

Una type assertion non esegue alcuna conversione a runtime: non genera codice, non controlla i valori e non trasforma nulla. Si limita a istruire il type checker a trattare un valore come se fosse di un tipo diverso da quello inferito. È quindi una promessa che il programmatore fa al compilatore, e in quanto tale può rivelarsi sbagliata: se l'asserzione non corrisponde al tipo reale del valore, l'errore non emerge in fase di compilazione ma si manifesta a runtime.

Per questa ragione una type assertion va usata con parsimonia e solo quando si dispone effettivamente di informazioni che il compilatore non possiede. Quando possibile, è preferibile ricorrere al narrowing tramite type guard, perché in quel caso la restrizione del tipo viene verificata.

### Le due sintassi disponibili

TypeScript mette a disposizione due forme equivalenti per esprimere una type assertion. La prima, e quella consigliata, utilizza la keyword `as`:

```ts
const valore: unknown = "TypeScript";
const lunghezza = (valore as string).length;
```

La seconda forma utilizza le parentesi angolari con il tipo prefisso al valore:

```ts
const valore: unknown = "TypeScript";
const lunghezza = (<string>valore).length;
```

Le due notazioni producono lo stesso risultato, ma non sono interscambiabili in tutti i contesti. La sintassi con parentesi angolari entra in conflitto con la sintassi JSX, dove `<Tipo>` verrebbe interpretato come l'apertura di un tag. Per questo motivo, nei file `.tsx` la forma `<Tipo>` non è ammessa e l'unica opzione disponibile è `as`. Dato che la forma `as` funziona ovunque ed evita ambiguità, viene adottata come convenzione preferita anche nei file `.ts` privi di JSX.

### Asserzioni consentite e double assertion

Il compilatore non accetta qualsiasi asserzione: consente di convertire un tipo solo verso un tipo sufficientemente affine, ovvero un supertipo o un sottotipo. Tentare di passare direttamente fra due tipi senza alcuna sovrapposizione produce un errore:

```ts
const n = 42;
// Errore: Conversion of type 'number' to type 'string' may be a mistake
// because neither type sufficiently overlaps with the other.
const s = n as string;
```

Quando si è davvero certi del risultato e si vuole forzare comunque la conversione, occorre passare per `unknown`, eseguendo una cosiddetta double assertion. Si tratta di una scappatoia da riservare a casi eccezionali, perché disattiva completamente il controllo:

```ts
const n = 42;
const s = n as unknown as string;
```

## Type casting con il DOM

Il caso d'uso più frequente delle type assertion riguarda l'interazione con il DOM. I metodi del browser sono tipizzati con firme volutamente generiche, dato che il compilatore non può conoscere il contenuto della pagina HTML. Il metodo `document.getElementById`, ad esempio, restituisce `HTMLElement | null`: il tipo generico `HTMLElement` non espone proprietà specifiche come `value`, che appartiene invece a `HTMLInputElement`.

```html
<input id="email" type="email" />
```

Accedendo alla proprietà `value` su un `HTMLElement` il compilatore segnala un errore, perché quella proprietà non è dichiarata sul tipo generico:

```ts
const elemento = document.getElementById("email");
// Errore: 'elemento' is possibly 'null'.
// Errore: Property 'value' does not exist on type 'HTMLElement'.
const indirizzo = elemento.value;
```

Per risolvere occorre gestire due aspetti distinti. Il primo è la possibilità che l'elemento non esista: la firma include `null`, quindi con i controlli rigorosi attivi per impostazione predefinita il valore va verificato prima dell'uso. Il secondo è la restrizione al tipo concreto `HTMLInputElement`, ottenuta tramite type assertion:

```ts
const elemento = document.getElementById("email") as HTMLInputElement | null;

if (elemento !== null) {
  // All'interno del blocco il tipo è ristretto a HTMLInputElement
  const indirizzo: string = elemento.value;
  console.log(indirizzo.toUpperCase());
}
```

Un'alternativa più espressiva è `document.querySelector`, che accetta un generic e permette di indicare direttamente il tipo dell'elemento atteso, restituendo `HTMLInputElement | null` senza bisogno di un'asserzione separata:

```ts
const campo = document.querySelector<HTMLInputElement>("#email");

if (campo) {
  console.log(campo.value);
}
```

Vale la pena ribadire che l'asserzione `as HTMLInputElement` non garantisce in alcun modo che l'elemento sia davvero un input: se l'`id` puntasse a un `<div>`, l'accesso a `value` restituirebbe `undefined` a runtime senza alcun segnale dal compilatore. La responsabilità della correttezza ricade interamente sul programmatore.

## const assertions

Una const assertion si esprime con `as const` ed è uno strumento distinto, anche se condivide con le type assertion la keyword `as`. Anziché reinterpretare un valore come un altro tipo, `as const` chiede al compilatore di inferire il tipo più specifico e immutabile possibile per il valore a cui è applicata.

L'effetto si comprende meglio osservando il comportamento del widening. Normalmente, una variabile dichiarata con `let` o le proprietà di un oggetto vengono inferite con il tipo allargato: un letterale `"submit"` diventa `string`, e un array diventa un array mutabile. Con `as const` questo allargamento viene soppresso.

### as const su un array: tuple readonly

Applicato a un array literal, `as const` produce una tupla readonly: la lunghezza e il tipo di ciascun elemento sono fissati, e l'array non può più essere modificato a livello di tipo.

```ts
// Tipo inferito: string[]
const colori = ["rosso", "verde", "blu"];

// Tipo inferito: readonly ["rosso", "verde", "blu"]
const coloriConst = ["rosso", "verde", "blu"] as const;

// Errore: Property 'push' does not exist on type
// 'readonly ["rosso", "verde", "blu"]'.
coloriConst.push("giallo");
```

Questa caratteristica è particolarmente utile per derivare union type da una lista di valori, evitando la duplicazione fra l'elenco dei dati e la dichiarazione del tipo:

```ts
const livelli = ["debug", "info", "warning", "error"] as const;

// Tipo: "debug" | "info" | "warning" | "error"
type Livello = (typeof livelli)[number];

function registra(messaggio: string, livello: Livello): void {
  console.log(`[${livello}] ${messaggio}`);
}
```

### as const su un oggetto: record readonly

Applicato a un oggetto literal, `as const` rende ogni proprietà readonly e ne preserva il tipo letterale, invece di allargarlo al tipo generale. Le proprietà non possono essere riassegnate, e il loro valore viene trattato come un literal type anziché come `string` o `number`.

```ts
const configurazione = {
  host: "localhost",
  porta: 8080,
  protocollo: "https",
} as const;

// Tipo di configurazione.porta: 8080 (non number)
// Tipo di configurazione.protocollo: "https" (non string)

// Errore: Cannot assign to 'porta' because it is a read-only property.
configurazione.porta = 9090;
```

### Differenza con Object.freeze

È importante non confondere il livello su cui agiscono `as const` e `Object.freeze`, perché operano in due piani completamente diversi.

`as const` interviene esclusivamente a livello di tipo, durante la compilazione. Marca le proprietà come readonly per il type checker, ma non emette alcun codice e non ha effetto a runtime: il valore JavaScript prodotto resta un oggetto ordinario, perfettamente mutabile se ci si aggira intorno al sistema dei tipi (ad esempio con una type assertion).

`Object.freeze`, al contrario, è una funzione JavaScript che agisce a runtime: rende l'oggetto effettivamente immutabile, impedendo l'aggiunta, la rimozione e la modifica delle proprietà durante l'esecuzione. Un suo limite, però, è la superficialità: congela solo il primo livello, lasciando modificabili gli oggetti annidati.

```ts
const conConst = { porta: 8080 } as const;
// Nessun errore di compilazione, perché l'asserzione aggira il tipo readonly;
// a runtime la mutazione avviene comunque.
(conConst as { porta: number }).porta = 9090;

const conFreeze = Object.freeze({ porta: 8080 });
// Tipo readonly E immutabilità a runtime: l'assegnazione viene ignorata
// (o lancia un TypeError in strict mode).
(conFreeze as { porta: number }).porta = 9090;
```

In sintesi, `as const` offre garanzie statiche a costo zero in fase di esecuzione, mentre `Object.freeze` offre garanzie effettive a runtime. I due strumenti sono complementari: TypeScript tipizza peraltro il risultato di `Object.freeze` proprio con proprietà readonly, combinando entrambi i piani quando serve un'immutabilità reale e verificata dal compilatore.

## Domande

<details>
<summary>Perché la sintassi `as` è preferibile alla sintassi `<Tipo>` per esprimere una type assertion?</summary>

Le due forme sono semanticamente equivalenti, ma la sintassi con parentesi angolari `<Tipo>valore` entra in conflitto con JSX, dove `<Tipo>` viene interpretato come l'apertura di un tag. Nei file `.tsx` la forma `<Tipo>` non è quindi ammessa, mentre `as` funziona in ogni contesto. Per uniformità e per evitare ambiguità, `as` è adottata come convenzione preferita anche nei file `.ts`.
</details>

<details>
<summary>Una type assertion modifica il valore a runtime o controlla che il tipo dichiarato sia corretto?</summary>

No. Una type assertion agisce esclusivamente in fase di compilazione: non genera codice, non esegue conversioni e non verifica nulla a runtime. Si limita a istruire il type checker a trattare il valore come un tipo diverso. Se l'asserzione è errata, il problema non viene segnalato dal compilatore e si manifesta soltanto durante l'esecuzione.
</details>

<details>
<summary>Perché, accedendo a `value` su un risultato di `document.getElementById`, il compilatore segnala due problemi distinti, e come si risolvono?</summary>

La firma di `document.getElementById` restituisce `HTMLElement | null`. Il primo problema è che il valore può essere `null`, quindi con i controlli rigorosi attivi di default va verificato prima dell'uso, tipicamente con un `if`. Il secondo è che `HTMLElement` è un tipo generico che non dichiara la proprietà `value`, specifica di `HTMLInputElement`: serve quindi una type assertion `as HTMLInputElement` (oppure `document.querySelector<HTMLInputElement>`) per restringere il tipo all'elemento concreto.
</details>

<details>
<summary>Qual è la differenza tra il tipo inferito per `["a", "b"]` e per `["a", "b"] as const`?</summary>

Senza `as const`, l'array viene inferito con il tipo allargato e mutabile `string[]`. Con `as const`, l'inferenza produce una tupla readonly `readonly ["a", "b"]`: la lunghezza e i tipi letterali degli elementi sono fissati e l'array non è più modificabile a livello di tipo, tanto che metodi come `push` non risultano disponibili.
</details>

<details>
<summary>Quale differenza esiste fra `as const` e `Object.freeze` in termini di livello di azione?</summary>

`as const` opera solo a livello di tipo, durante la compilazione: marca le proprietà come readonly per il type checker ma non produce alcun effetto a runtime, lasciando l'oggetto JavaScript di fatto mutabile. `Object.freeze` opera a runtime: rende l'oggetto effettivamente immutabile impedendo aggiunte, rimozioni e modifiche durante l'esecuzione, sebbene solo al primo livello (è superficiale). Il primo dà garanzie statiche a costo zero, il secondo garanzie effettive in esecuzione.
</details>
