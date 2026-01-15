---
title: "Architettura e Pattern per Test Scalabili"
date: 2025-01-23T10:00:00+01:00
description: "Page Object Model, Custom Fixtures e parallelizzazione: come strutturare test Playwright per progetti grandi"
menu:
  sidebar:
    name: "4. Architettura"
    identifier: playwright-4
    weight: 40
    parent: playwright-workshop
tags: ["Playwright", "Page Object Model", "Fixtures", "Parallelizzazione", "Architecture"]
categories: ["Testing", "Architecture", "Workshop"]
draft: false
---

*Tempo di lettura: ~12 minuti*

Quando la suite di test cresce a centinaia di test case, la manutenibilità diventa un problema. Questa sezione presenta pattern architetturali per strutturare test Playwright scalabili.

---

## Il Problema: Test Che Non Scalano

All'inizio i test sono semplici:

```javascript
test('login', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
});
```

Poi aggiungi altri test che fanno login:

```javascript
test('view profile', async ({ page }) => {
  // Duplicato
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  // Il test vero inizia qui
  await page.getByRole('link', { name: 'Profile' }).click();
});
```

**I problemi:**
1. Stesso codice in 50 test
2. Se il form cambia, modifichi 50 file
3. Ogni test esegue il login via UI (secondi persi)

---

## Pattern 1: Page Object Model

Incapsula la logica di una pagina in una classe.

**pages/LoginPage.ts**

```typescript
import { Page } from '@playwright/test';

export class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Login' }).click();
  }
}
```

**tests/login.spec.ts**

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('login con successo', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@test.com', 'password123');

  await expect(page).toHaveURL(/dashboard/);
});
```

**Vantaggi:**
- La logica è in un posto solo
- Se il form cambia, modifichi solo `LoginPage.ts`
- I test esprimono l'intento, non i dettagli

---

## Pattern 2: Custom Fixtures

Il login via UI è lento. Se hai 100 test, perdi minuti solo per autenticarti.

**La soluzione**: Salva lo stato del browser e riusalo.

**fixtures/auth.ts**

```typescript
import { test as base } from '@playwright/test';

export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    // Login una volta
    await page.goto('/login');
    await page.getByLabel('Email').fill('user@test.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();
    await page.waitForURL(/dashboard/);

    // Passa la pagina autenticata al test
    await use(page);

    // Cleanup
    await page.context().clearCookies();
  },
});
```

**tests/profile.spec.ts**

```typescript
import { test } from '../fixtures/auth';
import { expect } from '@playwright/test';

test('view profile', async ({ authenticatedPage }) => {
  // Già loggato!
  await authenticatedPage.getByRole('link', { name: 'Profile' }).click();
  await expect(authenticatedPage.getByText('Your Profile')).toBeVisible();
});
```

### Setup Globale

Per ottimizzare ancora: esegui il login **una volta** per tutta la suite.

**playwright.config.ts**

```typescript
export default defineConfig({
  projects: [
    // Progetto di setup
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },

    // Progetto principale: usa lo stato salvato
    {
      name: 'chromium',
      use: {
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

**tests/auth.setup.ts**

```typescript
import { test as setup } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
  await page.waitForURL(/dashboard/);

  // Salva lo stato
  await page.context().storageState({ path: authFile });
});
```

Il login viene eseguito **una sola volta**. Tutti i test partono già autenticati.

---

## Pattern 3: Isolamento per Parallelizzazione

```typescript
// playwright.config.ts
export default defineConfig({
  workers: process.env.CI ? 2 : 4,
  fullyParallel: true,
});
```

**Il problema**: Se due test paralleli usano lo stesso account, interferiscono.

```javascript
// Test 1: Aggiunge "Laptop" al carrello di user@test.com
// Test 2: Aggiunge "Phone" al carrello di user@test.com

// Il carrello ha 2 prodotti invece di 1!
```

**La soluzione**: Ogni test usa dati dedicati.

```typescript
export const test = base.extend({
  isolatedUser: async ({ request }, use) => {
    // Crea utente univoco via API
    const userId = `user-${Date.now()}-${Math.random()}`;
    const email = `${userId}@test.com`;

    await request.post('/api/users', {
      data: { email, password: 'test123' },
    });

    await use({ email, password: 'test123' });

    // Elimina utente
    await request.delete(`/api/users/${userId}`);
  },
});
```

**Utilizzo:**

```typescript
test('checkout', async ({ page, isolatedUser }) => {
  // Questo utente esiste solo per questo test
  await page.goto('/login');
  await page.getByLabel('Email').fill(isolatedUser.email);
  // ...
});
```

Nessuna interferenza. Parallelizzazione sicura.

---

## Pattern 4: Sincronizzazione con la Rete

Un click può scatenare una chiamata API. Se verifichi il risultato troppo presto, il test fallisce.

**Il problema:**

```javascript
await page.getByRole('button', { name: 'Salva' }).click();
await expect(page.getByText('Salvato')).toBeVisible();
// Potrebbe fallire se l'API è lenta!
```

**La soluzione**: Aspetta la risposta API.

```typescript
const [response] = await Promise.all([
  // Aspetta la risposta
  page.waitForResponse(
    (res) => res.url().includes('/api/products') && res.status() === 201
  ),
  // Scatena l'azione
  page.getByRole('button', { name: 'Salva' }).click(),
]);

// Ora l'operazione è completata
await expect(page.getByText('Salvato')).toBeVisible();
```

Il test prosegue solo quando la risposta API è stata ricevuta, garantendo determinismo.

---

## Struttura Consigliata

```text
playwright/
├── .auth/
│   └── user.json           # Stato autenticato
├── fixtures/
│   ├── auth.ts             # Fixture autenticazione
│   └── api.ts              # Fixture per setup API
├── pages/
│   ├── LoginPage.ts
│   └── CheckoutPage.ts
├── tests/
│   ├── auth.setup.ts       # Setup globale
│   ├── login.spec.ts
│   └── checkout.spec.ts
└── playwright.config.ts
```

---

## Cosa Ho Imparato

1. **Page Object Model**: Una classe per pagina. Manutenibilità.
2. **Fixtures**: Setup riutilizzabile. Autenticazione ottimizzata.
3. **Isolamento**: Dati dedicati per ogni test. Parallelizzazione sicura.
4. **waitForResponse**: Sincronizzazione con le API. Niente race condition.

Nel prossimo articolo: **CI/CD**, mobile testing, API testing, best practices.

---

## Repository

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

---

*Serie Playwright Workshop:*
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. **Architettura e Pattern per Test Scalabili** (questo articolo)
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
