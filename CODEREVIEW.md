# Code Review: codraw/process

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **Finding #1** — Added `"php": "^8.5"` to `require` in `composer.json`, matching the intent of the constraint used by sibling packages (`codraw-core`, `codraw-dependency-injection`) with the bounded form described above.
- **Finding #2** — Added a `"suggest"` section to `composer.json` with `"codraw/dependency-injection": "To use the ProcessIntegration"`, advertising the optional dependency needed by `DependencyInjection/ProcessIntegration.php` (kept in `require-dev` since the integration is optional).
- **Finding #4** — Fixed the stale copy-pasted docblock in `Tests/DependencyInjection/ProcessIntegrationTest.php`: `@property` now references `ProcessIntegration` and the unused `Draw\Component\Console\DependencyInjection\ConsoleIntegration` import was removed.

- **Finding #3** — Replaced `...\func_get_args()` forwarding in `ProcessFactory::create()` and `ProcessFactory::createFromShellCommandLine()` with explicit parameter forwarding (`$command, $cwd, $env, $input, $timeout`). Behavior is directly pinned by `Tests/ProcessFactoryTest.php`, which asserts all five forwarded values for both methods in both default and fully-specified invocations (all tests pass unchanged).

Validation pass (2026-07-20): `composer install`, PHPUnit (8 tests, 36 assertions, OK), PHPStan (level 5, no errors) and markdownlint all green with the fixes above; no test-expectation updates were required.

Not fixed (deliberately left as open items): #5 (Symfony 7 widening — repo-wide policy decision), #6 (phpunit.xml.dist schema pin — cosmetic, no single correct value while `^11.3 || ^12.0` is allowed), #7 (`docs/soup.md` stub — compliance-process content decision).

## Overall Assessment

`codraw/process` is a deliberately tiny package: a `ProcessFactory` (two proxy methods around `symfony/process`) and a `ProcessIntegration` DI integration class. The code is correct, matches its contract (`Draw\Contracts\Process\ProcessFactoryInterface`), and is essentially fully covered by tests. There are no security flaws or functional bugs in the package's own code. The findings below are packaging/metadata gaps and small maintainability items rather than defects likely to break anything in production.

## Findings

### Medium

#### 1. **[FIXED]** `composer.json` declares no `php` version requirement

- **File:** `composer.json:17-20`
- The `require` block lists only `codraw/contracts` and `symfony/process`. Sibling packages in the monorepo (e.g. `codraw-core`) declare `"php": ">=8.5"`, but this package can be installed on any PHP version Composer allows, even though the code uses PHP 8 syntax (trailing comma in parameter lists in `ProcessFactory::createFromShellCommandLine`, `ProcessFactory.php:28-34`). On an unsupported PHP runtime this fails with a parse error at use-time instead of a clear Composer conflict at install-time. Add an explicit `"php"` constraint consistent with the rest of the framework.

### Low

#### 2. **[FIXED]** `ProcessIntegration` depends on classes from a dev-only dependency

- **File:** `DependencyInjection/ProcessIntegration.php:5-6,13-15`, `composer.json:21-24`
- `ProcessIntegration` implements `IntegrationInterface` and uses `IntegrationTrait` from `codraw/dependency-injection`, which is only in `require-dev`. A consumer who installs `codraw/process` standalone and autoloads `ProcessIntegration` (e.g. via a container class-map scan or an over-eager `registerClasses`) gets a fatal "interface not found" error. This is a common pattern for optional integrations, but it should at least be advertised via a `"suggest"` entry (`codraw/dependency-injection: required to use ProcessIntegration`), or better a `conflict` guard against incompatible versions.

#### 3. **[FIXED]** `func_get_args()` proxying is fragile

- **File:** `ProcessFactory.php:25,35`
- Both methods forward with `...\func_get_args()`. This works today (PHP fills defaults for skipped parameters, including with named arguments), but it silently couples the factory's parameter order to `Process::__construct()` / `Process::fromShellCommandline()`, forwards any extra positional arguments the caller mistakenly passes, and is opaque to static analysis. Forwarding the named parameters explicitly (`new Process($command, $cwd, $env, $input, $timeout)`) is equally short and self-verifying.

#### 4. **[FIXED]** Stale copy-pasted docblock in the integration test

- **File:** `Tests/DependencyInjection/ProcessIntegrationTest.php:5,15`
- The class docblock declares `@property ConsoleIntegration $integration` and imports `Draw\Component\Console\DependencyInjection\ConsoleIntegration` — a class from `codraw/console`, which is not a dependency of this package at all (not even in `require-dev`). It is clearly copy-pasted from the console package's test. Harmless at runtime (docblock + use statement only), but misleading and it references a class that may not exist in a standalone checkout. It should say `@property ProcessIntegration $integration`.

#### 5. Symfony 7 not allowed

- **File:** `composer.json:19`
- `symfony/process` is constrained to `^6.4.0` only. This matches the repo-wide policy (siblings also pin `^6.4.0`), so it is a coordinated-upgrade concern rather than a package bug, but nothing in this package's code prevents `^6.4 || ^7.0`, and widening it would ease adoption.

#### 6. `phpunit.xml.dist` schema pinned to 11.3 while `require-dev` allows PHPUnit 12

- **File:** `phpunit.xml.dist:4`, `composer.json:23`
- `noNamespaceSchemaLocation` points at the 11.3 schema, but `phpunit/phpunit` allows `^12.0`. Under PHPUnit 12 the config still loads, but schema validation/IDE assistance is against the wrong version. Minor housekeeping.

#### 7. `docs/soup.md` is an empty stub

- **File:** `docs/soup.md`
- The Software-of-Unknown-Provenance table contains only the header row. If this document is part of a compliance process (the presence of `trivy.yaml` suggests so), it should either list `symfony/process`/`codraw/contracts` or be removed.

## Strengths

- **Focused, contract-driven design.** The whole point of the package — making process creation injectable and mockable — is achieved with a minimal surface. `ProcessFactory` exactly implements `Draw\Contracts\Process\ProcessFactoryInterface`, so consumers can depend on the contract and swap a test double for real process execution.
- **Safe-by-default API.** `create()` takes the command as an array (Symfony's injection-safe form) and both methods keep Symfony's sane 60-second default timeout rather than defaulting to unlimited. The shell-string variant is clearly named (`createFromShellCommandLine`) so the riskier path is opt-in and visible at call sites.
- **DI integration follows the framework conventions**: classes registered via the shared `IntegrationTrait` (which excludes `DependencyInjection/`, `Tests/`, `vendor/` automatically), interface aliased to the implementation, and definitions renamed to the stable `draw.process.*` id namespace (`DependencyInjection/ProcessIntegration.php:22-37`).
- **Clean static-analysis posture**: PHPStan level 5 with an empty baseline (`phpstan-baseline.neon` has `ignoreErrors: []`) — no suppressed debt.

## Test Coverage

Coverage is effectively complete for a package this size:

- `Tests/ProcessFactoryTest.php` covers both factory methods with both default and fully-specified arguments, asserting command line, cwd, env, input, and timeout on the resulting `Process`. Tests only construct the `Process` (never run it), so they are fast and portable.
- `Tests/DependencyInjection/ProcessIntegrationTest.php` exercises `ProcessIntegration::load()` through the shared `IntegrationTestCase`, asserting the `draw.process.process_factory` service id and the `ProcessFactoryInterface` alias.

Untested (all trivial): named-argument invocation of the factory methods, and behavior when `codraw/dependency-injection` is absent (finding #2). No gaps of practical concern.
