---
id: solid-lsp
title: "Liskov Substitution Principle"
related: [solid-ocp, solid-isp]
---

# Liskov Substitution Principle

An implementation must honor the shared contract's inputs, results, failures, invariants, and side effects. Compatible PHP signatures are not enough.

## Scenario

Checkout accepts any `Discount`: all non-negative subtotals are valid and the discount must be between zero and the subtotal.

## Problem

```php
<?php

declare(strict_types=1);

interface Discount
{
    /**
     * Accept every non-negative subtotal in minor currency units.
     * Return an amount in 0..subtotalMinor.
     * @throws InvalidArgumentException for a negative subtotal.
     */
    public function amount(int $subtotalMinor): int;
}

final class MinimumOrderDiscount implements Discount
{
    public function amount(int $subtotalMinor): int
    {
        // Stronger precondition: rejects inputs allowed by Discount.
        if ($subtotalMinor < 10000) {
            throw new InvalidArgumentException('Minimum subtotal is 10000');
        }

        return 1000;
    }
}

final class UncappedDiscount implements Discount
{
    public function amount(int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        return 1000; // Weaker guarantee: exceeds a subtotal of 500.
    }
}

function payableTotal(Discount $discount, int $subtotalMinor): int
{
    return $subtotalMinor - $discount->amount($subtotalMinor);
}

// payableTotal(new MinimumOrderDiscount(), 500) unexpectedly throws.
// payableTotal(new UncappedDiscount(), 500) returns -500.
```

## Improvement

```php
<?php

declare(strict_types=1);

interface Discount
{
    /**
     * Accept every non-negative subtotal in minor currency units.
     * Return an amount in 0..subtotalMinor.
     * @throws InvalidArgumentException for a negative subtotal.
     */
    public function amount(int $subtotalMinor): int;
}

final class MinimumOrderDiscount implements Discount
{
    public function amount(int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        // Preserve the original promotion: only qualifying orders get 1000.
        return $subtotalMinor >= 10000 ? 1000 : 0;
    }
}

final class CappedDiscount implements Discount
{
    public function __construct(private int $fixedMinor)
    {
        if ($fixedMinor < 0) {
            throw new InvalidArgumentException('Discount cannot be negative');
        }
    }

    public function amount(int $subtotalMinor): int
    {
        if ($subtotalMinor < 0) {
            throw new InvalidArgumentException('Subtotal cannot be negative');
        }

        return min($this->fixedMinor, $subtotalMinor);
    }
}

function payableTotal(Discount $discount, int $subtotalMinor): int
{
    return $subtotalMinor - $discount->amount($subtotalMinor);
}

$minimum = new MinimumOrderDiscount();
$capped = new CappedDiscount(1000);
// payableTotal($minimum, 500) returns 500: not eligible for the promotion.
// payableTotal($minimum, 10000) returns 9000: eligible orders keep the 1000 discount.
// payableTotal($capped, 500) returns 0: the fixed discount cannot exceed the subtotal.
// The same client works with both implementations without instanceof checks.
```

## Review checks

- **Flag:** show an input or operation sequence permitted by the base contract that breaks under a subtype.
- **Do not flag:** valid covariance/contravariance, implementation details, or rejection of inputs the contract forbids.
- **Fix:** meet the shared contract or redesign it. Returning zero below a minimum order preserves the promotion rule but changes the old exception behavior; agree on that correction before deployment.
- **Verify:** shared contract tests across implementations, including boundaries, exceptions, and state transitions.
