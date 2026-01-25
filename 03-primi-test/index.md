---
title: "I Tuoi Primi Test con Playwright"
date: 2025-01-22T10:00:00+01:00
description: "Hands-on con Playwright: sintassi base, Codegen per generare test, UI Mode e Trace Viewer per debugging"
menu:
  sidebar:
    name: "3. Primi Test"
    identifier: playwright-3
    weight: 30
    parent: playwright-workshop
tags: ["Playwright", "Codegen", "UI Mode", "Trace Viewer", "Tutorial"]
categories: ["Testing", "Tutorial", "Workshop"]
draft: false
---

*Tempo di lettura: ~12 minuti*

Questa sezione copre la sintassi base di Playwright, gli strumenti di generazione automatica dei test e le opzioni di debugging.

---

## La Sintassi

```javascript
import { test, expect } from '@playwright/test';

test('login con successo', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  await expect(page).toHaveURL(/dashboard/);
  await expect(page.getByText('Benvenuto')).toBeVisible();
});
```

**I pezzi:**
- `test()` definisce un test
- `{ page }` è una tab del browser
- `await` è obbligatorio (tutto è asincrono)
- `expect()` con `await` per le assertion

---

## Eseguire i Test

```bash
# Tutti i test
npx playwright test

# Un file specifico
npx playwright test login.spec.ts

# Per nome
npx playwright test -g "login"

# Un browser specifico
npx playwright test --project=chromium

# Debug (apre il browser)
npx playwright test --debug

# UI Mode (il migliore per sviluppo)
npx playwright test --ui
```

**Output:**

```bash
Running 3 tests using 3 workers

✓ login.spec.ts:3 › login con successo (2s)
✓ signup.spec.ts:5 › registrazione (3s)
✓ profile.spec.ts:7 › modifica profilo (2s)

3 passed (7s)
```

---

## Codegen: Non Scrivere, Registra

Scrivere test a mano può essere lento e ripetitivo. [**Codegen**](https://playwright.dev/docs/codegen) è lo strumento di Playwright che accelera questo processo, registrando le tue interazioni con l'applicazione e generando il codice del test.

```bash
npx playwright codegen localhost:3000
```

Questa istruzione apre una finestra del browser che punta all'URL specificato e la finestra di Codegen. Ogni azione che compi nel browser (click, input di testo, etc.) viene tradotta in tempo reale in una riga di codice.

![Codegen in azione mentre suggerisce un selettore](https://github.com/monte97/workshop-playwright/blob/main/slides/pages/capitolo-2-playwright/generator-demo-selector.png?raw=true)

La vera forza di Codegen è la sua capacità di **scegliere il selettore migliore possibile**. Dà la priorità a selettori resilienti e centrati sull'utente, come `getByRole`, `getByText`, e `getByLabel`, invece di selettori fragili come XPath o classi CSS.

Il codice generato è un ottimo punto di partenza:

```javascript
// Codice generato da Codegen
await page.goto('http://localhost:3000/');
await page.getByRole('link', { name: 'Products' }).click();
await page.getByPlaceholder('Search...').fill('laptop');
await page.getByRole('button', { name: 'Search' }).click();
await page.getByRole('heading', { name: 'Laptop Pro' }).click();
```

### Quando Usare Codegen

Codegen è ideale per "bootstrappare" un nuovo test.

**Usalo per:**
- Tracciare lo scheletro di un flusso utente (login, signup, acquisto).
- Scoprire rapidamente qual è il selettore giusto per un elemento.
- Generare il codice per form complesse o navigazioni multi-step.

**Il codice generato va poi rifinito:**
- Aggiungi assertion (`expect`) per verificare lo stato dell'applicazione.
- Estrai logica ripetuta in funzioni o Page Object.
- Gestisci dati di test e casi limite.

![Codegen genera anche le asserzioni](https://github.com/monte97/workshop-playwright/blob/main/slides/pages/capitolo-2-playwright/generator-demo-assertions.png?raw=true)

---

## UI Mode

L'[UI Mode](https://playwright.dev/docs/test-ui-mode) è l'ambiente di sviluppo e debugging integrato di Playwright, pensato per la scrittura di test.

```bash
npx playwright test --ui
```

Apre un'interfaccia grafica che rivoluziona il modo in cui si lavora con i test E2E.

![L'interfaccia di Playwright UI Mode](https://github.com/monte97/workshop-playwright/blob/main/slides/pages/capitolo-2-playwright/playwright-ui-mode.png?raw=true)

**È lo strumento principale per lo sviluppo locale perché permette di:**
- **Eseguire test singolarmente** con un clic, isolando il problema.
- **Usare il "Watch mode"**: il test viene rieseguito automaticamente a ogni salvataggio del file.
- **Navigare nel tempo (*Time Travel*)**: la timeline del test mostra screenshot per ogni azione. Cliccando su un punto della timeline, puoi vedere lo stato del DOM in quel preciso istante e "tornare indietro" per capire cosa è successo.
- **Ispezionare i selettori**: usando la funzione "Pick Locator" puoi passare il mouse sull'applicazione e vedere quale selettore usare, con la possibilità di testarlo in tempo reale.
- **Analizzare le chiamate di rete**, i log della console e vedere il codice sorgente del test, tutto in un unico posto.

L'UI Mode trasforma il debugging da un processo lento e frustrante a un'esperienza interattiva ed efficiente.

---

## Trace Viewer: L'analisi post-mortem

Il [Trace Viewer](https://playwright.dev/docs/trace-viewer) è uno strumento di debugging post-esecuzione. A differenza dell'UI Mode, che è interattivo, il Trace Viewer serve ad analizzare un'esecuzione **già completata**.

Il suo scopo principale è permettere di diagnosticare fallimenti, specialmente quelli che avvengono in ambienti non interattivi come le pipeline di CI/CD, ma è utilissimo anche in locale.

**Come si attiva:**

La configurazione più comune è registrarlo solo quando un test fallisce e viene ritentato.

```javascript
// playwright.config.ts
export default defineConfig({
  use: {
    // Registra un trace per ogni test, ma solo al primo tentativo fallito.
    trace: 'on-first-retry',
  },
});
```

Un file `trace.zip` viene generato per ogni test che corrisponde alla condizione.

**Cosa contiene un trace:**
- Lo **snapshot del DOM** per ogni istante del test (time travel).
- Gli **screenshot** di ogni azione.
- Le **chiamate di rete** (API requests/responses).
- I **log della console**.
- Il punto esatto del fallimento evidenziato.

**Come si apre un trace:**

```bash
# Esegui i test (se un test fallisce, viene generato il trace)
npx playwright test

# Apri il report dell'ultimo test fallito
npx playwright show-trace
```

Il trace è un'applicazione web locale, auto-contenuta e portabile. Puoi inviare il file `trace.zip` a un collega per mostrare esattamente cosa è successo, senza bisogno di riprodurre il bug. È la "scatola nera" dei tuoi test E2E.

---

## Un Test Completo

```javascript
import { test, expect } from '@playwright/test';

test('utente completa un acquisto', async ({ page }) => {
  // Naviga
  await page.goto('https://shop.example.com');

  // Login
  await page.getByRole('link', { name: 'Login' }).click();
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Accedi' }).click();
  await expect(page.getByText('Benvenuto')).toBeVisible();

  // Cerca prodotto
  await page.getByPlaceholder('Cerca...').fill('laptop');
  await page.getByRole('button', { name: 'Cerca' }).click();

  // Aggiungi al carrello
  await page.getByTestId('product-card').first().click();
  await page.getByRole('button', { name: 'Aggiungi al carrello' }).click();
  await expect(page.getByTestId('cart-count')).toHaveText('1');

  // Checkout
  await page.getByRole('link', { name: 'Carrello' }).click();
  await page.getByRole('button', { name: 'Checkout' }).click();
  await page.getByRole('button', { name: 'Conferma' }).click();

  // Verifica
  await expect(page.getByText('Ordine confermato')).toBeVisible();
});
```

Il test utilizza selettori semantici e non richiede attese manuali.

---

## Configurazione del Progetto (`playwright.config.ts`)

Un progetto Playwright è controllato dal file [`playwright.config.ts`](https://playwright.dev/docs/test-configuration) alla root. Una configurazione ben strutturata è fondamentale per un'esperienza di testing scalabile e manutenibile.

Vediamo una configurazione di base, simile a quella che puoi trovare nel [progetto demo (`demo/playwright.config.ts`)](https://github.com/monte97/workshop-playwright/blob/main/demo/playwright.config.ts).

```javascript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // 1. Dove trovare i file dei test.
  testDir: './tests',

  // 2. Timeout globale per ogni test (in millisecondi).
  timeout: 30 * 1000,

  // 3. Progetti: definisce i browser e i dispositivi da testare.
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
  ],

  // 4. Opzioni globali per l'esecuzione dei test.
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },

  // 5. Numero di worker per l'esecuzione parallela.
  workers: process.env.CI ? 1 : undefined, // In CI usa 1, localmente usa il default (metà delle CPU).

  // 6. Server web: avvia l'applicazione prima di eseguire i test.
  webServer: {
    command: 'npm run start', // Comando per avviare l'app
    url: 'http://localhost:3000', // URL da attendere prima di iniziare
    reuseExistingServer: !process.env.CI, // In locale, riusa il server se già attivo.
  },
});
```

### Analisi della Configurazione

1.  **`testDir`**: Indica a Playwright la cartella che contiene i file dei test (quelli che finiscono in `.spec.ts`).
2.  **`timeout`**: Imposta un tempo massimo per ogni singolo test. Se un test impiega più di 30 secondi, verrà interrotto. È una salvaguardia contro test bloccati.
3.  **`projects`**: Questa è una delle feature più potenti. Permette di definire diverse configurazioni di test. Qui definiamo due progetti, uno per `chromium` e uno per `firefox`. Eseguendo `npx playwright test --project=chromium` si userà solo Chrome. Playwright fornisce una lista di `devices` pre-configurati per simulare viewport e user agent di dispositivi comuni.
4.  **`use`**: Contiene opzioni che vengono applicate a tutti i progetti.
    *   `baseURL`: Permette di usare URL relativi nei test (es. `page.goto('/login')` invece di `page.goto('http://localhost:3000/login')`). Rende i test più portabili tra diversi ambienti.
    *   `trace`: Come abbiamo visto, `on-first-retry` è la strategia migliore per bilanciare risorse e capacità di debugging.
5.  **`workers`**: Controlla la parallelizzazione. Il valore di default è `undefined`, che significa che Playwright userà fino a metà dei core della CPU. In CI è prassi comune limitarlo a 1 o 2 per gestire meglio le risorse.
6.  **`webServer`**: Una feature comodissima. Playwright avvia automaticamente il server della tua applicazione prima di lanciare i test e lo spegne alla fine. `reuseExistingServer` è utile in locale per non dover riavviare il server a ogni esecuzione.

---

## Riepilogo

1. **Sintassi semplice**: `test()`, `page`, `await`, `expect()`
2. **Codegen**: Genera test registrando le azioni
3. **UI Mode**: Debugging interattivo con watch mode
4. **Trace Viewer**: Analisi dettagliata dei fallimenti

Nel prossimo articolo: come strutturare test per progetti grandi. **Page Object Model**, **Fixtures**, **parallelizzazione avanzata**.

---

## Prova

```bash
git clone https://github.com/monte97/workshop-playwright
cd workshop-playwright/demo
npm install
npx playwright test --ui
```

---

*Serie Playwright Workshop:*
1. [Perché i Test E2E Sono un Pilastro della Quality Assurance]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. **I Tuoi Primi Test con Playwright** (questo articolo)
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
