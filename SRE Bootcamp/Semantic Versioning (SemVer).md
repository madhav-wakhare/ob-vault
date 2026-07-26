
> **Semantic Versioning (SemVer)** is a standard versioning scheme that communicates the impact of software changes using version numbers.
>
> Official Format:
>
> ```text
> MAJOR.MINOR.PATCH
> ```

---

# Why SemVer Exists

Without a versioning standard, users cannot easily determine whether an update:

- Fixes bugs
- Adds features
- Breaks compatibility

SemVer allows developers to estimate upgrade risk just by looking at the version number.

Example:

```text
4.18.2 → 4.18.3
```

Likely safe (bug fixes only).

```text
4.18.2 → 5.0.0
```

Potentially risky (breaking changes).

---

# Version Structure

```text
MAJOR.MINOR.PATCH
```

Example:

```text
2.4.1
```

| Component | Value |
|------------|--------|
| MAJOR | 2 |
| MINOR | 4 |
| PATCH | 1 |

---

# MAJOR Version

## When to Increment?

Increase MAJOR when introducing **backward-incompatible changes**.

Example:

```text
1.5.2 → 2.0.0
```

---

## Examples of Breaking Changes

- Removing API endpoints
- Renaming functions
- Changing request formats
- Changing response formats
- Removing supported features
- Changing function signatures

---

## Rule

```text
MAJOR++
MINOR = 0
PATCH = 0
```

Example:

```text
1.7.5 → 2.0.0
```

---

# MINOR Version

## When to Increment?

Increase MINOR when adding **new functionality** without breaking existing users.

Example:

```text
1.4.2 → 1.5.0
```

---

## Examples

- New API endpoint
- New feature
- New optional parameter
- Performance enhancement
- New CLI command

Existing functionality continues to work.

---

## Rule

```text
MINOR++
PATCH = 0
```

Example:

```text
1.4.9 → 1.5.0
```

---

# PATCH Version

## When to Increment?

Increase PATCH when making **backward-compatible bug fixes**.

Example:

```text
1.4.2 → 1.4.3
```

---

## Examples

- Bug fixes
- Security patches
- Internal optimizations
- Logging improvements

No API changes.

---

## Rule

```text
PATCH++
```

Example:

```text
1.4.2 → 1.4.3
```

---

# Easy Memory Trick

```text
MAJOR.MINOR.PATCH

Break.New.Fix
```

Or:

```text
Breaking Changes
New Features
Bug Fixes
```

| Version Part | Meaning |
|-------------|---------|
| MAJOR | Breaking Changes |
| MINOR | New Features |
| PATCH | Bug Fixes |

---

# Before Version 1.0.0

Versions below:

```text
1.0.0
```

are considered:

```text
Initial Development
```

Examples:

```text
0.1.0
0.5.0
0.9.0
```

---

## Important Rule

Before 1.0.0:

- APIs may change anytime
- No backward compatibility guarantees
- Frequent breaking changes are acceptable

---

# Released Versions are Immutable

Once released:

```text
1.2.3
```

❌ Never modify it.

Instead release:

```text
1.2.4
```

---

## Why?

Ensures:

- Reproducible builds
- Reliable dependency management
- Consistent software behavior

---

# Pre-release Versions

Used before stable production releases.

Format:

```text
MAJOR.MINOR.PATCH-label
```

Examples:

```text
1.0.0-alpha
1.0.0-alpha.1
1.0.0-beta
1.0.0-rc.1
```

---

## Common Labels

### Alpha

```text
1.0.0-alpha
```

- Early development
- Highly unstable
- Features may be incomplete

---

### Beta

```text
1.0.0-beta
```

- Feature complete
- Under testing
- Minor bugs expected

---

### Release Candidate (RC)

```text
1.0.0-rc.1
```

- Potential final release
- Only critical fixes allowed

---

## Release Progression

```text
alpha
  ↓
beta
  ↓
rc
  ↓
final release
```

Example:

```text
1.0.0-alpha
1.0.0-beta
1.0.0-rc.1
1.0.0
```

---

# Build Metadata

Additional build information.

Format:

```text
1.0.0+build.45
```

Examples:

```text
1.0.0+jenkins.125
1.0.0+git.abc123
1.0.0+20260706
```

---

## Important

These versions are considered identical:

```text
1.0.0
```

```text
1.0.0+build.45
```

Build metadata does **not** affect version precedence.

---

# Public API Requirement

SemVer only works if software exposes a clearly defined API.

Examples:

- REST APIs
- SDKs
- Libraries
- CLI tools
- Frameworks

Without a public API, it is difficult to determine what constitutes a breaking change.

---

# Real-World Examples

## Bug Fix

Before:

```text
2.4.1
```

Fix login bug:

```text
2.4.2
```

Result:

```text
PATCH++
```

---

## New Feature

Before:

```text
2.4.2
```

Add dark mode:

```text
2.5.0
```

Result:

```text
MINOR++
```

---

## Breaking Change

Before:

```text
2.5.0
```

Remove legacy authentication API:

```text
3.0.0
```

Result:

```text
MAJOR++
```

---

# Dependency Management

SemVer helps estimate upgrade risk.

Example:

```json
{
  "express": "4.18.2"
}
```

---

## Upgrade Risk

| Upgrade | Risk Level |
|----------|------------|
| 4.18.2 → 4.18.3 | Low |
| 4.18.2 → 4.19.0 | Medium |
| 4.18.2 → 5.0.0 | High |

---

# Common Interview Questions

## What is Semantic Versioning?

A versioning scheme using:

```text
MAJOR.MINOR.PATCH
```

to communicate software compatibility and change impact.

---

## When Should MAJOR Be Incremented?

When introducing backward-incompatible changes.

---

## When Should MINOR Be Incremented?

When adding backward-compatible functionality.

---

## When Should PATCH Be Incremented?

When making backward-compatible bug fixes.

---

## What Does `1.0.0-beta.1` Mean?

A pre-release version intended for testing before the stable release.

---

# Examples Cheat Sheet

| Change | Version Update |
|----------|---------------|
| Bug Fix | 1.4.2 → 1.4.3 |
| New Feature | 1.4.2 → 1.5.0 |
| Breaking Change | 1.4.2 → 2.0.0 |
| Alpha Release | 1.0.0-alpha |
| Beta Release | 1.0.0-beta |
| Release Candidate | 1.0.0-rc.1 |
| Build Metadata | 1.0.0+build.45 |

---

# Visual Summary

```text
MAJOR.MINOR.PATCH
   │      │      │
   │      │      └── Bug Fixes
   │      │
   │      └──────── New Features
   │
   └─────────────── Breaking Changes
```

---

# Quick Revision (30 Seconds)

```text
MAJOR.MINOR.PATCH

MAJOR → Breaking Changes
MINOR → New Features
PATCH → Bug Fixes
```

### Rules

- Breaking change → MAJOR++
- New feature → MINOR++
- Bug fix → PATCH++
- Versions below 1.0.0 are unstable
- Use alpha, beta, rc for pre-releases
- Released versions should never be modified
- Build metadata (+build.xxx) does not affect version ordering

### Memory Trick

```text
Break.New.Fix

MAJOR.MINOR.PATCH
```