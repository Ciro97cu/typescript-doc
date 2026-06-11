# Librerie di terze parti e tipi

Quando si lavora con TypeScript è normale appoggiarsi a librerie di terze parti distribuite tramite npm. Molte di queste librerie sono scritte in JavaScript e, di per sé, non contengono alcuna informazione di tipo: il loro codice descrive cosa fanno le funzioni, ma non quali argomenti accettano né cosa restituiscono. Per colmare questa lacuna esiste un meccanismo dedicato, i file di dichiarazione `.d.ts`, che fanno da ponte tra il mondo JavaScript e il mondo TypeScript.

## I file di dichiarazione `.d.ts`

Un file con estensione `.d.ts` (declaration file) contiene esclusivamente informazioni di tipo: nessuna implementazione, nessun codice eseguibile, ma soltanto la descrizione delle firme di funzioni, classi, interfacce, costanti e moduli esposti da una libreria. In altre parole descrive la "forma" pubblica del codice JavaScript sottostante, senza ricompilarlo. Quando il compilatore incontra un import che punta a una libreria JavaScript, cerca un file `.d.ts` associato per sapere come tipizzare ciò che viene importato.

La sintassi tipica fa uso della keyword `declare`, che comunica al compilatore che un certo elemento esiste altrove (è fornito a runtime dalla libreria) e che qui se ne dichiara soltanto il tipo.

```ts
// matematica.d.ts — descrive una libreria JavaScript senza implementazione
declare function somma(a: number, b: number): number;

declare const VERSIONE: string;

export { somma, VERSIONE };
```

Il file di dichiarazione non emette alcun JavaScript in fase di compilazione: serve unicamente al type checker durante lo sviluppo.

## Pacchetti `@types` e risoluzione automatica

Per le librerie JavaScript più diffuse non è necessario scrivere i file di dichiarazione a mano. La community mantiene il repository **DefinitelyTyped**, da cui vengono pubblicati pacchetti dedicati sotto lo scope `@types`. Questi pacchetti contengono soltanto file `.d.ts` e si installano come dipendenze di sviluppo, affiancandoli alla libreria vera e propria.

Un esempio classico è `lodash`: la libreria è scritta in JavaScript, quindi i suoi tipi si ottengono installando `@types/lodash`.

```bash
npm install lodash
npm install --save-dev @types/lodash
```

Una volta installato, il pacchetto `@types` non va importato esplicitamente. TypeScript lo individua automaticamente cercando nella cartella `node_modules/@types` i tipi corrispondenti ai moduli effettivamente importati nel codice. È quindi sufficiente importare la libreria nel modo consueto, e l'autocompletamento e il type checking funzionano come se la libreria fosse stata scritta in TypeScript.

```ts
import { chunk } from "lodash";

// I tipi vengono forniti da @types/lodash, risolti automaticamente
const gruppi: number[][] = chunk([1, 2, 3, 4, 5], 2);
// gruppi -> [[1, 2], [3, 4], [5]]
```

Va osservato che, con i valori predefiniti di TypeScript 6.0, l'opzione `types` è impostata su `[]` solo per quanto riguarda i tipi globali da includere d'ufficio (come quelli di Node). I tipi degli `@types` associati a un modulo che viene effettivamente importato continuano invece a essere risolti automaticamente in base agli import presenti nel codice. Per i tipi globali di Node, ad esempio, occorre richiederli esplicitamente:

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "target": "es2025",
    "strict": true,
    "types": ["node"]
  }
}
```

In questo modo `@types/node` viene incluso a livello globale, mentre i tipi di librerie come `lodash` restano disponibili semplicemente importando la libreria.

## Librerie scritte in TypeScript

Una parte crescente delle librerie moderne è scritta direttamente in TypeScript e distribuisce i propri file `.d.ts` insieme al codice compilato. In questi casi non serve nulla: installando la libreria si ottengono già le definizioni di tipo, perché il pacchetto le include e ne indica la posizione tramite il campo `types` (o `typings`) del proprio `package.json`.

```bash
npm install zod
```

```ts
import { z } from "zod";

// I tipi sono inclusi nel pacchetto stesso: nessun @types da installare
const Utente = z.object({
  nome: z.string(),
  eta: z.number(),
});

type Utente = z.infer<typeof Utente>;
// type Utente = { nome: string; eta: number }

const risultato = Utente.parse({ nome: "Mario", eta: 30 });
```

Per riconoscere questo scenario è sufficiente verificare il `package.json` della libreria: la presenza di un campo come `"types": "./dist/index.d.ts"` indica che le dichiarazioni sono già a bordo.

## Librerie senza tipi

Resta il caso delle librerie JavaScript per le quali non esistono né tipi inclusi né un pacchetto `@types`. In questa situazione il compilatore, con la configurazione rigorosa predefinita, segnala un errore al momento dell'import, perché non sa come tipizzare ciò che viene importato.

```ts
import vecchiaLib from "vecchia-lib";
// Errore: Could not find a declaration file for module 'vecchia-lib'.
```

Esistono tre approcci per risolvere il problema, dal più rigoroso al più permissivo.

### Scrivere un file `.d.ts` manuale

L'approccio più solido consiste nel dichiarare manualmente i tipi della libreria in un file `.d.ts`, descrivendone soltanto le parti effettivamente utilizzate. Si crea un modulo di dichiarazione con `declare module`, indicando il nome esatto con cui la libreria viene importata.

```ts
// types/vecchia-lib.d.ts
declare module "vecchia-lib" {
  export function saluta(nome: string): string;

  export interface Opzioni {
    maiuscolo: boolean;
  }

  export function formatta(testo: string, opzioni?: Opzioni): string;
}
```

A questo punto l'import precedente acquisisce tipi corretti e completi per le funzioni dichiarate, con autocompletamento e controllo degli argomenti. È buona pratica raccogliere questi file in una cartella dedicata (ad esempio `types`) e assicurarsi che rientri tra i percorsi analizzati dal compilatore.

### Usare `any` o `unknown`

Quando i tipi precisi non sono noti o non vale la pena descriverli, si può ricorrere a una dichiarazione minima del modulo. Il modo più rapido attribuisce implicitamente il tipo `any` a tutto ciò che proviene dalla libreria.

```ts
// types/vecchia-lib.d.ts
declare module "vecchia-lib";
```

Con questa dichiarazione l'import smette di generare errori, ma ogni valore importato ha tipo `any`, con conseguente perdita di type safety e di autocompletamento. Per un compromesso più prudente è preferibile tipizzare esplicitamente con `unknown`, che obbliga a un controllo del tipo (narrowing) prima di utilizzare i valori.

```ts
// types/vecchia-lib.d.ts
declare module "vecchia-lib" {
  const contenuto: unknown;
  export default contenuto;
}
```

```ts
import vecchiaLib from "vecchia-lib";

// vecchiaLib è di tipo unknown: serve un controllo prima di usarlo
if (typeof vecchiaLib === "function") {
  vecchiaLib();
}
```

### Usare `declare` per variabili e moduli globali

Alcune librerie non si importano come moduli ma si caricano tramite un tag `<script>`, esponendo variabili nello scope globale. In questi casi non c'è alcun import da tipizzare: si dichiara direttamente l'esistenza della variabile globale con `declare`, così il compilatore la considera valida senza emettere codice.

```ts
// globali.d.ts
declare const ANALYTICS_ID: string;

declare function trackEvent(nome: string, dati?: Record<string, unknown>): void;
```

```ts
// La variabile globale è ora nota al type checker
trackEvent("pagina_vista", { percorso: "/home" });
console.log(ANALYTICS_ID);
```

La keyword `declare` è il filo conduttore di tutti questi scenari: comunica al compilatore che qualcosa esiste a runtime e ne fornisce soltanto la descrizione di tipo, senza generare alcun output JavaScript.

## Domande

<details>
<summary>A cosa serve un file `.d.ts` e che differenza ha rispetto a un normale file `.ts`?</summary>

Un file `.d.ts` (declaration file) contiene esclusivamente informazioni di tipo, senza alcuna implementazione né codice eseguibile, e non produce output JavaScript in fase di compilazione. Descrive la forma pubblica del codice JavaScript sottostante, facendo da ponte tra JS e TS. Un normale file `.ts` contiene invece codice che viene type-checked e compilato in JavaScript.

</details>

<details>
<summary>Dopo aver installato `@types/lodash`, è necessario importarlo esplicitamente nel codice?</summary>

No. I pacchetti `@types` contengono soltanto file `.d.ts` e vengono risolti automaticamente dal compilatore, che cerca in `node_modules/@types` i tipi corrispondenti ai moduli effettivamente importati. È sufficiente importare la libreria nel modo consueto (`import { chunk } from "lodash"`) perché i tipi vengano applicati.

</details>

<details>
<summary>Come si capisce se una libreria include già i propri tipi senza bisogno di un pacchetto `@types`?</summary>

Le librerie scritte in TypeScript distribuiscono i file `.d.ts` insieme al codice e lo segnalano nel proprio `package.json` tramite il campo `types` (o `typings`), ad esempio `"types": "./dist/index.d.ts"`. La presenza di questo campo indica che le definizioni sono già incluse e non occorre installare alcun pacchetto `@types`.

</details>

<details>
<summary>Quali sono le tre strategie per usare una libreria JavaScript priva di tipi, e quale offre la maggiore type safety?</summary>

Le tre strategie sono: scrivere manualmente un file `.d.ts` con `declare module`, ricorrere a `any` o `unknown` per la libreria, oppure usare `declare` per dichiarare variabili e moduli globali. La maggiore type safety si ottiene scrivendo un file `.d.ts` manuale, perché fornisce firme precise per le parti utilizzate; tra `any` e `unknown` è preferibile `unknown`, che impone un controllo del tipo prima dell'uso.

</details>

<details>
<summary>Perché si usa `declare module "nome-libreria"` senza corpo per silenziare l'errore di import, e qual è il suo costo?</summary>

La dichiarazione `declare module "nome-libreria";` senza corpo comunica al compilatore che il modulo esiste, eliminando l'errore "Could not find a declaration file for module". Il costo è che ogni valore importato dalla libreria assume implicitamente il tipo `any`, con conseguente perdita di type checking e di autocompletamento per tutto ciò che proviene da quel modulo.

</details>
