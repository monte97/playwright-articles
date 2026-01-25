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

[Playwright](https://playwright.dev/) è un framework open-source di Microsoft per l'automazione web, progettato specificamente per le applicazioni web moderne. Risolve i problemi storici del testing E2E attraverso tre pilastri: affidabilità, velocità e semplicità.

---

## Setup

```bash
npm init playwright@latest
npx playwright test
```

L'installazione scarica automaticamente tutte le dipendenze necessarie e non richiede configurazione di driver esterni. Playwright è pronto all'uso in pochi secondi.

---

## Pilastro 1: Affidabilità (Auto-Waiting)

Il problema principale dei test E2E è la gestione del timing. In un'applicazione web moderna, gli elementi non sono sempre immediatamente disponibili.

Un approccio tradizionale potrebbe fallire in questo scenario:
```javascript
// Causa del fallimento: l'elemento non è ancora visibile
button.click();
```

**Playwright aspetta automaticamente:**

```javascript
await page.getByRole('button', { name: 'Submit' }).click();
```

Prima di ogni azione, Playwright esegue una serie di [controlli di "azionabilità" (actionability checks)](https://playwright.dev/docs/actionability):
- L'elemento esiste nel DOM?
- È visibile?
- È stabile (nessuna animazione in corso)?
- Non è oscurato da altri elementi?
- È abilitato a ricevere eventi?

Solo quando **tutti** i controlli passano, Playwright esegue l'azione. Questo elimina la causa più comune di test "flaky".

**Anche le assertion sono intelligenti:**

```javascript
// Playwright attende che l'elemento diventi visibile
await expect(page.getByText('Ordine confermato')).toBeVisible();
```

Se l'elemento non è subito presente, `expect` attenderà (con un timeout configurabile), riprovando la condizione. Questo processo di attesa automatica è il cuore dell'affidabilità di Playwright.

---

## Pilastro 2: Velocità (Parallelizzazione)

Molti framework eseguono i test in sequenza per evitare interferenze. Playwright è progettato per eseguire i test in parallelo fin dal primo giorno.

```bash
# Esegue i test usando 4 worker in parallelo
npx playwright test --workers=4
```

Come è possibile? Ogni file di test viene eseguito in un **Browser Context** separato e isolato.

### Cos'è un Browser Context

Un [Browser Context](https://playwright.dev/docs/core-concepts#browser-contexts) è il concetto chiave che permette a Playwright di eseguire test in parallelo in modo sicuro.

Immagina di aprire una finestra di navigazione in incognito: ha i propri cookie, il proprio `localStorage`, il proprio `sessionStorage`, completamente separati dalla sessione principale. Un Browser Context funziona allo stesso modo, ma è più efficiente perché non richiede di avviare un nuovo processo.

In pratica:
- **Ogni test gira nel suo context isolato**: un test che effettua il login non influenza lo stato di un altro test.
- **I context condividono lo stesso processo**: questo rende la creazione di nuovi context quasi istantanea, a differenza dell'avvio di un nuovo processo.
- **Playwright crea automaticamente un context per ogni worker**: non devi gestire nulla manualmente.

```javascript
// File: test-1.spec.js
test('L_utente 1 aggiunge un prodotto al carrello', async ({ page }) => {
  // Questo test viene eseguito nel suo Browser Context #1
  // Ha i propri cookie e il proprio stato.
});

// File: test-2.spec.js
test('L_utente 2 naviga il suo profilo', async ({ page }) => {
  // Questo test è completamente isolato nel suo Browser Context #2
  // Non può vedere i dati dell'utente 1.
});
```

Questa architettura garantisce che i test siano eseguibili in parallelo in modo sicuro e affidabile, senza che lo stato di un test possa "inquinare" l'altro.

**L'impatto sulla velocità è notevole:**

```bash
# Esempio con una suite di test
workers=1: ~10 minuti
workers=4: ~2.5 minuti

# Un miglioramento di 4 volte
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

Il browser assegna a ogni elemento un [**ruolo ARIA**](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles) e un [**nome accessibile**](https://www.w3.org/WAI/ARIA/apg/practices/names-and-descriptions/):

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

### Accessibilità come Strategia di Test

L'approccio di Playwright ai selettori ha un vantaggio che va oltre la robustezza: **promuove l'accessibilità (a11y)**.

#### Perché i Selettori Semantici Migliorano l'Accessibilità

Quando scrivi un test che usa `getByRole('button', { name: 'Invia' })`, stai facendo esattamente quello che fa uno screen reader: cerchi un elemento con un ruolo specifico (button) e un nome accessibile (Invia).

Se il test non trova l'elemento, ci sono due possibilità:
1. L'elemento non esiste (bug nel codice)
2. L'elemento esiste ma non è accessibile (bug di accessibilità)

In entrambi i casi, il test ti sta dando informazioni utili.

#### Esempio Pratico

```html
<!-- Questo bottone NON è accessibile -->
<div class="btn" onclick="submit()">Invia</div>

<!-- Questo bottone È accessibile -->
<button type="submit">Invia</button>
```

Un test con `getByRole('button', { name: 'Invia' })` troverà solo il secondo elemento. Il primo `<div>` non ha il ruolo semantico di button, quindi non sarà trovato né dal test né da uno screen reader.

#### I Benefici Concreti

Adottare selettori semantici significa:

1. **Testare come un utente reale**: Inclusi gli utenti che usano tecnologie assistive.
2. **Feedback immediato sull'accessibilità**: Se `getByLabel('Email')` fallisce, probabilmente manca un'associazione `<label>` corretta.
3. **Test più stabili**: I selettori si basano su cosa l'elemento *è* (un button, un input), non su come *appare* (classi CSS, struttura DOM).
4. **Documentazione vivente**: Leggendo i test, capisci quali elementi sono interattivi e come sono etichettati.

Questo trasforma i test E2E da un semplice strumento di verifica a un meccanismo per costruire applicazioni più inclusive, in linea con le [linee guida WCAG (Web Content Accessibility Guidelines)](https://www.w3.org/WAI/standards-guidelines/wcag/).

---

## Approccio Tradizionale vs. Playwright

**Approccio Tradizionale:**

Un test scritto senza meccanismi di attesa automatica è spesso verboso e fragile. Richiede `sleep` manuali o `waitFor` espliciti per sincronizzare il test con lo stato dell'applicazione, rendendolo lento e inaffidabile.

```javascript
// Esempio concettuale di un test tradizionale
await page.goto('http://localhost:3000/login');

// Potrebbe essere necessario attendere esplicitamente che la pagina carichi
await waitForPageLoad(); 

await page.type('#email', 'user@example.com');
await page.type('#password', 'password123');

// Bisogna attendere il bottone prima di cliccarlo
await waitForElement('#login-button');
await page.click('#login-button');

// E attendere la navigazione o l'aggiornamento dell'UI
await sleep(3000); 
```

**Con Playwright:**

Il codice è più conciso, leggibile e robusto, perché Playwright gestisce la sincronizzazione in modo automatico.

```javascript
await page.goto('http://localhost:3000/login');

await page.getByLabel('Email').fill('user@example.com');
await page.getByLabel('Password').fill('password123');
await page.getByRole('button', { name: 'Login' }).click();

// Playwright attende implicitamente la navigazione
await expect(page).toHaveURL(/dashboard/);
```

Il risultato è un test che non solo è più pulito, ma anche più affidabile.

---

## Riepilogo

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
1. [Perché i Test E2E Sono un Pilastro della Quality Assurance]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. **Introduzione a Playwright: I Tre Pilastri** (questo articolo)
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
