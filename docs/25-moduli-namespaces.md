# Moduli e namespaces

Quando un progetto cresce, mantenere tutto il codice in un unico file diventa rapidamente ingestibile. TypeScript offre due meccanismi storici per organizzare il codice in unità riutilizzabili e isolate: i moduli ES e i namespaces. I moduli ES rappresentano lo standard attuale ed è l'approccio da adottare per qualsiasi codice nuovo; i namespaces vanno considerati un retaggio del passato, utile da conoscere ma non da preferire.

## Moduli ES

Un modulo è semplicemente un file che dichiara almeno un `import` o un `export`. Ogni modulo ha il proprio scope: le dichiarazioni di un file non inquinano lo scope globale e non sono visibili altrove finché non vengono esportate esplicitamente. Questo isolamento elimina i conflitti di nomi senza bisogno di alcuna struttura di raggruppamento aggiuntiva. Con i default di TypeScript 6.0 il sistema di moduli è già configurato su `"esnext"`, quindi i moduli ES funzionano senza alcuna impostazione particolare.

### Named export e named import

Un named export espone una o più dichiarazioni con il loro nome. Si può anteporre la keyword `export` direttamente alla dichiarazione, oppure raccogliere più nomi in un'unica clausola di export.

```ts
// file: matematica.ts
export const PI = 3.14159;

export function area(raggio: number): number {
  return PI * raggio ** 2;
}

interface Punto {
  x: number;
  y: number;
}

function distanza(a: Punto, b: Punto): number {
  return Math.hypot(b.x - a.x, b.y - a.y);
}

// Export raccolto in un'unica clausola; `type` marca un export di soli tipi
export { distanza, type Punto };
```

Per consumare un modulo si elencano tra parentesi graffe i nomi desiderati. I nomi importati devono corrispondere esattamente a quelli esportati, salvo l'uso di un alias introdotto con `as`.

```ts
// file: main.ts
import { area, PI, distanza, type Punto } from "./matematica";

const origine: Punto = { x: 0, y: 0 };
const altro: Punto = { x: 3, y: 4 };

console.log(area(2));            // utilizza l'export `area`
console.log(distanza(origine, altro)); // 5
console.log(PI);
```

Quando un nome importato entrerebbe in collisione con un identificatore locale, lo si rinomina con `as`.

```ts
// file: report.ts
import { area as calcolaArea } from "./matematica";

function area(): string {
  return "report sull'area";
}

console.log(area());            // funzione locale
console.log(calcolaArea(5));    // funzione importata e rinominata
```

### Default export

Ogni modulo può avere al massimo un default export, pensato per il "valore principale" del file. Si dichiara con `export default` e in fase di import non richiede le parentesi graffe: si sceglie liberamente il nome con cui riferirsi ad esso.

```ts
// file: contatore.ts
export default class Contatore {
  private valore = 0;

  incrementa(): number {
    return ++this.valore;
  }
}
```

```ts
// file: app.ts
import Contatore from "./contatore";

const c = new Contatore();
console.log(c.incrementa()); // 1
```

Default export e named export possono convivere nello stesso file e nello stesso import.

```ts
// file: logger.ts
export default function log(messaggio: string): void {
  console.log(`[log] ${messaggio}`);
}

export const LIVELLO = "info";
```

```ts
// file: avvio.ts
import log, { LIVELLO } from "./logger";

log(`livello corrente: ${LIVELLO}`);
```

### Import namespace con `import * as`

Per importare in blocco tutti i named export di un modulo sotto un unico identificatore si utilizza la forma `import * as`. L'oggetto risultante espone ogni export come proprietà; il default export, se presente, è accessibile tramite la proprietà `default`.

```ts
// file: geometria.ts
import * as matematica from "./matematica";

console.log(matematica.PI);
console.log(matematica.area(3));
```

Questa forma è comoda quando un modulo esporta molte funzioni correlate e si preferisce mantenerle raggruppate sotto un prefisso, ottenendo un effetto simile a quello dei namespaces ma restando nel sistema dei moduli ES.

### Re-export

Un modulo può ri-esportare ciò che importa da altri moduli, senza prima introdurlo nel proprio scope locale. Questo permette di costruire un punto di ingresso unico (spesso chiamato barrel) che aggrega le API di più file.

```ts
// file: index.ts
export { area, distanza } from "./matematica";
export { default as Contatore } from "./contatore";
export * from "./logger";

// Re-export con rinomina
export { PI as PI_GRECO } from "./matematica";
```

Chi consuma il package importa così da un unico percorso, senza conoscere la struttura interna delle cartelle.

```ts
// file: consumatore.ts
import { area, Contatore, PI_GRECO } from "./index";

console.log(area(1));
console.log(PI_GRECO);
const c = new Contatore();
```

## Namespaces

Prima che i moduli ES diventassero parte dello standard, i namespaces erano il meccanismo principale per raggruppare codice correlato — classi, interfacce, funzioni e variabili — all'interno di un unico blocco logico, così da prevenire i conflitti di nomi nello scope globale. Un namespace si dichiara con la keyword `namespace` e i suoi membri restano privati a meno che non vengano esposti con `export`. L'accesso dall'esterno avviene tramite la notazione `Nome.Elemento`.

```ts
namespace Validazione {
  // Membro interno, non visibile fuori dal namespace
  const espressioneEmail = /^[^@]+@[^@]+\.[^@]+$/;

  export interface Validatore {
    valida(testo: string): boolean;
  }

  export class ValidatoreEmail implements Validatore {
    valida(testo: string): boolean {
      return espressioneEmail.test(testo);
    }
  }
}

const validatore: Validazione.Validatore = new Validazione.ValidatoreEmail();
console.log(validatore.valida("utente@esempio.it")); // true
```

I namespaces possono anche essere annidati, esponendo sotto-raggruppamenti con la stessa notazione a punti.

```ts
namespace App {
  export namespace Util {
    export function formattaData(data: Date): string {
      return data.toISOString().slice(0, 10);
    }
  }
}

console.log(App.Util.formattaData(new Date(2026, 5, 11)));
```

Con l'adozione dei moduli ES l'uso dei namespaces è diventato decisamente meno comune. I moduli offrono lo stesso isolamento dello scope in modo nativo, gestiscono le dipendenze in maniera esplicita tramite `import` e si integrano con il caricamento dei moduli del runtime e con i bundler. Per questi motivi i namespaces vengono oggi considerati un approccio legacy: restano supportati per la manutenzione di codice esistente, ma per i nuovi progetti si preferisce organizzare il codice in moduli ES, eventualmente sfruttando `import * as` quando si desidera l'effetto di raggruppamento sotto un prefisso.

## triple-slash references

Le triple-slash references sono direttive in forma di commento, poste in cima a un file, che istruiscono il compilatore su una dipendenza. La forma più nota è quella che include un altro file, tipicamente un file di dichiarazione `.d.ts`.

```ts
/// <reference path="./tipi-globali.d.ts" />

const versione: AppVersione = { major: 6, minor: 0 };
```

Esiste anche la variante che dichiara una dipendenza da un pacchetto di tipi installato.

```ts
/// <reference types="node" />

import { readFileSync } from "node:fs";

const contenuto = readFileSync("dati.txt", "utf-8");
```

Queste direttive appartengono però a un'epoca precedente ai moduli ES e oggi risultano in larga parte superate. Per portare nello scope dichiarazioni e tipi si preferisce l'uso di `import`, mentre per i tipi delle librerie si fa affidamento sui pacchetti `@types` risolti automaticamente dal compilatore. Le triple-slash references conservano una nicchia residua nella scrittura manuale di alcuni file di dichiarazione, ma nel codice applicativo ordinario non andrebbero più impiegate.

## Librerie di terze parti e file di dichiarazione

I file di dichiarazione `.d.ts` contengono solo informazioni di tipo, senza implementazione, e fanno da ponte tra il codice JavaScript privo di tipi e il type system di TypeScript. Le librerie scritte direttamente in TypeScript spesso pubblicano i propri file di dichiarazione insieme al codice compilato, quindi funzionano senza ulteriori passaggi. Per le librerie JavaScript prive di tipi propri, la community mantiene pacchetti `@types` separati, installabili come dipendenze di sviluppo.

```bash
npm install lodash
npm install --save-dev @types/lodash
```

Una volta installato il pacchetto `@types`, non serve importarlo esplicitamente: il compilatore lo individua e ne applica le dichiarazioni automaticamente. È sufficiente importare la libreria come di consueto per ottenere il type checking completo.

```ts
import { chunk } from "lodash";

// `gruppi` è inferito come number[][] grazie a @types/lodash
const gruppi = chunk([1, 2, 3, 4, 5], 2);
console.log(gruppi); // [[1, 2], [3, 4], [5]]
```

Va tenuto presente che con i default di TypeScript 6.0 l'opzione `types` parte da un array vuoto: i pacchetti sotto `node_modules/@types` non vengono più enumerati implicitamente. Per i tipi globali di ambienti come Node serve dichiararli espressamente nel `tsconfig.json`.

```json
{
  "compilerOptions": {
    "types": ["node"]
  }
}
```

### Quando una libreria non ha tipi

Se per una libreria non esiste alcun pacchetto `@types` né un file di dichiarazione incluso, il compilatore segnala l'assenza delle definizioni.

```ts
import libreria from "libreria-senza-tipi";
// Errore: Could not find a declaration file for module 'libreria-senza-tipi'.
```

Si presentano allora tre strade. La prima, e più solida, consiste nello scrivere a mano un file di dichiarazione che descriva la forma del modulo, recuperando così la type safety.

```ts
// file: libreria-senza-tipi.d.ts
declare module "libreria-senza-tipi" {
  export function saluta(nome: string): string;
  const versione: string;
  export default versione;
}
```

La seconda, da usare solo come ripiego temporaneo, è ricorrere ad `any` o, meglio, ad `unknown`, accettando la perdita di controllo sui tipi. La terza è impiegare la keyword `declare` per affermare l'esistenza di variabili o moduli globali introdotti a runtime da uno script esterno, senza che il compilatore disponga del loro codice.

```ts
// Dichiara una variabile globale fornita da uno script caricato a parte
declare const ANALYTICS_ID: string;

console.log(ANALYTICS_ID.toUpperCase());
```

## Domande

<details>
<summary>Che cosa rende un file un modulo ES in TypeScript?</summary>

La presenza di almeno un `import` o un `export` a livello di file. In assenza di queste dichiarazioni il file è trattato come uno script che condivide lo scope globale; introducendo un qualsiasi `export` (o `import`) il file diventa un modulo con scope proprio, in cui le dichiarazioni non esportate restano private.

</details>

<details>
<summary>Qual è la differenza tra un default export e un named export al momento dell'import?</summary>

Un named export viene importato tra parentesi graffe e con il nome esatto con cui è stato esportato (eventualmente rinominato tramite `as`), mentre un default export viene importato senza graffe e con un nome scelto liberamente da chi importa. Un modulo può avere più named export ma al massimo un solo default export. I due possono comparire nello stesso import, ad esempio `import log, { LIVELLO } from "./logger";`.

</details>

<details>
<summary>Perché i namespaces sono considerati un approccio legacy rispetto ai moduli ES?</summary>

Perché i moduli ES forniscono nativamente l'isolamento dello scope che i namespaces ottenevano con il raggruppamento, gestiscono le dipendenze in modo esplicito tramite `import`/`export` e si integrano con il caricamento dei moduli del runtime e con i bundler. Con i moduli come standard attuale, l'uso dei namespaces è diventato raro e si raccomanda di organizzare il codice nuovo in moduli; per ottenere un effetto di raggruppamento sotto un prefisso è sufficiente la forma `import * as`.

</details>

<details>
<summary>Dopo aver installato un pacchetto `@types`, è necessario importarlo per usarne i tipi?</summary>

No. Il compilatore individua e applica automaticamente le dichiarazioni dei pacchetti presenti come `@types`, quindi è sufficiente importare la libreria vera e propria per ottenere il type checking. Va però ricordato che con i default di TypeScript 6.0 l'opzione `types` parte vuota, perciò i tipi globali di ambienti come Node vanno abilitati esplicitamente indicando `"types": ["node"]` nel `tsconfig.json`.

</details>

<details>
<summary>Quale alternativa moderna si preferisce alle triple-slash references e perché?</summary>

Si preferiscono gli `import` per portare nello scope dichiarazioni e tipi, e i pacchetti `@types` per i tipi delle librerie, dato che il compilatore li risolve automaticamente. Le triple-slash references appartengono a un'epoca precedente ai moduli ES e oggi sono in larga parte superate: conservano un impiego residuo nella scrittura manuale di alcuni file di dichiarazione, ma non andrebbero usate nel codice applicativo ordinario.

</details>
