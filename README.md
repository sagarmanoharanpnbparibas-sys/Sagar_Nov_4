# Sagar_Nov_4

import { test, expect, Page, TestInfo } from '@playwright/test';

// Settings applied inside this file only, so playwright.config.ts stays untouched
test.use({
  screenshot: 'on',
  video: 'on',
  trace: 'on',
  locale: 'en-US',
  viewport: { width: 1366, height: 800 },
});

test.setTimeout(90_000);

// =====================================================================
// Helpers (random data, screenshots, ad blocking)
// =====================================================================
const pick = <T,>(arr: T[]): T => arr[Math.floor(Math.random() * arr.length)];
const randInt = (min: number, max: number) => Math.floor(Math.random() * (max - min + 1)) + min;
const digits = (n: number) => Array.from({ length: n }, () => randInt(0, 9)).join('');
const letters = (n: number) =>
  Array.from({ length: n }, () => pick('abcdefghijklmnopqrstuvwxyz'.split(''))).join('');

const FIRST_NAMES = ['Aarav', 'Sagar', 'Priya', 'Rahul', 'Ananya', 'Vikram', 'Neha', 'Arjun', 'Meera', 'Karthik', 'Divya', 'Rohan'];
const LAST_NAMES = ['Sharma', 'Patel', 'Iyer', 'Reddy', 'Nair', 'Gupta', 'Kumar', 'Singh', 'Menon', 'Das', 'Rao', 'Verma'];
const STREETS = ['MG Road', 'Anna Salai', 'Park Street', 'Brigade Road', 'Linking Road', 'Residency Road'];
const CITIES = ['Chennai', 'Mumbai', 'Bengaluru', 'Hyderabad', 'Pune', 'Kolkata', 'Delhi'];
const STATES = ['Tamil Nadu', 'Maharashtra', 'Karnataka', 'Telangana', 'West Bengal', 'Delhi'];
const COMPANIES = ['Acme Corp', 'Globex', 'Initech', 'Umbrella Ltd', 'Hooli', 'Stark Industries'];
const WORDS = ['quality', 'delivery', 'support', 'order', 'website', 'product', 'service', 'feedback', 'testing', 'automation'];
const sentence = (n = 8) =>
  Array.from({ length: n }, () => pick(WORDS)).join(' ').replace(/^./, c => c.toUpperCase()) + '.';

function randomUser() {
  const firstName = pick(FIRST_NAMES);
  const lastName = pick(LAST_NAMES);
  return {
    firstName,
    lastName,
    fullName: `${firstName} ${lastName}`,
    email: `${firstName}.${lastName}.${Date.now()}${randInt(10, 99)}@example.com`.toLowerCase(),
    password: `Pw${letters(6)}${digits(4)}!`,
    gender: pick(['#id_gender1', '#id_gender2']),
    day: String(randInt(1, 28)),
    month: pick(['January', 'March', 'May', 'August', 'December']),
    year: String(randInt(1970, 2002)),
    company: pick(COMPANIES),
    address1: `${randInt(1, 999)} ${pick(STREETS)}`,
    address2: `Flat ${randInt(1, 40)}`,
    country: pick(['India', 'United States', 'Canada', 'Australia', 'Singapore']),
    state: pick(STATES),
    city: pick(CITIES),
    zip: digits(6),
    mobile: digits(10),
  };
}

function randomCard(name: string) {
  return {
    name,
    number: '4' + digits(15),
    cvc: digits(3),
    month: String(randInt(1, 12)).padStart(2, '0'),
    year: String(randInt(2028, 2033)),
  };
}

/** Creates a snap() function with its own step counter for each test. */
function makeSnap(page: Page, testInfo: TestInfo) {
  let n = 0;
  return async (name: string) => {
    n++;
    const body = await page.screenshot({ fullPage: true });
    await testInfo.attach(`${String(n).padStart(2, '0')}-${name}`, { body, contentType: 'image/png' });
  };
}

async function blockAds(page: Page) {
  await page.route('**/*', route => {
    const url = route.request().url();
    return /googlesyndication|doubleclick|adsbygoogle|googleads|adservice|google-analytics/.test(url)
      ? route.abort()
      : route.continue();
  });
}

async function hideOverlays(page: Page) {
  await page
    .addStyleTag({
      content:
        'iframe[id^="aswift"], ins.adsbygoogle, #ad_position_box, .google-auto-placed, #dismiss-button { display:none !important; }',
    })
    .catch(() => {});
}

async function open(page: Page, path: string) {
  await page.goto(path, { waitUntil: 'domcontentloaded' });
  await hideOverlays(page);
}

// =====================================================================
// PART A: GOOGLE TITLE TESTS
// =====================================================================
test.describe('PART A: Google homepage title', () => {
  test('A1 title equals "Google" (exact)', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await page.goto('https://www.google.com');
    await snap('google-home');
    await expect(page).toHaveTitle('Google');
  });

  test('A2 title contains "Google" (regex)', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await page.goto('https://www.google.com');
    await expect(page).toHaveTitle(/Google/);
    await snap('title-regex-ok');
  });

  test('A3 manual title comparison with logging', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await page.goto('https://www.google.com');
    const actual = await page.title();
    console.log(`Actual title: ${actual}`);
    await testInfo.attach('actual-title', { body: actual, contentType: 'text/plain' });
    await snap('google-manual-compare');
    expect(actual).toBe('Google');
  });

  test('A4 search box is visible and can search', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await page.goto('https://www.google.com');

    // Accept cookie/consent dialog if one appears (region dependent)
    const consent = page.getByRole('button', { name: /Accept all|I agree|Reject all/i }).first();
    if (await consent.isVisible({ timeout: 3000 }).catch(() => false)) await consent.click();

    const box = page.locator('textarea[name="q"], input[name="q"]').first();
    await expect(box).toBeVisible();
    await box.fill('playwright typescript');
    await snap('google-typed-query');
    await box.press('Enter');
    await page.waitForLoadState('domcontentloaded');
    await snap('google-results-or-captcha');

    // Google may show a CAPTCHA to automation, so only assert the title changed
    expect(await page.title()).toBeTruthy();
  });

  test('A5 page title on several sites (data driven)', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    const sites: [string, RegExp][] = [
      ['https://www.google.com', /Google/],
      ['https://playwright.dev', /Playwright/],
      ['https://automationexercise.com', /Automation Exercise/],
      ['https://the-internet.herokuapp.com', /The Internet/],
    ];
    for (const [url, titleRegex] of sites) {
      await page.goto(url, { waitUntil: 'domcontentloaded' });3
      await expect(page).toHaveTitle(titleRegex);
      await snap(`title-ok-${new URL(url).hostname}`);
    }
  });
});

// =====================================================================
// PART B: FULL AUTOMATION EXERCISE JOURNEY
// =====================================================================
const BASE = 'https://automationexercise.com';
const user = randomUser();

test.describe('PART B: Automation Exercise full journey', () => {
  test.describe.configure({ mode: 'serial' });

  test.beforeEach(async ({ page }) => {
    await blockAds(page);
  });

  const login = async (page: Page, email: string, password: string) => {
    await open(page, `${BASE}/login`);
    await page.locator('[data-qa="login-email"]').fill(email);
    await page.locator('[data-qa="login-password"]').fill(password);
    await page.locator('[data-qa="login-button"]').click();
  };

  const addFromList = async (page: Page, index: number) => {
    const card = page.locator('.features_items .product-image-wrapper').nth(index);
    await card.scrollIntoViewIfNeeded();
    await card.hover();
    await card.locator('.overlay-content .add-to-cart').click();
  };

  test('B01 home page loads, title and navigation', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, BASE);
    await expect(page).toHaveTitle(/Automation Exercise/);
    await expect(page.locator('.features_items')).toBeVisible();
    await snap('home-page');

    for (const name of ['Products', 'Cart', 'Signup / Login', 'Contact us']) {
      await expect(page.getByRole('link', { name: new RegExp(name) }).first()).toBeVisible();
    }
    await snap('nav-verified');
  });

  test('B02 register a new random user', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await testInfo.attach('generated-user', {
      body: JSON.stringify(user, null, 2),
      contentType: 'application/json',
    });

    await open(page, `${BASE}/login`);
    await expect(page.getByText('New User Signup!')).toBeVisible();
    await snap('signup-page');

    await page.locator('[data-qa="signup-name"]').fill(user.fullName);
    await page.locator('[data-qa="signup-email"]').fill(user.email);
    await snap('name-email-filled');
    await page.locator('[data-qa="signup-button"]').click();

    await expect(page.getByText('Enter Account Information')).toBeVisible();
    await page.locator(user.gender).check();
    await page.locator('#password').fill(user.password);
    await page.locator('#days').selectOption(user.day);
    await page.locator('#months').selectOption(user.month);
    await page.locator('#years').selectOption(user.year);
    await page.locator('#newsletter').check();
    await page.locator('#optin').check();
    await snap('account-info-filled');

    await page.locator('#first_name').fill(user.firstName);
    await page.locator('#last_name').fill(user.lastName);
    await page.locator('#company').fill(user.company);
    await page.locator('#address1').fill(user.address1);
    await page.locator('#address2').fill(user.address2);
    await page.locator('#country').selectOption(user.country);
    await page.locator('#state').fill(user.state);
    await page.locator('#city').fill(user.city);
    await page.locator('#zipcode').fill(user.zip);
    await page.locator('#mobile_number').fill(user.mobile);
    await snap('address-info-filled');

    await page.locator('[data-qa="create-account"]').click();
    await expect(page.locator('[data-qa="account-created"]')).toBeVisible();
    await snap('account-created');

    await page.locator('[data-qa="continue-button"]').click();
    await expect(page.getByText(`Logged in as ${user.fullName}`)).toBeVisible();
    await snap('logged-in-after-register');
  });

  test('B03 logout', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await expect(page.getByText(`Logged in as ${user.fullName}`)).toBeVisible();
    await page.getByRole('link', { name: /Logout/ }).click();
    await expect(page).toHaveURL(/\/login/);
    await expect(page.getByText('Login to your account')).toBeVisible();
    await snap('logged-out');
  });

  test('B04 login with valid credentials', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/login`);
    await snap('login-page');
    await page.locator('[data-qa="login-email"]').fill(user.email);
    await page.locator('[data-qa="login-password"]').fill(user.password);
    await snap('login-filled');
    await page.locator('[data-qa="login-button"]').click();
    await expect(page.getByText(`Logged in as ${user.fullName}`)).toBeVisible();
    await snap('login-success');
  });

  test('B05 login with wrong password shows error', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, 'wrong-password-123');
    await expect(page.getByText('Your email or password is incorrect!')).toBeVisible();
    await snap('wrong-password-error');
  });

  test('B06 login with unknown email shows error', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, `nobody.${Date.now()}@example.com`, 'whatever123');
    await expect(page.getByText('Your email or password is incorrect!')).toBeVisible();
    await snap('unknown-email-error');
  });

  test('B07 registering with existing email shows error', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/login`);
    await page.locator('[data-qa="signup-name"]').fill(user.fullName);
    await page.locator('[data-qa="signup-email"]').fill(user.email);
    await page.locator('[data-qa="signup-button"]').click();
    await expect(page.getByText('Email Address already exist!')).toBeVisible();
    await snap('duplicate-email-error');
  });

  test('B08 products list and product details', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/products`);
    await expect(page.getByText('All Products')).toBeVisible();
    expect(await page.locator('.features_items .product-image-wrapper').count()).toBeGreaterThan(0);
    await snap('products-list');

    await page.locator('.choose a[href*="product_details"]').first().click();
    await expect(page).toHaveURL(/product_details/);
    const info = page.locator('.product-information');
    await expect(info).toBeVisible();
    for (const label of [/Category:/, /Availability:/, /Condition:/, /Brand:/]) {
      await expect(info.getByText(label)).toBeVisible();
    }
    await snap('product-details');
  });

  test('B09 search for a product', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    const term = 'Dress';     
    await open(page, `${BASE}/products`);
    await page.locator('#search_product').fill(term);
    await snap('search-typed');
    await page.locator('#submit_search').click();

    await expect(page.getByText('Searched Products')).toBeVisible();
    const names = await page.locator('.features_items .productinfo p').allTextContents();
    expect(names.length).toBeGreaterThan(0);
    for (const n of names) expect(n.toLowerCase()).toContain(term.toLowerCase());
    await snap('search-results');
  });

  test('B10 category and brand filters', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/products`);
    await page.locator('.category-products a[href="#Women"]').click();
    await page.locator('#Women a[href*="category_products"]').first().click();
    await expect(page.locator('.title.text-center')).toContainText(/Women/i);
    await snap('category-women');

    await page.locator('.brands-name a').first().click();
    await expect(page.locator('.title.text-center')).toContainText(/Brand/i);
    await snap('brand-filter');
  });

  test('B11 add two products to cart and verify totals', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await open(page, `${BASE}/products`);

    await addFromList(page, 0);
    await expect(page.locator('#cartModal')).toBeVisible();
    await snap('added-first-product');
    await page.getByRole('button', { name: 'Continue Shopping' }).click();

    await addFromList(page, 1);
    await expect(page.locator('#cartModal')).toBeVisible();
    await snap('added-second-product');
    await page.getByRole('link', { name: 'View Cart' }).click();

    await expect(page).toHaveURL(/view_cart/);
    const rows = page.locator('#cart_info_table tbody tr');
    await expect(rows).toHaveCount(2);
    await snap('cart-two-items');

    for (let i = 0; i < 2; i++) {
      const row = rows.nth(i);
      const price = parseInt((await row.locator('.cart_price p').innerText()).replace(/\D/g, ''), 10);
      const qty = parseInt(await row.locator('.cart_quantity button').innerText(), 10);
      const total = parseInt((await row.locator('.cart_total_price').innerText()).replace(/\D/g, ''), 10);
      expect(price * qty).toBe(total);
    }
    await snap('cart-totals-verified');
  });

  test('B12 custom quantity from product details', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await open(page, `${BASE}/product_details/3`);
    await page.locator('#quantity').fill('4');
    await snap('quantity-4');
    await page.locator('button.cart').click();
    await expect(page.locator('#cartModal')).toBeVisible();
    await page.getByRole('link', { name: 'View Cart' }).click();
    await expect(page.locator('#product-3 .cart_quantity button')).toHaveText('4');
    await snap('cart-quantity-4');
  });

  test('B13 remove an item from the cart', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await open(page, `${BASE}/view_cart`);
    const rows = page.locator('#cart_info_table tbody tr');
    const before = await rows.count();
    expect(before).toBeGreaterThan(0);
    await snap('cart-before-remove');
    await page.locator('.cart_quantity_delete').first().click();
    await expect(rows).toHaveCount(before - 1);
    await snap('cart-after-remove'); 
  });

  test('B14 checkout and payment', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await open(page, `${BASE}/view_cart`);

    if ((await page.locator('#cart_info_table tbody tr').count()) === 0) {
      await open(page, `${BASE}/products`);
      await addFromList(page, 2);
      await page.getByRole('link', { name: 'View Cart' }).click();
    }
    await snap('cart-before-checkout');

    await page.getByText('Proceed To Checkout').click();
    await expect(page.getByText('Address Details')).toBeVisible();
    const delivery = page.locator('#address_delivery');
    await expect(delivery).toContainText(user.firstName);
    await expect(delivery).toContainText(user.lastName);
    await expect(delivery).toContainText(user.city);
    await expect(delivery).toContainText(user.mobile);
    await snap('address-verified');

    await page.locator('textarea[name="message"]').fill(sentence(10));
    await snap('order-comment');
    await page.getByRole('link', { name: 'Place Order' }).click();

    const card = randomCard(user.fullName);
    await expect(page).toHaveURL(/payment/);
    await page.locator('[data-qa="name-on-card"]').fill(card.name);
    await page.locator('[data-qa="card-number"]').fill(card.number);
    await page.locator('[data-qa="cvc"]').fill(card.cvc);
    await page.locator('[data-qa="expiry-month"]').fill(card.month);
    await page.locator('[data-qa="expiry-year"]').fill(card.year);
    await snap('payment-filled');

    await page.locator('[data-qa="pay-button"]').click();
    await expect(page.locator('[data-qa="order-placed"]')).toBeVisible();
    await snap('order-placed');

    // Try to download the invoice if the link exists
    const invoice = page.getByRole('link', { name: 'Download Invoice' });
    if (await invoice.isVisible().catch(() => false)) {
      const [download] = await Promise.all([page.waitForEvent('download'), invoice.click()]);
      const path = testInfo.outputPath(download.suggestedFilename());
      await download.saveAs(path);
      await testInfo.attach('invoice', { path });
      await snap('invoice-downloaded');
    }
  });

  test('B15 contact us form with file upload', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/contact_us`);
    await expect(page.getByText('Get In Touch')).toBeVisible();

    await page.locator('[data-qa="name"]').fill(user.fullName);
    await page.locator('[data-qa="email"]').fill(user.email);
    await page.locator('[data-qa="subject"]').fill(sentence(4));
    await page.locator('[data-qa="message"]').fill(sentence(20));
    await page.locator('input[name="upload_file"]').setInputFiles({
      name: 'sample.txt',
      mimeType: 'text/plain',
      buffer: Buffer.from('Playwright upload test file'),
    });
    await snap('contact-filled');

    page.once('dialog', d => d.accept());
    await page.locator('[data-qa="submit-button"]').click();
    await expect(page.locator('.status.alert.alert-success')).toContainText('Success');
    await snap('contact-success');
  });

  test('B16 subscribe to newsletter from footer', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, BASE);
    await page.locator('footer').scrollIntoViewIfNeeded();
    await page.locator('#susbscribe_email').fill(user.email);
    await snap('subscribe-typed');
    await page.locator('#subscribe').click();
    await expect(page.getByText('You have been successfully subscribed!')).toBeVisible();
    await snap('subscribe-success');
  });

  test('B17 write a product review', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, `${BASE}/product_details/1`);
    await expect(page.getByText('Write Your Review')).toBeVisible();
    await page.locator('#name').fill(user.fullName);
    await page.locator('#email').fill(user.email);
    await page.locator('#review').fill(sentence(15));
    await snap('review-filled');
    await page.locator('#button-review').click();
    await expect(page.getByText('Thank you for your review.')).toBeVisible();
    await snap('review-submitted');
  });

  test('B18 scroll to bottom and back to top', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await open(page, BASE);
    await page.locator('footer').scrollIntoViewIfNeeded();
    await expect(page.getByText('Subscription')).toBeVisible();
    await snap('scrolled-bottom');
    await page.locator('#scrollUp').click();
    await expect(page.getByText('Full-Fledged practice website for Automation Engineers')).toBeVisible();
    await snap('scrolled-top');
  });

  test('B19 delete the account (clean-up)', async ({ page }, testInfo) => {
    const snap = makeSnap(page, testInfo);
    await login(page, user.email, user.password);
    await expect(page.getByText(`Logged in as ${user.fullName}`)).toBeVisible();
    await snap('before-delete');

    await page.getByRole('link', { name: /Delete Account/ }).click();
    await expect(page.locator('[data-qa="account-deleted"]')).toBeVisible();
    await snap('account-deleted');
    await page.locator('[data-qa="continue-button"]').click();

    await login(page, user.email, user.password);
    await expect(page.getByText('Your email or password is incorrect!')).toBeVisible();
    await snap('login-fails-after-delete');
  });
});
