---
id: solid-isp
title: "Interface Segregation Principle"
related: [solid-srp, solid-lsp]
---

# Interface Segregation Principle

Clients should depend only on capabilities they need; implementations should not promise unsupported operations.

## Scenario

A preview reads archived reports, but an immutable snapshot cannot write. Missing reports return `null`.

## Problem

```php
<?php

declare(strict_types=1);

interface ReportStorage
{
    public function read(string $name): ?string;
    /** Store the report, replacing existing content with the same name. */
    public function write(string $name, string $content): void;
}

final class SnapshotArchive implements ReportStorage
{
    /** @param array<string, string> $reports */
    public function __construct(private array $reports) {}

    public function read(string $name): ?string
    {
        return $this->reports[$name] ?? null;
    }

    public function write(string $name, string $content): void
    {
        throw new LogicException('Archive is read-only');
    }
}

function preview(ReportStorage $reports, string $name): string
{
    return substr($reports->read($name) ?? 'Report unavailable', 0, 120);
}
```

## Improvement

```php
<?php

declare(strict_types=1);

interface ReportReader
{
    public function read(string $name): ?string;
}

interface ReportWriter
{
    /** Store the report, replacing existing content with the same name. */
    public function write(string $name, string $content): void;
}

final class SnapshotArchive implements ReportReader
{
    /** @param array<string, string> $reports */
    public function __construct(private array $reports) {}

    public function read(string $name): ?string
    {
        return $this->reports[$name] ?? null;
    }
}

function preview(ReportReader $reports, string $name): string
{
    return substr($reports->read($name) ?? 'Report unavailable', 0, 120);
}

$archive = new SnapshotArchive(['sales' => 'Sales report']);
// preview($archive, 'sales') still returns 'Sales report'.
// A preview test double needs only read(); the archive promises no write().
```

## Review checks

- **Flag:** a client depends on irrelevant capabilities or an implementation cannot honor a required method.
- **Do not flag:** a cohesive multi-method interface or genuinely documented optional behavior.
- **Fix:** split by client role, not by method count. Update consumers and implementations: this is an API migration.
- **Verify:** read/preview behavior, write clients, and supported operations on each implementation.
