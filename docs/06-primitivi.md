# Tipi primitivi

I tipi primitivi rappresentano i valori più elementari del linguaggio: non sono oggetti e non possiedono metodi propri (gli eventuali metodi sono forniti dai wrapper object di JavaScript). In TypeScript ogni primitivo ha un tipo dedicato, scritto rigorosamente in minuscolo: `boolean`, `number`, `bigint`, `string`, oltre a `symbol`, `null` e `undefined`. È importante non confonderli con i tipi `Boolean`, `Number` o `String` con l'iniziale maiuscola, che corrispondono ai wrapper object e che non vanno mai usati per annotare valori primitivi.

In questa pagina vengono trattati i tre primitivi più frequenti nel codice quotidiano: `boolean`, i tipi numerici (`number` e `bigint`) e `string`.

## Boolean

Il tipo `boolean` rappresenta i due soli valori logici `true` e `false`. È il tipo naturale per esprimere condizioni e per pilotare il controllo di flusso, ad esempio nelle istruzioni `if`, nei cicli `while` o negli operatori ternari.

Quando una variabile viene inizializzata con un letterale booleano, il tipo viene dedotto automaticamente tramite la type inference, quindi raramente è necessario annotarlo in modo esplicito. L'annotazione esplicita resta comunque utile sui parametri di funzione e sui valori di ritorno, dove l'inferenza da sola non basta.

```ts
let attivo: boolean = true;

// Tipo dedotto automaticamente come boolean grazie alla type inference
let inSpedizione = false;

function eMaggiorenne(eta: number): boolean {
  return eta >= 18;
}

if (eMaggiorenne(20)) {
  attivo = true;
}
```

Il sistema dei tipi è rigoroso: a una variabile `boolean` si possono assegnare solo `true` o `false`, mai un valore di un altro tipo, nemmeno se in JavaScript risulterebbe truthy o falsy.

```ts
let confermato: boolean;

confermato = true; // valido
// Errore: Type 'number' is not assignable to type 'boolean'.
confermato = 1;
```

Per ottenere un `boolean` a partire da un valore di altro tipo si ricorre a una conversione esplicita, ad esempio con la funzione `Boolean(...)` oppure con la doppia negazione `!!valore`. In questo modo l'intento di conversione resta evidente e il tipo risultante è correttamente `boolean`.

```ts
const testo = "ciao";

// Entrambe le espressioni producono un valore di tipo boolean
const haContenuto: boolean = Boolean(testo);
const haContenutoAbbreviato: boolean = !!testo;
```

## Number

In JavaScript, e di conseguenza in TypeScript, tutti i numeri tradizionali sono rappresentati come valori a virgola mobile a doppia precisione. Per questo motivo non esistono tipi separati per interi e decimali: il tipo `number` copre entrambi i casi, insieme a valori speciali come `Infinity`, `-Infinity` e `NaN`.

```ts
let intero: number = 42;
let decimale: number = 3.14;
let negativo = -7;          // tipo number dedotto automaticamente
let nonUnNumero: number = NaN;
```

Per migliorare la leggibilità dei letterali numerici molto lunghi si possono inserire underscore come separatori di cifre, senza che ciò ne alteri il valore.

```ts
const popolazione = 1_000_000; // equivalente a 1000000
```

### Letterali esadecimali, binari e ottali

Oltre alla consueta notazione decimale, `number` accetta letterali scritti in altre basi. Si usa il prefisso `0x` per l'esadecimale, `0b` per il binario e `0o` per l'ottale. In tutti i casi il tipo della costante resta `number`: la base è solo una comodità di scrittura, perché il valore memorizzato è sempre lo stesso.

```ts
const esadecimale = 0xff;     // 255
const binario = 0b1010;       // 10
const ottale = 0o17;          // 15

// Tutte le espressioni seguenti valgono true: la base non cambia il valore
console.log(esadecimale === 255);
console.log(binario === 0b1010);
console.log(ottale === 0o17);
```

### bigint

Il tipo `number` rappresenta in modo affidabile soltanto gli interi compresi fino a `Number.MAX_SAFE_INTEGER`, pari a `9007199254740991`. Oltre questa soglia i calcoli con interi perdono precisione. Per gestire interi arbitrariamente grandi in modo esatto esiste il tipo `bigint`.

Un letterale `bigint` si ottiene aggiungendo il suffisso `n` a un numero intero. Per emettere questi letterali è sufficiente un `target` pari a `"es2020"` o superiore; con le impostazioni predefinite di TypeScript 6.0 il `target` è già allineato a un valore moderno, quindi i `bigint` funzionano senza alcuna configurazione aggiuntiva.

```ts
const grande: bigint = 9_007_199_254_740_993n;

// Anche la notazione esadecimale ammette il suffisso n
const maschera: bigint = 0xffffffffffffffffn;

// Si può convertire un number in bigint con la funzione BigInt(...)
const daNumero: bigint = BigInt(255);
```

I tipi `number` e `bigint` sono distinti e non si mescolano: non è possibile combinarli direttamente in un'operazione aritmetica senza una conversione esplicita. Anche i confronti di uguaglianza stretta tra i due tipi restituiscono sempre `false`, perché i tipi differiscono.

```ts
const a: bigint = 10n;
const b: number = 5;

// Errore: Operator '+' cannot be applied to types 'bigint' and 'number'.
const somma = a + b;

// Conversione esplicita necessaria per operare
const corretta: bigint = a + BigInt(b);
```

## String

Il tipo `string` rappresenta sequenze di caratteri, ossia il testo. I letterali stringa si possono delimitare con apici singoli, con apici doppi oppure con i backtick. Le prime due forme sono equivalenti e si scelgono in base alle convenzioni del progetto; i backtick introducono invece i template literals, una variante più potente.

```ts
let nome: string = "Ada";
let cognome = 'Lovelace'; // tipo string dedotto automaticamente
```

Quando si usano apici singoli o doppi, per inserire un'andata a capo o altri caratteri speciali occorre ricorrere alle sequenze di escape, come `\n` per il newline e `\t` per la tabulazione.

```ts
const messaggio: string = "Prima riga\nSeconda riga";
```

### Template literals

I template literals, delimitati dai backtick, semplificano notevolmente la composizione del testo. Consentono di scrivere stringhe su più righe senza sequenze di escape e, soprattutto, di incorporare espressioni tramite la sintassi `${ ... }`: ogni espressione viene valutata e il suo risultato convertito in stringa e inserito al posto del segnaposto.

```ts
const nomeUtente = "Ada";
const eta = 36;

const saluto = `Ciao ${nomeUtente}, hai ${eta} anni.`;

// Le stringhe su più righe non richiedono \n
const blocco = `Riga uno
Riga due
Riga tre`;
```

All'interno di `${ ... }` può comparire qualsiasi espressione valida, comprese chiamate di funzione e operazioni aritmetiche. TypeScript verifica le espressioni interpolate come qualunque altro codice, segnalando eventuali errori di tipo.

```ts
function prezzoConIva(netto: number): number {
  return netto * 1.22;
}

const importo = `Totale: ${prezzoConIva(100).toFixed(2)} euro`;
```

Va inoltre tenuta presente la distinzione tra il tipo `string`, che descrive una qualunque stringa, e i literal type, che vincolano il valore a una stringa specifica. Una variabile dichiarata con `const` e inizializzata con un letterale stringa assume come tipo proprio quel letterale, mentre con `let` il tipo viene allargato (widening) a `string`.

```ts
const fisso = "submit"; // tipo "submit"
let variabile = "submit"; // tipo string
```

## Domande

<details>
<summary>Qual è la differenza tra il tipo `boolean` e il tipo `Boolean` in TypeScript?</summary>

`boolean`, in minuscolo, è il tipo primitivo che ammette esclusivamente i valori `true` e `false` ed è quello che si deve usare per annotare i valori logici. `Boolean`, con l'iniziale maiuscola, è il tipo del wrapper object di JavaScript: non va usato nelle annotazioni dei valori primitivi, perché un valore di tipo `boolean` non è la stessa cosa di un'istanza dell'oggetto wrapper. La funzione `Boolean(...)`, invece, è un modo lecito e comune per convertire un valore in `boolean`.
</details>

<details>
<summary>I letterali `0xff`, `0b1010` e `0o17` hanno tipi diversi tra loro?</summary>

No. Tutti e tre sono letterali numerici scritti in basi differenti (esadecimale, binaria e ottale), ma il loro tipo è sempre `number`. Il prefisso indica soltanto la notazione usata per scrivere la costante nel sorgente, mentre il valore memorizzato e il tipo non cambiano.
</details>

<details>
<summary>Di quale `target` minimo si ha bisogno per usare i letterali `bigint` come `10n`?</summary>

È sufficiente un `target` pari a `"es2020"` o superiore. Non occorre impostare `"esnext"`. Con le impostazioni predefinite di TypeScript 6.0, in cui il `target` è già allineato a un valore moderno, i letterali `bigint` funzionano senza alcuna configurazione aggiuntiva.
</details>

<details>
<summary>Perché l'espressione `10n + 5` produce un errore di compilazione?</summary>

Perché `10n` è di tipo `bigint` mentre `5` è di tipo `number`, e i due tipi numerici non si mescolano: l'operatore `+` non può essere applicato a un `bigint` e a un `number` contemporaneamente. Per eseguire l'operazione occorre una conversione esplicita, ad esempio `10n + BigInt(5)`.
</details>

<details>
<summary>Che vantaggio offrono i template literals rispetto agli apici singoli o doppi?</summary>

I template literals, delimitati dai backtick, permettono di scrivere stringhe su più righe senza usare la sequenza di escape `\n` e di incorporare espressioni tramite la sintassi `${ ... }`, che vengono valutate e convertite in stringa al posto del segnaposto. Le espressioni interpolate sono inoltre sottoposte al type checking come qualunque altro codice.
</details>
