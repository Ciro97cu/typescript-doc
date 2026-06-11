# Funzioni

Una funzione è un blocco di codice riutilizzabile che incapsula una determinata logica. In TypeScript ogni funzione può essere descritta in modo preciso indicando il tipo dei suoi parametri e il tipo del valore restituito: in questo modo il compilatore è in grado di verificare che la funzione venga sempre invocata con argomenti corretti e che il suo risultato venga usato in modo coerente. Si tratta di uno degli ambiti in cui i tipi statici offrono il maggior beneficio, poiché un errore nella firma di una funzione tende a propagarsi rapidamente in tutto il codice che la utilizza.

## Tipizzazione di parametri e valore di ritorno

Ogni parametro di una funzione può essere annotato con un tipo, scrivendolo dopo il nome del parametro e separato da due punti. Il tipo del valore di ritorno si indica invece dopo la parentesi chiusa della lista dei parametri. Annotare esplicitamente i parametri è particolarmente importante: in assenza di annotazione e di un contesto da cui dedurla, il compilatore non è in grado di inferire un tipo significativo e segnalerebbe un errore, dal momento che con le impostazioni predefinite di TypeScript 6.0 il controllo `noImplicitAny` è attivo.

```ts
function somma(a: number, b: number): number {
  return a + b;
}

const risultato = somma(3, 4); // risultato ha tipo number

somma(3, "4");
// Errore: Argument of type 'string' is not assignable to parameter of type 'number'.
```

Il tipo del valore di ritorno, al contrario dei parametri, può quasi sempre essere omesso: TypeScript lo deduce automaticamente analizzando le istruzioni `return` presenti nel corpo. Annotarlo resta comunque una buona pratica per le funzioni dell'interfaccia pubblica di un modulo, perché esplicita le intenzioni e impedisce che una modifica accidentale all'implementazione cambi silenziosamente il tipo restituito.

```ts
// Tipo di ritorno inferito: number
function quadrato(n: number) {
  return n * n;
}

// Tipo di ritorno annotato esplicitamente
function descrivi(nome: string, eta: number): string {
  return `${nome} ha ${eta} anni`;
}
```

### Il tipo void

Quando una funzione non restituisce alcun valore utile si utilizza il tipo `void` come tipo di ritorno. A runtime una funzione di questo genere restituisce comunque `undefined`, ma `void` comunica in modo esplicito che il valore prodotto non deve essere utilizzato. È il tipo tipico delle funzioni che eseguono effetti collaterali, come stampare a console o aggiornare uno stato.

```ts
function registra(messaggio: string): void {
  console.log(`[LOG] ${messaggio}`);
}
```

Va osservata una particolarità del tipo `void` come tipo di ritorno atteso: quando una funzione viene assegnata a un tipo che prevede ritorno `void`, è comunque ammesso che la sua implementazione restituisca un valore, il quale viene semplicemente ignorato. Questo comportamento, all'apparenza sorprendente, è ciò che consente di passare funzioni come `Array.prototype.push` a metodi come `forEach`, che si aspettano una callback con ritorno `void`.

```ts
type Callback = (valore: number) => void;

const numeri: number[] = [];

// La callback restituisce un number, ma il valore viene ignorato perché il tipo atteso è void
const callback: Callback = (valore) => numeri.push(valore);
```

## Parametri opzionali, valori di default e rest parameters

Un parametro può essere reso opzionale aggiungendo un punto interrogativo `?` dopo il suo nome. All'interno del corpo della funzione il tipo di un parametro opzionale include sempre `undefined`, motivo per cui il compilatore richiede di gestire l'eventuale assenza del valore prima di poterlo usare. I parametri opzionali devono comparire dopo quelli obbligatori.

```ts
function saluta(nome: string, titolo?: string): string {
  // titolo ha tipo string | undefined
  if (titolo === undefined) {
    return `Ciao, ${nome}`;
  }
  return `Ciao, ${titolo} ${nome}`;
}

saluta("Rossi");              // valido
saluta("Rossi", "Dott.");     // valido
```

In alternativa è possibile assegnare a un parametro un valore di default, indicandolo con `=` dopo il nome. Quando l'argomento viene omesso, o vale esplicitamente `undefined`, viene utilizzato il valore predefinito. In questo caso il tipo del parametro viene di norma inferito dal valore di default e all'interno della funzione il parametro non include `undefined`, poiché un valore è sempre garantito.

```ts
function incrementa(valore: number, passo: number = 1): number {
  // passo ha tipo number, mai undefined
  return valore + passo;
}

incrementa(10);     // 11
incrementa(10, 5);  // 15
```

Per accettare un numero variabile di argomenti si utilizza un rest parameter, introdotto dalla sintassi `...`. Esso raccoglie in un array tutti gli argomenti rimanenti e, di conseguenza, il suo tipo deve sempre essere un tipo di array. Un rest parameter può comparire una sola volta e deve trovarsi in ultima posizione nella lista dei parametri.

```ts
function sommaTutti(...numeri: number[]): number {
  return numeri.reduce((acc, n) => acc + n, 0);
}

sommaTutti();           // 0
sommaTutti(1, 2, 3, 4); // 10
```

## Function type

Una funzione è essa stessa un valore e, come tale, possiede un tipo. Il function type descrive la firma di una funzione, ovvero i tipi dei suoi parametri e il tipo del valore restituito, senza riferirsi a una implementazione specifica. La sintassi impiega una freccia: `(parametri) => tipoDiRitorno`. Questo tipo risulta indispensabile per tipizzare variabili che contengono funzioni e, soprattutto, per descrivere i parametri di tipo callback.

```ts
// Variabile il cui tipo è un function type
let operazione: (a: number, b: number) => number;

operazione = (a, b) => a + b;   // i tipi di a e b sono inferiti dal function type
operazione = (a, b) => a * b;

operazione = (a, b) => `${a}${b}`;
// Errore: Type 'string' is not assignable to type 'number'.
```

Quando una funzione viene assegnata a una variabile dotata di function type, i tipi dei parametri non vanno ripetuti: il compilatore li deduce dal contesto, secondo un meccanismo noto come contextual typing. Il function type si rivela particolarmente utile per descrivere le callback, rendendo esplicito ciò che ci si aspetta di ricevere e di restituire.

```ts
function applicaDue(
  valore: number,
  trasformazione: (n: number) => number
): number {
  return trasformazione(trasformazione(valore));
}

applicaDue(3, (n) => n + 1);  // 5
applicaDue(4, (n) => n * n);  // 256
```

Per nomi di firma ricorrenti conviene assegnare al function type un type alias, in modo da riutilizzarlo e migliorare la leggibilità.

```ts
type Comparatore<T> = (primo: T, secondo: T) => number;

const perLunghezza: Comparatore<string> = (a, b) => a.length - b.length;

const parole = ["casa", "a", "albero"];
parole.sort(perLunghezza); // ["a", "casa", "albero"]
```

## Function overloads

Gli function overloads consentono di assegnare a una stessa funzione più firme distinte, ciascuna con combinazioni di tipi di input e output potenzialmente diverse. Questo meccanismo è utile quando il tipo del valore restituito dipende dai tipi degli argomenti ricevuti, una relazione che una singola firma con union type non sarebbe in grado di esprimere con la medesima precisione.

La struttura prevede l'elenco delle firme di overload, prive di corpo, seguite da un'unica implementazione. La firma dell'implementazione deve essere compatibile con tutte le firme di overload, ma non è visibile dall'esterno: chi invoca la funzione può fare riferimento esclusivamente alle firme di overload dichiarate.

```ts
// Firme di overload
function combina(a: string, b: string): string;
function combina(a: number, b: number): number;
// Firma di implementazione: compatibile con tutte le precedenti, non richiamabile direttamente
function combina(a: string | number, b: string | number): string | number {
  if (typeof a === "string" && typeof b === "string") {
    return a + b;
  }
  if (typeof a === "number" && typeof b === "number") {
    return a + b;
  }
  throw new Error("I due argomenti devono essere dello stesso tipo");
}

const testo = combina("Type", "Script"); // tipo string
const numero = combina(2, 3);             // tipo number

combina("Type", 3);
// Errore: No overload matches this call.
```

Grazie agli overload il compilatore seleziona la firma più appropriata in base agli argomenti effettivamente passati e inferisce il tipo di ritorno corretto. Con una singola firma basata su union type, invece, il risultato avrebbe sempre tipo `string | number`, costringendo chi chiama la funzione a un controllo aggiuntivo anche quando non necessario.

Gli overload risultano particolarmente espressivi quando il numero o la natura dei parametri varia tra le diverse modalità d'uso, come nel caso di una funzione che può ricevere un identificatore singolo oppure un elenco.

```ts
interface Utente {
  id: number;
  nome: string;
}

const archivio: Utente[] = [
  { id: 1, nome: "Bianchi" },
  { id: 2, nome: "Verdi" },
];

function trova(id: number): Utente | undefined;
function trova(ids: number[]): Utente[];
function trova(arg: number | number[]): Utente | Utente[] | undefined {
  if (Array.isArray(arg)) {
    return archivio.filter((u) => arg.includes(u.id));
  }
  return archivio.find((u) => u.id === arg);
}

const singolo = trova(1);      // tipo Utente | undefined
const elenco = trova([1, 2]);  // tipo Utente[]
```

Va tenuto presente che, quando le firme alternative differiscono unicamente per il tipo di alcuni argomenti senza che ne dipenda il tipo di ritorno, soluzioni come i parametri opzionali, i valori di default o gli union type sono spesso più semplici e altrettanto efficaci. Gli overload andrebbero quindi riservati ai casi in cui esprimono una relazione che le altre tecniche non sanno catturare.

## Domande

<details>
<summary>Perché con le impostazioni predefinite di TypeScript 6.0 una funzione con un parametro privo di annotazione genera un errore?</summary>

Perché in TypeScript 6.0 l'opzione `strict` è attiva per impostazione predefinita e include `noImplicitAny`. In assenza di un'annotazione esplicita o di un contesto da cui dedurre il tipo, il compilatore non può inferire un tipo significativo per il parametro e, anziché assegnargli implicitamente `any`, segnala un errore come `Parameter 'x' implicitly has an 'any' type`.
</details>

<details>
<summary>Qual è la differenza tra un parametro opzionale dichiarato con ? e un parametro con valore di default?</summary>

Un parametro opzionale (`titolo?: string`) ha tipo `string | undefined` all'interno della funzione e deve quindi essere gestito tenendo conto della possibile assenza del valore. Un parametro con valore di default (`passo: number = 1`) garantisce invece sempre un valore: quando l'argomento è omesso o vale `undefined` si usa il default, perciò il suo tipo all'interno della funzione è `number`, senza includere `undefined`.
</details>

<details>
<summary>Come mai è possibile assegnare a un function type con ritorno void una funzione che in realtà restituisce un valore?</summary>

Si tratta di un comportamento intenzionale del type system: quando il tipo di ritorno atteso è `void`, una funzione che restituisce un valore è comunque assegnabile, e tale valore viene semplicemente ignorato. Questo consente, ad esempio, di passare metodi come `Array.prototype.push` a `forEach`, la cui callback è tipizzata con ritorno `void`, senza che il valore prodotto debba essere scartato manualmente.

```ts
const numeri: number[] = [];
[1, 2, 3].forEach((n) => numeri.push(n)); // valido
```
</details>

<details>
<summary>Perché la firma di implementazione di una funzione con overload non può essere richiamata direttamente?</summary>

Perché la firma di implementazione serve unicamente al compilatore per verificare che il corpo della funzione sia compatibile con tutte le firme di overload, ma non fa parte dell'interfaccia pubblica. Dall'esterno sono visibili solo le firme di overload dichiarate, quindi una chiamata che corrisponde alla sola firma di implementazione (come `combina("Type", 3)` con firme `(string, string)` e `(number, number)`) produce l'errore `No overload matches this call`.
</details>

<details>
<summary>Quale vantaggio offrono gli function overloads rispetto a una singola firma basata su union type?</summary>

Gli overload permettono di esprimere la dipendenza tra il tipo degli argomenti e il tipo del valore restituito. Con due firme `(a: string, b: string): string` e `(a: number, b: number): number`, chiamando la funzione con due stringhe il compilatore inferisce il ritorno come `string`, e con due numeri come `number`. Una singola firma con union, al contrario, restituirebbe sempre `string | number`, obbligando chi invoca la funzione a un controllo o a una type assertion per restringere il tipo del risultato.
</details>
