---
id: solid-dip
title: "Dependency Inversion Principle"
related: [solid-ocp, solid-srp]
---

# Dependency Inversion Principle

Application policy should depend on its own contracts; infrastructure supplies the implementations. Dependency injection alone is not dependency inversion.

## Scenario

Replenishment policy uses SQL today but must also support a warehouse API and database-free tests. Unknown SKUs are errors. Quantities and thresholds are expected to be non-negative. Both examples assume a PDO connection configured to throw on database errors.

## Problem

```php
<?php

declare(strict_types=1);

final class ReorderPolicy
{
    public function __construct(private PDO $db, private int $threshold)
    {
        if ($threshold < 0) {
            throw new InvalidArgumentException('Threshold cannot be negative');
        }
    }

    public function shouldReorder(string $sku): bool
    {
        // The policy is tied to SQL and the inventory table.
        $query = $this->db->prepare('SELECT available FROM inventory WHERE sku = :sku');
        $query->execute(['sku' => $sku]);
        $available = $query->fetchColumn();
        if ($available === false) {
            throw new OutOfBoundsException('Unknown product');
        }

        return (int) $available < $this->threshold;
    }
}
```

## Improvement

```php
<?php

declare(strict_types=1);

// Application-owned port: no PDO, SQL, or warehouse SDK types.
interface StockLevels
{
    /** Return stock for an existing SKU, or null if unknown.
     * The inventory source is expected to contain non-negative integer quantities.
     */
    public function availableFor(string $sku): ?int;
}

final class ReorderPolicy
{
    public function __construct(private StockLevels $stock, private int $threshold)
    {
        if ($threshold < 0) {
            throw new InvalidArgumentException('Threshold cannot be negative');
        }
    }

    public function shouldReorder(string $sku): bool
    {
        $available = $this->stock->availableFor($sku);
        if ($available === null) {
            throw new OutOfBoundsException('Unknown product');
        }

        return $available < $this->threshold;
    }
}

// Infrastructure adapter implements the application's contract.
final class PdoStockLevels implements StockLevels
{
    public function __construct(private PDO $db) {}

    public function availableFor(string $sku): ?int
    {
        $query = $this->db->prepare('SELECT available FROM inventory WHERE sku = :sku');
        $query->execute(['sku' => $sku]);
        $available = $query->fetchColumn();

        if ($available === false) {
            return null;
        }

        // Preserve the original PDO value conversion. Tightening stored-data
        // validation is a separate behavior change, not part of the DIP refactor.
        return (int) $available;
    }
}

// A test can run the real policy without a database or vendor SDK.
$stock = new class implements StockLevels {
    public function availableFor(string $sku): ?int
    {
        return ['LOW' => 2, 'ENOUGH' => 5][$sku] ?? null;
    }
};
$policy = new ReorderPolicy($stock, 5);
// shouldReorder('LOW') is true; shouldReorder('ENOUGH') is false.
// shouldReorder('UNKNOWN') throws the same OutOfBoundsException as before.
```

## Review checks

- **Flag:** policy depends on SQL, SDKs, or a service locator, obstructing a required alternative or isolation.
- **Do not flag:** PDO inside a storage adapter, bootstrap wiring, or concrete value objects.
- **Fix:** define a narrow application-owned port and wire infrastructure at bootstrap. Preserve existing error handling during the refactor.
- **Verify:** policy against a fake source, missing SKUs, database failures, and baseline casts. Tightening stored-value validation is a separate behavior change.
