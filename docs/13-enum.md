# enum

Un `enum` permette di definire un insieme di costanti con nomi leggibili, raggruppando sotto un'unica entità una serie di valori correlati. L'obiettivo principale è rendere il codice più espressivo e più sicuro: invece di disseminare numeri o stringhe "magici" nel programma, si fa riferimento a identificatori parlanti, e il compilatore limita i valori ammessi a quelli dichiarati nell'enum.

A differenza della maggior parte dei costrutti di tipo di TypeScript, un `enum` non vive soltanto a livello di tipi: viene emesso anche nel JavaScript generato come oggetto reale, presente quindi a runtime. Questa caratteristica lo distingue dalle alternative basate sui tipi (union di literal, oggetti `as const`) e merita attenzione nel momento in cui si sceglie come modellare un insieme finito di valori.

## Enum numerici

Quando i membri di un enum non hanno un valore esplicito, TypeScript assegna loro dei numeri interi a partire da `0`, incrementando di uno per ogni membro successivo. Questo è il comportamento predefinito.

```ts
enum Direzione {
  Su,    // 0
  Giu,   // 1
  Sinistra, // 2
  Destra,   // 3
}

const movimento: Direzione = Direzione.Su;
console.log(movimento); // 0
```

È possibile fissare il valore iniziale: l'auto-incremento riprende dal valore assegnato esplicitamente.

```ts
enum CodiceStato {
  Ok = 200,
  Creato,        // 201
  NoContent = 204,
  BadRequest = 400,
  NonAutorizzato, // 401
}

console.log(CodiceStato.Creato);        // 201
console.log(CodiceStato.NonAutorizzato); // 401
```

Una peculiarità degli enum numerici è il cosiddetto reverse mapping: l'oggetto generato a runtime contiene sia la corrispondenza nome → valore sia quella valore → nome. Risulta quindi possibile risalire dal numero al nome del membro.

```ts
enum Direzione {
  Su,
  Giu,
}

console.log(Direzione.Su);      // 0
console.log(Direzione[0]);      // "Su"
console.log(Direzione["Giu"]);  // 1
```

Va però osservato che un enum numerico è permissivo in fase di assegnazione: qualsiasi `number` viene accettato dove è atteso il tipo dell'enum, anche se non corrisponde ad alcun membro dichiarato. Questo riduce le garanzie offerte dal type system.

```ts
enum Direzione {
  Su,
  Giu,
}

const d: Direzione = 99; // Nessun errore: un number arbitrario è assegnabile a un enum numerico
```

## Enum stringa

Negli enum stringa ogni membro deve ricevere un valore esplicito, e quel valore deve essere una stringa letterale. Non esiste auto-incremento, proprio perché non c'è un concetto di "stringa successiva".

```ts
enum Ruolo {
  Admin = "ADMIN",
  Editor = "EDITOR",
  Lettore = "LETTORE",
}

const ruoloCorrente: Ruolo = Ruolo.Editor;
console.log(ruoloCorrente); // "EDITOR"
```

Gli enum stringa offrono due vantaggi pratici rispetto a quelli numerici. Il primo è la leggibilità in fase di debug: un valore serializzato come `"EDITOR"` è immediatamente comprensibile, a differenza di un `1` privo di contesto. Il secondo è una maggiore sicurezza in assegnazione: una stringa arbitraria non è assegnabile al tipo dell'enum, quindi il compilatore segnala l'errore.

```ts
enum Ruolo {
  Admin = "ADMIN",
  Editor = "EDITOR",
}

const r: Ruolo = "ADMIN"; // Errore: Type '"ADMIN"' is not assignable to type 'Ruolo'
const valido: Ruolo = Ruolo.Admin; // Corretto
```

Per contro, gli enum stringa non generano alcun reverse mapping: l'oggetto a runtime contiene solo la corrispondenza nome → valore.

## Enum come tipo e come valore

Ogni dichiarazione di `enum` introduce contemporaneamente un tipo e un valore con lo stesso nome. Il tipo si usa nelle annotazioni, mentre il valore è l'oggetto interrogabile a runtime per accedere ai membri. È inoltre possibile iterare sui valori di un enum stringa estraendone le chiavi, ed è comune usarlo nelle firme di funzione per vincolare un parametro a un insieme chiuso.

```ts
enum LivelloLog {
  Debug = "DEBUG",
  Info = "INFO",
  Errore = "ERRORE",
}

function registra(livello: LivelloLog, messaggio: string): void {
  console.log(`[${livello}] ${messaggio}`);
}

registra(LivelloLog.Info, "Avvio completato");

// Elenco dei valori dichiarati
const livelli = Object.values(LivelloLog);
console.log(livelli); // ["DEBUG", "INFO", "ERRORE"]
```

## Alternative moderne: union di literal e oggetti `as const`

In molti casi reali un `enum` può essere sostituito da una union di literal type. Si tratta di un costrutto puramente a livello di tipo, che non emette alcun codice JavaScript e non aggiunge peso al bundle. Il controllo sui valori ammessi resta identico, ma senza la presenza a runtime di un oggetto enum.

```ts
type Ruolo = "admin" | "editor" | "lettore";

function assegnaRuolo(ruolo: Ruolo): void {
  // ...
}

assegnaRuolo("admin");   // Corretto
assegnaRuolo("ospite");  // Errore: Argument of type '"ospite"' is not assignable to parameter of type 'Ruolo'
```

Quando, oltre al vincolo sui tipi, serve anche un oggetto navigabile a runtime, una soluzione idiomatica è un oggetto dichiarato con `as const`, dal quale si deriva il tipo tramite un'indicizzazione. In questo modo si ottiene sia la collezione di costanti sia la union dei valori, mantenendo il pieno controllo sul JavaScript prodotto.

```ts
const Ruolo = {
  Admin: "ADMIN",
  Editor: "EDITOR",
  Lettore: "LETTORE",
} as const;

// Union dei valori: "ADMIN" | "EDITOR" | "LETTORE"
type Ruolo = (typeof Ruolo)[keyof typeof Ruolo];

function assegnaRuolo(ruolo: Ruolo): void {
  // ...
}

assegnaRuolo(Ruolo.Admin); // Corretto, equivale a passare "ADMIN"
assegnaRuolo("EDITOR");    // Corretto: è un valore della union
assegnaRuolo("ALTRO");     // Errore: Argument of type '"ALTRO"' is not assignable to parameter of type 'Ruolo'
```

La preferenza per queste alternative nasce da diversi fattori: l'assenza di codice emesso, la trasparenza rispetto a JavaScript puro (un oggetto `as const` è semplicemente un oggetto) e il fatto che gli enum numerici accettano `number` arbitrari, indebolendo le garanzie. Resta comunque legittimo usare gli `enum`, in particolare quelli stringa, quando si desidera un'entità con un nome unico che funga sia da tipo sia da contenitore di costanti.

## Caveat dei const enum

Anteporre `const` alla dichiarazione produce un const enum. In questo caso TypeScript non emette alcun oggetto: tutti i riferimenti ai membri vengono sostituiti direttamente con i loro valori letterali (inlining) durante la compilazione, eliminando l'overhead a runtime.

```ts
const enum Direzione {
  Su,
  Giu,
  Sinistra,
  Destra,
}

const d = Direzione.Su;
// Il JavaScript emesso non contiene l'oggetto Direzione:
// const d = 0 /* Direzione.Su */;
```

Questo guadagno in efficienza ha però controindicazioni rilevanti. Poiché il valore viene inlinato, un const enum non esiste come oggetto a runtime: non è iterabile, non è interrogabile dinamicamente e non sopravvive all'interfaccia pubblica di un pacchetto. Per questo motivo l'uso dei const enum è sconsigliato nel codice di librerie: un consumatore che compila i file in modo isolato (come fanno Babel, esbuild e, in generale, gli strumenti che usano `isolatedModules`) non dispone delle informazioni necessarie a effettuare l'inlining e ottiene un errore o un comportamento scorretto. In presenza di una pipeline che compila ogni file singolarmente, l'opzione `isolatedModules` segnala esplicitamente l'incompatibilità.

Quando l'obiettivo è semplicemente azzerare l'overhead a runtime, una union di literal raggiunge lo stesso risultato senza i rischi del const enum, poiché non emette nulla per costruzione.

## Domande

<details>
<summary>Qual è la differenza, in termini di codice generato, tra un enum stringa e una union di literal type?</summary>

Un enum stringa viene emesso nel JavaScript come un oggetto reale, presente a runtime e interrogabile (per esempio con `Object.values`). Una union di literal type, invece, esiste soltanto a livello di tipi: non produce alcun codice JavaScript e quindi non aggiunge peso al bundle. Entrambi vincolano i valori ammessi in fase di compilazione allo stesso modo, ma solo l'enum lascia una traccia a runtime.

</details>

<details>
<summary>Perché un enum numerico è considerato meno sicuro di un enum stringa in fase di assegnazione?</summary>

In un enum numerico qualsiasi `number` è assegnabile al tipo dell'enum, anche un valore che non corrisponde ad alcun membro dichiarato: ad esempio `const d: Direzione = 99` non produce errori. Un enum stringa, al contrario, non accetta stringhe arbitrarie: solo i valori dei membri dichiarati sono assegnabili, quindi il compilatore intercetta gli errori. Questo rende gli enum stringa più rigorosi nel controllo dei valori.

</details>

<details>
<summary>Cosa si intende per reverse mapping e quali enum lo offrono?</summary>

Il reverse mapping è la presenza, nell'oggetto generato a runtime, sia della corrispondenza nome → valore sia di quella valore → nome. Lo offrono solo gli enum numerici: dato `enum Direzione { Su }`, si ottiene `Direzione.Su === 0` e `Direzione[0] === "Su"`. Gli enum stringa non generano alcun reverse mapping, ma soltanto la mappatura nome → valore.

</details>

<details>
<summary>Perché i const enum sono sconsigliati nel codice di una libreria?</summary>

Un const enum non viene emesso come oggetto: i suoi membri vengono inlinati con i rispettivi valori al momento della compilazione. Un consumatore che compila i file in isolamento (Babel, esbuild, o comunque con `isolatedModules` attivo) non dispone delle informazioni necessarie a effettuare l'inlining e ottiene un errore o un comportamento scorretto, dato che a runtime l'oggetto enum non esiste. Per questo, nelle librerie, conviene preferire un enum normale o una union di literal.

</details>

<details>
<summary>Come si ottiene una union dei valori a partire da un oggetto dichiarato con `as const`?</summary>

Si indicizza il tipo dell'oggetto con `keyof typeof` sull'oggetto stesso. Dato `const Ruolo = { Admin: "ADMIN", Editor: "EDITOR" } as const`, l'espressione `type Ruolo = (typeof Ruolo)[keyof typeof Ruolo]` produce la union `"ADMIN" | "EDITOR"`. Si ottiene così sia un oggetto navigabile a runtime sia un tipo che vincola i valori ammessi, senza ricorrere a un `enum`.

</details>
