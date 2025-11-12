# Laravel Billing — Open-Source Billing & Subscription Package

**Laravel Billing** is a composer-installable Laravel package that provides world-class, gateway-agnostic billing, plans, subscriptions, invoices, multi-currency support, usage metering and a Livewire-powered admin UI. The goal: a drop-in, extensible billing system any Laravel app can adopt — open source, modular, and future-proof.

> ⚡ This README is meant to be the public project landing page to attract contributors and users. It explains the vision, features, architecture, quickstart, API, how to help, roadmap, and governance.

---

## Table of contents

1. [Vision & Goals](#vision--goals)
2. [Key Features](#key-features)
3. [Who should care / Why it exists](#who-should-care--why-it-exists)
4. [Quickstart (for interested users)](#quickstart-for-interested-users)
5. [Installation & Setup (developer view)](#installation--setup-developer-view)
6. [Core Concepts & Models](#core-concepts--models)
7. [Public PHP API examples](#public-php-api-examples)
8. [Livewire Admin UI (out-of-the-box)](#livewire-admin-ui-out-of-the-box)
9. [Gateways & Extensibility](#gateways--extensibility)
10. [Events, Jobs & Webhooks](#events-jobs--webhooks)
11. [Security & Compliance notes](#security--compliance-notes)
12. [Testing & CI expectations](#testing--ci-expectations)
13. [Roadmap & Release Plan](#roadmap--release-plan)
14. [How to contribute](#how-to-contribute)
15. [Governance, Code of Conduct & License](#governance-code-of-conduct--license)
16. [Contact / Community](#contact--community)

---

# Vision & Goals

**Vision:** Build the world’s first widely-adopted, open-source, Laravel-first billing system that makes it trivial to add robust billing to any Laravel application: subscriptions, one-time charges, metered billing, invoices, reconciliation, multi-currency, and more — all with a developer-friendly, secure, and extensible architecture.

**Primary goals**

* Developer-first: easy installation, clear contracts, small API surface.
* Gateway-agnostic: adapter-based architecture for any payment provider.
* Composable: publishable Livewire admin UI and publishable views.
* Secure & compliant: minimize PCI scope and provide privacy controls.
* Maintainable & testable: strong test coverage, CI pipeline, clear docs.

---

# Key Features

**Core**

* Plans & Products: create plans with flexible billing intervals and per-currency pricing.
* Subscriptions: signups, trials, cancel/pause/resume, proration strategies.
* One-time payments: payments and invoices for single purchases.
* Invoicing: auto-generated invoices, status tracking, PDF generation.
* Transactions & Refunds: full transaction ledger and reconciliation utilities.
* Multi-currency: prices per currency + pluggable exchange-rate provider.
* Coupons & promotions: percent/flat/usage-limited discounts.
* Customer management: automatic linking to host `User` model, guest checkout support.
* Livewire Admin UI: CRUD for plans, customers, subscriptions, invoices, gateways.
* Events & hooks: `SubscriptionCreated`, `PaymentSucceeded`, `InvoiceGenerated`, etc.
* CLI & scheduler: artisan commands and scheduled jobs to run recurring billing and retries.

**Advanced / optional**

* Metered billing & usage records.
* Hosted checkout widget + embeddable checkout components.
* Accounting integrations (Xero/QuickBooks).
* Multi-tenant support recipe.
* KPI dashboard (MRR, churn, LTV) and metrics emitting.

---

# Who should care / Why it exists

* SaaS founders & teams who want a Laravel-native billing solution that’s not tied to one gateway or vendor.
* Laravel package authors who need a drop-in billing system for their packages.
* Companies wanting full control over billing logic, invoicing, and data privacy.
* Developers who want an extensible, auditable, testable billing core they can customize.

---

# Quickstart (for interested users)

> This is intentionally short — if there's demand we’ll publish a fully detailed installation and configuration guide.

1. Create a new Laravel project (or use an existing).
2. Add the package after we publish:

   ```bash
   composer require vendor/laravel-billing
   ```
3. Publish configuration, migrations, and views:

   ```bash
   php artisan vendor:publish --provider="Vendor\Billing\BillingServiceProvider"
   ```
4. Run migrations:

   ```bash
   php artisan migrate
   ```
5. Configure `.env`:

   * `BILLING_USER_MODEL="App\Models\User"`
   * Configure gateway API keys in `config/billing.php` or env.
6. Seed example plans (optional):

   ```bash
   php artisan billing:seed-plans
   ```
7. Schedule `php artisan schedule:run` in cron and run queue workers for async jobs.

---

# Installation & Setup (developer view)

> Full docs will be provided in the repository, this is the condensed dev installation.

1. **Require the package**

   ```bash
   composer require vendor/laravel-billing
   ```

2. **Register service provider** (if not auto-discovered)

   ```php
   // config/app.php
   'providers' => [
       // ...
       Vendor\Billing\BillingServiceProvider::class,
   ];
   ```

3. **Publish assets**

   ```bash
   php artisan vendor:publish --provider="Vendor\Billing\BillingServiceProvider" --tag="config"
   php artisan vendor:publish --provider="Vendor\Billing\BillingServiceProvider" --tag="migrations"
   php artisan vendor:publish --provider="Vendor\Billing\BillingServiceProvider" --tag="views"
   ```

4. **Migrate**

   ```bash
   php artisan migrate
   ```

5. **Environment variables** (example)

   ```env
   BILLING_USER_MODEL=App\Models\User
   BILLING_DEFAULT_CURRENCY=USD
   STRIPE_KEY=pk_test_...
   STRIPE_SECRET=sk_test_...
   -- and per-gateway keys --
   ```

6. **Scheduler & queue**

   * Add `* * * * * php /path-to-project/artisan schedule:run >> /dev/null 2>&1` to cron.
   * Start queue workers for background jobs:

     ```bash
     php artisan queue:work --tries=3
     ```

---

# Core Concepts & Models

* **Customer** — wrapper around host app user (nullable for guest customers). Stores billing preferences and external IDs.
* **Plan** — subscription offering (slug, description, features JSON, interval).
* **Price** — plan price per currency (amount in cents).
* **Subscription** — active relationship between Customer and Plan/Price.
* **Invoice** — billing document with line items and status (draft, sent, paid, refunded).
* **Transaction** — gateway-level charge / refund tracking.
* **PaymentMethod** — stored payment instruments (tokenized).
* **UsageRecord** — for metered billing.
* **Coupon** — discounts and promotions.

---

# Public PHP API examples

> The package exposes a service `Billing` (or facade) and a set of helper classes.

**Create subscription**

```php
use Vendor\Billing\Facades\Billing;

$subscription = Billing::createSubscription($user, 'pro-plan', [
    'price' => 'pro-monthly-usd',
    'payment_method' => $paymentMethodToken,
    'trial_days' => 14,
]);
```

**Charge a one-time payment**

```php
$charge = Billing::chargeOneTime($user, [
    'amount' => 4999, // cents
    'currency' => 'USD',
    'description' => 'E-book purchase',
    'payment_method' => $cardToken,
]);
```

**Generate invoice PDF**

```php
$invoice = Billing::generateInvoicePdf($invoiceId); // returns path or stream
```

**Refund**

```php
Billing::refundTransaction($transactionId, $amountInCents = null);
```

More detailed examples will be added in docs and the example/demo app.

---

# Livewire Admin UI (out-of-the-box)

The package ships a publishable Livewire admin UI covering:

* Plans & Prices CRUD
* Customers & Payment Methods
* Subscriptions (change plans, proration, cancel/resume)
* Invoices & PDFs
* Transactions & refunds
* Gateway settings (API keys)
* Audit trail & logs

Admin routes are prefixed (configurable) and protected by a `billing.admin` middleware group which the host app maps to its preferred auth/gate system (e.g., `auth` + `can:manage-billing`). Views are publishable for full customization.

---

# Gateways & Extensibility

**Architecture:** Adapter pattern — each gateway implements `Contracts/PaymentGatewayInterface`:

* `createCustomer(array $data)`
* `attachPaymentMethod($customer, $methodData)`
* `createPaymentIntent($params)`
* `capturePayment($params)`
* `refund($chargeId, $amount)`
* `createSubscription($customer, $priceOrPlan)`
* `cancelSubscription($subId, $atPeriodEnd)`
* `handleWebhook(Request $request)`

**Built-in adapters (initial):**

* Stripe (primary)
* PayPal
* A sandbox/mock gateway for testing

**How to add another gateway**

1. Create adapter class implementing the gateway contract.
2. Register adapter in `config/billing.php` or a service provider.
3. Add webhook route/handler that points to adapter webhook processor.

**Idempotency & webhooks**

* The package ensures idempotency by storing `external_id` from gateways and by using idempotency keys when calling gateway APIs.
* All webhooks are signature-verified where the gateway supports it.

---

# Events, Jobs & Webhooks

**Events published**

* `SubscriptionCreated`, `SubscriptionCancelled`, `SubscriptionResumed`, `InvoiceGenerated`, `PaymentSucceeded`, `PaymentFailed`, `RefundProcessed`, `WebhookReceived`.

**Jobs**

* `ProcessRecurringCharge`, `RetryFailedPayment`, `GenerateInvoicePdf`, `SendInvoiceEmail`, `SyncGatewayData`.

**Webhooks**

* Each gateway registers webhooks under `routes/vendor/billing/webhooks/{gateway}`.
* Webhooks pass through signature verification middleware, and all events are emitted into the app event bus so the host app can listen.

---

# Security & Compliance notes

* **PCI scope:** The package favors tokenization and hosted payment pages to keep PCI burden minimal. Payment data is not stored in raw form.
* **Secrets:** Gateway API keys should be stored in environment variables. The package may optionally encrypt keys in the DB using Laravel’s encryption helpers.
* **Webhooks:** Signature verification is enabled for supported gateways.
* **Privacy:** Customer PII is minimized; data export and deletion tools will be provided (GDPR/CCPA).
* **Access control:** Admin UI and CLI actions restricted by middleware/gates.

---

# Testing & CI expectations

* Unit tests for core billing logic and proration math.
* Integration tests that use a sandbox gateway adapter (mocked).
* Livewire component tests (Pest or PHPUnit).
* GitHub Actions pipeline: run tests, static analysis (PHPStan/Psalm), PHP-CS-Fixer, and publish docs.
* Aim for strong coverage of billing computations and webhook idempotency.

---

# Roadmap & Release Plan

**MVP (v0.x)**

* Plan/price model, subscriptions, Stripe adapter, basic Livewire UI, invoices, transactions, docs.

**v1.0**

* Multiple gateways (PayPal), proration, trials, coupons, PDF invoices, webhooks, transactional emails.

**v1.x**

* Metered billing, reconciliation tools, hosted checkout widget, multi-currency improvements, accounting integrations.

**v2.0**

* Multi-tenant capabilities, enterprise audit logs, advanced analytics dashboard.

Contributors: we will publish milestones and label issues for `good first issue`, `help wanted`, and `roadmap`.

---

# How to contribute

We welcome contributions of all kinds — code, documentation, tests, ideas, or translations.

**Before you contribute**

1. Read [CONTRIBUTING.md] (will be present in repo).
2. Open an issue to discuss larger features before writing code.
3. Follow PSR coding style; code must include tests for new behavior.

**Typical workflow**

1. Fork the repo, create a topic branch.
2. Run tests locally.
3. Open a PR with a clear description and link to any issue it addresses.
4. Add tests and docs for new features.

**Needed right now**

* README polish & design feedback
* Simple Stripe adapter improvements and test coverage
* Livewire admin scaffolding & UX feedback
* Translations and docs writers
* CI pipeline setup (GitHub Actions) and PHPStan config

---

# Governance, Code of Conduct & License

* **License:** MIT (default for open-source friendly adoption — can be adjusted if community prefers another).
* **Code of Conduct:** Contributor Covenant (we will add CODE_OF_CONDUCT.md).
* **Maintainers:** initially project creator + trusted collaborators. We will maintain a transparent decision log and release cadence.

---

# Support / Ways to help

* **Star the repo** — higher visibility helps attract contributors.
* **Comment** — tell us what gateway, feature, or region matters to you (eg. Razorpay, PayU, Mollie, Adyen, local/regional gateways).
* **Contribute code** — fork and make PRs. Small focused PRs are easiest to review.
* **Write docs / tutorials / integrations** — examples for Jetstream, Breeze, Inertia, Nova, Vapor, Forge.
* **Sponsor** — sponsorship helps pay for CI minutes, domain, docs hosting, and maintainers’ time.

---

# Example issue ideas to start

* Add unit tests for proration calculation with edge cases.
* Implement mock gateway that simulates network latencies and failures.
* Livewire CRUD for Coupons with validation and edge-case tests.
* Add localization for two languages (English + one other).
* Create a simple demo Laravel app showcasing the package with Stripe in test mode.

---

# Example `config/billing.php` (extract)

```php
return [
    'user_model' => env('BILLING_USER_MODEL', App\Models\User::class),
    'default_currency' => env('BILLING_DEFAULT_CURRENCY', 'USD'),
    'gateways' => [
        'stripe' => [
            'driver' => 'stripe',
            'key' => env('STRIPE_KEY'),
            'secret' => env('STRIPE_SECRET'),
            'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
        ],
        'paypal' => [
            'driver' => 'paypal',
            'client_id' => env('PAYPAL_CLIENT_ID'),
            'secret' => env('PAYPAL_SECRET'),
        ],
    ],
    'admin_route_prefix' => 'billing-admin',
    'admin_middleware' => ['web', 'auth', 'can:manage-billing'],
];
```

---

# FAQ (short)

**Q: Will this store credit cards?**
A: No raw card data — we rely on tokenization or hosted checkout flows. Full details in security docs.

**Q: What Laravel versions will be supported?**
A: We’ll target at least the current LTS and the latest minor — see `composer.json` for exact constraints. Backwards compatibility will be maintained where reasonable.

**Q: Can this support monthly and daily billing?**
A: Yes — billing intervals include days, weeks, months, years and custom intervals.

---

# Final call to action

If you care about this project, please:

1. **Star the repo** (when published).
2. **Open an issue** describing which features or gateways you care about most.
3. **Comment here** (or in the repo) with your use-case — SaaS, marketplace, accounting integrations, or country-specific payment needs.
4. **Join as a contributor** — even a small PR (typo fix, docs, unit test) is huge.
