# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`beelab/paypal-bundle` is a Symfony bundle that wraps the Omnipay PayPal Express gateway into a single injectable service plus a Doctrine entity for tracking transactions. It is a library, not an application — there is no runnable app, only the bundle code and its test suite. Supports PHP 7.1+ and Symfony 3.4 / 4.4 / 5.0.

## Commands

```bash
composer install                  # install dependencies (bin-dir is ./bin, not ./vendor/bin)
bin/phpunit                       # run the full test suite
bin/phpunit --filter testStart    # run a single test by name
bin/phpunit Tests/Paypal/ServiceTest.php   # run one test file
bin/php-cs-fixer fix              # apply code style fixes (if php-cs-fixer is installed)
```

There is no separate build or lint step beyond PHPUnit and php-cs-fixer.

## Architecture

The bundle has three moving parts that fit together at runtime:

1. **`Paypal\Service`** (`Paypal/Service.php`) — the only class consumers interact with. Wired in `Resources/config/services.xml` as `beelab_paypal.service` (and aliased to its FQCN for autowiring). It takes an `Omnipay\PayPal\ExpressGateway`, the Symfony router, and the merged bundle config. Usage is a two-call flow:
   - `setTransaction(Transaction $t, array $custom = [])` builds the Omnipay parameters (amount, currency, description, transactionId, return/cancel URLs generated from configured route names). Custom params override defaults via `array_merge`.
   - `start()` calls `gateway->purchase(...)->send()`, expects a redirect response, and stores the PayPal token on the transaction. `complete()` calls `completePurchase(...)`, inspects the `ACK` field, and routes to `Transaction::complete()` or `Transaction::error()`.
   - Both throw `RuntimeException` if `setTransaction()` wasn't called first. `start()` throws `Paypal\Exception` on a non-redirect response.

2. **`Entity\Transaction`** (`Entity/Transaction.php`) — a Doctrine `@MappedSuperclass`, **abstract by design**. Consumers extend it in their own app to get a concrete `@Entity`. It carries status (KO/STARTED/OK/ERROR constants), token, amount, timestamps, and the raw PayPal response. The hooks `getDescription()`, `getItems()`, and `getShippingAmount()` return empty/null defaults and are meant to be overridden by the subclass. When `getItems()` is non-empty the service also sends shipping amount and per-item data — but PayPal rejects mismatches between item totals and the transaction amount.

3. **DI layer** (`DependencyInjection/`) — `BeelabPaypalExtension` processes config into the `beelab_paypal.config` container parameter and loads `services.xml`. `Configuration` defines the config tree: `username`/`password`/`signature`/`return_route`/`cancel_route` are required; `currency` defaults to `EUR`, `test_mode` to `false`. The `service_class` option is **deprecated** — extending `Service` and registering it as a public service is the supported replacement (see the `TODO` markers in `services.xml` and the extension).

## Testing notes

- PHPUnit bootstrap is `Tests/bootstrap.php` (just loads Composer autoload). Test files end in `Test.php` under `Tests/`.
- The `Test/` directory (singular, shipped with the bundle) provides a reusable stub for **consumers' own test suites**: `TransactionStub` provides a concrete transaction with sample items. Do not confuse `Test/` (shipped helpers) with `Tests/` (this bundle's own tests).

## Conventions

- Code style is enforced by `.php_cs`: `@Symfony` + `@Symfony:risky` + `@PHP71Migration:risky` + `@PHPUnit60Migration:risky`, short array syntax, ordered imports, and `native_function_invocation` (note the leading `\` on global functions like `\array_merge`, `\is_file`). `declare_strict_types` is intentionally **off**.