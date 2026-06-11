# Type alias

Un type alias permette di assegnare un nome a un tipo già esistente. La parola chiave `type` introduce un identificatore che diventa sinonimo del tipo descritto alla destra del segno di uguale: da quel momento il nome può essere utilizzato ovunque sarebbe lecito scrivere il tipo per esteso. L'obiettivo non è creare un tipo nuovo dal nulla, ma dare un'etichetta leggibile e riutilizzabile a una forma che altrimenti andrebbe ripetuta in più punti.

L'utilità si manifesta soprattutto quando un tipo diventa lungo o ricorrente. Centralizzare la definizione in un alias riduce la duplicazione, migliora la leggibilità e rende le modifiche più semplici: aggiornando l'alias in un solo punto si aggiorna automaticamente ogni occorrenza. Un type alias non viene compilato in alcun costrutto JavaScript, perché esiste unicamente a livello di sistema dei tipi e scompare al momento dell'emissione del codice.

## Dare un nome a un tipo primitivo

Il caso più semplice consiste nel rinominare un tipo primitivo. Da solo questo offre un beneficio limitato, ma diventa significativo quando il nome aggiunge un valore semantico, rendendo esplicito il significato di un valore che altrimenti sarebbe genericamente un `string` o un `number`.

```ts
type Identificativo = string;
type Anno = number;

const codice: Identificativo = "ABC-123";
const annoNascita: Anno = 1990;
```

Va sottolineato che un alias di questo genere non introduce alcun controllo aggiuntivo: `Identificativo` resta a tutti gli effetti un `string` ed è interscambiabile con qualunque altra stringa. Per ottenere tipi nominali realmente distinti occorrono tecniche diverse, come i branded type, che esulano dal semplice alias.

## Union type con nome

Uno degli impieghi più diffusi consiste nell'assegnare un nome a una union type. Riunire sotto un'unica etichetta l'insieme dei valori ammessi rende la firma delle funzioni più chiara e consente di riutilizzare lo stesso insieme in più punti.

```ts
type Esito = "successo" | "errore" | "in attesa";

function descrivi(stato: Esito): string {
  switch (stato) {
    case "successo":
      return "Operazione completata";
    case "errore":
      return "Si è verificato un problema";
    case "in attesa":
      return "Elaborazione in corso";
  }
}

descrivi("successo");
// Errore: Argument of type '"annullato"' is not assignable to parameter of type 'Esito'.
descrivi("annullato");
```

La union può combinare anche tipi diversi tra loro, non solo literal type. È frequente, per esempio, accettare un identificativo espresso indifferentemente come stringa o come numero.

```ts
type Chiave = string | number;

function leggi(record: Record<Chiave, unknown>, chiave: Chiave): unknown {
  return record[chiave];
}
```

## Tuple con nome

Un type alias può descrivere una tuple, ovvero un array in cui la posizione, il numero e il tipo degli elementi sono noti in anticipo. Assegnare un nome alla tuple chiarisce il significato di ciascuna posizione, che altrimenti resterebbe implicito.

```ts
type Coordinata = [number, number];
type VoceRubrica = [nome: string, telefono: string];

const origine: Coordinata = [0, 0];
const contatto: VoceRubrica = ["Mario", "+39 333 1234567"];

// Errore: Type 'string' is not assignable to type 'number'.
const erroneo: Coordinata = ["0", 0];
```

Le etichette `nome` e `telefono` nell'esempio sono nomi descrittivi degli elementi della tuple: non influiscono sui controlli, ma migliorano la documentazione e i suggerimenti dell'editor.

## Oggetti

L'applicazione più ricorrente dei type alias riguarda la forma degli oggetti. Definire una volta la struttura attesa e riferirla tramite il suo nome evita di ripetere l'elenco delle proprietà in ogni firma. Le proprietà possono essere marcate come opzionali con `?` oppure come immutabili con `readonly`.

```ts
type Persona = {
  readonly id: number;
  nome: string;
  eta: number;
  email?: string;
};

function saluta(persona: Persona): string {
  return `Ciao ${persona.nome}, hai ${persona.eta} anni`;
}

const utente: Persona = { id: 1, nome: "Anna", eta: 30 };
saluta(utente);

// Errore: Cannot assign to 'id' because it is a read-only property.
utente.id = 2;
```

Gli alias possono inoltre comporsi tra loro, riferendosi l'uno all'altro per descrivere strutture annidate. Questo consente di costruire modelli articolati partendo da blocchi più piccoli e nominati.

```ts
type Indirizzo = {
  via: string;
  citta: string;
  cap: string;
};

type Cliente = {
  nome: string;
  indirizzo: Indirizzo;
};

const cliente: Cliente = {
  nome: "Luca",
  indirizzo: { via: "Via Roma 1", citta: "Milano", cap: "20100" },
};
```

## Rapporto con interface

Per descrivere la forma di un oggetto è possibile utilizzare sia un type alias sia un'interface, e nella maggior parte dei casi le due soluzioni risultano intercambiabili. Esistono però differenze rilevanti: un type alias non può essere riaperto per aggiungere proprietà in un secondo momento, mentre un'interface supporta il declaration merging; in compenso un type alias può dare un nome anche a primitivi, union type e tuple, cosa che un'interface non consente, perché si limita a descrivere forme di oggetti. Il confronto dettagliato tra i due strumenti, con i criteri per scegliere l'uno o l'altro, è trattato nella sezione Interfaces.

## Domande

<details>
<summary>Un type alias crea un nuovo tipo distinto rispetto a quello a cui fa riferimento?</summary>

No. Un type alias introduce soltanto un nome alternativo per un tipo esistente, senza creare un tipo nominalmente distinto. Per esempio, dato `type Identificativo = string`, un valore di tipo `Identificativo` resta a tutti gli effetti un `string` ed è pienamente interscambiabile con qualunque altra stringa. Non viene aggiunto alcun controllo supplementare e, dopo la compilazione, l'alias scompare perché esiste solo a livello di sistema dei tipi.

</details>

<details>
<summary>Quali categorie di tipi può nominare un type alias che un'interface non è in grado di descrivere?</summary>

Un type alias può dare un nome a tipi primitivi, a union type e a tuple, oltre che alla forma degli oggetti. Un'interface, invece, può descrivere unicamente forme di oggetti (incluse call signature e index signature), ma non può rinominare un primitivo né rappresentare direttamente una union type. Per questo motivo, di fronte a una union come `type Esito = "successo" | "errore"`, l'unica opzione tra le due è il type alias.

</details>

<details>
<summary>Cosa accade assegnando un valore non previsto a un parametro tipizzato con una union di literal type?</summary>

Il compilatore segnala un errore in fase di type checking, perché il valore non è assegnabile alla union dichiarata. Dato `type Esito = "successo" | "errore" | "in attesa"`, una chiamata come `descrivi("annullato")` produce un messaggio del tipo `Argument of type '"annullato"' is not assignable to parameter of type 'Esito'.`. Sono ammessi soltanto i literal elencati nella union.

</details>

<details>
<summary>Qual è la differenza principale, sul piano dell'estendibilità, tra un type alias e un'interface?</summary>

Un type alias non può essere riaperto: una volta definito, non è possibile aggiungere nuove proprietà dichiarandolo nuovamente. Un'interface, al contrario, supporta il declaration merging, ovvero più dichiarazioni con lo stesso nome vengono fuse automaticamente in un unico contratto. Per il resto, nella descrizione di forme di oggetti le due soluzioni sono in larga parte equivalenti; il confronto completo è approfondito nella sezione Interfaces.

</details>

<details>
<summary>Le etichette presenti in una tuple come `type VoceRubrica = [nome: string, telefono: string]` influenzano i controlli di tipo?</summary>

No. Le etichette `nome` e `telefono` hanno valore esclusivamente documentale: migliorano la leggibilità e i suggerimenti dell'editor, ma non modificano i controlli effettuati dal compilatore. Ciò che viene verificato sono il numero, l'ordine e il tipo degli elementi: la tuple ammette esattamente due elementi, entrambi di tipo `string`, indipendentemente dai nomi assegnati alle posizioni.

</details>
