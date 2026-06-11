# Narrowing e type guards

Quando una variabile ha un tipo ampio, come un union type oppure `unknown`, il compilatore non può sapere in anticipo quale forma concreta assuma quel valore in un determinato punto del programma. Il narrowing è il processo con cui TypeScript restringe progressivamente quel tipo a qualcosa di più specifico, basandosi sul flusso del controllo (control flow analysis). Lo strumento che innesca il narrowing è la type guard: un'espressione che esegue un controllo a runtime il cui esito permette al compilatore di trattare la variabile come un tipo più preciso all'interno del ramo corrispondente.

Il principio è importante perché su un union type si possono raggiungere direttamente solo le proprietà e i metodi comuni a tutti i membri dell'union. Per accedere ai membri specifici di una variante è necessario prima dimostrare al compilatore, tramite un controllo, di trovarsi effettivamente di fronte a quella variante. Il narrowing fa esattamente questo: dopo la type guard, dentro il blocco condizionato, il tipo risulta ristretto e l'accesso ai membri specifici diventa lecito e type-safe.

## Type guards

Esistono tre forme di type guard integrate nel linguaggio, ciascuna adatta a una categoria diversa di valori: `typeof`, `in` e `instanceof`. Tutte e tre operano allo stesso modo dal punto di vista del compilatore, ossia restringono il tipo della variabile all'interno del ramo in cui il controllo risulta vero (e, simmetricamente, nel ramo `else` rimuovono quella possibilità dall'union).

### typeof

L'operatore `typeof` restituisce a runtime una stringa che descrive il tipo primitivo di un valore (`"string"`, `"number"`, `"boolean"`, `"bigint"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`). È quindi la type guard naturale quando l'union è composto da tipi primitivi e occorre distinguerli per applicare logiche differenti.

```ts
function formatta(valore: string | number): string {
  if (typeof valore === "string") {
    // Qui il tipo è ristretto a string: i metodi di string sono accessibili
    return valore.trim().toUpperCase();
  }

  // Nel ramo rimanente il tipo è ristretto a number
  return valore.toFixed(2);
}

console.log(formatta("  ciao  ")); // "CIAO"
console.log(formatta(3.14159));    // "3.14"
```

Va ricordato che `typeof null` restituisce `"object"` per una nota peculiarità storica di JavaScript: per isolare `null` non si usa `typeof`, ma un confronto diretto con `null` o un controllo di veridicità del valore.

### in

L'operatore `in` verifica la presenza di una proprietà in un oggetto. È particolarmente utile quando un union è formato da forme di oggetti diverse che non condividono una gerarchia di classi, e che pertanto non possono essere distinte con `instanceof`. La presenza o l'assenza di una proprietà caratteristica diventa il criterio di discriminazione.

```ts
interface Cane {
  abbaia: () => void;
}

interface Gatto {
  miagola: () => void;
}

function emettiVerso(animale: Cane | Gatto): void {
  if ("abbaia" in animale) {
    // Il tipo è ristretto a Cane
    animale.abbaia();
  } else {
    // Il tipo è ristretto a Gatto
    animale.miagola();
  }
}
```

### instanceof

L'operatore `instanceof` verifica se un oggetto è stato creato a partire da una determinata classe (o da una sua sottoclasse), controllando la catena dei prototipi. È quindi la type guard appropriata quando si lavora con istanze di classi.

```ts
class Cerchio {
  constructor(public raggio: number) {}
}

class Rettangolo {
  constructor(public base: number, public altezza: number) {}
}

function area(figura: Cerchio | Rettangolo): number {
  if (figura instanceof Cerchio) {
    // Il tipo è ristretto a Cerchio
    return Math.PI * figura.raggio ** 2;
  }

  // Il tipo è ristretto a Rettangolo
  return figura.base * figura.altezza;
}
```

Poiché `instanceof` si appoggia ai prototipi, funziona solo con valori costruiti da classi o funzioni costruttore. Per i tipi rappresentati da interfacce o type alias, che non hanno una controparte a runtime, occorre ricorrere a `typeof`, a `in` oppure alle discriminated union.

## Discriminated union

Una discriminated union (chiamata anche tagged union) è un union di tipi oggetto che condividono una proprietà comune, detta proprietà discriminante, il cui valore è un literal type diverso per ciascuna variante. Quella proprietà funge da etichetta: confrontandone il valore, il compilatore è in grado di restringere automaticamente l'intero oggetto alla variante corrispondente. È il modo idiomatico in TypeScript per modellare un valore che può assumere un numero finito di forme alternative.

Il pattern si combina in modo naturale con un costrutto `switch` sulla proprietà discriminante: ogni `case` corrisponde a una variante e, al suo interno, il tipo è già ristretto a quella forma specifica.

```ts
interface Quadrato {
  tipo: "quadrato";
  lato: number;
}

interface Cerchio {
  tipo: "cerchio";
  raggio: number;
}

interface Rettangolo {
  tipo: "rettangolo";
  base: number;
  altezza: number;
}

type Figura = Quadrato | Cerchio | Rettangolo;

function calcolaArea(figura: Figura): number {
  switch (figura.tipo) {
    case "quadrato":
      // Ristretto a Quadrato
      return figura.lato ** 2;
    case "cerchio":
      // Ristretto a Cerchio
      return Math.PI * figura.raggio ** 2;
    case "rettangolo":
      // Ristretto a Rettangolo
      return figura.base * figura.altezza;
  }
}
```

Un vantaggio notevole di questo pattern emerge in combinazione con il tipo `never` per ottenere un controllo di esaustività (exhaustiveness check). Assegnando il valore residuo a una variabile di tipo `never` nel ramo di default, il compilatore segnala un errore qualora venisse aggiunta una nuova variante all'union senza gestirla, trasformando una svista in un errore individuato in fase di compilazione.

```ts
function descriviFigura(figura: Figura): string {
  switch (figura.tipo) {
    case "quadrato":
      return `Quadrato di lato ${figura.lato}`;
    case "cerchio":
      return `Cerchio di raggio ${figura.raggio}`;
    case "rettangolo":
      return `Rettangolo ${figura.base}x${figura.altezza}`;
    default:
      // Se l'union viene esteso senza aggiungere un case,
      // qui si ottiene: Type 'Figura' is not assignable to type 'never'.
      const esaustivo: never = figura;
      return esaustivo;
  }
}
```

## Type predicates per narrowing personalizzato

Le type guard integrate coprono i casi più comuni, ma talvolta la logica che determina il tipo di un valore è troppo articolata per essere espressa con `typeof`, `in` o `instanceof`, oppure va riutilizzata in più punti. In queste situazioni si definisce una funzione di type guard personalizzata, il cui tipo di ritorno non è `boolean` bensì un type predicate nella forma `parametro is Tipo`.

Quando una funzione del genere restituisce `true`, il compilatore non si limita a registrare un valore booleano: applica il narrowing e tratta l'argomento indicato come appartenente al tipo specificato nel predicate. In questo modo la conoscenza incapsulata nella funzione si propaga al codice chiamante.

```ts
interface Pesce {
  nuota: () => void;
}

interface Uccello {
  vola: () => void;
}

// Il tipo di ritorno `animale is Pesce` è il type predicate
function isPesce(animale: Pesce | Uccello): animale is Pesce {
  return (animale as Pesce).nuota !== undefined;
}

function muovi(animale: Pesce | Uccello): void {
  if (isPesce(animale)) {
    // Grazie al type predicate, qui il tipo è ristretto a Pesce
    animale.nuota();
  } else {
    // Nel ramo else il tipo è ristretto a Uccello
    animale.vola();
  }
}
```

I type predicate sono particolarmente preziosi nei filtri sugli array, dove permettono di trasformare il tipo dell'array risultante anziché lasciarlo invariato. Un `filter` con una funzione di type guard restituisce un array già ristretto al tipo voluto.

```ts
const animali: (Pesce | Uccello)[] = [
  { nuota: () => {} },
  { vola: () => {} },
  { nuota: () => {} },
];

// Senza il type predicate il risultato sarebbe (Pesce | Uccello)[]:
// con isPesce, invece, il tipo inferito è Pesce[]
const soloPesci: Pesce[] = animali.filter(isPesce);
```

È bene sottolineare che il type predicate è un'asserzione di cui il compilatore si fida: spetta a chi scrive la funzione garantire che il controllo a runtime corrisponda effettivamente al tipo dichiarato. Un predicate scritto in modo scorretto non produce errori di compilazione, ma sposta il problema all'esecuzione, vanificando la sicurezza che il type system intende offrire.

## Domande

<details>
<summary>Perché su un union type non è possibile accedere direttamente ai membri specifici di una sola variante senza una type guard?</summary>

Perché in un punto qualsiasi del programma il compilatore considera il valore come potenzialmente appartenente a uno qualunque dei membri dell'union. Sono perciò garantiti come accessibili solo i membri comuni a tutte le varianti. Per usare un membro specifico occorre prima restringere il tipo con una type guard, così da dimostrare al compilatore di trovarsi davanti a quella precisa variante.
</details>

<details>
<summary>Quando conviene usare `in` al posto di `instanceof`?</summary>

L'operatore `instanceof` funziona solo con valori costruiti da classi o funzioni costruttore, perché si basa sulla catena dei prototipi. Quando l'union è formato da interfacce o type alias, che non hanno una controparte a runtime, `instanceof` non è applicabile: si ricorre allora a `in`, distinguendo le varianti in base alla presenza di una proprietà caratteristica.
</details>

<details>
<summary>Che cosa caratterizza la proprietà discriminante di una discriminated union?</summary>

È una proprietà presente in tutte le varianti dell'union il cui tipo è un literal type distinto per ciascuna variante (ad esempio `tipo: "cerchio"` contro `tipo: "quadrato"`). Confrontandone il valore, tipicamente in uno `switch`, il compilatore restringe automaticamente l'intero oggetto alla variante corrispondente.
</details>

<details>
<summary>A cosa serve assegnare il valore residuo a una variabile di tipo `never` nel ramo di default di uno `switch` su una discriminated union?</summary>

Serve a ottenere un controllo di esaustività. Se tutte le varianti sono gestite, nel default il tipo residuo è `never` e l'assegnazione è lecita. Se in seguito si aggiunge una nuova variante all'union senza un `case` dedicato, quel valore non è più assegnabile a `never` e il compilatore segnala un errore, individuando la mancanza in fase di compilazione.

```ts
const esaustivo: never = figura; // errore se manca un case
```
</details>

<details>
<summary>In che cosa si distingue una funzione che restituisce `boolean` da una che restituisce un type predicate `arg is Tipo`?</summary>

Una funzione che restituisce `boolean` comunica soltanto l'esito vero o falso del controllo, senza influire sul tipo della variabile nel chiamante. Una funzione con type predicate `arg is Tipo`, invece, quando restituisce `true` induce il compilatore a restringere l'argomento al tipo dichiarato nei rami successivi. Il narrowing diventa così riutilizzabile, ma la correttezza del controllo a runtime resta responsabilità di chi scrive la funzione, dato che il compilatore si fida del predicate.
</details>
