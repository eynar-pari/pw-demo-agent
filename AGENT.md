# Agent Guidelines — pw-demo-agent

## Stack

- **Framework:** Playwright Test (`@playwright/test`)
- **Language:** TypeScript
- **Pattern:** Page Object Model (POM)

---

## Rules for writing tests

### 1. Always use the Page Object Model (POM)

Every test must interact with the UI exclusively through a **Page Object class**. Raw `page` calls (`page.locator`, `page.fill`, `page.click`, etc.) are not allowed inside spec files.

**Structure:**

```
tests/
  pages/          ← Page Object classes live here
    LoginPage.ts
    InventoryPage.ts
    ...
  login.spec.ts   ← Spec files import and use Page Objects
  ...
```

**Page Object rules:**

- One class per page or major UI section.
- Declare all locators as `readonly` class properties in the constructor.
- Expose high-level action methods (`login()`, `addToCart()`) instead of leaking low-level details to specs.
- Inherit from no base class unless explicitly agreed; keep them plain TypeScript classes.

```typescript
// tests/pages/LoginPage.ts  ✅
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.locator('#user-name');
    this.passwordInput = page.locator('#password');
    this.loginButton  = page.locator('#login-button');
  }

  async goto() {
    await this.page.goto('https://www.saucedemo.com/');
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
}
```

```typescript
// tests/login.spec.ts  ✅
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test('login satisfactorio', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('standard_user', 'secret_sauce');

  await expect(page).toHaveURL(/inventory/);   // ← mandatory expect
});
```

---

### 2. The `test.describe` title must match the spec filename

The outermost `test.describe` block must use the filename (without `.spec.ts`) as its title. This keeps the Playwright HTML report readable and makes it trivial to locate the source file from a failing test.

| File | Required `test.describe` title |
|---|---|
| `login.spec.ts` | `'login'` |
| `shopping-cart.spec.ts` | `'shopping-cart'` |
| `checkout-flow.spec.ts` | `'checkout-flow'` |

```typescript
// tests/login.spec.ts  ✅
test.describe('login', () => {
  test('login satisfactorio', async ({ page }) => { ... });
});

// tests/login.spec.ts  ❌ title does not match filename
test.describe('Login Page', () => {
  test('login satisfactorio', async ({ page }) => { ... });
});
```

---

### 3. Every test must contain at least one `expect`

A test with no assertion is not a test — it is a script. **Every `test()` block must include at least one `expect()`** from `@playwright/test`.

- Prefer Playwright's auto-retrying matchers (`toHaveURL`, `toBeVisible`, `toHaveText`, `toContainText`, `toHaveValue`, …) over manual waits followed by a boolean check.
- Place assertions at the end of the test, after all actions, unless an intermediate state must be verified.
- A single logical outcome → one focused `expect`. Multiple outcomes → multiple `expect` calls, each describing a distinct assertion.

```typescript
// ❌ No assertion — not allowed
test('navigate to inventory', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('standard_user', 'secret_sauce');
});

// ✅ At least one expect
test('navigate to inventory', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('standard_user', 'secret_sauce');

  await expect(page).toHaveURL(/inventory/);
});
```

---

## Quick checklist before submitting a test

- [ ] The spec imports a Page Object class from `tests/pages/`.
- [ ] No raw `page.locator` / `page.click` calls appear inside the spec file.
- [ ] The outermost `test.describe` title matches the filename (without `.spec.ts`).
- [ ] The `test()` block contains at least one `expect()`.
- [ ] Locators are defined in the Page Object constructor, not inside action methods.
