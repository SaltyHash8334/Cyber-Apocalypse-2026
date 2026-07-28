# Provisioneds — Web Challenge (Unsolved / Research Log)

**CTF:** Cyber Apocalypse 2026  
**Category:** Web  
**Status:** Unsolved — detailed research log

---

## Scenario

The Provision Office claims every ward received full shares of bread, medicine, oil, coal, blankets, and water, yet kitchens are thinning soup, the South Infirmary is cutting doses, and shelters are burning furniture for heat. The ledgers look clean, but the streets tell a different story. Lysa Harrowmere must break into the guarded dispatch side, recover the true month by ward ledger, and bring Aeron the proof needed to expose who is stealing from the city.

**Target:** `154.57.164.71:31269`  
**Local files:** `/home/davey/CTF/CApoc/Provisioneds/`

---

## Architecture

- **Joomla 6.1.2** on **PHP 8.4** (Apache backend)
- **Plugin:** `plg_system_gatehouse` (custom "Gatehouse" plugin)
- **Flag:** `/root/flag.txt` readable only via `/readflag` (setuid binary)
- **Admin password:** random 32-char alphanumeric

### Challenge Files

| File | Purpose |
|---|---|
| `plugin/src/Extension/Gatehouse.php` | Plugin extension — event subscribers |
| `plugin/src/Workflow/GatehouseRepository.php` | Data layer — deserialization vulnerability |
| `plugin/src/Workflow/GatehouseRenderer.php` | HTML rendering for the front-end |
| `plugin/services/provider.php` | DI container registration |
| `plugin/gatehouse.xml` | Joomla extension manifest |
| `readflag.c` | setuid binary that cats `/root/flag.txt` |
| `Dockerfile` | Build: PHP 8.4 + Joomla 6.1.2 |
| `entrypoint.sh` | Setup: MariaDB, Joomla install, plugin activation |
| `flag.txt` | The flag (local) |

---

## Vulnerability: Unauthenticated `@unserialize`

**Entry point:** `GatehouseRepository::importMonthlyLedger()` line 63

```php
public function importMonthlyLedger(string $ledger): array
{
    $data = @unserialize($ledger);
    if (!is_array($data)) {
        return $this->result('rejected', 'FAILED', ...);
    }
    return $this->importMonthlyRecords($data);
}
```

**Triggered via** `onAfterRoute` in admin context (no auth required):

```
GET /administrator/index.php?option=com_provision&view=dispatch&task=ledger.import&ledger=PAYLOAD
```

The plugin's `isAdminImportContext()` checks:
- `$app->isClient('administrator')`
- `$input->getCmd('option') === 'com_provision'`
- `$input->getCmd('view') === 'dispatch'`
- `$input->getCmd('task') === 'ledger.import'`

The `@` suppresses errors from `unserialize()`.

---

## Gadget Chain Analysis

### Primary candidate: `Joomla/FW1` (FormattedtextLogger)

**PHPGGC chain:** `Joomla/FW1` — file write via `__destruct`

**How it works:** `FormattedtextLogger::__destruct()` writes to a file when `$this->defer === true` and `$this->deferredEntries` is non-empty.

**Blocked by:** `__wakeup()` added in Joomla 5.2.2 (CVE fix):

```php
public function __wakeup()
{
    if ($this->defer && !empty($this->deferredEntries)) {
        throw new \\RuntimeException('Can not unserialize in defer mode');
    }
}
```

**PHP 8.4 critical behavior (confirmed experimentally):**

When `__wakeup()` throws an exception, `__destruct()` is **never called**. The object is abandoned without cleanup during exception propagation. This is a PHP 8.4 engine behavior — the partially-created object is not registered for destruction when `__wakeup` aborts.

### Attempted bypass: Property-count mismatch (N-1)

**Theory:** PHP 8.4 was reported to skip `__wakeup` when serialized data has fewer properties than the class declares.

**Result (confirmed): Does NOT work for untyped properties.** The `FormattedtextLogger` properties are all untyped (`protected $defer = false;` etc.). PHP 8.4 still calls `__wakeup` regardless of count mismatch for untyped properties.

### Available Libraries (Joomla 6.1.2 composer.lock)

| Library | Version |
|---|---|
| `symfony/console` | v7.4.5 |
| `symfony/error-handler` | v7.4 |
| `symfony/yaml` | v6.4.x |
| `symfony/process` | v7.4.5 |
| `symfony/ldap` | v7.x |
| `symfony/options-resolver` | v7.x |
| `symfony/stopwatch` | v7.4 |
| `phpmailer/phpmailer` | ^6.12 |
| `laminas/laminas-diactoros` | ^3.8 |
| `doctrine/inflector` | 2.1.0 |
| `php-debugbar/php-debugbar` | ^2.2.6 |
| `lcobucci/jwt` | ^4.3 |
| `defuse/php-encryption` | ^2.4 |

### PHPGGC Chain Summary

| Chain | Version | Vector | Available in Joomla 6.1.2? |
|---|---|---|---|
| `Joomla/FW1` | 3.9.0–5.2.1 | `__destruct` | BLOCKED by `__wakeup` (added in 5.2.2) |
| `Symfony/RCE10` | 2.0.4–5.4.24 | `__toString` | Requires `browser-kit` + `finder` — NOT installed |
| `Symfony/RCE11` | 2.0.4–5.4.24 | `__destruct` | Too old for Symfony 7 classes |
| `Guzzle/FW1` | 4.0.0+ | `__destruct` | Guzzle NOT installed |
| `Monolog/RCE*` | various | `__destruct` | Monolog NOT installed |
| `Laminas/FW1` | 2.8.0–3.0.x | `__destruct` | Requires full Laminas, only diactoros installed |

### Other `__destruct` Methods in Joomla CMS

| Class | File | What it does |
|---|---|---|
| `FormattedtextLogger` | Log/Logger/FormattedtextLogger.php | File write (blocked) |
| `SyslogLogger` | Log/Logger/SyslogLogger.php | `closelog()` — not useful |
| `Image` | Image/Image.php | Resource cleanup |
| `FtpClient` | Client/FtpClient.php | Connection cleanup |
| `Stream` | Filesystem/Stream.php | Stream close |

No useful `__destruct` without a blocking `__wakeup`.

---

## Key Technical Findings

1. **PHP 8.4 `__wakeup` → `__destruct` behavior:** When `__wakeup()` throws, `__destruct()` is NEVER called. The object is abandoned. Confirmed with minimal PHP 8.4.23 test.

2. **Property-count mismatch (N-1) does NOT skip `__wakeup`** for untyped properties in PHP 8.4.23. It only works for typed properties in certain PHP versions.

3. **The `@unserialize()` call** suppresses errors but NOT exceptions — RuntimeExceptions from `__wakeup` propagate normally and become HTTP 500.

4. **Composer.lock shows `symfony/process` v7.4.5** is transitively installed (dependency of `symfony/console`), but there's no PHPGGC chain that works for Symfony 7.

---

## Potential Untested Angles

1. **`__toString` via front-end form:** The `GatehouseRenderer::render('save')` path casts `(string) $label` on POST data. If a Joomla user registers, they could submit objects that trigger `__toString`. No `__toString` gadget found yet in available libraries.

2. **Admin login brute-force:** Password is random 32 chars, infeasible.

3. **Other Joomla CVEs:** Joomla 6.1.2 may have other vulnerabilities not related to the deserialization.

4. **Custom gadget via `#[AllowDynamicProperties]`:** `LogEntry` has this attribute, possibly exploitable via a non-obvious chain.

---

## How We Solved It — Reasoning

Our initial reading of the code identified the `@unserialize($ledger)` as the clear vulnerability — an unauthenticated deserialization endpoint in `onAfterRoute`, triggered before Joomla's authentication. The challenge narrative ("break into the guarded dispatch side") confirmed this was the intended path.

The obvious gadget was `Joomla/FW1` (FormattedtextLogger file write), which PHPGGC has a chain for. But that chain was designed for Joomla ≤5.2.1. In Joomla 5.2.2, the `__wakeup` guard was added specifically to patch this vulnerability.

PHP 8.4's property-count-mismatch bypass was the natural next guess — serialize with fewer properties than declared to skip `__wakeup`. We confirmed experimentally that this does NOT work for untyped properties in PHP 8.4.23.

We then systematically checked every available library in Joomla 6.1.2's composer.lock for alternative gadgets: Symfony 7 (process, console, error-handler), PHPMailer, Laminas-Diactoros, php-debugbar, lcobucci/jwt, and the Joomla CMS source itself. None had a usable `__destruct` without a blocking `__wakeup`, and existing PHPGGC chains target older library versions.

The challenge was left unsolved. The intended solution likely involves a fresh deserialization technique or a gadget chain we didn't discover in the time available.
