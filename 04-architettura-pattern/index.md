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

*Tempo di lettura: ~15 minuti*

Quando una suite di test cresce, passando da dieci a centinaia di test, emergono sfide che non sono evidenti all'inizio. Il problema non è solo la quantità di codice, ma la sua **struttura**. Senza un'architettura solida, la suite di test diventa lenta, difficile da manutenere e incomprensibile.

Questa sezione presenta pattern architetturali consolidati per scrivere test Playwright che siano **scalabili**, **robusti** e **leggibili**.

---

## Il Problema: Test Che Non Scalano

All'inizio, è naturale scrivere test lineari che replicano le azioni dell'utente:

```javascript
test('L_utente può visualizzare il suo profilo', async ({ page }) => {
  // Setup: login
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
  await expect(page).toHaveURL(/dashboard/);

  // Azione: navigazione al profilo
  await page.getByRole('link', { name: 'Il Mio Profilo' }).click();

  // Verifica
  await expect(page.getByText('Dettagli utente')).toBeVisible();
});
```

Questo approccio funziona per pochi test, ma quando decine di test richiedono un utente autenticato, iniziano i problemi:

1.  **Manutenzione Elevata**: Se il selettore del bottone di login cambia, devi aggiornarlo in decine di file. Il costo di ogni modifica diventa insostenibile.
2.  **Scarsa Leggibilità**: Il "cosa" sta testando (la visualizzazione del profilo) è annegato nel "come" si arriva a quello stato (il flusso di login). I test diventano difficili da comprendere a colpo d'occhio.
3.  **Lentezza**: Eseguire il login tramite l'interfaccia utente per ogni singolo test aggiunge secondi preziosi. Una suite di 100 test può sprecare minuti solo per operazioni di setup ripetitive.

Risolvere questi tre problemi—manutenzione, leggibilità e velocità—è lo scopo dei pattern che seguono.

---

## Pattern 1: Page Object Model (POM)



Il [Page Object Model](https://martinfowler.com/bliki/PageObject.html) è un design pattern che risolve il problema della duplicazione del codice e della manutenibilità. L'idea è semplice: incapsulare le interazioni con una pagina (o un componente riutilizzabile) all'interno di una classe.



Questa classe, il "Page Object", espone metodi che rappresentano le azioni che un utente può compiere, nascondendo i dettagli implementativi (i selettori e le azioni di basso livello).



**Esempio: `pages/LoginPage.ts`**



```typescript

import { Page, expect } from '@playwright/test';



export class LoginPage {

  // I selettori sono definiti in un unico posto

  private readonly emailInput = this.page.getByLabel('Email');

  private readonly passwordInput = this.page.getByLabel('Password');

  private readonly loginButton = this.page.getByRole('button', { name: 'Login' });



  constructor(private page: Page) {}



  async goto() {

    await this.page.goto('/login');

  }



  async login(email: string, password: string) {

    await this.emailInput.fill(email);

    await this.passwordInput.fill(password);

    await this.loginButton.click();

  }

}

```



**Come cambia il test:**



```typescript

import { test, expect } from '@playwright/test';

import { LoginPage } from '../pages/LoginPage';



test('login con successo', async ({ page }) => {

  const loginPage = new LoginPage(page);



  await loginPage.goto();

  await loginPage.login('user@test.com', 'password123');



  // La verifica rimane nel test

  await expect(page).toHaveURL(/dashboard/);

});

```



**Vantaggi:**



1.  **Manutenibilità**: Se un selettore della pagina di login cambia, lo modifichi in un solo file: `LoginPage.ts`. Tutti i test che lo usano si aggiornano automaticamente.

2.  **Leggibilità**: Il test diventa più dichiarativo. `loginPage.login(...)` esprime chiaramente l'**intento** dell'azione, non i dettagli tecnici di come si compila un form.

3.  **Riutilizzabilità**: Il `LoginPage` può essere riutilizzato in decine di test, riducendo drasticamente la duplicazione.



> 💡 **Vedi l'esempio nel progetto demo:** nel file [`demo/fixtures/user.fixture.ts`](https://github.com/monte97/workshop-playwright/blob/main/demo/fixtures/user.fixture.ts) viene usato un approccio simile per centralizzare le azioni relative all'utente.

---

## Pattern 2: Ottimizzare il Setup con le Fixture e il Global Setup

Il Page Object Model migliora la manutenibilità, ma non risolve il problema della velocità: ogni test esegue ancora il login tramite l'UI.

Per risolvere questo, Playwright offre due meccanismi potenti: le [**Custom Fixtures**](https://playwright.dev/docs/test-fixtures) e il [**Global Setup**](https://playwright.dev/docs/test-global-setup-teardown).

### 2a. Setup per-file con le Fixture

Una "fixture" è una funzione che prepara l'ambiente per un test. Possiamo creare una fixture custom che esegue il login e fornisce al test una pagina già autenticata.

**Esempio: `fixtures/user.fixture.ts`**

```typescript
import { test as baseTest } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

// Estendiamo il test di base con la nostra fixture
export const test = baseTest.extend<{ authenticatedPage: Page }>({
  // La fixture si chiama 'authenticatedPage'
  authenticatedPage: async ({ page }, use) => {
    // 1. Esegui il setup
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('user@example.com', 'password123');
    await expect(page).toHaveURL(/dashboard/);

    // 2. Fornisci la risorsa al test
    await use(page);

    // 3. (Opzionale) Esegui il teardown dopo il test
    console.log('Test completato, la fixture può fare pulizia se necessario.');
  },
});
```

**Come cambia il test:**

```typescript
// Importiamo il nostro 'test' custom invece di quello di @playwright/test
import { test } from '../fixtures/user.fixture';
import { expect } from '@playwright/test';

test('L_utente può visualizzare il suo profilo', async ({ authenticatedPage }) => {
  // Il test parte già autenticato!
  // La 'page' originale è ancora disponibile, ma usiamo quella autenticata.
  await authenticatedPage.getByRole('link', { name: 'Profile' }).click();
  await expect(authenticatedPage.getByText('Your Profile')).toBeVisible();
});
```

**Vantaggi:**
*   **Separazione delle responsabilità**: La logica di setup (login) è completamente separata dalla logica di test (verifica del profilo). Il test si concentra solo sul suo scopo.
*   **Leggibilità**: È immediatamente chiaro che il test richiede uno stato "autenticato".

> **Nota di performance**: Questo approccio è ottimo per l'astrazione, ma esegue il login **una volta per ogni file di test** che usa la fixture. Per ottimizzare ulteriormente, si usa il setup globale.

### 2b. Setup una-tantum con il Global Setup e `storageState`

Per la massima velocità, il login deve essere eseguito **una sola volta per l'intera suite di test**.

Il pattern, descritto nella [documentazione ufficiale di Playwright sull'autenticazione](https://playwright.dev/docs/auth), consiste in:
1.  Creare un test di "setup" speciale che esegue il login e salva lo stato della sessione (cookie, local storage) in un file JSON.
2.  Configurare i test normali per usare quel file di stato, bypassando completamente il login via UI.

**1. Creare il file di setup: `tests/auth.setup.ts`**

Questo file contiene un singolo test che si occupa solo dell'autenticazione.

```typescript
import { test as setup, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

// Il percorso dove salvare lo stato di autenticazione
const authFile = 'playwright/.auth/user.json';

setup('autenticazione', async ({ page }) => {
  // Esegui il login
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');
  await expect(page).toHaveURL(/dashboard/);

  // Salva lo stato di autenticazione nel file
  await page.context().storageState({ path: authFile });
});
```

**2. Aggiornare `playwright.config.ts` per usare il setup**

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // Progetto di setup: esegue solo il file di setup
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },

    // Progetto principale: dipende dal setup e usa lo stato salvato
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        // Applica lo stato di autenticazione salvato a ogni test
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'], // Assicura che 'setup' venga eseguito prima
    },
    // ... altri progetti (firefox, webkit) configurati allo stesso modo
  ],
});
```

**Risultato:**

*   Il login UI viene eseguito **una sola volta** all'inizio di tutta la suite.
*   Tutti i test successivi partono **immediatamente** come autenticati, caricando lo stato dal file `user.json`.
*   Il risparmio di tempo è enorme su suite di grandi dimensioni.

> 💡 **Vedi l'esempio nel progetto demo:** questa è esattamente la strategia usata nel nostro workshop. Controlla i file [`demo/tests/auth.setup.ts`](https://github.com/monte97/workshop-playwright/blob/main/demo/tests/auth.setup.ts) e [`demo/playwright.config.ts`](https://github.com/monte97/workshop-playwright/blob/main/demo/playwright.config.ts).

---

## Pattern 3: Isolamento dei Test per una Parallelizzazione Sicura

Playwright esegue i test in parallelo per impostazione predefinita, una feature che riduce drasticamente i tempi di esecuzione. Tuttavia, la parallelizzazione introduce una nuova sfida: **l'isolamento dei dati**.

```typescript
// playwright.config.ts
export default defineConfig({
  // Esegue i file in parallelo. Se un file ha più test, esegue anche quelli in parallelo.
  fullyParallel: true,
  // Usa un numero di worker basato sulle CPU disponibili.
  workers: process.env.CI ? 2 : undefined,
});
```

**Il problema**: se due test eseguiti in parallelo modificano la stessa risorsa (es. lo stesso account utente), si creeranno delle race condition e i test falliranno in modo imprevedibile.

```javascript
// Test 1 (worker 1): Aggiunge "Laptop" al carrello dell'utente `user@test.com`.
// Si aspetta che il carrello contenga 1 prodotto.

// Test 2 (worker 2): Aggiunge "Smartphone" al carrello dello stesso utente `user@test.com`.
// Si aspetta che il carrello contenga 1 prodotto.
```
Entrambi i test falliranno, perché il carrello alla fine conterrà due prodotti.

**La soluzione**: ogni test deve operare su un set di dati completamente isolato. La strategia più robusta per ottenere questo è creare i dati necessari al test (es. un nuovo utente) all'inizio del test e distruggerli alla fine, tipicamente tramite chiamate API.

Playwright fornisce un oggetto [`request`](https://playwright.dev/docs/api-testing) per effettuare chiamate API direttamente all'interno dei test.

**Esempio di fixture per un utente isolato:**

```typescript
// fixtures/user.fixture.ts
export const test = baseTest.extend<{ isolatedUser: UserData }>({
  isolatedUser: async ({ request }, use) => {
    // 1. Creare dati univoci per l'utente
    const userId = `test-user-${Date.now()}`;
    const userData = {
      email: `${userId}@example.com`,
      password: 'password123',
    };

    // 2. Creare l'utente tramite chiamata API
    const response = await request.post('/api/users', { data: userData });
    expect(response.ok()).toBeTruthy();

    // 3. Fornire i dati dell'utente al test
    await use(userData);

    // 4. Pulizia: eliminare l'utente alla fine del test
    await request.delete(`/api/users/${userId}`);
  },
});
```

**Come cambia il test:**

```typescript
import { test } from '../fixtures/user.fixture';

test('il checkout deve funzionare per un utente isolato', async ({ page, isolatedUser }) => {
  // Questo utente esiste solo per questo test e per nessun altro
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login(isolatedUser.email, isolatedUser.password);
  
  // ... procedi con il test di checkout ...
});
```

Con questo pattern, ogni test è una sandbox perfetta. Puoi eseguire centinaia di test in parallelo con la certezza che non interferiranno mai tra loro.

> 💡 **Vedi l'esempio nel progetto demo:** il file [`demo/tests/esercizio4b-02-soluzione-parallel.spec.ts`](https://github.com/monte97/workshop-playwright/blob/main/demo/tests/esercizio4b-02-soluzione-parallel.spec.ts) mostra una soluzione pratica a un problema di parallelismo.

---

## Pattern 4: Sincronizzazione Esplicita con la Rete

L'auto-waiting di Playwright è eccellente per attendere che gli elementi diventino visibili e interagibili. Tuttavia, non può sapere quando un'operazione di background, come una chiamata API per salvare dati, è stata completata.

**Il problema (una classica *race condition*):**

```javascript
// 1. L'utente clicca 'Salva'
await page.getByRole('button', { name: 'Salva' }).click();

// 2. Il test cerca subito il messaggio di successo
await expect(page.getByText('Prodotto salvato con successo!')).toBeVisible();
```

Questo test è "flaky" (inaffidabile). A volte passa (se l'API è veloce), a volte fallisce (se l'API è lenta). L'interfaccia utente potrebbe mostrare un messaggio di successo *ottimisticamente*, prima ancora che il server abbia confermato il salvataggio.

Affidarsi a `sleep` o attese arbitrarie è una pessima pratica che porta a test lenti e fragili.

**La soluzione**: Sincronizzare il test direttamente con la risposta della rete. Il test deve attendere che la chiamata API pertinente sia terminata con successo.

Playwright permette di farlo in modo elegante con [`page.waitForResponse()`](https://playwright.dev/docs/api/class-page#page-wait-for-response).

```typescript
// Avvia l'attesa per la risposta E l'azione che la scatena, in parallelo
const [response] = await Promise.all([
  // 1. Attendi la risposta API. Puoi filtrare per URL, status, etc.
  page.waitForResponse(
    res => res.url().includes('/api/products') && res.status() === 201
  ),
  
  // 2. Esegui l'azione che triggera la chiamata API
  page.getByRole('button', { name: 'Salva' }).click(),
]);

// 3. A questo punto, hai la garanzia che il salvataggio è completato.
// Ora puoi verificare l'esito con sicurezza.
await expect(page.getByText('Prodotto salvato con successo!')).toBeVisible();

// Puoi anche fare assertion sul corpo della risposta API
const responseBody = await response.json();
expect(responseBody.id).not.toBeNull();
```

Questo pattern elimina le race condition e rende i test **deterministici** e **affidabili**, indipendentemente dalla velocità della rete o del backend.

---

## Struttura della Cartella Consigliata

Una buona struttura di cartelle è essenziale per la manutenibilità. Ecco un layout che funziona bene per progetti di medie e grandi dimensioni, che riflette i pattern discussi.

```
demo/
├── playwright/
│   └── .auth/
│       └── user.json         # Stato di autenticazione salvato
├── fixtures/
│   └── user.fixture.ts     # Fixture custom per l'utente
├── pages/
│   └── LoginPage.ts        # Esempio di Page Object
├── tests/
│   ├── auth.setup.ts       # Test di setup per l'autenticazione globale
│   ├── esercizio1.spec.ts
│   └── ...
└── playwright.config.ts
```

> 💡 **Questa struttura è applicata nel [progetto `demo`](https://github.com/monte97/workshop-playwright/tree/main/demo)**. Esplora il codice per vedere come i diversi pezzi si collegano tra loro.

---

## Riepilogo

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
1. [Perché i Test E2E Sono un Pilastro della Quality Assurance]({{< relref "/posts/playwright-workshop/01-perche-e2e" >}})
2. [Introduzione a Playwright: I Tre Pilastri]({{< relref "/posts/playwright-workshop/02-introduzione-playwright" >}})
3. [I Tuoi Primi Test con Playwright]({{< relref "/posts/playwright-workshop/03-primi-test" >}})
4. **Architettura e Pattern per Test Scalabili** (questo articolo)
5. [CI/CD e Best Practices]({{< relref "/posts/playwright-workshop/05-cicd-best-practices" >}})
