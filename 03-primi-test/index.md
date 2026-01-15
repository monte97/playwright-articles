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

Scrivere test a mano è lento. **Codegen** registra le tue azioni e genera il codice.

```bash
npx playwright codegen localhost:3000
```

Si apre un browser. Interagisci con l'app. Playwright genera il codice:

```javascript
// Generato automaticamente
await page.goto('http://localhost:3000/');
await page.getByRole('link', { name: 'Products' }).click();
await page.getByPlaceholder('Search...').fill('laptop');
await page.getByRole('button', { name: 'Search' }).click();
await page.getByRole('heading', { name: 'Laptop Pro' }).click();
```

Nota: Playwright **sceglie automaticamente i selettori migliori** (`getByRole`, `getByPlaceholder`).

### Quando Usare Codegen

**Funziona bene per:**
- Flussi di login/signup lineari
- Navigazione tra pagine
- Form semplici
- Click su bottoni visibili

**Poi migliora il codice:**
- Aggiungi assertion significative
- Estrai parti ripetute
- Gestisci casi edge

---

## UI Mode

```bash
npx playwright test --ui
```

È un'interfaccia grafica per sviluppare e debuggare test.

**Cosa puoi fare:**
- Eseguire singoli test con un click
- Watch mode (riesegue al salvataggio)
- Vedere screenshot step-by-step
- Ispezionare il DOM in ogni momento
- Vedere le chiamate di rete
- Leggere i log della console

**I pannelli principali:**

- **Timeline**: Cronologia delle azioni. Clicca per vedere lo stato in quel momento.
- **Azioni**: Lista di click, fill, goto con tempo e locator usato.
- **Pick Locator**: Passa il mouse sul DOM per vedere il selettore suggerito.
- **Network**: Tutte le richieste API.
- **Console**: Log dal browser e dal test.

---

## Trace Viewer

Quando un test fallisce in CI, il Trace Viewer permette di analizzare l'esecuzione passo per passo.

**Attivazione:**

```javascript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry',
  },
});
```

**Cosa registra:**
- Screenshot di ogni azione
- Stato del DOM
- Chiamate di rete
- Log della console
- Punto esatto dell'errore

**Come usarlo:**

```bash
# Esegui con trace
npx playwright test --trace on

# Apri il trace di un test fallito
npx playwright show-trace trace.zip
```

Il trace è un file che puoi aprire nel browser. Vedi esattamente cosa è successo, anche se il test è girato in CI.

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

## Configurazione Base

```javascript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30 * 1000,

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

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },

  workers: process.env.CI ? 1 : 4,

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

**Cosa fa:**
- `testDir`: Dove sono i test
- `projects`: Browser da testare
- `baseURL`: URL base (evita hardcoding)
- `trace`: Registra trace al primo retry
- `workers`: Parallelizzazione
- `webServer`: Avvia l'app prima dei test

---

## Cosa Ho Imparato

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
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. **I Tuoi Primi Test con Playwright** (questo articolo)
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
