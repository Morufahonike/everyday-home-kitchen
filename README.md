# Everyday Home Kitchen – e-commerce website

Online shop for **Everyday Home Kitchen Services Limited**: affordable, practical kitchen essentials and selected home products.

Built with **Python 3.13, Django 5.2 LTS, PostgreSQL, Django templates, hand-written mobile-first CSS and vanilla JavaScript**. Prepared for Railway hosting but **not deployed**.

> **Status: test-mode build.** Products are clearly labelled samples, payments use Paystack in test mode (or a simulated provider locally), and no supplier is connected. See [What is a placeholder](#what-is-a-placeholder-or-needs-external-access).

---

## Contents

1. [What works](#what-works)
2. [What is a placeholder or needs external access](#what-is-a-placeholder-or-needs-external-access)
3. [Local setup](#local-setup)
4. [Environment variables](#environment-variables)
5. [Database migrations and sample data](#database-migrations-and-sample-data)
6. [Running the tests](#running-the-tests)
7. [Staff roles and the dashboard](#staff-roles-and-the-dashboard)
8. [Payments: Paystack](#payments-paystack)
9. [Supplier integration](#supplier-integration)
10. [Brand identity](#brand-identity)
11. [GitHub repository](#github-repository)
12. [Deploying to Railway (requires owner approval)](#deploying-to-railway-requires-owner-approval)
13. [Project structure](#project-structure)
14. [Information still needed from the business](#information-still-needed-from-the-business)

---

## What works

**Storefront**
- Homepage: header with logo, prominent search, account and basket; a scrolling category "shelf" bar; hero with shopping call to action; category tiles; *Everyday bestsellers* (computed from real paid orders only – until there is enough sales history the section is honestly titled *Everyday favourites*, a staff-picked collection); *Small kitchen solutions*; *Storage and organisation*; *Affordable finds*; optional promotional banner (off by default); customer reviews (only shown when approved, verified-purchase reviews exist); newsletter signup with consent; footer with contact details and policy links.
- Category, collection and all-products pages with price range, in-stock and category filters, five sort orders and pagination. On phones the filters open as a bottom sheet (keyboard accessible, closes with Escape).
- Search across names, descriptions, categories and SKUs, with live suggestions (arrow keys + Enter supported).
- Product pages with variants (size, set size), live price/stock update when an option is chosen, quantity stepper, stock status, description/features/care, related products, verified-purchase reviews.

**Shopping**
- Session basket: add (with confirmation toast), change quantity, remove. Prices, names and stock are always read from the database; the browser only sends IDs and quantities. Quantities are clamped to available stock.
- Checkout for guests or signed-in customers: contact, delivery address (Nigerian states list), delivery option, notes, promo code, terms acceptance. Totals are recomputed on the server; double-submitting the form does not create duplicate orders.
- Order confirmation page, confirmation email (printed to the log by default), order progress timeline, courier/tracking details when staff add them, guest order lookup by order number + email.

**Customer accounts**
- Register, sign in (email + password), sign out, password reset by email, change password, edit details, address book (default address pre-fills checkout), order history and order detail.

**Staff**
- Dashboard (`/dashboard/`): paid-sales summary (today / 7 / 30 days) and 14-day chart, orders needing action, low-stock list, supplier integration status, recent orders.
- Orders: search and filter, order detail, controlled status changes (only valid transitions allowed), add courier and tracking link, public or internal notes, optional customer email, cancelling restocks items.
- Stock and prices: quick edit per variant.
- Customers list (no passwords or payment details are ever shown).
- Full management of products, variants, categories, collections, promo codes, banners, delivery options, policy pages, store contact details and review moderation in the Django admin (`/store-admin/` by default; configurable).

**Security**
- Django's password hashing and validators (min. 10 characters), CSRF protection on every form, server-side permission checks on every staff action, customers can only see their own orders and addresses.
- Secure, HTTP-only, SameSite cookies; HTTPS redirect and HSTS in production; strict Content Security Policy (no inline scripts or styles), `X-Frame-Options: DENY`, referrer and permissions policies, `no-store` caching on account/checkout/dashboard pages.
- Login and order-lookup rate limiting, open-redirect protection, input validation on every form, safe error pages, structured logging.
- Payment verification server-to-server, webhook signature checks, duplicate-event protection, amount/currency mismatch detection, no card data ever touches the site.

## What is a placeholder or needs external access

| Area | Current state | Needed to go live |
|---|---|---|
| **Brand assets** | Applied from the brand guidelines sheet. Logo icon redrawn as SVG from a low-resolution image. | Official vector logo files and product/lifestyle photography – see [Brand identity](#brand-identity). |
| **Products** | 36 invented **development samples** (`is_sample=True`, "Sample" label, illustrated placeholders instead of photos). | Real products, photos you own or have written permission to use, real prices and stock. |
| **Payments** | **Paystack integrated** (server-side initialise, verify, signed webhooks), tested against simulated Paystack responses. The mock provider remains for local work. | Paystack **test keys** entered in Railway, one real test-mode transaction, then business activation and your explicit approval for live mode. |
| **Supplier** | Interface only, set to "No supplier connected". Paid orders are recorded as "not sent" for manual fulfilment. | Supplier approval and API documentation/credentials. |
| **Delivery options** | Two options with **placeholder fees** stated as such on the checkout page. | Real delivery areas, fees and wording. |
| **Policy pages** | Draft wording marked "Draft – for review" with bracketed gaps. | Review by the business (and ideally a lawyer), especially returns window and NDPA privacy details. |
| **Contact details** | Empty – nothing is invented; the contact page says details will be published shortly. | Phone, email, WhatsApp, address, hours (edit in admin → Store settings). |
| **Email** | Console backend: emails appear in the server log only. | An email service (SMTP or provider) – may have a cost. |
| **Currency** | Assumed Nigerian naira (₦). | Confirm with the business and payment provider. |
| **Test promo code** | `TESTSAVE10` (10% off over ₦10,000) exists for testing. | Deactivate or delete before launch. |

## Local setup

Requirements: Python 3.13 (3.12+ should work), PostgreSQL 14+.

```bash
git clone <repository> everyday-home-kitchen
cd everyday-home-kitchen
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

createdb everyday_home_kitchen     # or create it in pgAdmin
cp .env.example .env               # then edit DATABASE_URL and DJANGO_SECRET_KEY

python manage.py migrate
python manage.py seed_sample_data  # optional: sample products, delivery options, draft pages
python manage.py setup_roles       # staff groups
python manage.py createsuperuser   # your own admin account
python manage.py runserver
```

Open <http://127.0.0.1:8000>. Staff dashboard: <http://127.0.0.1:8000/dashboard/>. Admin: <http://127.0.0.1:8000/store-admin/>.

The `.env` file is read automatically in development (real environment variables take priority). Never commit it.

To try a purchase: add items, check out with any test details, and on the **Simulated payment** page choose *Simulate a successful payment* (or *declined* to test the retry path).

## Environment variables

All names and placeholder values are in [`.env.example`](.env.example). Summary:

| Variable | Required | Purpose |
|---|---|---|
| `DJANGO_SECRET_KEY` | **Production** | Long random string. Generate with `python -c "import secrets; print(secrets.token_urlsafe(50))"`. |
| `DJANGO_DEBUG` | No (default `false`) | `true` only on your own computer. |
| `DATABASE_URL` | **Production** | PostgreSQL connection string. Railway provides this. |
| `DJANGO_ALLOWED_HOSTS` | **Production** | Comma-separated domain names. Railway's own domain is added automatically from `RAILWAY_PUBLIC_DOMAIN`. |
| `DJANGO_CSRF_TRUSTED_ORIGINS` | With a custom domain | e.g. `https://www.example.com`. |
| `DJANGO_ADMIN_URL` | No | Admin path; change from the default in production. |
| `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS`, `SECURE_HSTS_INCLUDE_SUBDOMAINS` | No | HTTPS hardening (on by default in production). Increase HSTS once the final domain is settled. |
| `STORE_CURRENCY`, `STORE_CURRENCY_SYMBOL`, `STORE_TIME_ZONE`, `LOW_STOCK_THRESHOLD` | No | Store settings (defaults: NGN, ₦, Africa/Lagos, 5). |
| `EMAIL_*`, `DEFAULT_FROM_EMAIL` | No | Outgoing email. Default prints to the log. |
| `PAYMENT_PROVIDER` | No (default `mock`) | `mock` or `paystack`. |
| `PAYMENT_LIVE_MODE` | No (default `false`) | Must stay `false` until you approve live payments. The app refuses to start if set with the mock provider. |
| `ALLOW_MOCK_PAYMENTS` | No | Defaults to on in debug, **off in production**. Set `true` on a private staging site to allow simulated payments; otherwise checkout shows "temporarily unavailable". |
| `MOCK_PAYMENT_WEBHOOK_SECRET` | With mock webhooks | Signing secret for simulated webhooks. |
| `PAYSTACK_SECRET_KEY`, `PAYSTACK_PUBLIC_KEY` | With Paystack | Test keys first (`sk_test_`, `pk_test_`). Enter only in hosting environment settings. |
| `PAYSTACK_CHANNELS`, `PAYSTACK_TIMEOUT_SECONDS` | No | Defaults `card` and `15`. |
| `SUPPLIER_ADAPTER`, `SUPPLIER_SYNC_ENABLED`, `SUPPLIER_ORDER_SUBMISSION_ENABLED`, `SUPPLIER_API_BASE_URL`, `SUPPLIER_API_KEY` | Later | Supplier integration; all off by default. |
| `MEDIA_ROOT`, `SERVE_MEDIA` | For uploads | Where product photos are stored; see Railway notes. |
| `LOG_LEVEL` | No | `INFO` by default. |

## Database migrations and sample data

```bash
python manage.py migrate                # apply all migrations
python manage.py makemigrations         # after changing models, then commit the new files
python manage.py seed_sample_data       # safe to run again; existing records untouched
python manage.py setup_roles            # safe to run again; refreshes group permissions
```

Before launch, delete sample products in the admin (filter by **Is sample: Yes**) or deactivate them, and deactivate the `TESTSAVE10` code.

## Running the tests

Tests run against PostgreSQL (Django creates and removes a temporary test database; the database user needs permission to create databases).

```bash
python manage.py test
```

**Last run: 90 tests, all passing**, on PostgreSQL 16 with Python 3.13 and Django 5.2.18. They cover:

- Storefront: listings, filters, sorting, bad input, search and suggestions, hidden products, honest bestseller labelling, security headers.
- Basket: server-side prices, stock clamping, update/remove, inactive items, malformed sessions, open-redirect protection.
- Checkout: server totals with tampered fields ignored, promo codes, terms, stock changes, inactive delivery options, duplicate-submission protection, checkout closed when no payment provider is available.
- Paystack: amount sent in kobo from the server, Mastercard payment verified and card brand recorded, callback URL cannot fake success, underpayment and wrong currency flagged, decline then retry with a new reference, Paystack outage at checkout and on return, webhook signature, duplicate and retry handling, reversal flagged, live-key/test-key safety rules, CSP allows the Paystack redirect.
- Mock payments: successful payment, replayed returns not double-processed, browser parameters can't mark an order paid, decline and retry, amount mismatch, other visitors can't view or pay an order, mock disabled in production, webhook signatures, duplicate webhook events, webhook payload alone can't mark paid.
- Accounts: registration, weak/duplicate rejection, login/logout (POST only), basket kept after login, login rate limit, cross-customer order and address access blocked, password reset email, password hashing.
- Staff permissions: anonymous, customer and unassigned staff blocked; order handlers can progress orders but not edit stock or view customers; managers can; invalid status changes and unsafe tracking links rejected; cancellations restock.
- Supplier safety: no supplier contacted by default, sync skipped, command runs; sample data labelled and idempotent; newsletter consent.

**Browser checks performed (headless Chromium):** homepage, listings, category, product, basket, sign-in, register, order lookup, contact, policy and empty-search pages at 320, 375, 430, 768, 1024 and 1440 px with no horizontal overflow and no console errors; full guest journey (filter sheet, add to basket, variant choice, quantity stepper, promo code, checkout, simulated payment, confirmation) completed at 320, 375, 768 and 1440 px; staff dashboard pages checked at 375 and 1440 px. A production-mode run under Gunicorn (debug off) was also checked locally for security headers and compressed, cache-busted static files.

Not yet tested: real devices (iOS Safari, Android Chrome), screen readers, and load testing.

## Staff roles and the dashboard

`python manage.py setup_roles` creates two groups. Give staff accounts **Staff status** and add them to a group in the admin (Users → choose user):

- **Store managers** – products, prices, stock, promotions, delivery options, pages, store settings, orders; can view customers, payments and supplier status.
- **Order handlers** – view and update orders and shipments; view products and stock.

Every dashboard view checks these permissions on the server. Superusers can do everything; keep superuser accounts to a minimum.

## Payments: Paystack

Payment providers sit behind one interface (`apps/payments/providers/base.py`): **initiate**, **verify** and **parse_webhook**. Two providers are registered:

- `mock` – simulated payments for local development (never usable with live mode);
- `paystack` – Paystack (`apps/payments/providers/paystack.py`).

**How the Paystack flow works**

1. Checkout creates the order and a `Payment` with the server-calculated total, then calls Paystack's *Initialize Transaction* API from the server with the amount in kobo, currency NGN, our unique reference, channel `card` and a callback URL. The customer is redirected to Paystack's hosted checkout, where they enter their **Mastercard, Visa or Verve** card. Card details never touch this website.
2. Paystack sends the customer back to `/payments/return/<reference>/`. The site ignores anything in that URL and calls *Verify Transaction* server-to-server. An order is marked paid only when Paystack reports `success` **and** the reference, amount and currency all match. Mismatches are flagged for staff; reversals (refunds/chargebacks) are flagged too.
3. Paystack's `charge.success` webhook (`/payments/webhook/paystack/`) is checked with HMAC-SHA512 of the raw body using the secret key, de-duplicated, and re-verified with step 2. If Paystack can't be reached, the webhook returns 503 so Paystack retries.
4. Processing is idempotent: repeats never double-charge stock, send duplicate emails or re-mark orders. Card brand and last four digits (from Paystack) are stored for reporting; the full card number is never received.

**Key safety built in:** keys come only from environment variables. A live key (`sk_live_…`) is refused unless `PAYMENT_LIVE_MODE=true`; live mode is refused with a test key; a public key in the secret slot is refused. Checkout closes politely if Paystack is not configured. `python manage.py payments_status` shows the configuration without printing any secret.

**Entering your Paystack test keys (never in chat, email or GitHub)**

1. Sign in at <https://dashboard.paystack.com>, make sure the **Test mode** switch is on, and open **Settings → API Keys & Webhooks**.
2. In Railway, open the web service → **Variables** → **New variable**, and add:
   - `PAYMENT_PROVIDER` = `paystack`
   - `PAYSTACK_SECRET_KEY` = your **test secret key** (starts `sk_test_`)
   - `PAYSTACK_PUBLIC_KEY` = your **test public key** (starts `pk_test_`)
   - `PAYMENT_LIVE_MODE` = `false`
3. Back in Paystack (same page), set **Test Webhook URL** to `https://<your-railway-domain>/payments/webhook/paystack/` and **Test Callback URL** to `https://<your-railway-domain>/`. The site sends its own callback per payment, which takes precedence.
4. Railway redeploys automatically with the new variables.

**Test transaction (Mastercard)** — use Paystack's published test cards (<https://paystack.com/docs/payments/test-payments/>). `5192 6027 2058 4796`, CVV `123`, any future expiry, is in the Mastercard number range (51–55) and simulates a bank-authorised success; `4084 0800 0000 5408` (CVV `001`) is a declined card. After paying, the staff dashboard's Payments panel and the order page show the card brand Paystack reported.

**Going live (only with the owner's explicit approval):** activate the business on Paystack (compliance documents, corporate bank account for a registered business), confirm card payments are enabled, replace both keys with live keys, set `PAYMENT_LIVE_MODE=true`, set the live webhook URL in Paystack, and make one small real purchase and refund.

**Eligibility and costs (from Paystack's published pages, October 2026 — confirm on your dashboard):**

- Paystack accepts debit and credit cards issued by Mastercard, Visa and Verve. Foreign-issued cards need **international payments** enabled on the business.
- Nigerian businesses settle in NGN by default; USD settlement needs a Zenith Bank USD domiciliary account. Payouts go to a corporate account matching the registration documents (registered business) or a personal account (starter business; lifetime collection limit of ₦8,000,000).
- Local card fees: 1.5% + ₦100 (₦100 waived under ₦2,500), capped at ₦2,000. International cards in NGN: 3.9% + ₦100.

## Supplier integration

`apps/suppliers/adapters/base.py` defines `fetch_products`, `fetch_stock`, `submit_order` and `fetch_tracking`. The registered adapter is `none`, which never contacts anything. Products and variants already carry `supplier`, `supplier_product_id` and `supplier_variant_id` fields, and each paid order gets a `SupplierOrder` record ("not sent").

Order submission happens only when **all** of these are true: a live adapter is configured, `SUPPLIER_ORDER_SUBMISSION_ENABLED=true`, and the order is paid. Stock sync (`python manage.py supplier_sync`, dry run unless `--apply`) also needs `SUPPLIER_SYNC_ENABLED=true`. Supplier failures are logged and never break checkout. Tracking updates from a supplier should be written as `OrderEvent` entries with `source="supplier"` so customers see genuine updates only.

Do not import supplier images or descriptions until the supplier has given written permission (tick *Image rights confirmed* on the supplier record).

## Brand identity

The official brand guidelines sheet is stored at `docs/brand/brand-guidelines-sheet.png` and has been applied:

| Brand role | Value | Where it is used |
|---|---|---|
| Terracotta Orange (primary) | `#E86838` | Logo, large "Shop all products" button, icons, cart badge accents, sales chart |
| Warm Yellow (secondary) | `#FFC457` | Test-mode strip, offer banner, footer tagline, highlights |
| Cream (background) | `#FFF7EE` | Page background |
| Leaf Green (accent) | `#2E7D32` | In-stock and success states, keyboard focus ring, logo leaf |
| Soft Peach (support) | `#FDE0D2` | Hero panel, category bar, selected options, newsletter band |
| Warm Brown (text) | `#8B5E3C` | Secondary text; a deeper shade `#4A3222` is used for headings, body text and the footer |
| Headings | Poppins (500/600/700) | Self-hosted, SIL Open Font License |
| Body | Nunito (400–800) | Self-hosted, SIL Open Font License |

All values live in the token block at the top of `static/css/main.css`.

**Accessibility adjustment:** white text on Terracotta Orange is 3.25:1, below the WCAG AA minimum of 4.5:1 for normal-size text. Normal buttons and links therefore use a deeper shade of the same orange (`#B9481F`, 5.2:1). The true brand orange is used for the logo, icons and the large hero button (19px bold counts as large text, where 3:1 is enough).

**Logo:** the sheet is a 501 × 679 px image, too small to cut a clean logo from. The house-and-leaf icon was redrawn as SVG (`static/img/brand/logo-icon.svg`, also the favicon) with the wordmark set in Poppins. Replace it with the official vector files (SVG or high-resolution PNG) when available: swap the SVG file, or edit `templates/partials/logo.html` to use the supplied horizontal logo.

**From the website concept, deliberately not used yet:** the "Fast delivery across Nigeria" and "Secure payments" badges (not supportable while delivery is unconfirmed and payments are simulated) and a "Deals" menu item (no genuine deals exist). The lifestyle photography in the concept needs licensed or commissioned photos; the site uses illustrated placeholders until then.

## GitHub repository

The code is a git repository with a `.gitignore` that excludes `.env`, uploaded media, collected static files and local databases. `.env.example` lists every variable with placeholders only. A GitHub Actions workflow (`.github/workflows/tests.yml`) runs migrations checks and the full test suite against PostgreSQL on every push to `main` and on pull requests (it uses GitHub's included Actions minutes for private repositories).

Never commit real keys. If a secret is ever committed, rotate it at the provider immediately; deleting the commit is not enough.

## Deploying to Railway (requires owner approval)

**Costs:** Railway needs a card on file. Per Railway's pricing page (October 2026): a one-time $5 trial credit; Hobby plan $5/month including $5 of usage; usage billed at about $10 per GB RAM/month, $20 per vCPU/month and $0.15 per GB/month for volumes. A small Django app plus PostgreSQL typically fits within or near the Hobby allowance, but this is an estimate — check your Railway usage page after a few days.

**Steps (after approval):**

1. In Railway: **New project → Deploy from GitHub repo** → choose `everyday-home-kitchen` (authorise the Railway GitHub app for that repository if asked).
2. In the same project: **+ Create → Database → PostgreSQL**.
3. On the web service → **Variables**, add:
   - `DATABASE_URL` = `${{Postgres.DATABASE_URL}}`
   - `DJANGO_SECRET_KEY` = a new long random value (generate one locally with `python -c "import secrets; print(secrets.token_urlsafe(50))"`)
   - `DJANGO_ADMIN_URL` = a non-obvious path, e.g. `manage-ehk-7q2/`
   - `PAYMENT_PROVIDER` = `paystack`, `PAYMENT_LIVE_MODE` = `false`, plus the two Paystack **test** keys (see above)
   - Supplier switches stay unset (off).
4. `railway.json` already configures: build `collectstatic`, pre-deploy `migrate`, start Gunicorn on `$PORT`, health check `/healthz/`, restart on failure. Railway's own domain is trusted automatically (`RAILWAY_PUBLIC_DOMAIN`).
5. **Settings → Networking → Generate Domain** to get the `*.up.railway.app` address.
6. Open the service's shell (or `railway run`) and run `python manage.py setup_roles`, `python manage.py createsuperuser`, and optionally `python manage.py seed_sample_data` for test products.
7. **Product photos:** add a volume to the web service mounted at `/data/media`, then set `MEDIA_ROOT=/data/media` and `SERVE_MEDIA=true` (volume storage is billed per GB).
8. Turn on PostgreSQL backups in Railway before taking real orders.
9. With a custom domain later: add it in Railway, then set `DJANGO_ALLOWED_HOSTS` and `DJANGO_CSRF_TRUSTED_ORIGINS`, and raise `SECURE_HSTS_SECONDS` to `31536000` once HTTPS is confirmed.

## Project structure

```
config/            settings, URLs, WSGI
apps/core/         homepage, policy pages, contact, newsletter, security headers, rate limiting, seed command
apps/catalog/      categories, collections, products, variants, reviews, search and listings
apps/cart/         session basket
apps/orders/       checkout, orders, order events, shipments, delivery options, emails
apps/payments/     provider interface, mock provider, verification service, webhooks
apps/promotions/   promo codes and homepage banner
apps/accounts/     custom email-login user, addresses, account pages, staff roles command
apps/suppliers/    supplier models, adapter interface, safe services, sync command
apps/dashboard/    staff dashboard views
templates/         all HTML templates
static/            CSS, JavaScript, icons, fonts
```

## Information still needed from the business

- Official vector logo files (SVG or high-resolution PNG) and brand photography.
- Real product range, photos with usage rights, prices and stock.
- Paystack business activation status, whether international card payments are enabled, and the settlement account.
- Supplier name, approval, API documentation and test credentials.
- Delivery areas, fees, timings and courier(s).
- Returns policy details, privacy/data-protection contact (NDPA), registered address and company details for the footer and terms.
- Contact phone, email, WhatsApp and opening hours.
- Domain name and email sending address.
