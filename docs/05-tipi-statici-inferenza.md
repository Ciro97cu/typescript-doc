# Tipizzazione statica e inferenza

Uno degli aspetti che distingue maggiormente TypeScript da JavaScript riguarda il modo in cui i tipi vengono trattati. Comprendere la differenza tra tipizzazione dinamica e tipizzazione statica, e capire come TypeScript assegni e deduca i tipi, è il punto di partenza per sfruttare appieno il type system.

## Tipi statici e tipi dinamici

JavaScript è un linguaggio a tipizzazione dinamica: il tipo di una variabile non è fissato al momento della scrittura del codice, ma viene risolto a runtime e può cambiare nel corso dell'esecuzione. La stessa variabile può contenere prima una stringa e poco dopo un numero, senza che nulla lo impedisca. Questa flessibilità ha un costo: molti errori di tipo emergono soltanto durante l'esecuzione del programma, spesso in produzione, e risultano difficili da individuare.

```ts
// JavaScript: la stessa variabile cambia tipo a runtime
let valore = "ciao";   // qui contiene una stringa
valore = 42;            // poco dopo contiene un numero, nessun errore
valore = true;          // e ora un booleano

// L'errore si manifesta solo quando il codice viene eseguito
console.log(valore.toUpperCase()); // TypeError: valore.toUpperCase is not a function
```

TypeScript adotta invece la tipizzazione statica: il tipo viene determinato durante lo sviluppo e gli errori di tipo vengono segnalati dal compilatore prima ancora che il programma venga eseguito. Una variabile a cui è associato il tipo `string` non potrà mai ricevere un numero, e qualsiasi tentativo in tal senso viene intercettato in fase di compilazione.

```ts
let valore = "ciao"; // il tipo viene fissato a string
valore = 42;          // Errore: Type 'number' is not assignable to type 'string'.
```

È importante sottolineare che l'analisi statica non aggiunge alcun controllo a runtime: il JavaScript generato non contiene verifiche di tipo, perché tutto il lavoro avviene in fase di compilazione. I tipi servono allo sviluppatore e al compilatore, ma scompaiono nel codice eseguibile.

## Type assignment e type inference

Esistono due modi per associare un tipo a una variabile in TypeScript.

Il primo è il type assignment esplicito, ovvero l'indicazione diretta del tipo tramite una type annotation, scritta dopo il nome della variabile e preceduta dai due punti. In questo caso è lo sviluppatore a dichiarare in modo inequivocabile quale tipo la variabile dovrà rispettare.

```ts
let nome: string = "Anna";
let eta: number = 30;
let attivo: boolean = true;

eta = "trenta"; // Errore: Type 'string' is not assignable to type 'number'.
```

Il secondo modo è la type inference: quando una variabile viene dichiarata e inizializzata contestualmente, TypeScript deduce automaticamente il tipo dal valore assegnato, rendendo superflua l'annotazione esplicita. Il risultato è identico, ma il codice resta più conciso.

```ts
let citta = "Roma";   // il tipo viene dedotto: string
let punti = 10;        // il tipo viene dedotto: number
let confermato = false; // il tipo viene dedotto: boolean

citta = 99; // Errore: Type 'number' is not assignable to type 'string'.
```

La regola pratica generalmente adottata consiste nel lasciare lavorare la type inference quando il valore iniziale rende il tipo evidente, riservando l'annotazione esplicita ai casi in cui la deduzione non sarebbe possibile o produrrebbe un tipo troppo ampio rispetto alle intenzioni. Annotare esplicitamente un tipo che il compilatore avrebbe comunque dedotto correttamente è ridondante.

```ts
// Annotazione ridondante: il tipo string sarebbe stato dedotto comunque
let cognome: string = "Rossi";

// Forma preferita: si lascia agire la type inference
let cognome2 = "Rossi";
```

Un caso in cui l'annotazione esplicita è invece necessaria si presenta quando una variabile viene dichiarata senza inizializzazione: in assenza di un valore da cui dedurre il tipo, l'annotazione diventa l'unico modo per fornire l'informazione al compilatore.

```ts
let descrizione: string; // senza annotazione il tipo sarebbe any implicito
descrizione = "in lavorazione"; // assegnazione valida
```

## Parametri di funzione e any implicito

I parametri di una funzione rappresentano il caso più frequente in cui la type inference non dispone di informazioni sufficienti: al momento della dichiarazione non esiste alcun valore da cui dedurre il tipo, poiché il valore arriverà soltanto al momento della chiamata. Un parametro privo di annotazione assumerebbe quindi il tipo `any` in modo implicito, perdendo ogni garanzia offerta dal type system.

Con `strict` attivo per impostazione predefinita in TypeScript 6.0, l'opzione `noImplicitAny` è anch'essa abilitata di default: di conseguenza, un parametro senza tipo esplicito non viene tollerato silenziosamente, ma genera un errore di compilazione.

```ts
// Errore: Parameter 'a' implicitly has an 'any' type.
// Errore: Parameter 'b' implicitly has an 'any' type.
function somma(a, b) {
  return a + b;
}
```

La correzione consiste nell'annotare esplicitamente i parametri. Il tipo di ritorno, al contrario, viene generalmente dedotto in modo affidabile dalla type inference a partire dall'espressione restituita, per cui la sua annotazione resta facoltativa.

```ts
// I parametri sono tipizzati esplicitamente; il tipo di ritorno (number) viene dedotto
function somma(a: number, b: number) {
  return a + b;
}

somma(3, 4);       // 7
somma(3, "4");     // Errore: Argument of type 'string' is not assignable to parameter of type 'number'.
```

Lo stesso principio vale per le arrow function e per i parametri di una callback, dove l'annotazione esplicita dei parametri è altrettanto necessaria in assenza di un contesto che ne consenta la deduzione.

```ts
const raddoppia = (n: number): number => n * 2;

const nomi = ["anna", "luca", "marco"];
// Qui il tipo di 'nome' viene dedotto da quello dell'array (string),
// quindi l'annotazione esplicita non è richiesta
const maiuscoli = nomi.map((nome) => nome.toUpperCase());
```

Vale la pena osservare che, quando una funzione viene passata in un contesto che ne descrive già la firma attesa, entra in gioco la cosiddetta contextual typing: in quei casi TypeScript è in grado di dedurre il tipo dei parametri dal contesto e l'errore di any implicito non si presenta. Resta invece sempre necessaria l'annotazione quando una funzione viene dichiarata in modo indipendente, senza un contesto che ne suggerisca la forma.

## Domande

<details>
<summary>Qual è la differenza fondamentale tra la tipizzazione dinamica di JavaScript e quella statica di TypeScript?</summary>

Nella tipizzazione dinamica di JavaScript il tipo di una variabile è risolto a runtime e può cambiare durante l'esecuzione, per cui gli errori di tipo emergono soltanto quando il codice viene eseguito. Nella tipizzazione statica di TypeScript il tipo è determinato in fase di sviluppo e gli errori vengono segnalati dal compilatore prima dell'esecuzione. I controlli di tipo non lasciano traccia nel JavaScript generato, poiché avvengono interamente in fase di compilazione.

</details>

<details>
<summary>Che differenza c'è tra type assignment e type inference?</summary>

Con il type assignment il tipo viene indicato esplicitamente tramite una type annotation (ad esempio `let nome: string = "Anna"`). Con la type inference il tipo non viene scritto, ma dedotto automaticamente dal compilatore a partire dal valore assegnato al momento della dichiarazione (ad esempio `let nome = "Anna"`, dedotto come `string`). Il risultato sul piano dei tipi è identico; cambia solo se l'informazione è fornita dallo sviluppatore o dedotta dal compilatore.

</details>

<details>
<summary>Perché un parametro di funzione senza tipo provoca un errore in TypeScript 6.0?</summary>

Un parametro privo di annotazione assumerebbe il tipo `any` in modo implicito, perché al momento della dichiarazione non esiste alcun valore da cui dedurre il tipo. In TypeScript 6.0 l'opzione `strict` è attiva per impostazione predefinita e include `noImplicitAny`, che trasforma l'any implicito in un errore di compilazione: viene segnalato un messaggio del tipo `Parameter 'a' implicitly has an 'any' type.`. La soluzione è annotare esplicitamente il parametro, ad esempio `function somma(a: number, b: number)`.

</details>

<details>
<summary>È necessario annotare sempre anche il tipo di ritorno di una funzione?</summary>

No. Il tipo di ritorno viene generalmente dedotto in modo affidabile dalla type inference a partire dall'espressione restituita, quindi la sua annotazione è facoltativa. Annotarlo esplicitamente può comunque risultare utile come forma di documentazione o per imporre un contratto preciso, ma non è imposto dal compilatore. I parametri, invece, richiedono quasi sempre un'annotazione esplicita perché non dispongono di un valore da cui dedurre il tipo.

</details>

<details>
<summary>Esiste un caso in cui i parametri di una funzione non richiedono un'annotazione esplicita?</summary>

Sì, quando entra in gioco la contextual typing. Se una funzione viene passata in un contesto che ne descrive già la firma attesa (ad esempio il callback di `Array.prototype.map`, dove il tipo dell'elemento è noto dal tipo dell'array), TypeScript deduce il tipo dei parametri dal contesto e non viene sollevato alcun errore di any implicito. L'annotazione resta invece necessaria quando la funzione è dichiarata in modo indipendente, senza un contesto che ne suggerisca la forma.

</details>
