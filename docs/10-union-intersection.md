# Union e intersection types

I union type e gli intersection type sono due dei costrutti più potenti del type system di TypeScript. Permettono di comporre tipi esistenti per descrivere valori che, nel codice reale, raramente appartengono a una sola categoria rigida. Un union type esprime un'alternativa ("uno tra"), mentre un intersection type esprime una combinazione ("tutto insieme"). Comprenderne la differenza e il modo in cui il compilatore tratta i membri risultanti è fondamentale per scrivere codice flessibile ma comunque type-safe.

## Union types

Un union type descrive un valore che può assumere uno tra più tipi. Si dichiara elencando i tipi membri separati dall'operatore `|`. È il modo idiomatico per modellare una variabile o un parametro che, a seconda del contesto, accetta più forme diverse: ad esempio un identificatore che può arrivare come `string` o come `number`.

```ts
function formattaId(id: string | number): string {
  return `ID-${id}`;
}

formattaId("42a");
formattaId(42);
// Errore: Argument of type 'boolean' is not assignable to parameter of type 'string | number'.
formattaId(true);
```

La caratteristica più importante dei union type riguarda l'accesso ai membri. Quando una variabile ha un tipo union, il compilatore consente di accedere soltanto alle proprietà e ai metodi comuni a tutti i tipi che compongono l'unione. Il motivo è di sicurezza: dato che a runtime il valore potrebbe essere di uno qualsiasi dei tipi membri, solo ciò che è garantito da tutti può essere usato senza ulteriori controlli.

```ts
function descrivi(valore: string | number): string {
  // Consentito: toString() esiste sia su string sia su number
  return valore.toString();
}

function lunghezza(valore: string | number): number {
  // Errore: Property 'length' does not exist on type 'string | number'.
  //         Property 'length' does not exist on type 'number'.
  return valore.length;
}
```

Per usare un membro specifico di uno solo dei tipi serve restringere l'unione a un tipo più preciso. Questa operazione prende il nome di narrowing e, per i tipi primitivi, la si ottiene comunemente con l'operatore `typeof`. All'interno del ramo in cui `typeof` ha verificato il tipo, il compilatore considera la variabile come appartenente a quel solo tipo e ne abilita tutti i membri.

```ts
function descrivi(valore: string | number): string {
  if (typeof valore === "string") {
    // In questo ramo il tipo è ristretto a string
    return `Stringa di ${valore.length} caratteri`;
  }
  // Qui resta soltanto number
  return `Numero arrotondato a ${valore.toFixed(2)}`;
}

descrivi("ciao");   // "Stringa di 4 caratteri"
descrivi(3.14159);  // "Numero arrotondato a 3.14"
```

Il narrowing con `typeof` funziona perché TypeScript analizza il flusso del codice (control flow analysis): riconosce che dentro il blocco `if` il controllo a runtime garantisce un determinato tipo, e propaga questa informazione fino alla fine del ramo. Lo stesso vale per il ramo complementare, dove per esclusione rimane l'altro membro dell'unione.

## Intersection types

Un intersection type combina più tipi in uno solo, ottenuto con l'operatore `&`. Il tipo risultante possiede simultaneamente tutte le proprietà di tutti i tipi combinati: un valore che soddisfa un'intersezione deve quindi essere conforme a ciascuno dei tipi che la compongono. Mentre l'union rappresenta un'alternativa, l'intersection rappresenta un'aggregazione, ed è particolarmente utile per comporre forme di oggetto a partire da blocchi riutilizzabili.

```ts
type Identificabile = {
  id: number;
};

type ConTimestamp = {
  creatoIl: Date;
  aggiornatoIl: Date;
};

type Entita = Identificabile & ConTimestamp;

const utente: Entita = {
  id: 1,
  creatoIl: new Date("2026-01-01"),
  aggiornatoIl: new Date("2026-06-11"),
};

// Errore: Property 'creatoIl' is missing in type '{ id: number; }' but required in type 'ConTimestamp'.
const incompleto: Entita = {
  id: 2,
};
```

A differenza dei union type, su un valore di tipo intersection si possono usare direttamente tutti i membri provenienti da ciascun tipo combinato, senza alcun narrowing: poiché il valore è garantito conforme a tutti i tipi contemporaneamente, ogni proprietà è sempre presente.

```ts
function registra(entita: Entita): void {
  // Tutti i membri sono accessibili senza controlli
  console.log(entita.id, entita.creatoIl.toISOString(), entita.aggiornatoIl.toISOString());
}
```

Gli intersection type sono spesso preferibili all'ereditarietà tra interfacce quando si vogliono comporre tipi in modo orizzontale, mescolando capacità diverse in maniera simile ai mixin. Va però considerato che, se i tipi combinati dichiarano la stessa proprietà con tipi incompatibili tra loro, l'intersezione di quella proprietà diventa `never`, rendendo di fatto impossibile soddisfare il tipo.

```ts
type A = { valore: string };
type B = { valore: number };

// La proprietà 'valore' diventa string & number, ovvero never
type Impossibile = A & B;

// Errore: Type 'string' is not assignable to type 'never'.
const x: Impossibile = { valore: "testo" };
```

## Intersection di union types

Quando l'operatore `&` viene applicato a due union type, il risultato non è la somma dei membri ma il tipo comune a entrambe le unioni, cioè ciò che è assegnabile a tutte e due. Sui tipi primitivi questo equivale a calcolare l'intersezione insiemistica dei membri: i tipi presenti in una sola delle due unioni vengono scartati, perché un valore deve appartenere contemporaneamente a entrambe.

```ts
// L'unico tipo presente in entrambe le union è number
type Comune = (string | number) & (number | boolean);
// Comune è esattamente: number

const n: Comune = 42;

// Errore: Type 'string' is not assignable to type 'number'.
const s: Comune = "no";
```

Lo stesso principio si applica alle union di literal type, dove il risultato è l'insieme dei valori condivisi. Questa proprietà permette di restringere un insieme di valori ammessi intersecando le alternative di due union differenti.

```ts
type Primarie = "rosso" | "verde" | "blu";
type Fredde = "verde" | "blu" | "viola";

// Restano solo i valori presenti in entrambe le union
type PrimarieFredde = Primarie & Fredde;
// PrimarieFredde è: "verde" | "blu"

const colore: PrimarieFredde = "blu";

// Errore: Type '"rosso"' is not assignable to type '"verde" | "blu"'.
const errato: PrimarieFredde = "rosso";
```

Se le due union non hanno alcun tipo membro in comune, l'intersezione si riduce a `never`, a segnalare che nessun valore può soddisfare entrambe le unioni simultaneamente. Riconoscere questo comportamento aiuta a interpretare correttamente i tipi calcolati che, apparentemente senza motivo, risultano inutilizzabili.

```ts
type Numeri = 1 | 2 | 3;
type Lettere = "a" | "b" | "c";

// Nessun membro condiviso: il risultato è never
type Vuoto = Numeri & Lettere;

// Errore: Type 'number' is not assignable to type 'never'.
const v: Vuoto = 1;
```

## Domande

<details>
<summary>Perché su un valore di tipo `string | number` non è possibile accedere alla proprietà `length`?</summary>

Perché su un union type il compilatore consente di accedere solo ai membri comuni a tutti i tipi dell'unione. La proprietà `length` esiste su `string` ma non su `number`, quindi non è garantita per ogni possibile valore. Per usarla occorre prima restringere il tipo con il narrowing, ad esempio con `if (typeof valore === "string")`, e accedere a `length` all'interno di quel ramo.

</details>

<details>
<summary>Qual è la differenza concettuale tra l'operatore `|` e l'operatore `&` sui tipi?</summary>

L'operatore `|` crea un union type, che descrive un valore appartenente a uno tra i tipi membri (un'alternativa). L'operatore `&` crea un intersection type, che descrive un valore conforme contemporaneamente a tutti i tipi combinati (un'aggregazione). Su un union sono accessibili solo i membri comuni, mentre su un intersection sono accessibili tutti i membri di tutti i tipi.

</details>

<details>
<summary>Qual è il tipo risultante di `(string | number) & (number | boolean)` e perché?</summary>

Il risultato è `number`. Intersecando due union type si ottiene il tipo comune a entrambe, cioè ciò che è assegnabile a tutte e due le unioni. L'unico membro presente sia in `string | number` sia in `number | boolean` è `number`, quindi gli altri vengono scartati.

</details>

<details>
<summary>Cosa succede se si interseca un type `{ valore: string }` con `{ valore: number }`?</summary>

La proprietà condivisa `valore` assume il tipo `string & number`, che si riduce a `never` perché nessun valore è contemporaneamente stringa e numero. Il tipo risultante resta formalmente valido, ma diventa di fatto impossibile da soddisfare: ogni tentativo di assegnare un valore alla proprietà `valore` genera un errore di assegnabilità a `never`.

</details>

<details>
<summary>Come fa il compilatore a sapere quale membro di un union è effettivamente in uso dentro un ramo `if`?</summary>

Tramite la control flow analysis. Quando un controllo a runtime come `typeof valore === "string"` compare in un `if`, il compilatore propaga l'informazione lungo il flusso del codice e considera la variabile ristretta a quel tipo all'interno del ramo, abilitandone tutti i membri. Nel ramo complementare, per esclusione, resta l'altro membro dell'unione.

</details>
