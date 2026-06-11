# Optional chaining e nullish coalescing

Quando si lavora con strutture dati provenienti da API, configurazioni o input dell'utente, capita spesso che alcune proprietà intermedie possano mancare o che un valore possa essere assente. TypeScript, in coppia con la sintassi moderna di JavaScript, mette a disposizione due operatori pensati esattamente per questi casi: l'optional chaining (`?.`) per l'accesso sicuro alle proprietà annidate e il nullish coalescing (`??`) per fornire un valore di ripiego soltanto quando l'operando è effettivamente assente.

## Optional chaining (`?.`)

L'operatore di optional chaining permette di accedere a proprietà annidate, chiamare metodi o indicizzare array senza il rischio che una proprietà intermedia `null` o `undefined` faccia lanciare un'eccezione a runtime. Quando l'operando immediatamente a sinistra di `?.` è `null` oppure `undefined`, l'intera espressione si interrompe (short-circuit) e restituisce `undefined`; in caso contrario l'accesso prosegue normalmente.

Si consideri una struttura in cui alcuni livelli sono opzionali. Senza optional chaining sarebbe necessario verificare manualmente ogni passaggio della catena.

```ts
interface Indirizzo {
  citta: string;
  cap?: string;
}

interface Persona {
  nome: string;
  indirizzo?: Indirizzo;
}

const persona: Persona = { nome: "Anna" };

// Accesso sicuro: indirizzo è assente, quindi l'espressione restituisce undefined
const citta: string | undefined = persona.indirizzo?.citta;

console.log(citta); // undefined
```

Il valore inferito da TypeScript per `citta` è `string | undefined`: il tipo `undefined` viene aggiunto proprio perché la catena può interrompersi. Questo riflette nel sistema dei tipi ciò che accade a runtime, costringendo a gestire il caso in cui il valore non sia presente.

L'optional chaining si applica anche alla chiamata di metodi e all'accesso tramite indice. Per i metodi si utilizza `?.()`, mentre per gli elementi di un array o per le proprietà calcolate si utilizza `?.[]`.

```ts
interface Logger {
  scrivi(messaggio: string): void;
}

interface Servizio {
  logger?: Logger;
  tag?: string[];
}

function elabora(servizio: Servizio): void {
  // Il metodo viene invocato solo se logger è presente
  servizio.logger?.scrivi("avvio");

  // Accesso sicuro al primo elemento: se tag è assente, primoTag è undefined
  const primoTag: string | undefined = servizio.tag?.[0];
  console.log(primoTag);
}

elabora({}); // nessuna chiamata al logger, nessun errore
```

È importante non confondere `?.()` con la garanzia che il valore sia una funzione: l'optional chiama il metodo solo se l'oggetto che lo contiene non è nullish. Se la proprietà esiste ma non è invocabile, l'errore resta. Va inoltre evidenziato che lo short-circuit coinvolge l'intera espressione a destra del punto in cui la catena si interrompe: appena un anello risulta nullish, le valutazioni successive non vengono eseguite affatto.

```ts
interface Configurazione {
  database?: {
    host: string;
    porta: number;
  };
}

const config: Configurazione = {};

// Se database è undefined, porta non viene nemmeno letta: il risultato è undefined
const porta: number | undefined = config.database?.porta;
```

## Nullish coalescing (`??`)

L'operatore di nullish coalescing restituisce l'operando di destra soltanto quando l'operando di sinistra è `null` o `undefined`; in tutti gli altri casi restituisce l'operando di sinistra. Serve quindi a fornire un valore di default, ma in modo più preciso rispetto al tradizionale OR logico.

```ts
function saluta(nome: string | null | undefined): string {
  // Se nome è null o undefined si usa "ospite", altrimenti il nome fornito
  const effettivo = nome ?? "ospite";
  return `Benvenuto, ${effettivo}`;
}

console.log(saluta("Marco"));    // Benvenuto, Marco
console.log(saluta(null));       // Benvenuto, ospite
console.log(saluta(undefined));  // Benvenuto, ospite
```

Il tipo del risultato di `??` esclude la parte nullish dell'operando di sinistra. Nell'esempio precedente, `nome ?? "ospite"` ha tipo `string`, perché `null` e `undefined` vengono rimpiazzati dal valore di destra.

### Differenza tra `??` e `||`

La distinzione fondamentale riguarda il trattamento dei valori falsy. L'operatore `||` restituisce l'operando di destra ogni volta che quello di sinistra è falsy, e in JavaScript sono falsy non solo `null` e `undefined`, ma anche `0`, la stringa vuota `''`, `false` e `NaN`. L'operatore `??`, invece, considera "mancante" esclusivamente `null` e `undefined`, lasciando intatti tutti gli altri valori. Questa differenza è cruciale quando `0`, `''` o `false` sono valori legittimi da preservare.

```ts
interface Impostazioni {
  volume?: number;
  etichetta?: string;
  attivo?: boolean;
}

function applica(impostazioni: Impostazioni) {
  // Con ??: solo undefined viene sostituito, lo zero viene mantenuto
  const volume = impostazioni.volume ?? 50;

  // Con ||: lo zero è falsy, quindi verrebbe erroneamente sostituito con 50
  const volumeErrato = impostazioni.volume || 50;

  return { volume, volumeErrato };
}

console.log(applica({ volume: 0 }));
// { volume: 0, volumeErrato: 50 }
```

Nell'esempio, un utente che imposta deliberatamente il volume a `0` vede il proprio valore conservato solo grazie a `??`. Con `||` lo zero verrebbe scartato come se fosse assente, introducendo un bug sottile. Lo stesso ragionamento vale per la stringa vuota e per `false`.

```ts
const etichetta: string | undefined = "";

// '' è falsy: con || si ottiene il default
console.log(etichetta || "predefinita"); // "predefinita"

// '' non è nullish: con ?? si conserva la stringa vuota
console.log(etichetta ?? "predefinita"); // ""
```

### Combinare i due operatori

Optional chaining e nullish coalescing si completano a vicenda: il primo produce `undefined` quando la catena si interrompe, il secondo intercetta quell'`undefined` per fornire un valore di ripiego. La loro combinazione consente di leggere proprietà profonde e assegnare un default in un'unica espressione concisa.

```ts
interface Profilo {
  preferenze?: {
    tema?: "chiaro" | "scuro";
  };
}

function temaCorrente(profilo: Profilo): "chiaro" | "scuro" {
  // Se la catena si interrompe, ?. restituisce undefined e ?? fornisce il default
  return profilo.preferenze?.tema ?? "chiaro";
}

console.log(temaCorrente({}));                              // "chiaro"
console.log(temaCorrente({ preferenze: { tema: "scuro" } })); // "scuro"
```

Va ricordato che per ragioni di precedenza non è consentito combinare direttamente `??` con `&&` o `||` senza parentesi: il compilatore segnala l'ambiguità e impone di rendere esplicito il raggruppamento.

```ts
const a: number | null = null;
const b = false;

// Errore: '??' and '||' cannot be mixed without parentheses
// const risultato = a ?? b || 10;

// Corretto: il raggruppamento è esplicito
const risultato = (a ?? b) || 10;
```

## Domande

<details>
<summary>Qual è il tipo inferito da TypeScript per l'espressione `persona.indirizzo?.citta`, dato che `indirizzo` è una proprietà opzionale di tipo `Indirizzo` e `citta` è `string`?</summary>

Il tipo inferito è `string | undefined`. L'optional chaining aggiunge `undefined` al tipo del risultato perché, se `indirizzo` è `null` o `undefined`, l'intera espressione interrompe la valutazione (short-circuit) e restituisce `undefined` anziché tentare di leggere `citta`.

</details>

<details>
<summary>Perché in molti scenari l'operatore `??` è preferibile a `||` per assegnare un valore di default?</summary>

Perché `||` restituisce l'operando di destra per qualsiasi valore falsy a sinistra, inclusi `0`, `''`, `false` e `NaN`, che spesso sono valori legittimi da conservare. L'operatore `??`, invece, sostituisce l'operando di sinistra solo quando è `null` o `undefined`, preservando tutti gli altri valori. Per esempio, `0 ?? 50` restituisce `0`, mentre `0 || 50` restituisce `50`.

</details>

<details>
<summary>Cosa succede a runtime quando si valuta `config.database?.porta` e `database` è `undefined`?</summary>

L'espressione subisce uno short-circuit: appena l'operando a sinistra di `?.` risulta `undefined`, la valutazione si interrompe, la proprietà `porta` non viene nemmeno letta e l'intera espressione restituisce `undefined`, senza lanciare alcuna eccezione.

</details>

<details>
<summary>Perché il codice `a ?? b || 10` viene segnalato dal compilatore, e come si risolve?</summary>

L'operatore `??` non può essere combinato direttamente con `||` (o `&&`) nella stessa espressione senza parentesi, perché la precedenza risulterebbe ambigua. Il compilatore emette un errore del tipo `'??' and '||' cannot be mixed without parentheses`. La soluzione consiste nell'esplicitare il raggruppamento con le parentesi, per esempio `(a ?? b) || 10` oppure `a ?? (b || 10)`, a seconda della semantica desiderata.

</details>

<details>
<summary>Qual è la differenza tra `servizio.logger?.scrivi("avvio")` e una garanzia che `scrivi` sia effettivamente chiamabile?</summary>

L'optional chaining su `?.` controlla soltanto che l'oggetto a sinistra (`logger`) non sia `null` o `undefined`; solo in tal caso procede con l'invocazione del metodo. Non verifica in alcun modo che `scrivi` esista e sia una funzione: se `logger` è presente ma `scrivi` non è invocabile, l'errore a runtime si verifica comunque. L'optional protegge contro l'assenza dell'oggetto contenitore, non contro un membro presente ma non valido.

</details>
