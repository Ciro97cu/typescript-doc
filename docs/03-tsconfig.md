# tsconfig.json

Il file `tsconfig.json` è il file di configurazione che guida il compilatore su come compilare un progetto. La sua presenza nella radice di una cartella segnala a TypeScript che quella cartella è la root di un progetto: a quel punto è sufficiente eseguire `tsc` senza argomenti perché il compilatore individui automaticamente i file da elaborare e applichi le opzioni indicate. Il file viene normalmente generato con `npx tsc --init`, che produce uno scheletro commentato pronto da personalizzare.

Le opzioni si suddividono in due grandi famiglie: quelle che determinano *quali* file fanno parte del progetto (`include`, `exclude`, `files`) e quelle, raccolte sotto `compilerOptions`, che determinano *come* quei file vengono compilati. Un esempio minimale e moderno, valido su TypeScript 6.0, ha questo aspetto.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "dist",
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Va tenuto presente che molte impostazioni hanno valori predefiniti già orientati allo sviluppo moderno: in TypeScript 6.0 la modalità rigorosa è attiva di default, il sistema di moduli predefinito è `esnext` e il `target` è allineato all'anno corrente. Le opzioni descritte di seguito servono quindi soprattutto a deviare consapevolmente da questi default o a renderli espliciti.

## include & exclude

`include` è un array di pattern glob che indica quali file entrano nella compilazione. Quando viene omesso, il compilatore include in modo implicito tutti i file `.ts`, `.tsx` (e, se abilitato, `.js`) presenti nella cartella del progetto. Il pattern più diffuso raccoglie ricorsivamente tutto ciò che si trova sotto `src`.

`exclude` è anch'esso un array di pattern e rimuove dei file dall'insieme calcolato da `include`; non agisce sui file richiesti esplicitamente da `files`. Se viene omesso, vengono comunque scartati `node_modules`, la cartella indicata in `outDir` e i file di dichiarazione `.d.ts` generati. È buona pratica escludere sempre almeno `node_modules` e la directory di output.

```json
{
  "include": ["src/**/*", "tests/**/*.spec.ts"],
  "exclude": ["node_modules", "dist", "src/**/*.legacy.ts"]
}
```

## files

`files` elenca singoli file in modo puntuale, indicandone il percorso relativo o assoluto. A differenza di `include`, non accetta pattern glob: ogni file va nominato esplicitamente. Risulta comodo nei progetti molto piccoli o quando si vuole controllare in modo chirurgico l'ordine e l'insieme dei file compilati. Un file già presente in `files` non ha bisogno di essere ripetuto in `include`.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "esnext"
  },
  "files": ["src/main.ts", "src/utils.ts"]
}
```

## target

`target` specifica la versione di ECMAScript del JavaScript generato. Il compilatore traduce le sintassi più recenti negli equivalenti supportati dalla versione richiesta: scegliendo un target più basso, costrutti moderni vengono trasformati in forme più verbose ma compatibili con ambienti datati. La scelta va fatta in base agli ambienti di esecuzione di destinazione.

Su TypeScript 6.0 il valore predefinito è allineato all'anno corrente, attualmente `es2025`. Per i progetti nuovi si utilizzano valori moderni come `es2022`, `es2025` o `esnext`. Il valore `es5` è considerato legacy e va evitato.

```json
{
  "compilerOptions": {
    "target": "es2022"
  }
}
```

Con un target moderno, sintassi come l'operatore di optional chaining o i campi privati delle classi vengono emessi così come sono, senza polyfill o trasformazioni superflue.

```ts
class Contatore {
  #valore = 0;

  incrementa(): void {
    this.#valore++;
  }

  get valore(): number {
    return this.#valore;
  }
}

const c = new Contatore();
c.incrementa();
console.log(c.valore); // 1
```

## lib

`lib` indica quali librerie di dichiarazione descrivono l'ambiente di runtime disponibile. Queste librerie definiscono i tipi delle API a disposizione: le API del DOM come `document` e `window`, oppure le API di ECMAScript come `Array.prototype.flat` o `Promise`. Quando l'opzione non viene specificata, l'insieme delle librerie viene dedotto automaticamente dal `target`.

Specificare `lib` esplicitamente è utile, ad esempio, in un progetto che gira solo lato server e non deve accedere al DOM, oppure quando si vuole un target di sintassi diverso dal livello di API tipizzate.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "lib": ["es2022", "dom", "dom.iterable"]
  }
}
```

Per un servizio Node senza accesso al browser è sufficiente includere solo le librerie ECMAScript.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "lib": ["es2022"]
  }
}
```

## allowJs & checkJs

`allowJs` consente di includere nella compilazione anche file JavaScript accanto a quelli TypeScript, scenario tipico delle migrazioni graduali da un codebase JS esistente. `checkJs` attiva il type checking anche all'interno dei file `.js`, basandosi sull'inferenza e sui commenti JSDoc; ha effetto solo quando `allowJs` è a sua volta abilitato. Entrambe le opzioni sono disattivate di default.

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "outDir": "dist"
  },
  "include": ["src/**/*"]
}
```

Con `checkJs` attivo, anche un file JavaScript annotato con JSDoc viene verificato dal compilatore.

```ts
// file legacy.js con checkJs attivo
/** @param {number} a @param {number} b */
function somma(a, b) {
  return a + b;
}

somma(2, "3"); // Errore: Argument of type 'string' is not assignable to parameter of type 'number'.
```

## jsx

`jsx` controlla come viene trattato il codice JSX presente nei file `.tsx`. Il valore `preserve` lascia il JSX intatto nell'output, demandando la trasformazione a uno strumento esterno; `react` lo converte nelle chiamate classiche a `React.createElement`; `react-native` lo conserva ma con estensione `.js`. Le modalità `react-jsx` e `react-jsxdev` trasformano il JSX in chiamate a `jsx` e `jsxs` importate automaticamente da `react/jsx-runtime`, eliminando la necessità di avere `React` in scope; `react-jsxdev` aggiunge informazioni utili al debug ed è pensato per le build di sviluppo.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "lib": ["es2022", "dom"]
  }
}
```

Con `react-jsx` non serve alcun import esplicito di `React` nel componente.

```tsx
type Props = {
  nome: string;
};

function Saluto({ nome }: Props) {
  return <h1>Ciao, {nome}</h1>;
}

export default Saluto;
```

## declaration & declarationMap

`declaration` istruisce il compilatore a generare, accanto al JavaScript, i file di dichiarazione `.d.ts` che descrivono i tipi pubblici del codice. Questi file sono indispensabili per distribuire una libreria, poiché permettono ai consumatori di beneficiare del type checking senza avere accesso ai sorgenti TypeScript. `declarationMap` genera inoltre delle source map per i file di dichiarazione, in modo che gli editor possano risalire dalla definizione `.d.ts` al sorgente originale; ha effetto solo se `declaration` è attivo. Entrambe sono disattivate di default.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "declaration": true,
    "declarationMap": true,
    "outDir": "dist"
  },
  "include": ["src/**/*"]
}
```

## sourceMap

`sourceMap` genera i file `.js.map` che collegano ogni porzione del JavaScript emesso alla riga corrispondente del sorgente TypeScript. Questi file consentono ai debugger del browser o di Node.js di mostrare e mettere in pausa il codice TypeScript originale anziché l'output compilato, rendendo il debug molto più immediato. L'opzione è disattivata di default.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "sourceMap": true,
    "outDir": "dist"
  }
}
```

## rootDir & outDir

`rootDir` indica la directory radice dei sorgenti e determina come la struttura delle cartelle viene riprodotta nell'output. In TypeScript 6.0 il valore predefinito è la cartella che contiene il `tsconfig.json`. `outDir` indica invece la directory in cui viene scritto il JavaScript compilato; in sua assenza l'output viene generato accanto ai sorgenti. La combinazione tipica fa partire i file da `src` e li scrive in `dist`, mantenendo la gerarchia interna.

```json
{
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "target": "es2022",
    "module": "nodenext"
  },
  "include": ["src/**/*"]
}
```

Con questa configurazione, un file `src/servizi/auth.ts` viene compilato in `dist/servizi/auth.js`, preservando la sottocartella `servizi`.

## removeComments

`removeComments` elimina tutti i commenti dal JavaScript generato. Risulta utile per ridurre la dimensione dell'output destinato alla produzione. L'opzione è disattivata di default.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "esnext",
    "removeComments": true,
    "outDir": "dist"
  }
}
```

## downlevelIteration

`downlevelIteration` fa emettere al compilatore del codice aggiuntivo per far funzionare correttamente le iterazioni — come `for...of` su `Map`, `Set` o generatori — quando si compila verso target ECMAScript molto vecchi privi di iteratori nativi. Si tratta di un'opzione legacy, legata a `target` ormai datati: con i target moderni gli iteratori sono supportati nativamente e l'opzione non serve. Su TypeScript 6.0 è considerata deprecata e non va utilizzata nei progetti nuovi.

## noEmitOnError

`noEmitOnError` impedisce al compilatore di produrre qualsiasi file JavaScript quando la compilazione rileva degli errori di tipo. Senza questa opzione, TypeScript per impostazione predefinita emette comunque l'output nonostante gli errori, sulla base del principio che il JavaScript prodotto potrebbe comunque essere eseguibile. Attivarla è consigliato nelle pipeline di build, dove un errore di tipo deve interrompere il processo. L'opzione è disattivata di default.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "noEmitOnError": true,
    "outDir": "dist"
  }
}
```

## strict

`strict` è un'opzione ombrello che abilita in blocco l'intera famiglia dei controlli rigorosi: `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables` e `alwaysStrict`. In TypeScript 6.0 questa modalità è attiva di default, perché rappresenta la base per scrivere codice solido e privo di insidie legate ai valori nulli o ai tipi impliciti. Ciascun controllo può comunque essere riattivato o disattivato singolarmente, ponendo l'opzione specifica dopo `strict`.

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": false
  }
}
```

Con `strictNullChecks` in vigore, l'accesso a un valore potenzialmente assente viene segnalato dal compilatore.

```ts
function lunghezza(testo: string | undefined): number {
  return testo.length; // Errore: 'testo' is possibly 'undefined'.
}

function lunghezzaSicura(testo: string | undefined): number {
  return testo?.length ?? 0; // corretto: il caso undefined viene gestito
}
```

## Additional checks

Oltre alla famiglia `strict`, esistono controlli supplementari che intercettano errori di disattenzione frequenti. `noUnusedLocals` segnala le variabili locali dichiarate ma mai utilizzate; `noUnusedParameters` fa lo stesso per i parametri di funzione non usati; `noImplicitReturns` impone che tutti i percorsi di una funzione restituiscano un valore, quando almeno uno lo fa; `noFallthroughCasesInSwitch` impedisce che un `case` non vuoto "scivoli" nel successivo per mancanza di un `break` o di un `return`.

```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

Con `noImplicitReturns` attivo, una funzione che dimentica di restituire un valore in un ramo viene rifiutata.

```ts
function segno(n: number): string {
  if (n > 0) {
    return "positivo";
  } else if (n < 0) {
    return "negativo";
  }
  // Errore: Not all code paths return a value.
}
```

## module

`module` determina il sistema di moduli usato per il codice generato, ovvero la forma in cui vengono emessi gli `import` e gli `export`. In TypeScript 6.0 il valore predefinito è `esnext`. Per i progetti nuovi i valori consigliati sono `esnext` quando l'output viene gestito da un bundler, `nodenext` per applicazioni Node che combinano moduli ES e CommonJS seguendo le regole di risoluzione di Node, `commonjs` quando serve esplicitamente l'output CommonJS, e `preserve` quando si desidera che il compilatore lasci la sintassi dei moduli il più possibile inalterata. I valori storici `amd`, `umd`, `system` e `none` sono deprecati e non vanno scelti per codice nuovo.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext"
  }
}
```

Le dichiarazioni di modulo ES vengono compilate in coerenza con il sistema scelto.

```ts
// file matematica.ts
export function quadrato(n: number): number {
  return n * n;
}

// file main.ts
import { quadrato } from "./matematica.js";

console.log(quadrato(9)); // 81
```

## moduleResolution

`moduleResolution` definisce la strategia con cui il compilatore localizza i file a partire dagli specificatori usati negli `import`. La scelta va coordinata con l'opzione `module`. I valori moderni sono `nodenext` (e `node16`), che applicano le regole di risoluzione di Node basate sul campo `exports` e sul tipo di modulo dichiarato in `package.json`, e `bundler`, pensato per ambienti gestiti da strumenti come Vite, esbuild o webpack, dove la risoluzione finale è delegata al bundler. Per le applicazioni Node si consiglia `nodenext`; per i progetti basati su un bundler si consiglia `bundler`.

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["es2022", "dom"]
  }
}
```

Con la strategia `bundler` gli import possono omettere l'estensione del file, lasciando che sia il bundler a risolverla.

```ts
// con moduleResolution "bundler" l'estensione non è obbligatoria
import { quadrato } from "./matematica";

console.log(quadrato(4)); // 16
```

## Domande

<details>
<summary>Quale impostazione di moduleResolution corrisponde al vecchio comportamento "classic" in TypeScript 6.0?</summary>

Nessuna. La strategia `classic` è stata rimossa e non è più un valore valido per `moduleResolution`. Anche l'alias `node` (cioè `node10`) è ormai deprecato. Le strategie da utilizzare sono `nodenext` (o `node16`) per le applicazioni Node e `bundler` per i progetti gestiti da un bundler come Vite, esbuild o webpack.

</details>

<details>
<summary>Perché in molti progetti su TypeScript 6.0 il tsconfig.json non riporta esplicitamente "strict": true?</summary>

Perché in TypeScript 6.0 la modalità rigorosa è attiva di default. L'opzione `strict` abilita in blocco controlli come `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes` e `strictPropertyInitialization`. Indicarla esplicitamente serve solo a renderne evidente l'attivazione, oppure a riattivarla quando si desidera disattivare uno dei controlli individuali ponendolo dopo `strict`.

</details>

<details>
<summary>Qual è la differenza tra `files` e `include` nel selezionare i file del progetto?</summary>

`include` accetta pattern glob e seleziona insiemi di file in modo ricorsivo, ad esempio `"src/**/*"`. `files` elenca invece singoli percorsi in modo puntuale e non supporta i pattern: ogni file deve essere nominato esplicitamente. Un file già presente in `files` non ha bisogno di comparire anche in `include`.

</details>

<details>
<summary>Cosa accade di default se la compilazione presenta errori di tipo, e come si modifica questo comportamento?</summary>

Per impostazione predefinita TypeScript emette comunque i file JavaScript anche in presenza di errori di tipo, partendo dal presupposto che l'output potrebbe essere eseguibile. Attivando `noEmitOnError` si impedisce la generazione di qualsiasi output quando esiste almeno un errore, comportamento consigliato nelle pipeline di build affinché un errore di tipo interrompa il processo.

</details>

<details>
<summary>Perché l'opzione `downlevelIteration` non è più necessaria nei progetti nuovi?</summary>

Perché serviva a far funzionare le iterazioni come `for...of` su `Map`, `Set` o generatori quando si compilava verso target ECMAScript molto vecchi, privi di iteratori nativi. Con i target moderni gli iteratori sono supportati direttamente, quindi l'opzione non aggiunge nulla. Su TypeScript 6.0 è considerata legacy e deprecata, così come i target più datati a cui era legata.

</details>
