---
title: "CI/CD e Best Practices"
date: 2025-01-24T10:00:00+01:00
description: "Integrazione Playwright in CI/CD, mobile testing, API testing e best practices per team"
menu:
  sidebar:
    name: "5. CI/CD & Best Practices"
    identifier: playwright-5
    weight: 50
    parent: playwright-workshop
tags: ["Playwright", "CI/CD", "GitHub Actions", "Mobile Testing", "API Testing", "Best Practices"]
categories: ["Testing", "DevOps", "Workshop"]
draft: false
---

*Tempo di lettura: ~10 minuti*

L'integrazione dei test in una pipeline CI/CD garantisce l'esecuzione automatica ad ogni push. Questa sezione copre la configurazione per GitHub Actions e GitLab CI, il mobile testing, l'API testing e le best practices.

---

## GitHub Actions

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

> ⚠️ **Ottimizzazione**: Per semplificare ulteriormente la pipeline, è possibile usare l'action ufficiale `uses: microsoft/playwright-github-action@v1`. Questa action gestisce in modo automatico il caching dei browser e l'installazione delle dipendenze, riducendo i tempi di esecuzione e la complessità dello script.

**GitLab CI:**

```yaml
playwright:
  image: mcr.microsoft.com/playwright:v1.40.0-jammy
  stage: test
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
```

---

## Configurazione per CI

```typescript
// playwright.config.ts
export default defineConfig({
  // Meno worker in CI
  workers: process.env.CI ? 2 : 4,

  // Retry in CI
  retries: process.env.CI ? 2 : 0,

  // Reporter
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['github']]
    : 'html',

  use: {
    // Trace solo su retry
    trace: 'on-first-retry',

    // Screenshot solo su fallimento
    screenshot: 'only-on-failure',
  },
});
```

---

## Mobile Testing

Playwright emula device mobili senza configurazione extra.

```typescript
import { devices } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'iPhone 13',
      use: { ...devices['iPhone 13'] },
    },
    {
      name: 'Pixel 5',
      use: { ...devices['Pixel 5'] },
    },
  ],
});
```

**Touch events:**

```typescript
test('mobile menu', async ({ page }) => {
  await page.tap('#menu-button');
  await expect(page.getByRole('navigation')).toBeVisible();
});
```

---

## API Testing

Playwright testa anche API direttamente.

**GET:**

```typescript
test('GET products', async ({ request }) => {
  const response = await request.get('/api/products');

  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);

  const products = await response.json();
  expect(products).toHaveLength(10);
});
```

**POST:**

```typescript
test('POST create product', async ({ request }) => {
  const response = await request.post('/api/products', {
    data: {
      name: 'New Product',
      price: 99.99,
    },
  });

  expect(response.status()).toBe(201);
});
```

**Setup via API** (più veloce della UI):

```typescript
test.beforeEach(async ({ request }) => {
  await request.post('/api/products', {
    data: { name: 'Test Product', price: 50 },
  });
});

test('product in list', async ({ page }) => {
  await page.goto('/products');
  await expect(page.getByText('Test Product')).toBeVisible();
});
```

---

## Best Practices

### Testa comportamento, non implementazione

```typescript
// Fragile
await page.click('div.container > button.btn-primary');

// Robusto
await page.getByRole('button', { name: 'Acquista' }).click();
```

### Priorità selettori

```typescript
// 1. Role (migliore)
await page.getByRole('button', { name: 'Submit' });

// 2. Label
await page.getByLabel('Email');

// 3. Placeholder
await page.getByPlaceholder('Cerca...');

// 4. Text
await page.getByText('Continua');

// 5. Test ID (per elementi senza semantica)
await page.getByTestId('checkout-button');

// Evitare
await page.click('.btn.btn-primary.mt-2');
```

### Assertion significative

```typescript
// Troppo generico
await expect(page.getByText('Success')).toBeVisible();

// Specifico
await expect(page.getByRole('alert')).toContainText('Ordine confermato');
await expect(page.getByTestId('order-id')).toHaveText(/ORD-\d+/);
```

### Test isolati

```typescript
// Fragile: dipende dal test precedente
test('create product', async ({ page }) => {
  // Crea "Laptop"
});

test('find product', async ({ page }) => {
  // Assume che "Laptop" esista
});

// Robusto: indipendente
test('create and find product', async ({ page }) => {
  // Setup
  // Test
  // Cleanup
});
```

---

## Quando Usare Playwright

**Ottimo per:**
- Nuovi progetti
- Web app moderne (SPA, PWA)
- Cross-browser testing
- CI/CD intensive
- Team che vuole velocità

**Considerazioni:**
- Nessun supporto per IE11
- Solo web mobile (emulazione), non app native
- Curva di apprendimento iniziale

---

## Riepilogo della Serie

**Fondamenti:**
- Piramide dei test e ruolo dei test E2E
- Sfide storiche del testing E2E

**Pilastri Playwright:**
- Auto-waiting per test affidabili
- Selettori semantici per test resilienti
- Parallelizzazione per velocità

**Pratica:**
- Sintassi base
- Codegen per generazione automatica
- UI Mode e Trace Viewer per debugging

**Architettura:**
- Page Object Model
- Custom Fixtures
- Isolamento per parallelizzazione

**Enterprise:**
- CI/CD
- Mobile e API testing
- Best practices

---

## Risorse Utili

* **Playwright Docs**: [playwright.dev](https://playwright.dev)
* **Best Practices**: [playwright.dev/docs/best-practices](https://playwright.dev/docs/best-practices)
* **VS Code Extension**: [marketplace.visualstudio.com](https://marketplace.visualstudio.com/items?itemName=ms-playwright.playwright)
* **Discord**: [aka.ms/playwright/discord](https://aka.ms/playwright/discord)
* **GitHub**: [github.com/microsoft/playwright](https://github.com/microsoft/playwright)

---

## Repository

Tutto il codice del workshop:

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

```bash
git clone https://github.com/monte97/workshop-playwright
cd workshop-playwright

# Demo app
cd infrastructure-demo && npm install && npm start

# Test
cd ../demo && npm install && npx playwright test --ui
```

---

*Serie Playwright Workshop:*
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. **CI/CD e Best Practices** (questo articolo)
