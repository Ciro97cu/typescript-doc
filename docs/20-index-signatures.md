# Index signatures

Le index signatures permettono di descrivere oggetti dei quali non si conoscono in anticipo i nomi delle proprietà, ma se ne conosce la struttura: il tipo delle chiavi e il tipo dei valori associati. Si tratta dello strumento naturale per modellare strutture dati tipo mappa o dizionario, ossia raccolte in cui le proprietà vengono aggiunte dinamicamente e il loro numero non è fissato a priori.

Quando si dichiara un tipo oggetto con un elenco esplicito di proprietà, ogni nome deve essere noto al momento della definizione. Una index signature ribalta questa logica: anziché elencare le singole proprietà, si descrive una regola generale che vale per qualsiasi chiave di un certo tipo. La sintassi prevede di indicare, tra parentesi quadre, un nome simbolico per la chiave seguito dal suo tipo, e poi il tipo dei valori.

```ts
interface DizionarioErrori {
  [codice: string]: string;
}

const errori: DizionarioErrori = {
  E001: "Campo obbligatorio mancante",
  E002: "Formato email non valido",
};

// Qualsiasi chiave di tipo string è ammessa, anche se non dichiarata esplicitamente
errori.E003 = "Sessione scaduta";
errori["E004"] = "Permesso negato";
```

Il nome scelto per la chiave (`codice` nell'esempio) è puramente documentativo: serve a chiarire l'intento, ma non vincola in alcun modo i nomi effettivi delle proprietà. Ciò che conta sono i due tipi coinvolti, quello della chiave e quello del valore.

## Vincolo di omogeneità

Una index signature impone un vincolo di omogeneità sia sulle chiavi sia sui valori. Tutte le chiavi devono condividere lo stesso tipo, dichiarato nella signature, e tutti i valori devono essere assegnabili al tipo dichiarato. Questo significa che un oggetto descritto da `[key: string]: number` non può contenere valori che non siano numerici.

```ts
interface Punteggi {
  [giocatore: string]: number;
}

const classifica: Punteggi = {
  anna: 120,
  marco: 95,
};

// Errore: Type 'string' is not assignable to type 'number'.
const erronea: Punteggi = {
  anna: "centoventi",
};
```

Il vincolo si estende anche alle proprietà dichiarate esplicitamente accanto a una index signature. È possibile combinare proprietà note e una index signature nello stesso tipo, ma il tipo di ciascuna proprietà esplicita deve essere compatibile con il tipo dei valori della signature. In caso contrario il compilatore segnala l'incongruenza.

```ts
interface Configurazione {
  [opzione: string]: string;
  versione: string; // Compatibile: string è assegnabile a string
}

interface ConfigurazioneNonValida {
  [opzione: string]: string;
  // Errore: Property 'attiva' of type 'boolean' is not assignable to 'string' index type 'string'.
  attiva: boolean;
}
```

Quando si desidera ammettere valori di natura diversa pur mantenendo una struttura aperta, la soluzione corretta non è violare l'omogeneità, bensì ampliare il tipo dei valori con un union type. In questo modo il vincolo resta soddisfatto, perché ogni valore continua a essere assegnabile al tipo dichiarato.

```ts
interface Impostazioni {
  [chiave: string]: string | number | boolean;
}

const preferenze: Impostazioni = {
  tema: "scuro",
  dimensioneFont: 14,
  notifiche: true,
};
```

## Chiavi numeriche trattate come stringhe

Le chiavi degli oggetti in JavaScript sono sempre, di fatto, delle stringhe (o dei `symbol`). Per questo motivo, anche quando si dichiara una index signature con chiavi numeriche tramite `[indice: number]`, le chiavi numeriche vengono normalizzate a stringhe a runtime: scrivere `oggetto[0]` equivale ad accedere alla proprietà `"0"`.

TypeScript modella questo comportamento imponendo una regola di coerenza: se un tipo possiede sia una index signature numerica sia una index signature di tipo stringa, il tipo dei valori della signature numerica deve essere assegnabile al tipo dei valori della signature di stringa. La signature numerica viene quindi considerata un caso più ristretto di quella di stringa.

```ts
interface Mappa {
  [chiaveStringa: string]: string;
  // Errore: 'number' index type 'number' is not assignable to 'string' index type 'string'.
  [chiaveNumerica: number]: number;
}
```

La conseguenza pratica è che un oggetto con index signature di tipo stringa accetta indifferentemente l'accesso tramite chiave numerica o tramite chiave stringa, perché la chiave numerica viene comunque convertita in stringa prima della risoluzione della proprietà.

```ts
interface Etichette {
  [id: string]: string;
}

const etichette: Etichette = {
  "1": "primo",
  "2": "secondo",
};

// Entrambi gli accessi sono validi e si riferiscono alla stessa proprietà "1"
const a: string = etichette[1];
const b: string = etichette["1"];
```

## Index signatures e iterazione

Poiché l'insieme delle chiavi non è noto al momento della compilazione, l'accesso a una proprietà tramite index signature restituisce sempre il tipo dei valori dichiarato, anche per chiavi che potrebbero non esistere a runtime. Per ottenere un controllo più rigoroso conviene abilitare `noUncheckedIndexedAccess` nel `tsconfig.json`: con questa opzione, l'accesso indicizzato produce un tipo che include `undefined`, costringendo a gestire il caso di chiave assente.

```json
{
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

```ts
interface Inventario {
  [articolo: string]: number;
}

const magazzino: Inventario = { viti: 200, bulloni: 150 };

// Con noUncheckedIndexedAccess il tipo è number | undefined
const quantita = magazzino["rondelle"];

// Si rende necessario un controllo prima dell'uso aritmetico
if (quantita !== undefined) {
  const totale: number = quantita + 10;
  console.log(totale);
}
```

L'iterazione su un oggetto con index signature avviene tramite i consueti costrutti del linguaggio. `for...in` scorre le chiavi enumerabili, mentre `Object.keys`, `Object.values` e `Object.entries` restituiscono rispettivamente chiavi, valori e coppie. Le chiavi prodotte sono sempre di tipo `string`, coerentemente con la normalizzazione descritta in precedenza.

```ts
interface Prezzi {
  [prodotto: string]: number;
}

const listino: Prezzi = { pane: 2, latte: 1, caffe: 5 };

for (const prodotto in listino) {
  // prodotto è di tipo string
  console.log(`${prodotto}: ${listino[prodotto]}`);
}

const totale = Object.values(listino).reduce((somma, prezzo) => somma + prezzo, 0);
console.log(`Totale: ${totale}`);
```

Quando l'insieme delle chiavi è invece noto e ristretto, è preferibile evitare una index signature aperta e ricorrere all'utility type `Record`, che consente di vincolare le chiavi a un union di literal type mantenendo la stessa ergonomia di una mappa.

```ts
type Ruolo = "admin" | "editor" | "viewer";

const permessi: Record<Ruolo, boolean> = {
  admin: true,
  editor: true,
  viewer: false,
};

// Errore: Object literal may only specify known properties,
// and 'guest' does not exist in type 'Record<Ruolo, boolean>'.
const permessiErrati: Record<Ruolo, boolean> = {
  admin: true,
  editor: true,
  viewer: false,
  guest: false,
};
```

## Domande

<details>
<summary>Quale vincolo impone una index signature sui valori di un oggetto?</summary>

Impone l'omogeneità: tutti i valori associati alle chiavi devono essere assegnabili al tipo dichiarato nella signature. In un tipo `[key: string]: number` non è ammesso alcun valore non numerico. Se si vogliono ammettere valori di tipi diversi, occorre ampliare il tipo dei valori con un union type, ad esempio `[key: string]: string | number | boolean`, senza violare il vincolo.
</details>

<details>
<summary>Cosa accade quando si accede a un oggetto con index signature di tipo stringa usando una chiave numerica, ad esempio oggetto[1]?</summary>

La chiave numerica viene normalizzata a stringa, poiché in JavaScript le chiavi degli oggetti sono sempre stringhe o `symbol`. L'accesso `oggetto[1]` è quindi equivalente a `oggetto["1"]` e si riferisce alla stessa proprietà. Entrambi gli accessi sono validi e restituiscono il tipo dei valori dichiarato dalla signature.
</details>

<details>
<summary>Perché una proprietà esplicita dichiarata accanto a una index signature può generare un errore del compilatore?</summary>

Perché il tipo della proprietà esplicita deve essere compatibile con il tipo dei valori della index signature. Data una signature `[opzione: string]: string`, una proprietà `attiva: boolean` produce un errore del tipo "Property 'attiva' of type 'boolean' is not assignable to 'string' index type 'string'.", in quanto `boolean` non è assegnabile a `string`. La proprietà esplicita deve rientrare nel tipo dei valori imposto dalla signature.
</details>

<details>
<summary>Che effetto ha l'opzione noUncheckedIndexedAccess sull'accesso tramite index signature?</summary>

Aggiunge `undefined` al tipo restituito dall'accesso indicizzato. Senza l'opzione, `magazzino["rondelle"]` ha tipo `number` anche se la chiave potrebbe non esistere a runtime; con `noUncheckedIndexedAccess` attiva il tipo diventa `number | undefined`, obbligando a verificare la presenza del valore prima di utilizzarlo. In questo modo il type system riflette il fatto che la chiave potrebbe essere assente.
</details>

<details>
<summary>Quando conviene usare Record al posto di una index signature aperta?</summary>

Quando l'insieme delle chiavi è noto e ristretto. Una index signature come `[key: string]: T` accetta qualsiasi chiave stringa, mentre `Record<K, T>` consente di vincolare le chiavi a un union di literal type, ad esempio `Record<"admin" | "editor" | "viewer", boolean>`. In questo modo il compilatore segnala sia le chiavi mancanti sia quelle non previste, offrendo un controllo più stretto rispetto alla index signature aperta.
</details>
