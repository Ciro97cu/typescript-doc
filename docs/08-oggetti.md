# Oggetti

In TypeScript un object literal permette di descrivere in modo preciso la forma di un valore non primitivo: per ciascuna proprietà viene indicato un tipo, e il compilatore verifica che il valore assegnato rispetti quella struttura. La verifica avviene già in fase di sviluppo, prima dell'esecuzione, e copre sia la presenza delle proprietà attese sia la correttezza dei loro tipi.

## Tipi oggetto con proprietà tipizzate

Il modo più diretto per tipizzare un oggetto consiste nell'annotare la variabile con un object type, ovvero un blocco fra parentesi graffe in cui ogni proprietà è associata al proprio tipo. Una volta dichiarata questa forma, ogni assegnazione viene confrontata con essa: una proprietà mancante, una proprietà con il tipo sbagliato o una proprietà non prevista producono un errore di compilazione.

```ts
const persona: { nome: string; eta: number; attivo: boolean } = {
  nome: "Ada",
  eta: 36,
  attivo: true,
};

// Accesso type-safe: il compilatore conosce il tipo di ogni proprietà
const messaggio: string = persona.nome.toUpperCase();
```

Quando l'oggetto viene inizializzato direttamente con un valore letterale, spesso l'annotazione esplicita non è necessaria: la type inference deduce automaticamente la forma a partire dai valori assegnati. Il tipo dedotto è comunque un object type completo, con le stesse garanzie di verifica.

```ts
// Tipo inferito: { nome: string; eta: number }
const utente = {
  nome: "Bruno",
  eta: 29,
};

utente.eta = 30; // Corretto: number assegnato a number

// Errore: Type 'string' is not assignable to type 'number'.
utente.eta = "trenta";
```

L'annotazione esplicita resta utile quando si vuole fissare un contratto indipendente dal valore iniziale, ad esempio per i parametri di una funzione che riceve un oggetto strutturato. In questo caso il tipo descrive esattamente cosa la funzione si aspetta di ricevere.

```ts
function descrivi(prodotto: { titolo: string; prezzo: number }): string {
  return `${prodotto.titolo} costa ${prodotto.prezzo} euro`;
}

descrivi({ titolo: "Tastiera", prezzo: 49.9 });

// Errore: Object literal may only specify known properties,
// and 'sconto' does not exist in type '{ titolo: string; prezzo: number; }'.
descrivi({ titolo: "Mouse", prezzo: 19.9, sconto: 5 });
```

Quest'ultimo errore deriva dall'excess property checking: quando un object literal viene passato direttamente dove è atteso un tipo specifico, TypeScript segnala le proprietà in eccesso, perché quasi sempre indicano un refuso o un'aspettativa errata.

### Proprietà opzionali, readonly e annidate

Una proprietà può essere dichiarata opzionale con il simbolo `?`: in tal caso può essere assente, e il suo tipo include implicitamente `undefined`. Il modificatore `readonly`, invece, impedisce la riassegnazione della proprietà dopo l'inizializzazione. Gli object type possono inoltre annidarsi, descrivendo strutture complesse a più livelli.

```ts
type Configurazione = {
  readonly id: string;
  etichetta: string;
  descrizione?: string; // Proprietà opzionale: string | undefined
  limiti: {
    minimo: number;
    massimo: number;
  };
};

const config: Configurazione = {
  id: "cfg-01",
  etichetta: "Predefinita",
  limiti: { minimo: 0, massimo: 100 },
};

config.etichetta = "Personalizzata"; // Corretto

// Errore: Cannot assign to 'id' because it is a read-only property.
config.id = "cfg-02";

// La proprietà opzionale richiede un controllo prima dell'uso
const testo = config.descrizione ?? "Nessuna descrizione";
```

## Object type specifico contro il tipo generico `object`

Accanto agli object type specifici esiste il tipo generico `object`, che rappresenta qualsiasi valore non primitivo: oggetti, array, funzioni e istanze di classi. Il confine è quindi con i tipi primitivi (`string`, `number`, `boolean`, `bigint`, `symbol`, `null`, `undefined`), che non sono assegnabili a `object`.

```ts
let qualsiasiOggetto: object;

qualsiasiOggetto = { nome: "Carla" }; // Corretto
qualsiasiOggetto = [1, 2, 3];         // Corretto: anche un array è un object
qualsiasiOggetto = () => {};          // Corretto: anche una funzione

// Errore: Type 'number' is not assignable to type 'object'.
qualsiasiOggetto = 42;
```

La differenza sostanziale riguarda l'accesso ai membri. Un object type specifico descrive proprietà ben definite, e il compilatore le conosce: l'accesso è verificato e l'autocompletamento è disponibile. Il tipo `object`, al contrario, garantisce soltanto che il valore non sia un primitivo, ma non espone alcuna proprietà specifica; di conseguenza ogni tentativo di leggere una proprietà concreta viene respinto.

```ts
function stampaNome(valore: object): void {
  // Errore: Property 'nome' does not exist on type 'object'.
  console.log(valore.nome);
}

function stampaNomeTipizzato(valore: { nome: string }): void {
  // Corretto: la proprietà 'nome' è parte del tipo
  console.log(valore.nome.toUpperCase());
}
```

Per questo motivo `object` è una scelta appropriata solo quando interessa unicamente distinguere un valore non primitivo da uno primitivo, mentre la verifica delle singole proprietà viene rimandata o non serve. Nella maggior parte dei casi è preferibile dichiarare un object type specifico, oppure ricorrere a un `type` alias o a un'interface per dare un nome riutilizzabile alla forma e ottenere così il controllo completo del compilatore.

```ts
type Punto = { x: number; y: number };

function distanzaOrigine(p: Punto): number {
  // Le proprietà sono verificate e tipizzate come number
  return Math.sqrt(p.x ** 2 + p.y ** 2);
}

distanzaOrigine({ x: 3, y: 4 }); // 5
```

Va infine distinto `object` (con la `o` minuscola) dal tipo `Object` e dal tipo vuoto `{}`: questi ultimi accettano anche i valori primitivi tramite boxing e descrivono soltanto i membri di `Object.prototype`, risultando troppo permissivi. Per indicare un generico non primitivo si utilizza `object`, mentre per una forma concreta si utilizza sempre un object type specifico.

## Domande

<details>
<summary>Qual è la differenza pratica tra un object type specifico e il tipo generico `object` quando si accede a una proprietà?</summary>

Un object type specifico, come `{ nome: string }`, dichiara proprietà note al compilatore: l'accesso a `valore.nome` è verificato, tipizzato e supportato dall'autocompletamento. Il tipo `object` garantisce solo che il valore non sia un primitivo, ma non espone alcuna proprietà concreta, quindi un accesso come `valore.nome` produce l'errore `Property 'nome' does not exist on type 'object'`. Con `object` non si verificano le proprietà specifiche.

</details>

<details>
<summary>Perché passare un object literal con una proprietà in più rispetto al tipo atteso genera un errore, mentre assegnarlo prima a una variabile intermedia spesso no?</summary>

Quando un object literal viene passato o assegnato direttamente dove è atteso un tipo specifico, interviene l'excess property checking, che segnala le proprietà non previste con un messaggio come `Object literal may only specify known properties`. Assegnando prima il letterale a una variabile, il controllo sulle proprietà in eccesso non si applica e conta solo la compatibilità strutturale: se la variabile contiene tutte le proprietà richieste, l'assegnazione è ammessa anche in presenza di proprietà aggiuntive.

</details>

<details>
<summary>Cosa cambia tra dichiarare una proprietà come `descrizione?: string` e come `descrizione: string | undefined`?</summary>

Con `descrizione?: string` la proprietà è opzionale: può essere completamente assente dall'oggetto, e il suo tipo effettivo diventa `string | undefined`. Con `descrizione: string | undefined`, invece, la proprietà deve essere presente, pur potendo valere `undefined`: ometterla genera un errore perché la chiave è obbligatoria. In entrambi i casi, prima di usare il valore come stringa, occorre un controllo o un operatore come `??`.

</details>

<details>
<summary>Quali valori sono assegnabili al tipo `object` e quali no?</summary>

Al tipo `object` sono assegnabili tutti i valori non primitivi: object literal, array, funzioni e istanze di classi. Non sono assegnabili i valori primitivi, ovvero `string`, `number`, `boolean`, `bigint`, `symbol`, `null` e `undefined`. Per esempio `qualsiasiOggetto = [1, 2, 3]` è corretto, mentre `qualsiasiOggetto = 42` produce `Type 'number' is not assignable to type 'object'`.

</details>

<details>
<summary>Perché per descrivere un generico valore non primitivo si preferisce `object` rispetto a `Object` o a `{}`?</summary>

I tipi `Object` e `{}` accettano anche i valori primitivi tramite boxing e descrivono soltanto i membri di `Object.prototype`, risultando quindi troppo permissivi per indicare un valore non primitivo. Il tipo `object` con la `o` minuscola esclude i primitivi e cattura esattamente l'insieme dei valori non primitivi. Quando però servono le singole proprietà, nessuno di questi è adatto: occorre un object type specifico, un `type` alias o un'interface.

</details>
