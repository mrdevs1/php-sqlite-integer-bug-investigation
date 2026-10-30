
# PHP SQLite Large Integer Bug Investigation

![PHP](https://img.shields.io/badge/PHP-8.x-blue)
![SQLite](https://img.shields.io/badge/SQLite-Database-blue)
![C](https://img.shields.io/badge/Language-C-orange)
![Status](https://img.shields.io/badge/Status-Under%20Investigation-yellow)

## Overview

This project investigates a reported PHP PDO SQLite issue involving large integer corruption when using custom SQLite functions through `sqliteCreateFunction()`.

The purpose is to reproduce the reported behavior, investigate the underlying C implementation, understand integer conversion between PHP and SQLite, develop a potential solution, and validate the changes through regression testing.

**Original issue:** https://github.com/php/php-src/issues/25021

**Official repository:** https://github.com/php/php-src

## 1. Problem Description

SQLite supports 64-bit signed integer values. However, incorrect use of a 32-bit integer conversion function can cause large values to be truncated.

For example, the reported issue involves a value such as:

```php
1234000000000
```

The value may be received incorrectly inside a custom PHP SQLite function.

### Reproduction Example

```php
<?php

$db = new PDO('sqlite::memory:');

$db->sqliteCreateFunction(
    'cust',
    function ($value) {
        var_dump($value);
        return $value;
    },
    1
);

$stmt = $db->query(
    'SELECT cust(1234000000000)'
);

var_dump($stmt->fetchColumn());

?>
```

### Expected Behavior

The custom function should receive and return the original integer:

```text
1234000000000
```

### Reported Incorrect Behavior

The reported issue describes an incorrectly converted value:

```text
1344386048
```

The actual output should be verified against the affected PHP version and environment.

## 2. Root Cause Analysis

The investigation focuses on the PDO SQLite extension:

`ext/pdo_sqlite/sqlite_driver.c`

SQLite provides different functions for reading integer values:

- `sqlite3_value_int()` returns a 32-bit integer.
- `sqlite3_value_int64()` returns a 64-bit integer.

Similarly, SQLite provides different functions for returning integer values:

- `sqlite3_result_int()` accepts a 32-bit integer.
- `sqlite3_result_int64()` accepts a 64-bit integer.

Using a 32-bit conversion function for a value outside the signed 32-bit range can result in truncation.

The investigation therefore examines both conversion directions:

1. SQLite to PHP.
2. PHP to SQLite.

## 3. Proposed Source Code Fix

Open the PHP source file:

`ext/pdo_sqlite/sqlite_driver.c`

### Fix A: SQLite-to-PHP conversion

Locate the integer handling inside the SQLite callback argument conversion.

Before:

```c
case SQLITE_INTEGER:
    ZVAL_LONG(
        &zargs[i + is_agg],
        sqlite3_value_int(argv[i])
    );
    break;
```

Proposed change:

```c
case SQLITE_INTEGER:
    ZVAL_LONG(
        &zargs[i + is_agg],
        sqlite3_value_int64(argv[i])
    );
    break;
```

This uses SQLite's 64-bit integer accessor instead of the 32-bit accessor.

### Fix B: PHP-to-SQLite conversion

Investigate the callback return-value conversion.

Before:

```c
case IS_LONG:
    sqlite3_result_int(
        context,
        Z_LVAL(retval)
    );
    break;
```

Proposed change:

```c
case IS_LONG:
    sqlite3_result_int64(
        context,
        Z_LVAL(retval)
    );
    break;
```

This preserves large PHP integer return values when they are passed back to SQLite.

**Important:** These are proposed changes, not a confirmed upstream patch. Verify the exact source code, surrounding logic, PHP platform compatibility, and callback behavior before applying them.

## 4. Regression Testing

A regression test should verify that large integers survive the complete conversion process.

Example:

```php
<?php

$db = new PDO('sqlite::memory:');

$db->sqliteCreateFunction(
    'cust',
    function ($value) {
        return $value;
    },
    1
);

$value = 1234000000000;

$stmt = $db->query(
    'SELECT cust(1234000000000) AS result'
);

$result = $stmt->fetch(PDO::FETCH_ASSOC);

var_dump($result['result']);
var_dump($result['result'] === $value);

?>
```

Expected output:

```text
int(1234000000000)
bool(true)
```

### Additional Test Cases

| Test value | Purpose |
|---|---|
| 100 | Small integer |
| 2147483647 | Maximum signed 32-bit integer |
| 2147483648 | First integer above signed 32-bit range |
| 1234000000000 | Reported example |
| 9223372036854775807 | Maximum signed 64-bit integer |

The final test requires a 64-bit PHP build. Test values should also be checked for correct behavior on 32-bit platforms.

## 5. Development Environment

### Requirements

- PHP development environment
- C compiler
- SQLite development libraries
- Git
- Linux or another supported PHP build environment

### Clone PHP Source Code

```bash
git clone https://github.com/php/php-src.git

cd php-src
```

### Configure PHP

```bash
./buildconf

./configure --enable-cli --with-pdo-sqlite
```

### Compile PHP

```bash
make -j$(nproc)
```

Check the compiled PHP version:

```bash
sapi/cli/php -v
```

Check PDO SQLite support:

```bash
sapi/cli/php -m
```

## 6. Add a PHP Regression Test

Create a test in:

`ext/pdo_sqlite/tests/sqlite_create_function_large_integer.phpt`

Test content:

```php
--TEST--
PDO SQLite custom functions preserve large integers
--EXTENSIONS--
pdo_sqlite
--FILE--
<?php

$db = new PDO('sqlite::memory:');

$db->sqliteCreateFunction(
    'cust',
    fn($value) => $value,
    1
);

$value = 1234000000000;

$result = $db->query(
    'SELECT cust(1234000000000)'
)->fetchColumn();

var_dump($result);
var_dump($result === $value);

?>
--EXPECT--
int(1234000000000)
bool(true)
```

Run the test:

```bash
sapi/cli/php run-tests.php \
ext/pdo_sqlite/tests/sqlite_create_function_large_integer.phpt
```

Check whether the test passes with the original source and after applying the proposed changes.

## 7. Verify the Fix

The fix should be considered successful only after verifying:

- The original reproduction works correctly.
- Large integer arguments reach PHP without truncation.
- Large integer callback return values reach SQLite correctly.
- Existing PDO SQLite tests still pass.
- Small integer behavior remains unchanged.
- Supported 32-bit and 64-bit platform behavior is handled correctly.

Do not assume that changing one conversion function automatically fixes every integer conversion problem.

## 8. GitHub Repository Structure

This independent investigation can be organized as follows:

```text
php-sqlite-integer-bug-investigation/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── examples/
│   └── reproduce.php
│
├── tests/
│   └── regression.phpt
│
└── patches/
    └── proposed-fix.patch
```

The README contains the investigation, reproduction, technical explanation, proposed fix, and testing instructions.

## 9. Git Commands

Initialize your independent GitHub repository:

```bash
git init

git add .

git commit -m "Add PHP SQLite integer bug investigation"

git branch -M main
```

Connect your GitHub repository:

```bash
git remote add origin https://github.com/YOUR_USERNAME/php-sqlite-integer-bug-investigation.git
```

Push the project:

```bash
git push -u origin main
```

Replace `YOUR_USERNAME` with your GitHub username.

## 10. Contributing to PHP

After validating the proposed changes:

1. Fork the official PHP repository.
2. Create a dedicated bug-fix branch.
3. Apply the verified source changes.
4. Add the regression test.
5. Run the relevant PDO SQLite test suite.
6. Review the changes for compatibility.
7. Submit a pull request referencing the original issue.

Official repository:

https://github.com/php/php-src

## 11. Project Goals

- Investigate PHP core behavior.
- Understand PDO SQLite internals.
- Learn C programming in a real-world codebase.
- Understand 32-bit and 64-bit integer conversion.
- Reproduce a reported software bug.
- Develop and validate a potential fix.
- Write regression tests.
- Contribute to open-source development.

## 12. Technologies Used

- PHP
- C Programming
- SQLite
- PDO
- Git
- GitHub
- Linux

## 13. Project Status

**Current status:** Research and proposed solution.

The issue is being investigated independently. The proposed source changes and regression tests must be verified before describing the bug as fixed.

## 14. References

- PHP Source Code: https://github.com/php/php-src
- Original Issue #25021: https://github.com/php/php-src/issues/25021
- SQLite Integer API: https://www.sqlite.org/c3ref/value_blob.html
- PDO SQLite Documentation: https://www.php.net/manual/en/ref.pdo-sqlite.php
- PHP Documentation: https://www.php.net/docs.php

## Disclaimer

This is an independent educational and technical investigation. It is not an official PHP project and is not affiliated with or endorsed by the PHP development team.

---
