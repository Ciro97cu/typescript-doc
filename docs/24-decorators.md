# Decorators

Un decorator è una dichiarazione speciale che si applica a una classe oppure a uno dei suoi membri (metodo, accessor o field) tramite la sintassi `@espressione`. La sua funzione è quella di osservare, annotare o modificare l'elemento decorato, intervenendo sul suo comportamento senza disperdere la logica all'interno del corpo della classe. In TypeScript 6.0 i decorator seguono lo standard ECMAScript (Stage 3) e sono abilitati per impostazione predefinita: non è più necessaria alcuna opzione del compilatore per utilizzarli.

Il punto fondamentale da comprendere è il momento in cui un decorator viene eseguito. Un decorator non viene invocato quando si crea un'istanza con `new`, bensì quando la classe viene definita, cioè una sola volta, nel momento in cui il motore valuta la dichiarazione `class`. Questo significa che il codice contenuto direttamente nel decorator viene eseguito a tempo di definizione, mentre la logica che deve operare sulle singole istanze va registrata separatamente, come si vedrà più avanti con `addInitializer`.

Nello standard un decorator è una funzione che riceve sempre due argomenti: il valore decorato e un oggetto di contesto. La forma generale è `(value, context)`. Il tipo del primo argomento e quello del `context` dipendono dall'elemento decorato: per una classe si usa `ClassDecoratorContext`, per un metodo `ClassMethodDecoratorContext`, per un field `ClassFieldDecoratorContext`, per gli accessor `ClassGetterDecoratorContext`, `ClassSetterDecoratorContext` e `ClassAccessorDecoratorContext`. L'oggetto di contesto espone informazioni utili come `kind` (il tipo di elemento), `name` (il nome del membro), `static` e `private` (booleani che indicano la natura del membro), il metodo `addInitializer` e, per gli accessor, l'oggetto `access`.

```ts
// Class decorator: viene eseguito alla definizione della classe Persona,
// non quando si scrive `new Persona(...)`.
function logged(value: Function, context: ClassDecoratorContext) {
  console.log(`Definizione della classe: ${String(context.name)}`);
}

@logged
class Persona {
  constructor(public nome: string) {}
}

// In console compare "Definizione della classe: Persona" subito,
// ancora prima di qualunque istanziazione.
const p = new Persona("Ada");
```

## Decorator Factory

Spesso è necessario configurare il comportamento di un decorator con dei parametri. Poiché la sintassi `@espressione` valuta l'espressione e ne usa il risultato come decorator, è sufficiente che quell'espressione sia una chiamata di funzione che restituisce a sua volta il decorator vero e proprio. Una funzione di questo genere prende il nome di decorator factory: riceve gli argomenti di configurazione e ritorna la funzione `(value, context)` che verrà effettivamente applicata.

```ts
// La factory riceve un prefisso e restituisce il decorator.
function logger(prefisso: string) {
  return function (value: Function, context: ClassDecoratorContext) {
    console.log(`[${prefisso}] registrata la classe ${String(context.name)}`);
  };
}

@logger("MODELLO")
class Prodotto {
  constructor(public titolo: string) {}
}

// In console: "[MODELLO] registrata la classe Prodotto"
```

L'uso di `@logger("MODELLO")` chiarisce la distinzione fra i due livelli: la factory `logger` viene invocata immediatamente con l'argomento `"MODELLO"`, e il suo valore di ritorno è il decorator che TypeScript applica alla classe.

## Multiple decorators and execution order

A un singolo elemento è possibile applicare più decorator, impilandoli uno sopra l'altro. In presenza di più factory l'ordine di esecuzione si articola in due fasi distinte. Prima vengono valutate tutte le decorator factory dall'alto verso il basso (top-down), nell'ordine in cui appaiono nel codice, perché si tratta di normali chiamate di funzione che producono i decorator. Successivamente i decorator risultanti vengono applicati dal basso verso l'alto (bottom-up): il decorator più vicino all'elemento viene applicato per primo, e il suo risultato viene a sua volta passato a quello immediatamente superiore.

```ts
function primo() {
  console.log("primo(): factory valutata");
  return function (value: Function, context: ClassDecoratorContext) {
    console.log("primo(): decorator applicato");
  };
}

function secondo() {
  console.log("secondo(): factory valutata");
  return function (value: Function, context: ClassDecoratorContext) {
    console.log("secondo(): decorator applicato");
  };
}

@primo()
@secondo()
class Esempio {}

// Output prodotto, nell'ordine:
// primo(): factory valutata
// secondo(): factory valutata
// secondo(): decorator applicato
// primo(): decorator applicato
```

L'esempio mostra chiaramente le due direzioni: le factory `primo()` e `secondo()` vengono eseguite nell'ordine di lettura, mentre l'applicazione effettiva avviene in ordine inverso, partendo da quello più interno.

## Property, Accessor, Method Decorators

Oltre alle classi, lo standard prevede decorator per i metodi, per i field (le proprietà di istanza dichiarate nel corpo della classe), per i getter e i setter e per gli accessor `accessor`. Ciascuno riceve la coppia `(value, context)`, ma il significato di `value` cambia in base all'elemento.

Un method decorator riceve come `value` la funzione del metodo e un contesto di tipo `ClassMethodDecoratorContext`. Può restituire una nuova funzione che sostituisce il metodo originale, tecnica con cui si realizzano facilmente comportamenti come il logging delle chiamate.

```ts
// Method decorator: avvolge il metodo registrando argomenti e risultato.
function tracciaChiamate(
  value: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  return function (this: unknown, ...args: any[]) {
    console.log(`Chiamata a ${String(context.name)} con`, args);
    const risultato = value.apply(this, args);
    console.log(`${String(context.name)} ha restituito`, risultato);
    return risultato;
  };
}

class Calcolatrice {
  @tracciaChiamate
  somma(a: number, b: number): number {
    return a + b;
  }
}

new Calcolatrice().somma(2, 3);
// Chiamata a somma con [ 2, 3 ]
// somma ha restituito 5
```

Un field decorator riceve `undefined` come `value`, dal momento che al momento della decorazione il valore della proprietà non esiste ancora, e un contesto `ClassFieldDecoratorContext`. Può restituire una funzione di inizializzazione che prende il valore iniziale e ne restituisce uno trasformato; tale funzione viene eseguita per ogni istanza, al momento dell'inizializzazione del field.

```ts
// Field decorator: trasforma il valore iniziale assicurandone la non-negatività.
function nonNegativo(
  value: undefined,
  context: ClassFieldDecoratorContext<unknown, number>
) {
  return function (valoreIniziale: number): number {
    return valoreIniziale < 0 ? 0 : valoreIniziale;
  };
}

class Conto {
  @nonNegativo
  saldo: number = -50;
}

console.log(new Conto().saldo); // 0
```

Per i getter e i setter si usano rispettivamente `ClassGetterDecoratorContext` e `ClassSetterDecoratorContext`, e il decorator può restituire una nuova funzione di accesso. Esiste inoltre la parola chiave `accessor`, che dichiara un campo dotato automaticamente di un get e di un set: il relativo decorator riceve un oggetto con i metodi `get` e `set` e un contesto `ClassAccessorDecoratorContext`, e può restituire una versione modificata di tale coppia.

```ts
// Accessor decorator: intercetta lettura e scrittura di un campo `accessor`.
function osservato<This, T>(
  value: ClassAccessorDecoratorTarget<This, T>,
  context: ClassAccessorDecoratorContext<This, T>
): ClassAccessorDecoratorResult<This, T> {
  return {
    get(this: This) {
      const v = value.get.call(this);
      console.log(`Lettura di ${String(context.name)}:`, v);
      return v;
    },
    set(this: This, nuovo: T) {
      console.log(`Scrittura di ${String(context.name)}:`, nuovo);
      value.set.call(this, nuovo);
    },
  };
}

class Termostato {
  @osservato
  accessor temperatura: number = 20;
}

const t = new Termostato();
t.temperatura = 22; // Scrittura di temperatura: 22
t.temperatura;      // Lettura di temperatura: 22
```

Vale la pena ribadire che lo standard Stage 3 non prevede decorator per i parametri: la decorazione si applica esclusivamente alla classe, ai metodi, ai field e agli accessor.

## Returning (and changing) a Class in a Class Decorator

Un class decorator può non limitarsi a osservare la classe, ma può restituire un nuovo valore che la sostituisce. Quando il valore restituito è una classe, esso prende il posto di quella originale per tutto il resto del programma. Questa possibilità consente di estendere o avvolgere la classe decorata, ad esempio aggiungendo membri o eseguendo logica al momento della costruzione di ogni istanza. Per tipizzare correttamente il costruttore in ingresso si vincola il generic alla forma di una classe costruibile, ossia `new (...args: any[]) => object`.

```ts
// Class decorator che restituisce una sottoclasse: aggiunge un timestamp
// di creazione a ogni istanza e registra un messaggio in fase di costruzione.
function conTimestamp<T extends new (...args: any[]) => object>(
  value: T,
  context: ClassDecoratorContext
) {
  return class extends value {
    creatoIl = new Date();
    constructor(...args: any[]) {
      super(...args);
      console.log(`Istanza di ${String(context.name)} creata`);
    }
  };
}

@conTimestamp
class Documento {
  constructor(public titolo: string) {}
}

const d = new Documento("Relazione");
console.log(d.titolo);     // "Relazione"
console.log(d.creatoIl);   // data e ora della creazione
```

La classe restituita estende quella ricevuta come `value`, ne preserva quindi proprietà e metodi originali, e vi aggiunge il nuovo campo `creatoIl`. Poiché la sostituzione avviene a livello della definizione, ogni successivo `new Documento(...)` produce in realtà un'istanza della sottoclasse generata dal decorator.

## Autobinding Decorator

Un problema ricorrente con i metodi di classe riguarda il valore di `this`. Quando un metodo viene passato come callback, ad esempio ad `addEventListener` o a `setTimeout`, perde il legame con l'istanza e `this` diventa `undefined` in strict mode. La soluzione consiste nel legare stabilmente il metodo all'istanza, e lo standard Stage 3 permette di farlo in modo pulito tramite `context.addInitializer`. Questo metodo, disponibile sull'oggetto di contesto, registra una funzione che verrà eseguita su ciascuna istanza durante la sua inizializzazione, con `this` correttamente impostato sull'istanza stessa. All'interno dell'initializer si può quindi sostituire il metodo con la sua versione legata.

```ts
// Method decorator di autobinding: lega `this` all'istanza in modo
// che il metodo conservi il contesto anche se passato come callback.
function autobind(
  value: (...args: any[]) => any,
  context: ClassMethodDecoratorContext
) {
  context.addInitializer(function (this: any) {
    this[context.name] = value.bind(this);
  });
}

class Pulsante {
  messaggio = "Cliccato";

  @autobind
  gestisciClick() {
    console.log(this.messaggio);
  }
}

const pulsante = new Pulsante();
const fn = pulsante.gestisciClick;
fn(); // "Cliccato": `this` resta legato all'istanza
```

L'initializer viene eseguito una volta per ogni istanza: nel momento in cui `Pulsante` viene costruito, il metodo `gestisciClick` viene rimpiazzato sull'istanza da una sua copia con `this` permanentemente legato. Di conseguenza, anche estraendo il metodo in una variabile e invocandolo separatamente, il contesto rimane quello corretto. Un'alternativa equivalente, che non richiede alcun decorator, consiste nel dichiarare il metodo come un arrow function assegnato a un field, ma l'approccio con `addInitializer` ha il vantaggio di essere riutilizzabile e dichiarativo.

## Domande

<details>
<summary>In quale momento viene eseguita la funzione di un class decorator e perché è rilevante?</summary>

La funzione di un class decorator viene eseguita una sola volta, quando la classe viene definita, e non a ogni istanziazione con `new`. È rilevante perché qualsiasi logica scritta direttamente nel corpo del decorator opera a tempo di definizione: per agire sulle singole istanze occorre invece registrare una funzione tramite `context.addInitializer`, che viene eseguita durante l'inizializzazione di ciascuna istanza.

</details>

<details>
<summary>Qual è la differenza fra una decorator factory e il decorator vero e proprio?</summary>

Una decorator factory è una funzione che riceve argomenti di configurazione e restituisce il decorator. Quando si scrive `@logger("MODELLO")`, la factory `logger` viene chiamata subito con `"MODELLO"` e il suo valore di ritorno, cioè la funzione `(value, context)`, è il decorator effettivamente applicato all'elemento. La factory serve quindi a parametrizzare il comportamento del decorator.

</details>

<details>
<summary>Date più factory impilate sullo stesso elemento, in quale ordine vengono valutate le factory e in quale vengono applicati i decorator?</summary>

Le decorator factory vengono valutate dall'alto verso il basso (top-down), nell'ordine di lettura, perché sono normali chiamate di funzione. I decorator risultanti vengono invece applicati dal basso verso l'alto (bottom-up): viene applicato per primo quello più vicino all'elemento e il suo risultato viene passato a quello superiore.

</details>

<details>
<summary>Cosa riceve come primo argomento un field decorator e cosa può restituire?</summary>

Un field decorator riceve `undefined` come primo argomento, dato che al momento della decorazione il valore della proprietà non esiste ancora, e un contesto di tipo `ClassFieldDecoratorContext`. Può restituire una funzione di inizializzazione con firma `(initialValue) => newValue`, eseguita per ogni istanza, che riceve il valore iniziale del field e ne restituisce una versione eventualmente trasformata.

</details>

<details>
<summary>Come si realizza l'autobinding di un metodo nello standard Stage 3 e perché serve?</summary>

Serve perché un metodo passato come callback perde il legame con l'istanza, e `this` diventa `undefined` in strict mode. Lo si realizza con un method decorator che chiama `context.addInitializer`, registrando una funzione eseguita su ogni istanza che sostituisce il metodo con la sua versione legata, ad esempio `this[context.name] = value.bind(this)`. In questo modo il contesto resta corretto anche quando il metodo viene estratto e invocato separatamente.

</details>
