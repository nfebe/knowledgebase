---
description: Scaffold a new whilesmart/eloquent-<noun> Laravel package to the WhileSmart conventions
argument-hint: <noun> "<one-line description>" [domain fields/notes]
allowed-tools: Read, Write, Edit, Bash, WebFetch
---

Scaffold a new standalone Laravel package **`whilesmart/eloquent-$1`** under `~/work/whilesmartphp/eloquent-$1/`, following the WhileSmart package conventions exactly. Arguments: `$ARGUMENTS` (first token is the noun, then a quoted description, then any domain fields/behaviours to model).

## Ground the work first (do not invent)
1. Read the conventions: https://github.com/whilesmart/conventions/blob/main/php/laravel/packages/README.md (use WebFetch).
2. Mirror an existing exemplar package as the structural template: read `~/work/whilesmartphp/eloquent-assets/` in full (composer.json, ServiceProvider, Model, migration, config, Http/{Controllers,Requests,Resources}, Enums, Traits, routes, database/factories, tests, Makefile, docker-compose.yml, phpunit.xml, .github/workflows, README, CHANGELOG, LICENSE).
3. Read `~/work/whilesmartphp/eloquent-owner-access/` for the authz contract and concerns.
4. If the domain needs live data / an external integration, also skim `~/work/whilesmartphp/eloquent-holdings/` for how a provider is abstracted behind a contract.

## Hard rules (the conventions, encoded)
- **Naming/namespace:** package `whilesmart/eloquent-<noun>`, namespace `Whilesmart\<Noun>\…`, provider `Whilesmart\<Noun>\<Noun>ServiceProvider` registered via `extra.laravel.providers`. PSR-4 maps `src/` and `database/factories/`.
- **Polymorphic owner, never user_id.** Migration uses `$table->morphs('owner')`; model has `owner(): MorphTo`. An owner trait `Has<Noun>s` exposes `morphMany`.
- **Authorization via `whilesmart/eloquent-owner-access`** (require `^1.0`). Controllers `use AuthorizesOwnerController` and call `scopeAccessibleOwners()` in index and `authorizeAccessTo()` in show/update/destroy/actions. FormRequests `use AuthorizesOwnerRequest`: store calls `authorizeOwnerInRequest()`, update calls `authorizeOwnerOfBoundModel('<routeKey>')`. Never reference the host's User model.
- **Config-driven & host-agnostic.** `config/<noun>.php` with `register_routes`, `route_prefix`, `route_middleware`, and `<noun>_table`. ServiceProvider `mergeConfigFrom` in `register()`, `loadMigrationsFrom` + conditional route registration gated by `config('<noun>.register_routes')` in `boot()`, plus publishes for config and migrations. Model `getTable()` returns `config('<noun>.<noun>_table', '<noun>s')`. Migration table name reads the same config.
- **No integration hard-coded in the schema or package.** If the domain pulls external data (prices, rates, geocoding…), define a `Contracts\<X>Provider` interface, ship a `Null<X>Provider` bound via `bindIf`, and let the host bind a real adapter. Store provider-neutral columns (`provider`, `external_ref`), not vendor names. Emit a domain event (e.g. `<Noun>sUpdated`) for the host to bridge — don't import app classes.
- **Soft deletes**, sensible per-owner indexes, `metadata` json column when useful. Enums are string-backed with a `values(): array` helper; the model casts to them.
- **Tests** use Orchestra Testbench: `tests/TestCase.php` extends `Orchestra\Testbench\TestCase`, `use RefreshDatabase`, `defineDatabaseMigrations()` loads the package migrations, `getPackageProviders()` returns `[OwnerAccessServiceProvider::class, <Noun>ServiceProvider::class]`, and `getEnvironmentSetUp()` sets `route_middleware` to `['api']`. Cover: CRUD through the HTTP boundary, owner-scoping/403 with a strict bound `OwnerAuthorizer`, and any provider/contract behaviour with a fake bound implementation. `phpunit.xml` uses sqlite `:memory:`.
- **Meta files:** `composer.json` (php `^8.2`, laravel `^11.0|^12.0`, dev: `orchestra/testbench ^9|^10`, `laravel/pint ^1.22`, `fakerphp/faker`; scripts `test`/`pint`/`pint:test`; `minimum-stability: dev`, `prefer-stable: true`), `docker-compose.yml` (image `ghcr.io/whilesmartphp/laravel-dev:package-8.4`, mount `.:/app`, `APP_ENV=testing`), `Makefile` (`-include package.mk` + a `bootstrap` target), `.gitignore` (vendor, composer.lock, package.mk), `README.md`, `CHANGELOG.md`, MIT `LICENSE` (WhileSmart LTD), and `.github/workflows/{checks,commits,release}.yml` copied from the exemplar (checks runs matrix PHP 8.2/8.3/8.4 + `vendor/bin/testbench package:test` + `composer pint:test`; release uses `whilesmart/workflows/package/release@main` with `package: whilesmart/eloquent-<noun>`).

## Build, then verify in a container
After writing every file, validate without needing the package-specific image: run a one-off `laravel-dev` container (the `:8.2` tag is already local) with the package mounted, install deps, lint, and run the suite:

```
cd ~/work/whilesmartphp/eloquent-<noun>
docker run --rm -v "$PWD":/app -w /app ghcr.io/whilesmartphp/laravel-dev:8.2 \
  bash -lc "composer install --no-interaction --no-progress && vendor/bin/pint && vendor/bin/testbench package:test"
```

Iterate until Pint is clean and all tests pass. Do not commit, tag, or publish — leave the package ready for review. Report the file tree, the endpoints/contract surface, and the passing test count. No AI attribution anywhere.
