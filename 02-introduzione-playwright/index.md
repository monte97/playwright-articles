---
title: "Introduzione a Playwright: I Tre Pilastri"
date: 2025-01-21T10:00:00+01:00
description: "Come Playwright risolve i problemi storici del testing E2E con affidabilità, velocità e semplicità"
menu:
  sidebar:
    name: "2. Intro Playwright"
    identifier: playwright-2
    weight: 20
    parent: playwright-workshop
tags: ["Playwright", "E2E Testing", "Auto-waiting", "Selettori Semantici"]
categories: ["Testing", "Workshop"]
draft: false
---

*Tempo di lettura: ~10 minuti*

Playwright è un framework open-source di Microsoft per l'automazione web.

Ma non è "un altro Selenium". È costruito su tre pilastri che risolvono i problemi storici del testing E2E.

---

## Setup: 30 Secondi

```bash
npm init playwright@latest
npx playwright test
```

Fatto. Nessuna configurazione complessa, nessun driver da scaricare.

---

## Pilastro 1: Affidabilità (Auto-Waiting)

Il problema principale dei framework vecchi è il timing.

```javascript
// Selenium - Il problema
await driver.findElement(By.id('button')).click();
// Crash! L'elemento non è ancora visibile
```

**Playwright aspetta automaticamente:**

```javascript
await page.getByRole('button', { name: 'Submit' }).click();
```

Prima di ogni azione, Playwright verifica:
- L'elemento esiste nel DOM?
- È visibile?
- È stabile (nessuna animazione)?
- Non è coperto da altri elementi?
- È abilitato?

Solo quando **tutti** i controlli passano, esegue l'azione.

**Le assertion funzionano allo stesso modo:**

```javascript
// Retry automatico fino a 30 secondi
await expect(page.getByText('Ordine confermato')).toBeVisible();
```

Se l'elemento non appare subito, Playwright ritenta. Niente più `sleep(5000)`.

---

## Pilastro 2: Velocità (Parallelizzazione)

Selenium e Cypress eseguono i test in sequenza. Playwright li esegue in parallelo.

```bash
npx playwright test --workers=4
```

Come? Ogni test ha il suo **browser context isolato**:

```javascript
test('user 1 aggiunge al carrello', async ({ page }) => {
  // Browser context #1
});

test('user 2 aggiunge al carrello', async ({ page }) => {
  // Browser context #2 - completamente isolato
});
```

Nessuna condivisione di stato. Ogni test è una sandbox.

**I numeri:**

```bash
workers=1: ~10 minuti
workers=4: ~2.5 minuti

# 4x più veloce
```

---

## Pilastro 3: Semplicità (Una API, Tre Browser)

```bash
npx playwright test --project=chromium
npx playwright test --project=firefox
npx playwright test --project=webkit
```

Una sola API per tutti i browser. Niente configurazioni separate.

---

## Selettori Semantici

Questo è il cuore di Playwright.

**Il problema dei selettori tradizionali:**

```javascript
// Fragile - si rompe con ogni refactoring CSS
await page.click('.btn.btn-primary.mt-2');
await page.click('#submit-form-123');
await page.click('div > form > button:nth-child(3)');
```

**Selettori semantici:**

```javascript
// Robusto
await page.getByRole('button', { name: 'Submit' });
await page.getByLabel('Email');
await page.getByPlaceholder('Cerca...');
await page.getByText('Continua');
await page.getByTestId('checkout-button');
```

Perché funzionano:
- **Resilienti**: Non si rompono con refactoring CSS
- **Leggibili**: Sembrano inglese
- **Accessibili**: Seguono le best practice a11y

### Come Funziona

Il browser assegna a ogni elemento un **ruolo ARIA** e un **nome accessibile**:

```html
<button aria-label="Invia modulo">
  <svg>...</svg> Invia
</button>
```

Playwright vede:

```text
ROLE: button, NAME: "Invia modulo"
```

Indipendentemente dal CSS o dalla struttura HTML.

---

## Prima e Dopo

**Senza Playwright:**

```javascript
await driver.get('http://localhost:3000/login');

// Sleep perché a volte non funziona
await driver.sleep(1000);

let input = driver.findElement(By.id('email'));
await input.sendKeys('user@example.com');

input = driver.findElement(By.name('password'));
await input.sendKeys('password123');

let button = driver.findElement(
  By.xpath('//button[contains(text(), "Login")]')
);
await button.click();

// Sleep ancora...
await driver.sleep(3000);
```

**Con Playwright:**

```javascript
await page.goto('http://localhost:3000/login');

await page.getByLabel('Email').fill('user@example.com');
await page.getByLabel('Password').fill('password123');
await page.getByRole('button', { name: 'Login' }).click();

await expect(page).toHaveURL(/dashboard/);
```

Zero `sleep()`. Codice leggibile. Selettori stabili.

---

## Cosa Ho Imparato

1. **Auto-waiting** elimina i test flaky legati al timing
2. **Parallelizzazione** riduce i tempi di 4x
3. **Selettori semantici** rendono i test resistenti ai cambiamenti
4. **Una API unificata** semplifica il cross-browser testing

Nel prossimo articolo mettiamo le mani sul codice: sintassi, **Codegen**, **UI Mode** e **Trace Viewer**.

---

## Repository

👉 **[workshop-playwright](https://github.com/monte97/workshop-playwright)**

---

*Serie Playwright Workshop:*
1. [Il Gap che Nessun Unit Test Può Colmare]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. **Introduzione a Playwright: I Tre Pilastri** (questo articolo)
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
