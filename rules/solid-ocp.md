---
id: solid-ocp
title: "Open/Closed Principle"
related: [solid-lsp, solid-dip]
---

# Open/Closed Principle

Use extension points when independent variants are a confirmed requirement or repeatedly change stable consumer logic.

## Scenario

Checkout adds independently maintained shipping tariffs. Callers select a tariff per quote; amounts are non-negative minor units.

## Problem

```php
<?php

declare(strict_types=1);

final class ShippingCosts
{
    public function quote(string $method, int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        // Every new tariff requires editing this consumer.
        return match ($method) {
            'standard' => $subtotalMinor >= 5000 ? 0 : 500,
            'express' => 1500,
            default => throw new InvalidArgumentException('Unknown shipping method'),
        };
    }
}
```

## Improvement

```php
<?php

declare(strict_types=1);

interface ShippingRate
{
    /** For a non-negative subtotal, return a non-negative shipping fee. */
    public function cost(int $subtotalMinor): int;
}

final class StandardRate implements ShippingRate
{
    public function cost(int $subtotalMinor): int
    {
        return $subtotalMinor >= 5000 ? 0 : 500;
    }
}

final class ExpressRate implements ShippingRate
{
    public function cost(int $subtotalMinor): int
    {
        return 1500;
    }
}

final class ShippingCosts
{
    /** @param array<string, ShippingRate> $rates */
    public function __construct(private array $rates) {}

    public function quote(string $method, int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        $rate = $this->rates[$method] ?? null;
        if ($rate === null) {
            throw new InvalidArgumentException('Unknown shipping method');
        }

        return $rate->cost($subtotalMinor);
    }
}

// Composition root: enable a new tariff without changing ShippingCosts.
final class PickupRate implements ShippingRate
{
    public function cost(int $subtotalMinor): int
    {
        return 0;
    }
}

$shipping = new ShippingCosts([
    'standard' => new StandardRate(),
    'express' => new ExpressRate(),
    'pickup' => new PickupRate(),
]);
// quote('standard', 4999) remains 500; quote('standard', 5000) remains 0.
// quote('pickup', 1000) adds a new supported variant.
```

## Review checks

- **Flag:** repeated edits to stable consumer logic, or a design that would make independent variants hard to add when they are a confirmed requirement.
- **Do not flag:** a fixed `match` or `switch`, a bug fix, or a one-off variation on its own.
- **Fix:** inject a strategy or register implementations when per-call selection is required; configuration may still change.
- **Verify:** existing tariffs, thresholds, unknown-method errors, and a newly registered tariff.
