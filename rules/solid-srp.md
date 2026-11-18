---
id: solid-srp
title: "Single Responsibility Principle"
related: [solid-isp, solid-dip]
---

# Single Responsibility Principle

Separate a module when it has independent reasons to change, not because it has many methods.

## Scenario

Billing owns tax calculation; integrations owns invoice JSON. Both change independently, and tax is also needed elsewhere. Amounts are non-negative minor units; 20% half-up tax is illustrative.

## Problem

```php
<?php

declare(strict_types=1);

final class InvoiceExporter
{
    public function export(int $subtotalMinor): string
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        // Tax policy and the integration's JSON schema live together.
        $taxMinor = intdiv($subtotalMinor, 5) + ($subtotalMinor % 5 >= 3 ? 1 : 0);
        if ($subtotalMinor > PHP_INT_MAX - $taxMinor) {
            throw new OverflowException('Invoice total exceeds integer range');
        }

        return json_encode([
            'subtotal_minor' => $subtotalMinor,
            'tax_minor' => $taxMinor,
            'total_minor' => $subtotalMinor + $taxMinor,
        ], JSON_THROW_ON_ERROR);
    }
}
```

## Improvement

```php
<?php

declare(strict_types=1);

final class InvoiceTax
{
    public function calculate(int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        return intdiv($subtotalMinor, 5) + ($subtotalMinor % 5 >= 3 ? 1 : 0);
    }
}

final class InvoiceExporter
{
    public function __construct(private InvoiceTax $tax) {}

    public function export(int $subtotalMinor): string
    {
        $taxMinor = $this->tax->calculate($subtotalMinor);
        if ($subtotalMinor > PHP_INT_MAX - $taxMinor) {
            throw new OverflowException('Invoice total exceeds integer range');
        }

        return json_encode([
            'subtotal_minor' => $subtotalMinor,
            'tax_minor' => $taxMinor,
            'total_minor' => $subtotalMinor + $taxMinor,
        ], JSON_THROW_ON_ERROR);
    }
}

$tax = new InvoiceTax();
$exporter = new InvoiceExporter($tax);
// $tax->calculate(10000) returns 2000, independently of JSON export.
// $exporter->export(10000) preserves the original JSON schema and amounts.
```

## Review checks

- **Flag:** identify independent change drivers and the concrete coupling they cause.
- **Do not flag:** large classes, multiple cohesive methods, or use-case coordinators.
- **Fix:** extract independently changing policy; no interface without a need for alternatives.
- **Verify:** JSON, errors, rounding boundaries, and overflow behavior before/after.
