# Interfaces

Un'interface descrive la forma di un valore: definisce quali proprietà e quali metodi un oggetto deve possedere, senza specificarne l'implementazione. Si tratta a tutti gli effetti di un contratto. Chi dichiara di rispettare quel contratto si impegna a fornire esattamente i membri richiesti, con i tipi indicati; il compilatore verifica che l'impegno sia mantenuto e segnala ogni discrepanza in fase di compilazione, prima che il codice venga eseguito.

A differenza di una classe, un'interface non genera alcun codice JavaScript: esiste soltanto durante il type checking e scompare nell'output compilato. Il suo scopo è puramente descrittivo, e per questo viene impiegata per dichiarare le forme degli oggetti che attraversano un'applicazione, dai parametri delle funzioni ai valori di ritorno delle API.

```ts
interface Persona {
  nome: string;
  eta: number;
  saluta(): string;
}

const utente: Persona = {
  nome: "Aurora",
  eta: 30,
  saluta() {
    return `Ciao, mi chiamo ${this.nome}`;
  },
};
```

Qualsiasi oggetto assegnato a `Persona` deve esporre tutte e tre le proprietà con i tipi corretti. Se ne manca una, o se un tipo non corrisponde, il compilatore interrompe la compilazione.

```ts
const incompleto: Persona = {
  nome: "Bruno",
  eta: 42,
  // Errore: Property 'saluta' is missing in type
  // '{ nome: string; eta: number; }' but required in type 'Persona'.
};
```

## Implementare un'interface con una classe

Una classe dichiara di rispettare un contratto tramite la keyword `implements`. Il compilatore controlla allora che la classe definisca tutti i membri descritti dall'interface, con firme compatibili. L'aspetto più rilevante è che una classe può implementare più interface contemporaneamente, elencandole separate da virgola: in questo modo si compongono contratti diversi senza ricorrere all'ereditarietà.

```ts
interface Identificabile {
  id: number;
}

interface Serializzabile {
  toJSON(): string;
}

class Documento implements Identificabile, Serializzabile {
  constructor(
    public id: number,
    public titolo: string,
  ) {}

  toJSON(): string {
    return JSON.stringify({ id: this.id, titolo: this.titolo });
  }
}
```

`Documento` soddisfa sia `Identificabile` sia `Serializzabile`. Qualora uno dei membri richiesti venisse omesso, il compilatore lo segnalerebbe puntualmente.

```ts
class Bozza implements Identificabile, Serializzabile {
  id = 0;
  // Errore: Class 'Bozza' incorrectly implements interface 'Serializzabile'.
  // Property 'toJSON' is missing in type 'Bozza' but required in type 'Serializzabile'.
}
```

## Modificatori in un'interface: solo readonly

Un'interface descrive un contratto pubblico, ovvero ciò che un valore espone verso l'esterno. Per questa ragione non ammette gli access modifier delle classi: `public`, `protected` e `private` non possono comparire al suo interno. Concetti come la visibilità ristretta appartengono all'implementazione di una classe, non alla forma osservabile da chi la consuma.

L'unico modificatore consentito è `readonly`, che vincola una proprietà a non essere riassegnata dopo l'inizializzazione. Si rivela utile per valori che devono restare stabili per tutta la vita dell'oggetto, come un identificatore.

```ts
interface Configurazione {
  readonly chiave: string;
  valore: number;
}

const config: Configurazione = { chiave: "timeout", valore: 5000 };

config.valore = 8000; // consentito

config.chiave = "retry";
// Errore: Cannot assign to 'chiave' because it is a read-only property.
```

Va sottolineato che `readonly` agisce esclusivamente a livello di tipo durante la compilazione: non produce alcun controllo a runtime e non equivale al `private` di una classe, che riguarda invece l'accessibilità del membro.

## Estendere interface con extends

Un'interface può ereditare i membri di un'altra tramite `extends`, ottenendo un contratto più ampio che comprende sia i propri membri sia quelli ereditati. Qui emerge una differenza sostanziale rispetto alle classi: mentre una classe può estendere una sola classe (ereditarietà singola), un'interface può estendere più interface insieme, fondendone i contratti in uno unico.

```ts
interface Animale {
  nome: string;
  emettiVerso(): string;
}

interface Domestico {
  proprietario: string;
}

interface Cane extends Animale, Domestico {
  razza: string;
}

const fido: Cane = {
  nome: "Fido",
  emettiVerso: () => "Bau",
  proprietario: "Elena",
  razza: "Border Collie",
};
```

`Cane` raccoglie tutti i membri di `Animale`, di `Domestico` e i propri. Una classe, al contrario, non potrebbe estendere due classi: l'eredità multipla è una prerogativa delle interface, ed è proprio per questo che il meccanismo dei contratti risulta più flessibile della gerarchia di classi.

```ts
class A {}
class B {}

// Errore: Classes can only extend a single class.
// class C extends A, B {}
```

Una classe rimane comunque libera di implementare più interface tramite `implements`, recuperando per altra via la composizione di contratti che l'ereditarietà singola le nega.

## Interface come function type

Oltre alle forme di oggetto, un'interface può descrivere il tipo di una funzione attraverso una call signature: una firma priva di nome che indica i tipi dei parametri e il tipo di ritorno. Un valore compatibile con quell'interface è dunque una funzione invocabile con quella firma.

```ts
interface Comparatore {
  (a: number, b: number): number;
}

const ascendente: Comparatore = (a, b) => a - b;

const numeri = [5, 2, 8, 1];
numeri.sort(ascendente); // [1, 2, 5, 8]
```

I tipi dei parametri `a` e `b` non vanno ripetuti nell'assegnazione: il compilatore li deduce dalla call signature dichiarata in `Comparatore`. Un'interface di questo genere può inoltre combinare la call signature con proprietà aggiuntive, descrivendo una funzione che porta con sé dati accessori.

```ts
interface Contatore {
  (incremento: number): number;
  totale: number;
}

function creaContatore(): Contatore {
  const fn = ((incremento: number) => (fn.totale += incremento)) as Contatore;
  fn.totale = 0;
  return fn;
}
```

## Proprietà e metodi opzionali

Non tutti i membri di un contratto sono obbligatori. Apponendo un punto interrogativo `?` dopo il nome di una proprietà o di un metodo, si dichiara che quel membro è opzionale: può essere presente oppure assente, e nel secondo caso il suo tipo include implicitamente `undefined`.

```ts
interface Ordine {
  id: number;
  importo: number;
  note?: string;
  applicaSconto?(percentuale: number): number;
}

const ordineMinimo: Ordine = { id: 1, importo: 100 };

const ordineCompleto: Ordine = {
  id: 2,
  importo: 200,
  note: "Consegna urgente",
  applicaSconto: (percentuale) => 200 * (1 - percentuale / 100),
};
```

Poiché un membro opzionale potrebbe non esistere, ogni accesso va protetto. L'optional chaining `?.` consente di invocare un metodo opzionale in modo sicuro, restituendo `undefined` se il metodo non è stato fornito invece di sollevare un errore a runtime.

```ts
function totaleOrdine(ordine: Ordine): number {
  return ordine.applicaSconto?.(10) ?? ordine.importo;
}

totaleOrdine(ordineMinimo); // 100
totaleOrdine(ordineCompleto); // 180
```

## interface o type: quando scegliere

Per descrivere la forma di un oggetto, `interface` e `type` sono in larga misura intercambiabili e producono lo stesso risultato in termini di type checking. Le due definizioni seguenti impongono il medesimo contratto.

```ts
interface PuntoI {
  x: number;
  y: number;
}

type PuntoT = {
  x: number;
  y: number;
};
```

La differenza più significativa riguarda la riapribilità. Un'interface supporta il declaration merging: dichiarando più volte un'interface con lo stesso nome, le sue dichiarazioni si fondono e i membri si sommano. Questo comportamento è prezioso per arricchire tipi forniti da librerie esterne o per estendere dichiarazioni globali.

```ts
interface Finestra {
  larghezza: number;
}

interface Finestra {
  altezza: number;
}

// Le due dichiarazioni si fondono in un unico contratto
const f: Finestra = { larghezza: 1920, altezza: 1080 };
```

Un `type`, al contrario, non è riapribile: dichiararlo due volte con lo stesso nome produce un errore.

```ts
type Schermo = { larghezza: number };

type Schermo = { altezza: number };
// Errore: Duplicate identifier 'Schermo'.
```

In compenso un `type` è più versatile, perché può dare un nome a qualsiasi tipo, non soltanto alle forme di oggetto: union type, intersection type, tuple, primitivi rinominati e tipi calcolati sono esprimibili con `type` ma non con `interface`.

```ts
type Id = string | number;
type Coordinate = [number, number];
type Risposta = "si" | "no" | "forse";
```

Ne deriva un criterio pratico: si preferisce `interface` per descrivere la forma di oggetti e classi, soprattutto quando il contratto è destinato a essere implementato o esteso e quando si desidera sfruttare il declaration merging; si ricorre invece a `type` per union type, intersection type, tuple e per assegnare un alias a tipi che non siano semplici forme di oggetto. Le interface dichiarano comunque sempre e solo forme di oggetto o di funzione, e non possono essere usate per rinominare un primitivo.

## Domande

<details>
<summary>Perché un'interface non ammette i modificatori public, protected e private mentre consente readonly?</summary>

Un'interface descrive il contratto pubblico di un valore, ossia ciò che è osservabile dall'esterno: la visibilità ristretta (`private`, `protected`) è un dettaglio implementativo che appartiene alle classi, non alla forma esposta, quindi non avrebbe senso in un contratto. `readonly` invece esprime un vincolo sulla forma stessa, indicando che una proprietà non può essere riassegnata dopo l'inizializzazione; agisce solo a livello di tipo in compilazione, senza effetti a runtime.
</details>

<details>
<summary>In che modo extends per le interface differisce da extends per le classi?</summary>

Un'interface può estendere più interface contemporaneamente con `extends A, B, C`, fondendone i contratti, mentre una classe può estendere una sola classe (ereditarietà singola). Una classe può però implementare più interface con `implements`, recuperando così la composizione di contratti che l'ereditarietà singola non le consente.
</details>

<details>
<summary>Come si descrive il tipo di una funzione tramite un'interface?</summary>

Si utilizza una call signature: una firma senza nome interna all'interface che indica i tipi dei parametri e il tipo di ritorno.

```ts
interface Comparatore {
  (a: number, b: number): number;
}

const ascendente: Comparatore = (a, b) => a - b;
```

I tipi dei parametri non vanno ripetuti nell'assegnazione, perché vengono dedotti dalla call signature.
</details>

<details>
<summary>Qual è la differenza pratica tra interface e type quanto a riapribilità?</summary>

Un'interface supporta il declaration merging: dichiararla più volte con lo stesso nome fonde le dichiarazioni e somma i membri, il che è utile per estendere tipi di librerie o dichiarazioni globali. Un `type` non è riapribile e dichiararlo due volte genera l'errore `Duplicate identifier`. In compenso `type` può rappresentare anche union type, intersection type, tuple e primitivi rinominati, cosa che un'interface non può fare.
</details>

<details>
<summary>Cosa accade quando si accede a un metodo opzionale di un'interface e come gestirlo in sicurezza?</summary>

Un membro opzionale dichiarato con `?` potrebbe essere assente, e il suo tipo include implicitamente `undefined`. Invocarlo direttamente non è sicuro; si usa l'optional chaining `?.`, che restituisce `undefined` se il membro non è presente anziché sollevare un errore a runtime, spesso combinato con `??` per fornire un valore di fallback.

```ts
return ordine.applicaSconto?.(10) ?? ordine.importo;
```
</details>
