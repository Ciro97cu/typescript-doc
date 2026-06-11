# Configurare un progetto

Avviare un progetto TypeScript da zero richiede pochi passaggi, ma la scelta degli strumenti incide sulla qualità del flusso di lavoro quotidiano. L'obiettivo è ottenere un ambiente in cui il codice viene controllato dai tipi durante lo sviluppo, compilato in JavaScript valido e ricaricato automaticamente a ogni modifica, senza dipendere da pacchetti datati o configurazioni manuali fragili.

## Inizializzare il package.json

Il primo passo consiste nel creare un file `package.json`, che descrive il progetto e tiene traccia delle dipendenze. Si esegue `npm init` nella cartella del progetto; con il flag `-y` vengono accettati tutti i valori predefiniti, generando immediatamente un file pronto all'uso.

```bash
npm init -y
```

Il risultato è un `package.json` minimale, che verrà arricchito man mano con dipendenze e script.

## Installare TypeScript

TypeScript viene installato come dipendenza di sviluppo, perché serve soltanto in fase di scrittura e compilazione, non a runtime. Si utilizza il flag `--save-dev` (abbreviabile in `-D`) in modo che il pacchetto venga registrato fra le `devDependencies`.

```bash
npm install --save-dev typescript
```

Installando TypeScript localmente, anziché in modo globale, ogni progetto resta vincolato alla propria versione del compilatore, evitando incompatibilità fra progetti diversi sulla stessa macchina. Il compilatore `tsc` diventa così accessibile tramite `npx`, che esegue il binario presente in `node_modules`.

## Generare il tsconfig.json

La configurazione del compilatore risiede nel file `tsconfig.json`. Lo si genera con il comando `init` di `tsc`, eseguito attraverso `npx`:

```bash
npx tsc --init
```

Il file prodotto adotta impostazioni moderne già sicure per default. In TypeScript 6.0 la modalità `strict` è attiva senza interventi manuali, il `module` predefinito è `"esnext"` e il `target` è allineato all'anno corrente. Una configurazione tipica per un progetto destinato a un bundler come Vite si presenta nella forma seguente.

```json
{
  "compilerOptions": {
    "target": "es2025",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["es2025", "dom"],
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "noEmitOnError": true
  },
  "include": ["src/**/*"]
}
```

Per un progetto eseguito direttamente su Node.js, invece, conviene allineare risoluzione dei moduli e tipi all'ambiente di esecuzione, indicando esplicitamente i tipi di Node tramite `"types": ["node"]`, dato che in 6.0 i pacchetti in `node_modules/@types` non vengono più inclusi automaticamente.

```json
{
  "compilerOptions": {
    "target": "es2025",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "types": ["node"],
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true
  },
  "include": ["src/**/*"]
}
```

## Compilare in watch mode

Durante lo sviluppo è scomodo invocare il compilatore manualmente a ogni modifica. L'opzione `--watch` mantiene `tsc` in esecuzione e ricompila automaticamente i file appena cambiano, fornendo un riscontro immediato sugli errori di tipo.

```bash
npx tsc --watch
```

Conviene registrare i comandi ricorrenti come script nel `package.json`, in modo da invocarli con nomi brevi e condivisi nel team. Un esempio tipico distingue una build singola da una build continua.

```json
{
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch"
  }
}
```

A questo punto `npm run build` produce l'output una sola volta, mentre `npm run build:watch` avvia la compilazione incrementale.

## Server di sviluppo e runtime

Per vedere il codice in esecuzione servono strumenti che vadano oltre la semplice compilazione. Per un'applicazione che gira nel browser, la scelta moderna è un bundler con dev server integrato come Vite, che offre ricaricamento istantaneo e gestione nativa di TypeScript. Si crea un nuovo progetto con il comando di scaffolding ufficiale, scegliendo il template desiderato durante la procedura interattiva.

```bash
npm create vite@latest
```

Una volta generato il progetto, gli script tipici avviano il dev server e producono la build di produzione. Vite si occupa internamente della trasformazione di TypeScript, mentre il controllo dei tipi resta affidato a `tsc` eseguito separatamente, così da non rallentare il dev server.

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

Quando invece l'obiettivo è eseguire uno script TypeScript lato server, senza configurare un bundler, lo strumento adatto è tsx, che esegue un file direttamente, applicando al volo la trasformazione necessaria.

```bash
npx tsx src/index.ts
```

tsx supporta anche una modalità di esecuzione continua, che riavvia il processo a ogni salvataggio, comoda per lo sviluppo di script e servizi.

```bash
npx tsx watch src/index.ts
```

Un possibile file di partenza, type-correct su TypeScript 6.0, mostra come i tipi vengano applicati anche al codice eseguito tramite tsx.

```ts
// src/index.ts
interface Saluto {
  nome: string;
  lingua: "it" | "en";
}

function componiSaluto(s: Saluto): string {
  // Narrowing sul literal type per scegliere il messaggio
  return s.lingua === "it" ? `Ciao, ${s.nome}` : `Hello, ${s.nome}`;
}

const messaggio = componiSaluto({ nome: "Ada", lingua: "it" });
console.log(messaggio);
```

Combinando questi strumenti si ottiene un flusso completo: `npx tsc --watch` o `tsc --noEmit` per il controllo dei tipi, Vite per le applicazioni nel browser e tsx per gli script di runtime, lasciando da parte i server di sviluppo datati a favore di soluzioni più rapide e meglio integrate con l'ecosistema attuale.

## Domande

<details>
<summary>Perché TypeScript viene installato con --save-dev anziché come dipendenza di produzione?</summary>

Perché il compilatore serve soltanto in fase di scrittura e di build, non quando l'applicazione gira. Il codice distribuito è il JavaScript già generato, quindi TypeScript non deve far parte delle dipendenze di runtime. Registrarlo fra le `devDependencies` con `--save-dev` lo rende disponibile localmente al progetto, vincolandolo a una versione specifica del compilatore ed evitando conflitti con altri progetti sulla stessa macchina.

</details>

<details>
<summary>Quale valore di moduleResolution conviene scegliere per un progetto basato su Vite e quale per uno eseguito su Node.js?</summary>

Per un progetto servito da un bundler come Vite si usa `"bundler"`, che riflette il modo in cui questi strumenti risolvono gli import. Per un'applicazione eseguita direttamente su Node.js si usa `"nodenext"`, allineata alle regole di risoluzione dei moduli di Node. L'opzione `classic` non esiste più in TypeScript 6.0 e non va presa in considerazione.

</details>

<details>
<summary>A cosa serve l'opzione --watch del compilatore e in cosa differisce dalla modalità watch di tsx?</summary>

`tsc --watch` mantiene il compilatore in esecuzione e rigenera il JavaScript a ogni modifica dei file sorgente, fornendo un riscontro continuo sugli errori di tipo. La modalità `tsx watch` riavvia invece l'esecuzione dello script a ogni salvataggio: non produce file compilati su disco, ma rilancia direttamente il programma. Il primo si occupa di build e type checking, il secondo del ciclo di esecuzione durante lo sviluppo.

</details>

<details>
<summary>Perché in un progetto Node.js può essere necessario aggiungere "types": ["node"] al tsconfig.json?</summary>

Perché in TypeScript 6.0 l'opzione `types` ha come default un array vuoto, e i pacchetti presenti in `node_modules/@types` non vengono più inclusi automaticamente. Per disporre delle dichiarazioni globali di Node, come `process` o i moduli del core, occorre quindi elencarli esplicitamente con `"types": ["node"]`.

</details>

<details>
<summary>Nello script "build": "tsc && vite build", qual è il ruolo di ciascun comando?</summary>

`tsc` esegue il controllo dei tipi sull'intero progetto e blocca il processo se trova errori, grazie all'operatore `&&` che impedisce il proseguimento in caso di esito negativo. Solo a controllo superato `vite build` produce il bundle di produzione. In questo modo la trasformazione veloce di Vite resta separata dalla verifica dei tipi, che rimane affidata al compilatore ufficiale.

</details>
