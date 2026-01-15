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

**Playwright**, sviluppato da Microsoft, non è semplicemente un'altra libreria di automazione. È un ripensamento architetturale del testing browser.

In questo articolo esploriamo i tre pilastri che lo rendono diverso da tutto ciò che è venuto prima.

---

## Setup in 30 Secondi

```bash
# Installa Playwright
npm init playwright@latest

# Esegui i test
npx playwright test
```

Questo è tutto. Nessuna configurazione complessa, nessun driver da scaricare, nessuna dipendenza da gestire manualmente.

Opzionalmente, l'estensione VS Code migliora ulteriormente l'esperienza:

```bash
code --install-extension ms-playwright.playwright
```

---

## I Tre Pilastri di Playwright

### 1. Affidabilità: Auto-Waiting

Il problema principale dei framework tradizionali è il timing. Quando interagisci con un elemento:

```javascript
// Selenium - Il problema
await driver.findElement(By.id('button')).click();
// Crash! L'elemento non è ancora visibile
```

Playwright risolve questo con **Auto-waiting**:

```javascript
// Playwright - La soluzione
await page.getByRole('button', { name: 'Submit' }).click();
```

Dietro le quinte, Playwright esegue automaticamente una serie di controlli prima di ogni azione:

- L'elemento esiste nel DOM?
- È visibile all'utente?
- È stabile (nessuna animazione in corso)?
- Non è coperto da altri elementi?
- È abilitato?

Solo quando **tutti** questi controlli passano, Playwright esegue l'azione.

**Web-First Assertions** seguono la stessa logica:

```javascript
// Retry automatico fino a 30 secondi (configurabile)
await expect(page.getByText('Order Confirmed')).toBeVisible();
```

Se l'elemento non è visibile, Playwright ritenta automaticamente. Niente più `sleep(5000)` in attesa che il server risponda.

---

### 2. Velocità: Parallelizzazione Nativa

Diversamente da Selenium e Cypress (che eseguono i test sequenzialmente per evitare race condition), Playwright supporta **parallelizzazione vera**:

```bash
npx playwright test --workers=4
```

Come? Grazie all'isolamento del **browser context**:

```javascript
// Ogni test ha il suo browser context isolato
test('user 1 adds to cart', async ({ page }) => {
  // Browser context #1
});

test('user 2 adds to cart', async ({ page }) => {
  // Browser context #2 (completamente isolato)
});
```

Non c'è condivisione di stato. Ogni test è una sandbox completa. Questo significa:

- 4 test in 10 secondi (vs 40 secondi in sequenza)
- Zero race condition tra test
- Debug più semplice (ogni test è indipendente)

**Risultati tipici:**

```bash
# workers=1: ~10 minuti
# workers=4: ~2.5 minuti

# 4x più veloce!
```

---

### 3. Semplicità: Un'API, Tre Browser

```bash
# Lo stesso test su Chromium, Firefox, WebKit
npx playwright test --project=chromium
npx playwright test --project=firefox
npx playwright test --project=webkit
```

Una sola API, supporto nativo per tre motori browser. Niente configurazione per-browser, niente compatibility issues.

---

## Selettori Semantici: Il Cuore del Successo

Uno dei segreti di Playwright è l'uso di **selettori semantici** basati sull'accessibility tree del browser.

### Il Problema dei Selettori Tradizionali

```javascript
// Fragile - si rompe con ogni refactoring CSS
await page.click('.btn.btn-primary.mt-2');
await page.click('#submit-form-123');
await page.click('div > form > button:nth-child(3)');
```

### La Soluzione: Selettori Semantici

```javascript
// Robusto e leggibile
await page.getByRole('button', { name: 'Submit' });
await page.getByLabel('Email Address');
await page.getByPlaceholder('Enter search query');
await page.getByText('Click here to continue');
await page.getByTestId('checkout-button');
```

Questi selettori sono:

- **Resilienti**: Non si rompono con refactoring CSS
- **Semantici**: Leggibili e comprensibili
- **Accessibili**: Rispecchiano come uno screen reader vedrebbe la pagina
- **Stabili**: `data-testid` per elementi dinamici

### Come Funziona: Accessibility Tree

Il browser assegna a ogni elemento un **ruolo ARIA** e un **nome accessibile**:

```html
<main>
  <h1>Accedi al tuo account</h1>

  <form>
    <div>
      <label for="user">Username:</label>
      <input id="user" type="text" placeholder="mario.rossi">
    </div>

    <button aria-label="Invia modulo di login">
      <svg>...</svg> Entra
    </button>
  </form>
</main>
```

Playwright vede:

```text
ROLE: generic (corrisponde a <main>)
 |
 +-- ROLE: heading, NAME: "Accedi al tuo account"
 |
 +-- ROLE: form
      |
      +-- ROLE: textbox, NAME: "Username:"
      |
      +-- ROLE: button, NAME: "Invia modulo di login"
```

Questo modello permette di trovare elementi in modo affidabile, indipendentemente da CSS, struttura HTML, o implementazione.

---

## Un Esempio Pratico: Prima e Dopo

### Senza Playwright (Il Problema)

```javascript
// Selenium/Cypress tradizionale
describe('E-commerce flow', () => {
  it('should complete purchase', async () => {
    await driver.get('http://localhost:3000/login');

    // Aggiungi sleep perché a volte non funziona
    await driver.sleep(1000);

    let input = driver.findElement(By.id('email'));
    await input.sendKeys('user@example.com');

    input = driver.findElement(By.name('password'));
    await input.sendKeys('password123');

    let button = driver.findElement(
      By.xpath('//button[contains(text(), "Login")]')
    );
    await button.click();

    // Aspetta la dashboard (sleep arbitrario!)
    await driver.sleep(3000);

    // Continua con selettori fragili...
  });
});
```

**Problemi:**
- Molti `sleep()` arbitrari
- Selettori fragili (classe, xpath)
- Timing issues
- Manutenzione difficile

### Con Playwright (La Soluzione)

```javascript
import { test, expect } from '@playwright/test';

test('should complete e-commerce flow', async ({ page }) => {
  await page.goto('http://localhost:3000/login');

  // Auto-waiting automatico
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  // Aspetta automaticamente il caricamento
  await expect(page).toHaveURL(/dashboard/);

  // Continua con selettori semantici...
  await page.getByPlaceholder('Search...').fill('laptop');
  await page.getByRole('button', { name: 'Search' }).click();
  await page.getByRole('link', { name: 'Laptop Pro' }).click();
  await page.getByRole('button', { name: 'Add to Cart' }).click();

  // Verifica con retry automatico
  await expect(page.getByTestId('cart-count')).toHaveText('1');
});
```

**Vantaggi:**
- Zero `sleep()`
- Codice leggibile (sembra inglese)
- Retry automatico sulle assertion
- Selettori semantici e stabili
- Nessuna race condition

---

## Architettura: Perché Funziona

A differenza dei suoi predecessori, Playwright stabilisce una connessione **WebSocket diretta** con il browser (utilizzando il Chrome DevTools Protocol per Chromium).

```text
┌─────────────────┐     WebSocket      ┌─────────────────┐
│  Test Script    │ ←───────────────→  │     Browser     │
│  (Node.js)      │   (bassa latenza)  │   (Chromium)    │
└─────────────────┘                    └─────────────────┘
```

Questo garantisce:
- **Bassa latenza**: Comunicazione quasi istantanea
- **Accesso privilegiato**: Controllo totale su rete, storage e eventi del browser

---

## Cosa Abbiamo Imparato

1. **Auto-waiting** elimina la fragilità legata al timing
2. **Parallelizzazione nativa** accelera il feedback loop
3. **Selettori semantici** rendono i test resilienti ai cambiamenti
4. **Un'API unificata** semplifica il cross-browser testing

Nel prossimo articolo metteremo le mani sul codice: vedremo la sintassi base, come usare **Codegen** per generare test automaticamente, e come debuggare con **UI Mode** e **Trace Viewer**.

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
