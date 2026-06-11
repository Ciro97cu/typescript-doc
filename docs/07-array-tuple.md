# Array e tuple

Quando i dati da gestire non sono valori singoli ma collezioni ordinate, TypeScript mette a disposizione due costrutti distinti: gli array, pensati per sequenze omogenee di lunghezza variabile, e le tuple, pensate per insiemi a lunghezza fissa in cui ogni posizione ha un significato e un tipo precisi. Pur condividendo la stessa rappresentazione a runtime (entrambi sono array JavaScript), il type system li tratta in modo molto diverso, ed è proprio questa differenza a renderli utili in scenari complementari.

## Array

Un array contiene più valori dello stesso tipo, ordinati e accessibili tramite un indice con base zero. Il primo elemento occupa la posizione `0`, il secondo la posizione `1`, e così via. TypeScript verifica il tipo degli elementi: qualsiasi tentativo di inserire un valore incompatibile con quello dichiarato viene segnalato come errore già in fase di compilazione.

Per annotare un array esistono due sintassi del tutto equivalenti. La prima, più diffusa e concisa, consiste nel posporre `[]` al tipo degli elementi (`tipo[]`). La seconda utilizza il generic `Array<tipo>`. La scelta è puramente stilistica: il tipo risultante è identico in entrambi i casi.

```ts
// Sintassi tipo[]
const nomi: string[] = ["Anna", "Luca", "Marco"];

// Sintassi Array<tipo>, perfettamente equivalente
const eta: Array<number> = [34, 28, 41];

// L'accesso avviene tramite indice a base zero
const primo: string = nomi[0]; // "Anna"

// Errore: Argument of type 'number' is not assignable to parameter of type 'string'.
nomi.push(42);
```

Quando un array viene inizializzato direttamente con dei valori, non è necessario annotarlo: la type inference deduce il tipo degli elementi a partire dal contenuto. Un letterale che contiene valori di tipi differenti produce automaticamente un union type.

```ts
// Tipo inferito: number[]
const punteggi = [10, 20, 30];

// Tipo inferito: (string | number)[]
const misti = ["uno", 2, "tre"];

misti.push(4);        // ammesso: number fa parte dell'union
misti.push("quattro"); // ammesso: anche string fa parte dell'union

// Errore: Argument of type 'boolean' is not assignable to parameter of type 'string | number'.
misti.push(true);
```

La forma `Array<tipo>` diventa particolarmente leggibile quando il tipo degli elementi è a sua volta articolato, ad esempio un union type o un tipo oggetto. In questi casi le parentesi angolari delimitano in modo netto il contenuto dell'array senza richiedere parentesi aggiuntive.

```ts
type Coordinata = { x: number; y: number };

// Più chiaro di { x: number; y: number }[]
const percorso: Array<Coordinata> = [
  { x: 0, y: 0 },
  { x: 3, y: 4 },
];

// Iterazione type-safe: ogni punto è tipizzato come Coordinata
for (const punto of percorso) {
  console.log(punto.x + punto.y);
}
```

Quando occorre un array che non deve essere modificato dopo la creazione, si può ricorrere al tipo `readonly`. Un array `readonly` non espone i metodi che alterano la collezione, come `push`, `pop` o `splice`, e ogni tentativo di assegnazione tramite indice viene rifiutato dal compilatore.

```ts
const colori: readonly string[] = ["rosso", "verde", "blu"];

// Errore: Property 'push' does not exist on type 'readonly string[]'.
colori.push("giallo");

// Errore: Index signature in type 'readonly string[]' only permits reading.
colori[0] = "nero";
```

## Tuple

Una tupla è un array in cui il numero degli elementi e il tipo di ciascuna posizione sono noti in anticipo e fissati nella dichiarazione. Risulta lo strumento ideale per raggruppare valori di tipi diversi quando l'ordine ha un significato preciso e la lunghezza è determinata: una coppia chiave/valore, una riga di una tabella, una coordinata con etichetta. A differenza dell'array, dove ogni posizione condivide lo stesso tipo, nella tupla ciascun indice possiede il proprio tipo.

```ts
// Tupla a lunghezza e tipi noti: prima un string, poi un number
let utente: [string, number];

utente = ["Anna", 34]; // corretto

// Errore: Type 'number' is not assignable to type 'string'.
//         Type 'string' is not assignable to type 'number'.
utente = [34, "Anna"];

// L'accesso a ogni posizione restituisce il tipo dichiarato per quell'indice
const id: string = utente[0]; // string
const punti: number = utente[1]; // number
```

L'accesso a un indice valido restituisce esattamente il tipo dichiarato per quella posizione, mentre l'accesso a un indice fuori dai limiti dichiarati viene segnalato come errore. Questa è la differenza sostanziale rispetto a un array, dove l'accesso a qualunque indice è sempre lecito a livello di tipo.

```ts
const riga: [string, number, boolean] = ["totale", 100, true];

// Errore: Tuple type '[string, number, boolean]' of length '3'
//         has no element at index '3'.
const inesistente = riga[3];
```

Gli elementi di una tupla possono essere resi opzionali apponendo `?` al tipo, ma soltanto in coda: una volta introdotto un elemento opzionale, tutti quelli successivi devono essere a loro volta opzionali. Non è consentito far seguire un elemento obbligatorio a uno opzionale, perché ciò renderebbe ambigua la posizione degli elementi.

```ts
// Il terzo elemento è opzionale: la tupla può avere lunghezza 2 o 3
type Punto = [number, number, number?];

const punto2D: Punto = [10, 20];      // corretto: z omesso
const punto3D: Punto = [10, 20, 30];  // corretto: z presente

// Errore: A required element cannot follow an optional element.
type NonValida = [number, string?, boolean];
```

Le tuple supportano anche gli elementi etichettati, che assegnano un nome a ciascuna posizione. Le etichette non cambiano il comportamento a runtime né i tipi, ma migliorano la leggibilità e compaiono nei suggerimenti dell'editor, rendendo evidente il ruolo di ogni elemento.

```ts
// Le etichette documentano il significato di ciascuna posizione
type IntervalloDate = [inizio: Date, fine: Date];

const vacanze: IntervalloDate = [new Date("2026-08-01"), new Date("2026-08-15")];
```

Va però tenuta presente una particolarità ereditata da JavaScript: a runtime una tupla resta un normale array, e il metodo `push` consente di aggiungere elementi oltre la lunghezza dichiarata senza che il compilatore lo impedisca. Il controllo sulla lunghezza viene applicato all'assegnazione tramite letterale e all'accesso per indice, ma non alle chiamate di `push`. Per questo motivo l'uso di `push` su una tupla va evitato quando si vuole preservare la garanzia sulla lunghezza fissa.

```ts
const coppia: [string, number] = ["punteggio", 10];

// Nessun errore di compilazione, anche se la tupla "dovrebbe" avere lunghezza 2:
coppia.push(99);

// A runtime coppia vale ["punteggio", 10, 99]: la lunghezza dichiarata è stata bypassata.
// Tuttavia l'accesso per indice resta vincolato al tipo dichiarato:
// Errore: Tuple type '[string, number]' of length '2' has no element at index '2'.
const terzo = coppia[2];
```

Per ottenere una garanzia più forte si può ricorrere a una tupla `readonly`, che non espone `push` né gli altri metodi che modificano la collezione. In questo modo la lunghezza fissa diventa effettivamente inviolabile a livello di tipo.

```ts
const coordinate: readonly [number, number] = [45.46, 9.19];

// Errore: Property 'push' does not exist on type 'readonly [number, number]'.
coordinate.push(0);

// Errore: Cannot assign to '0' because it is a read-only property.
coordinate[0] = 0;
```

Le tuple si rivelano comode anche come tipo di ritorno di funzioni che devono restituire più valori eterogenei in un'unica struttura ordinata, un pattern reso ancora più espressivo dalla destrutturazione abbinata alle etichette.

```ts
function dividi(dividendo: number, divisore: number): [quoziente: number, resto: number] {
  return [Math.floor(dividendo / divisore), dividendo % divisore];
}

const [quoziente, resto] = dividi(17, 5);
console.log(quoziente, resto); // 3 2
```

## Domande

<details>
<summary>Quale differenza esiste tra le sintassi `tipo[]` e `Array<tipo>`?</summary>

Nessuna differenza semantica: le due forme sono completamente equivalenti e producono lo stesso tipo. `string[]` e `Array<string>` descrivono entrambe un array di stringhe. La scelta è soltanto stilistica; la forma con `Array<tipo>` tende a risultare più leggibile quando il tipo degli elementi è articolato, ad esempio un union type, perché le parentesi angolari ne delimitano chiaramente il contenuto.

</details>

<details>
<summary>Qual è la differenza principale, a livello di type system, tra un array e una tupla?</summary>

In un array tutti gli elementi condividono lo stesso tipo e la lunghezza è variabile: l'accesso a qualunque indice è lecito a livello di tipo. In una tupla, invece, il numero degli elementi e il tipo di ciascuna posizione sono noti e fissati nella dichiarazione, quindi ogni indice ha il proprio tipo e l'accesso a un indice fuori dai limiti dichiarati viene segnalato come errore di compilazione.

</details>

<details>
<summary>Perché la dichiarazione `type T = [number, string?, boolean]` non è valida?</summary>

Perché in una tupla gli elementi opzionali sono ammessi soltanto in coda: una volta introdotto un elemento opzionale con `?`, tutti quelli successivi devono essere a loro volta opzionali. Far seguire l'elemento obbligatorio `boolean` all'elemento opzionale `string?` renderebbe ambigua la posizione degli elementi, perciò il compilatore produce l'errore "A required element cannot follow an optional element".

</details>

<details>
<summary>Che cosa accade chiamando `push` su una tupla e perché rappresenta un'insidia?</summary>

A runtime una tupla è un normale array JavaScript, e il metodo `push` ne eredita il comportamento: consente di aggiungere elementi oltre la lunghezza dichiarata senza che il compilatore segnali alcun errore. Il controllo sulla lunghezza viene applicato all'assegnazione tramite letterale e all'accesso per indice, ma non a `push`. La garanzia sulla lunghezza fissa risulta così aggirabile; per renderla effettiva si può usare una tupla `readonly`, che non espone `push`.

```ts
const coppia: [string, number] = ["x", 1];
coppia.push(99); // ammesso dal compilatore, lunghezza dichiarata bypassata
```

</details>

<details>
<summary>Come si rende immodificabile un array o una tupla a livello di tipo?</summary>

Anteponendo il modificatore `readonly` al tipo, ad esempio `readonly string[]` oppure `readonly [number, number]`. Un array o una tupla `readonly` non espongono i metodi che alterano la collezione, come `push`, `pop` o `splice`, e rifiutano l'assegnazione tramite indice. Si tratta di un vincolo a livello di type system: per garantire l'immutabilità anche a runtime occorre invece ricorrere a `Object.freeze`.

</details>
