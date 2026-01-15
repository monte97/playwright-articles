---
title: "Architettura e Pattern per Test Scalabili"
date: 2025-01-23T10:00:00+01:00
description: "Page Object Model, Custom Fixtures e parallelizzazione: come strutturare test Playwright per progetti enterprise"
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

Un test che funziona non è abbastanza. Quando la suite cresce a centinaia di test, servono pattern che garantiscano **manutenibilità**, **riusabilità** e **scalabilità**.

In questo articolo esploriamo tre pattern fondamentali: **Page Object Model**, **Custom Fixtures**, e **parallelizzazione avanzata**.

---

## Il Problema: Test Che Non Scalano

Quando inizi, i test sono semplici:

```javascript
test('login', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
  await expect(page).toHaveURL(/dashboard/);
});
```

Ma poi aggiungi altri test che fanno login:

```javascript
test('view profile', async ({ page }) => {
  // Duplicato: stesso codice di login
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  // Il test vero inizia qui
  await page.getByRole('link', { name: 'Profile' }).click();
  // ...
});

test('edit settings', async ({ page }) => {
  // Di nuovo duplicato...
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  // ...
});
```

**Problemi:**

1. **Duplicazione**: Lo stesso codice di login in 50 test
2. **Manutenzione**: Se il form cambia, devi modificare 50 file
3. **Lentezza**: Ogni test esegue il login via UI (secondi persi)

---

## Pattern 1: Page Object Model (POM)

Il Page Object Model incapsula la logica di interazione con una pagina in una classe dedicata.

### Struttura

```text
tests/
├── pages/
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   └── ProfilePage.ts
├── specs/
│   ├── login.spec.ts
│   └── profile.spec.ts
└── fixtures/
    └── auth.ts
```

### Implementazione

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

  async getErrorMessage() {
    return this.page.getByTestId('error-message');
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

test('login con credenziali errate', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@test.com', 'wrong-password');

  await expect(loginPage.getErrorMessage()).toBeVisible();
});
```

### Vantaggi

- **DRY**: La logica di login è in un solo posto
- **Manutenibilità**: Se il form cambia, modifichi solo `LoginPage.ts`
- **Leggibilità**: I test esprimono l'intento, non i dettagli

---

## Pattern 2: Custom Fixtures

Le Fixture di Playwright permettono di creare setup/teardown riutilizzabili.

### Il Problema

Eseguire il login via UI prima di ogni test è:

- **Lento**: 2-3 secondi per test
- **Ridondante**: Stai testando il login 100 volte

### La Soluzione: Storage State

Playwright può salvare lo stato del browser (cookie, localStorage) e riutilizzarlo.

**fixtures/auth.ts**

```typescript
import { test as base } from '@playwright/test';

export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    // Setup: esegui login una volta
    await page.goto('/login');
    await page.getByLabel('Email').fill('user@test.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();
    await page.waitForURL(/dashboard/);

    // Passa la pagina autenticata al test
    await use(page);

    // Teardown: cleanup automatico
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

### Setup Globale con Project Dependencies

Per ottimizzare ulteriormente, puoi definire un progetto di setup:

**playwright.config.ts**

```typescript
export default defineConfig({
  projects: [
    // Progetto di setup: esegue login e salva lo stato
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
import { test as setup, expect } from '@playwright/test';

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

**Risultato**: Il login viene eseguito **una sola volta** per l'intera suite.

---

## Pattern 3: Parallelizzazione e Isolamento

### Configurazione Base

```typescript
// playwright.config.ts
export default defineConfig({
  workers: process.env.CI ? 2 : 4,
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
});
```

**Risultati tipici:**

```bash
# workers=1: ~10 minuti
# workers=4: ~2.5 minuti

# 4x più veloce!
```

### Il Problema: Resource Contention

Se due test paralleli usano lo stesso account utente:

```javascript
// Test 1: Aggiunge "Laptop" al carrello di user@test.com
// Test 2: Aggiunge "Phone" al carrello di user@test.com

// Interferiscono! Il carrello ha 2 prodotti invece di 1.
```

### La Soluzione: Fixture-Based Isolation

Ogni test deve operare su dati dedicati:

```typescript
export const test = base.extend({
  isolatedUser: async ({ request }, use) => {
    // Setup: crea utente univoco via API
    const userId = `user-${Date.now()}-${Math.random()}`;
    const email = `${userId}@test.com`;

    await request.post('/api/users', {
      data: { email, password: 'test123' },
    });

    // Passa l'utente al test
    await use({ email, password: 'test123' });

    // Teardown: elimina utente e dati
    await request.delete(`/api/users/${userId}`);
  },
});
```

**Utilizzo:**

```typescript
test('checkout flow', async ({ page, isolatedUser }) => {
  // Questo test ha un utente tutto suo
  await page.goto('/login');
  await page.getByLabel('Email').fill(isolatedUser.email);
  await page.getByLabel('Password').fill(isolatedUser.password);
  // ...
});
```

### Sharding per CI/CD

Per distribuire i test su più macchine:

```bash
# Macchina 1
npx playwright test --shard=1/4

# Macchina 2
npx playwright test --shard=2/4

# Macchina 3
npx playwright test --shard=3/4

# Macchina 4
npx playwright test --shard=4/4
```

---

## Pattern 4: Sincronizzazione con Eventi di Rete

Le moderne applicazioni sono asincrone. Un click può scatenare una chiamata API.

### Il Problema: Race Conditions

```javascript
// Click sul bottone "Salva"
await page.getByRole('button', { name: 'Salva' }).click();

// Verifica il messaggio di successo
await expect(page.getByText('Salvato')).toBeVisible();
// ❌ Potrebbe fallire se l'API è lenta!
```

### La Soluzione: waitForResponse

```typescript
// Pattern robusto: attesa esplicita della risposta API
const [response] = await Promise.all([
  // 1. Definisci l'evento da attendere
  page.waitForResponse(
    (res) => res.url().includes('/api/products') && res.status() === 201
  ),
  // 2. Scatena l'azione
  page.getByRole('button', { name: 'Salva' }).click(),
]);

// 3. Ora siamo certi che l'operazione è conclusa
await expect(page.getByText('Salvato')).toBeVisible();
```

---

## Struttura Consigliata

```text
playwright/
├── .auth/                    # Storage state salvato
│   └── user.json
├── fixtures/
│   ├── auth.ts               # Fixture autenticazione
│   └── api.ts                # Fixture per setup via API
├── pages/
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   └── CheckoutPage.ts
├── tests/
│   ├── auth.setup.ts         # Setup globale
│   ├── login.spec.ts
│   ├── checkout.spec.ts
│   └── profile.spec.ts
└── playwright.config.ts
```

---

## Cosa Abbiamo Imparato

1. **Page Object Model**: Incapsula la logica di pagina, migliora manutenibilità
2. **Custom Fixtures**: Setup/teardown riutilizzabili, autenticazione ottimizzata
3. **Parallelizzazione**: Worker multipli con isolamento dei dati
4. **Sincronizzazione**: `waitForResponse` per gestire chiamate API asincrone

Nel prossimo articolo vedremo come integrare Playwright in **CI/CD**, testing mobile, API testing e le **best practices** per team enterprise.

---

## Repository

Esplora gli esempi nel repository del workshop:

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

---

*Serie Playwright Workshop:*
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. **Architettura e Pattern per Test Scalabili** (questo articolo)
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
