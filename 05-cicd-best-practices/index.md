---
title: "CI/CD e Best Practices"
date: 2025-01-24T10:00:00+01:00
description: "Integrazione Playwright in CI/CD, mobile testing, API testing e le best practices per team enterprise"
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

Un test che funziona solo in locale non vale molto. Il vero valore dei test E2E emerge quando girano **automaticamente** ad ogni push, in **CI/CD**.

In questo articolo finale vediamo l'integrazione con GitHub Actions, mobile testing, API testing, e le best practices per team enterprise.

---

## Integrazione CI/CD

### GitHub Actions

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

### GitLab CI

```yaml
# .gitlab-ci.yml
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
    expire_in: 1 week
```

### Configurazione Ottimizzata per CI

```typescript
// playwright.config.ts
export default defineConfig({
  // Meno worker in CI (risorse limitate)
  workers: process.env.CI ? 2 : 4,

  // Retry in CI per gestire flakiness occasionale
  retries: process.env.CI ? 2 : 0,

  // Report HTML
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['github']]
    : 'html',

  use: {
    // Trace solo su retry (riduce storage)
    trace: 'on-first-retry',

    // Screenshot solo su fallimento
    screenshot: 'only-on-failure',
  },
});
```

---

## Mobile Testing

Playwright supporta l'emulazione di device mobili senza configurazione aggiuntiva.

### Device Emulation

```typescript
import { devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // Desktop
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] },
    },

    // Mobile
    {
      name: 'iPhone 13',
      use: { ...devices['iPhone 13'] },
    },
    {
      name: 'Pixel 5',
      use: { ...devices['Pixel 5'] },
    },

    // Tablet
    {
      name: 'iPad Pro',
      use: { ...devices['iPad Pro 11'] },
    },
  ],
});
```

### Touch Events

```typescript
test('mobile menu', async ({ page }) => {
  // Tap (equivalente al touch)
  await page.tap('#menu-button');

  // Swipe
  await page.touchscreen.tap(100, 100);

  // Verifica responsive
  await expect(page.getByRole('navigation')).toBeVisible();
});
```

**Device preconfigurati:**

- iPhone (varie versioni)
- iPad
- Android (Pixel, Galaxy)
- Tablets

---

## API Testing

Playwright non è solo per UI. Può testare API direttamente.

### GET Request

```typescript
test('GET products', async ({ request }) => {
  const response = await request.get('/api/products');

  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);

  const products = await response.json();
  expect(products).toHaveLength(10);
});
```

### POST Request

```typescript
test('POST create product', async ({ request }) => {
  const response = await request.post('/api/products', {
    data: {
      name: 'New Product',
      price: 99.99,
    },
  });

  expect(response.status()).toBe(201);

  const product = await response.json();
  expect(product.id).toBeDefined();
});
```

### Setup via API

Un pattern comune è usare le API per setup veloce:

```typescript
test.beforeEach(async ({ request }) => {
  // Crea dati via API (più veloce della UI)
  await request.post('/api/products', {
    data: { name: 'Test Product', price: 50 },
  });
});

test('product appears in list', async ({ page }) => {
  await page.goto('/products');
  await expect(page.getByText('Test Product')).toBeVisible();
});
```

---

## Best Practices

### 1. Test Philosophy

**Testa comportamento, non implementazione:**

```typescript
// ❌ Fragile: dipende dalla struttura HTML
await page.click('div.container > button.btn-primary');

// ✅ Robusto: testa l'intento
await page.getByRole('button', { name: 'Acquista' }).click();
```

**User-centric testing:**

```typescript
// ❌ Test tecnico
expect(await page.evaluate(() => localStorage.getItem('cart'))).toBe('...');

// ✅ Test dall'ottica utente
await expect(page.getByTestId('cart-count')).toHaveText('3');
```

### 2. Selettori Priorità

```typescript
// 1. Role-based (migliore)
await page.getByRole('button', { name: 'Submit' });

// 2. Label (ottimo per form)
await page.getByLabel('Email');

// 3. Placeholder
await page.getByPlaceholder('Cerca...');

// 4. Text
await page.getByText('Continua');

// 5. Test ID (per elementi senza semantica)
await page.getByTestId('checkout-button');

// ❌ Evitare: CSS selectors fragili
await page.click('.btn.btn-primary.mt-2');
```

### 3. Assertions Significative

```typescript
// ❌ Troppo generico
await expect(page).toHaveURL(/.*/);

// ✅ Specifico
await expect(page).toHaveURL('/dashboard');

// ❌ Solo presenza
await expect(page.getByText('Success')).toBeVisible();

// ✅ Verifica completa
await expect(page.getByRole('alert')).toContainText('Ordine confermato');
await expect(page.getByTestId('order-id')).toHaveText(/ORD-\d+/);
```

### 4. DRY con Fixtures e POM

```typescript
// ❌ Duplicazione
test('test 1', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name=email]', 'user@test.com');
  // ... ripetuto in 50 test
});

// ✅ Centralizzato
test('test 1', async ({ authenticatedPage }) => {
  // Già loggato grazie alla fixture
});
```

### 5. Isolamento dei Test

```typescript
// ❌ Test che dipendono l'uno dall'altro
test('create product', async ({ page }) => {
  // Crea "Laptop"
});

test('find product', async ({ page }) => {
  // Assume che "Laptop" esista dal test precedente
  // FRAGILE! Cosa succede se il primo test fallisce?
});

// ✅ Test indipendenti
test('create and find product', async ({ page }) => {
  // Setup: crea il prodotto
  // Test: verifica che sia visibile
  // Teardown: pulisci
});
```

---

## Quando Usare Playwright

### Ottimo Per

- Nuovi progetti (o refactor testing)
- Web app moderne (SPA, PWA)
- Cross-browser testing
- CI/CD intensive
- Team che vuole velocità di sviluppo

### Considerazioni

- Nessun supporto per browser legacy (IE11)
- Solo web mobile (emulazione), non app native
- Curva di apprendimento iniziale
- Costo di migrazione da altri framework

---

## Riepilogo della Serie

In questa serie abbiamo costruito competenze complete sul testing E2E:

### Fondamenti

- La piramide dei test e il ruolo dei test E2E
- Le sfide storiche del testing E2E
- Perché Playwright cambia le regole

### Pilastri Playwright

- Auto-waiting per test affidabili
- Selettori semantici per test resilienti
- Parallelizzazione per velocità

### Pratica

- Sintassi base e primi test
- Codegen per generazione automatica
- UI Mode e Trace Viewer per debugging

### Architettura

- Page Object Model per manutenibilità
- Custom Fixtures per setup riutilizzabile
- Isolamento per parallelizzazione sicura

### Enterprise

- Integrazione CI/CD
- Mobile e API testing
- Best practices per team scalabili

---

## Risorse

### Documentazione

- [Playwright Docs](https://playwright.dev)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [API Reference](https://playwright.dev/docs/api/class-playwright)

### Tools

- [VS Code Extension](https://marketplace.visualstudio.com/items?itemName=ms-playwright.playwright)
- [Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Codegen](https://playwright.dev/docs/codegen)

### Community

- [Discord](https://aka.ms/playwright/discord)
- [GitHub](https://github.com/microsoft/playwright)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/playwright)

---

## Repository del Workshop

Tutto il codice, le slide e gli esercizi sono disponibili:

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

Include:

- **Demo app** (TechStore) per esercitazioni pratiche
- **Test di esempio** per ogni concetto
- **Slide** della presentazione

```bash
git clone https://github.com/monte97/workshop-playwright
cd workshop-playwright

# Avvia la demo app
cd infrastructure-demo && npm install && npm start

# Esegui i test
cd ../demo && npm install && npx playwright test --ui
```

---

*Serie Playwright Workshop:*
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. **CI/CD e Best Practices** (questo articolo)
