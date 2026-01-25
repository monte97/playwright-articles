---
title: "CI/CD, Strategie Avanzate e Futuro del Testing"
date: 2025-01-24T10:00:00+01:00
description: "Integrazione in CI/CD, sharding, mobile e API testing, e uno sguardo al futuro con l'Agent-Driven Development"
menu:
  sidebar:
    name: "5. CI/CD & Strategie"
    identifier: playwright-5
    weight: 50
    parent: playwright-workshop
tags: ["Playwright", "CI/CD", "GitHub Actions", "Sharding", "Agent-Driven Development"]
categories: ["Testing", "DevOps", "Workshop"]
draft: false
---

*Tempo di lettura: ~15 minuti*

Abbiamo visto come scrivere test robusti e come architettare una suite di test scalabile. L'ultimo passo è integrare questi test nel ciclo di sviluppo, in modo che vengano eseguiti automaticamente e forniscano feedback rapidi a ogni modifica del codice.

Questa sezione si concentra su come portare i test Playwright in produzione, integrandoli in pipeline di **Continuous Integration / Continuous Deployment (CI/CD)**. Vedremo anche strategie avanzate come il testing su dispositivi mobili, l'API testing per velocizzare il setup e uno sguardo al futuro del testing.

---

## Integrazione in CI/CD

L'obiettivo della CI è semplice: eseguire la suite di test in un ambiente pulito e automatico ogni volta che viene proposto un cambiamento (es. in una Pull Request). Se i test passano, il codice è considerato sicuro da integrare.

Playwright è progettato per la CI. Fornisce [reporter dedicati](https://playwright.dev/docs/ci-intro#ci-reporters) (es. per GitHub Actions), configurazioni specifiche e strumenti per analizzare i fallimenti.

### Esempio con GitHub Actions

[GitHub Actions](https://github.com/features/actions) è uno degli strumenti di CI/CD più popolari. Ecco un workflow completo e commentato per eseguire i test Playwright.

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

# Trigger: esegui su push a 'main'/'develop' e su Pull Request verso 'main'
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: 'Playwright Tests'
    # Esegui su una macchina Ubuntu fresca
    runs-on: ubuntu-latest
    # Timeout per l'intero job per evitare che rimanga bloccato
    timeout-minutes: 30

    steps:
      # 1. Clona il repository
      - uses: actions/checkout@v4

      # 2. Imposta l'ambiente Node.js
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      # 3. Installa le dipendenze del progetto
      - name: Install dependencies
        run: npm ci

      # 4. Installa i browser richiesti da Playwright
      #    --with-deps installa anche le dipendenze di sistema (un must in CI)
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      # 5. Esegui i test Playwright
      - name: Run Playwright tests
        run: npx playwright test

      # 6. Carica il report dei test come artefatto
      #    'if: always()' assicura che il report venga caricato anche se i test falliscono
      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7 # Conserva il report per 7 giorni
```

> 💡 **Azione Ufficiale Playwright**: Per semplificare ulteriormente, puoi usare l'azione ufficiale [`microsoft/playwright-github-action`](https://github.com/microsoft/playwright-github-action). Questa action ottimizza l'installazione e il caching delle dipendenze del browser, rendendo la pipeline più veloce e concisa.

### Altri Sistemi di CI

Il pattern è simile per altri sistemi come [GitLab CI](https://docs.gitlab.com/ee/ci/), Jenkins, o CircleCI. La maggior parte dei fornitori offre container Docker pre-configurati con Playwright, come `mcr.microsoft.com/playwright`, che semplificano ulteriormente il setup. La [guida ufficiale di Playwright sulla CI](https://playwright.dev/docs/ci-intro) contiene esempi per diverse piattaforme.

---

## Strategie per la CI

Eseguire test in CI richiede strategie diverse rispetto all'ambiente locale. L'obiettivo è ottenere feedback nel modo più rapido e affidabile possibile.

### Configurazione Specifica per la CI

È buona norma avere impostazioni diverse per la CI nel file `playwright.config.ts`.

```typescript
// playwright.config.ts
const isCI = !!process.env.CI;

export default defineConfig({
  // In CI, usa meno worker perché le macchine virtuali hanno meno CPU.
  workers: isCI ? 2 : undefined,

  // Abilita i tentativi solo in CI. In locale, un test deve passare al primo colpo.
  retries: isCI ? 2 : 0,

  // Usa reporter diversi. In CI, vogliamo un output per la console ('list' o 'github')
  // e il report HTML da caricare come artefatto.
  reporter: isCI ? 'github' : 'list',

  use: {
    // Registra un trace solo quando un test viene ritentato, per non sprecare risorse.
    trace: 'on-first-retry',
  },
});
```

### Parallelizzazione su Larga Scala: Sharding

Quando una suite di test cresce fino a richiedere più di 10-15 minuti, anche con la parallelizzazione su una singola macchina, è il momento di usare lo **sharding**.

Lo [sharding](https://playwright.dev/docs/test-parallel#sharding) è una tecnica che permette di dividere la suite di test per eseguirla su **più macchine in parallelo**. Ad esempio, se hai 200 test e usi 4 macchine (shard), ogni macchina eseguirà solo 50 test, riducendo il tempo totale di circa 4 volte.

**Esempio di sharding con GitHub Actions:**

```yaml
jobs:
  test:
    name: 'Playwright Tests'
    runs-on: ubuntu-latest
    # Definisci una 'matrix' per creare job paralleli
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4] # Crea 4 job, uno per ogni shard

    steps:
      # ... (checkout, setup-node, etc.) ...

      - name: Run Playwright tests
        # Passa lo shard corrente e il numero totale di shard a Playwright
        run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
      
      # ... (upload artifact) ...
```
Questo è il modo più efficace per mantenere i tempi di esecuzione bassi anche con migliaia di test.

## Strategie Avanzate di Test

Oltre ai test E2E standard, Playwright eccelle in altre due aree che sono fondamentali per una strategia di test completa: il testing su dispositivi mobili e il testing delle API.

### Mobile Testing: Emulazione di Viewport e Touch

Il [testing su dispositivi mobili](https://playwright.dev/docs/emulation) con Playwright non richiede un dispositivo fisico. Playwright utilizza l'emulazione integrata di Chromium per simulare viewport, user agent, e eventi touch di centinaia di dispositivi mobili.

Questo si integra perfettamente nella configurazione a progetti.

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // Progetto per desktop
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] },
    },
    // Progetto per iPhone
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13 Pro'] },
    },
    // Progetto per Android
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
});
```
Eseguendo `npx playwright test --project="Mobile Safari"`, i test verranno eseguiti in un ambiente che simula un iPhone. Questo permette di verificare la reattività del layout e il corretto funzionamento degli eventi touch (`page.tap()`) direttamente nella pipeline di CI.

### API Testing: Stabilità e Velocità

Come visto nell'articolo precedente, l'interazione diretta con le API è uno strumento potentissimo. Il [testing delle API con Playwright](https://playwright.dev/docs/api-testing) non serve solo a verificare i backend, ma anche a rendere i test UI più veloci e robusti.

**Casi d'uso principali:**
1.  **Verifica diretta degli endpoint**: Scrivere test che controllano lo stato, la risposta e lo schema di un'API, indipendentemente dall'interfaccia utente.
2.  **Setup e teardown dello stato**: Invece di usare l'UI per creare un utente, aggiungere un prodotto al carrello o configurare uno stato specifico, fallo con una chiamata API all'inizio del test. È ordini di grandezza più veloce e meno fragile.

**Esempio di setup via API:**

```typescript
test.beforeEach('Crea un prodotto di test', async ({ request }) => {
  // Questa chiamata API assicura che il prodotto esista prima che il test parta
  await request.post('/api/products', {
    data: { id: 'test-product-123', name: 'Prodotto di Test', price: 99 },
  });
});

test('il prodotto creato via API è visibile nella UI', async ({ page }) => {
  await page.goto('/products/test-product-123');
  await expect(page.getByRole('heading', { name: 'Prodotto di Test' })).toBeVisible();
});
```
Questa strategia ibrida (setup via API, verifica via UI) è una delle best practice più efficaci per una suite di test E2E matura.

## Oltre i Test Tradizionali: Uno Sguardo al Futuro

Finora abbiamo discusso pratiche consolidate, ma il mondo dello sviluppo è in continua evoluzione. L'approccio che Playwright promuove, basato su selettori semantici e test che descrivono il comportamento dell'utente, apre la porta a due paradigmi emergenti: il **Documentation-Driven Development** e l'**Agent-Driven Development**.

### Documentation-Driven Development

Quando i test E2E sono scritti in modo leggibile e dichiarativo, non sono più solo uno strumento di verifica, ma diventano una **documentazione vivente ed eseguibile** del sistema. Un test come:

```javascript
test('Un utente non autenticato viene rediretto alla pagina di login', ...)
```
è più chiaro di qualsiasi documentazione statica, perché è costantemente verificato contro il sistema reale.

Un team che adotta questo approccio può usare la propria suite di test come fonte primaria di verità sul comportamento dell'applicazione.

### Agent-Driven Development

Guardando ancora più avanti, una suite di test ben strutturata e basata sull'accessibilità è il prerequisito per l'**Agent-Driven Development**. Immaginiamo un futuro in cui un agente AI (un "developer agent") riceve un compito come: *"Implementa la possibilità per un utente di aggiornare il proprio indirizzo email dalla pagina del profilo."*

Per completare questo task, l'agente ha bisogno di:
1.  **Comprendere lo stato attuale del sistema**: l'agente potrebbe "leggere" i test esistenti per capire come funziona il flusso di login, come accedere alla pagina del profilo, e quali elementi sono presenti. I selettori semantici (`getByRole`, `getByLabel`) sono un linguaggio che sia l'uomo che la macchina possono interpretare.
2.  **Verificare il proprio lavoro**: una volta implementata la funzionalità, l'agente deve poter scrivere un nuovo test E2E per validare il suo operato, seguendo i pattern già presenti nella codebase.

Senza test chiari e semantici, questo livello di automazione è impossibile. Investire oggi in una suite di test di alta qualità non è solo una buona pratica, ma un modo per preparare la propria codebase alla prossima generazione di strumenti di sviluppo software.

Playwright, con la sua enfasi sull'esperienza utente e sull'accessibilità, non è solo un ottimo strumento per il testing di oggi, ma una base solida per l'automazione di domani.

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
1. [Perché i Test E2E Sono un Pilastro della Quality Assurance]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. [Architettura e Pattern per Test Scalabili]({{< relref "/posts/playwright-workshop/04-architettura-pattern" >}})
5. **CI/CD, Strategie Avanzate e Futuro del Testing** (questo articolo)
