---
title: "I Tuoi Primi Test con Playwright"
date: 2025-01-22T10:00:00+01:00
description: "Hands-on con Playwright: sintassi base, Codegen per generare test automaticamente, UI Mode e Trace Viewer per debugging"
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

Basta teoria. È ora di scrivere codice.

In questo articolo vedremo come scrivere il tuo primo test Playwright, come usare **Codegen** per generare test automaticamente, e come debuggare con **UI Mode** e **Trace Viewer**.

---

## La Sintassi Base

Un test Playwright ha questa struttura:

```javascript
import { test, expect } from '@playwright/test';

test('login con successo', async ({ page }) => {
  // Naviga alla pagina
  await page.goto('/login');

  // Interagisci con gli elementi
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  // Verifica il risultato
  await expect(page).toHaveURL(/dashboard/);
  await expect(page.getByText('Benvenuto')).toBeVisible();
});
```

**Punti chiave:**

- `test()` definisce un test case
- `{ page }` è una fixture che rappresenta una tab del browser
- `await` è obbligatorio per ogni operazione (Playwright è asincrono)
- `expect()` con `await` per le web-first assertions

---

## Eseguire i Test

### Comandi Base

```bash
# Tutti i test
npx playwright test

# Test specifico
npx playwright test login.spec.ts

# Per nome
npx playwright test -g "login"

# Browser specifico
npx playwright test --project=chromium

# Debug mode (apre il browser)
npx playwright test --debug

# UI Mode (raccomandato per sviluppo)
npx playwright test --ui
```

### Output Tipico

```bash
Running 3 tests using 3 workers

✓ login.spec.ts:3 › login con successo (2s)
✓ signup.spec.ts:5 › registrazione utente (3s)
✓ profile.spec.ts:7 › modifica profilo (2s)

3 passed (7s)
```

### HTML Reporter

```bash
# Apri il report dettagliato
npx playwright show-report
```

Il report HTML mostra ogni test con screenshot, trace e dettagli di esecuzione.

---

## Codegen: Genera Test Automaticamente

Scrivere test manualmente è tedioso. **Codegen** registra le tue azioni nel browser e genera il codice corrispondente.

### Come Funziona

```bash
# Avvia Codegen
npx playwright codegen localhost:3000
```

1. Si apre un browser con la tua app
2. Interagisci normalmente (clicca, scrivi, naviga)
3. Playwright genera il codice in tempo reale

### Codice Generato

```javascript
// Generato automaticamente da Codegen
await page.goto('http://localhost:3000/');

await page.getByRole('link', { name: 'Products' }).click();

await page.getByPlaceholder('Search...').fill('laptop');

await page.getByRole('button', { name: 'Search' }).click();

await page.getByRole('heading', { name: 'Laptop Pro' }).click();
```

Nota come Playwright **sceglie automaticamente i selettori migliori** (`getByRole`, `getByPlaceholder`) invece di usare selettori CSS fragili.

### Limitazioni e Best Practice

**Situazioni problematiche:**

- Navigazione via Tab non viene registrata bene
- Asserzioni su tooltip/balloon instabili
- Dropdown con contenuto dinamico
- Form multi-step con validazione

**Usa Codegen per:**

- Flussi di login/signup lineari
- Navigazione base tra pagine
- Compilazione di form semplici
- Click su bottoni visibili

**Poi migliora il codice:**

- Refactor selettori se necessario
- Aggiungi asserzioni significative
- Estrai fixture e helper
- Aggiungi gestione errori

---

## UI Mode: Debugging Interattivo

UI Mode è lo strumento più potente per lo sviluppo di test.

```bash
npx playwright test --ui
```

### Features

**Development:**
- Watch mode attivo (riesegue al salvataggio)
- Hot reload
- Esegui singoli test con un click

**Debugging:**
- Inspector integrato
- Timeline visuale delle azioni
- Video playback

**Analysis:**
- Screenshot step-by-step
- Network tab (come DevTools)
- Console logs

### Pannelli Principali

- **Elenco Test (Sidebar)**: Mostra tutti i file e test. Puoi eseguire, osservare (watch) o debuggare ogni singolo test.

- **Timeline**: Vista cronologica delle azioni. Evidenzia navigazioni e azioni con colori diversi.

- **Azioni**: Lista di click, fill, goto con tempo impiegato e locator usato.

- **Pick Locator**: Passa il mouse sul DOM per vedere il selettore suggerito. Clicca per copiarlo.

- **Source**: Codice del test con evidenziazione della riga corrente.

- **Network**: Tutte le richieste API con header, payload e response.

- **Console**: Log dal browser e dal test.

---

## Trace Viewer: L'Autopsia dei Test Falliti

Quando un test fallisce in CI, il **Trace Viewer** ti mostra esattamente cosa è successo.

### Attivazione

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // Registra trace al primo fallimento
  },
});
```

### Come Usarlo

```bash
# Eseguire test con trace sempre attivo
npx playwright test --trace on

# Visualizzare trace di un test fallito
npx playwright show-trace trace.zip
```

### Cosa Mostra

**Timeline:**
- Cronologia di azioni e navigazioni
- Seek per esplorare ogni momento

**Azioni:**
- Lista di click, fill, goto
- Tempo impiegato per azione
- Locator utilizzato

**Source:**
- Codice del test sincronizzato
- Evidenzia la riga dell'errore

**Network:**
- Richieste API/risorse
- Headers, payload, response

**Console:**
- Log dal browser e dal test

**Errors:**
- Messaggio di errore completo
- Punto esatto del fallimento

**DOM Snapshot:**
- HTML in quel momento
- Ispezionabile con DevTools

---

## Un Test Completo: E-Commerce Flow

Mettiamo insieme tutto quello che abbiamo visto:

```javascript
import { test, expect } from '@playwright/test';

test('utente completa un acquisto', async ({ page }) => {
  // 1. Naviga alla home
  await page.goto('https://shop.example.com');

  // 2. Login
  await page.getByRole('link', { name: 'Login' }).click();
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Accedi' }).click();

  // 3. Verifica login
  await expect(page.getByText('Benvenuto')).toBeVisible();

  // 4. Cerca un prodotto
  await page.getByPlaceholder('Cerca prodotti...').fill('laptop');
  await page.getByRole('button', { name: 'Cerca' }).click();

  // 5. Seleziona il prodotto
  await page.getByTestId('product-card').first().click();

  // 6. Aggiungi al carrello
  await page.getByRole('button', { name: 'Aggiungi al carrello' }).click();

  // 7. Verifica carrello
  await expect(page.getByTestId('cart-count')).toHaveText('1');

  // 8. Vai al checkout
  await page.getByRole('link', { name: 'Carrello' }).click();
  await page.getByRole('button', { name: 'Procedi al checkout' }).click();

  // 9. Conferma ordine
  await page.getByRole('button', { name: 'Conferma ordine' }).click();

  // 10. Verifica successo
  await expect(page.getByText('Ordine confermato')).toBeVisible();
});
```

**Nota**: I selettori sono **semantici** (`getByRole`, `getByLabel`, `getByTestId`), il codice è **leggibile**, e non c'è nessun `sleep()`.

---

## Configurazione: playwright.config.ts

Una configurazione base:

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // Directory dei test
  testDir: './tests',

  // Timeout globale per test (ms)
  timeout: 30 * 1000,

  // Browser da testare
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

  // Configurazione globale
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },

  // Parallelizzazione
  workers: process.env.CI ? 1 : 4,

  // Web server da avviare prima dei test
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Cosa Abbiamo Imparato

1. **Sintassi base**: `test()`, `page`, `await`, `expect()`
2. **Codegen**: Genera test automaticamente registrando le azioni
3. **UI Mode**: Debugging interattivo con watch mode
4. **Trace Viewer**: Analisi dettagliata dei test falliti

Nel prossimo articolo vedremo come strutturare test per progetti enterprise: **Page Object Model**, **Custom Fixtures**, e **parallelizzazione avanzata**.

---

## Repository

Prova gli esempi nel repository del workshop:

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

```bash
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
