## What is a Wheel?

A **wheel** (`.whl`) is a built distribution format for Python packages.

Think of it as a **ready-to-install package** that `pip` can directly install without building the package from source code.

---

## Simple Analogy

### Source Package (`.tar.gz`)

Like receiving:
- Wood
- Screws
- Instructions

You must assemble the chair yourself.

### Wheel Package (`.whl`)

Like receiving:
- Fully assembled chair

Just unpack and use it.

---

## Why Wheels Exist

Many Python packages contain code written in:
- C
- C++
- Rust

Examples:
- NumPy
- Pandas
- SciPy
- cryptography

Without wheels, your machine must:

1. Download source code
2. Compile the code
3. Build the package
4. Install it

This requires:
- GCC/Clang/MSVC compiler
- Development libraries
- Additional installation time

---

## Installation Flow

### With Wheel Available

```bash
pip install numpy
```

Output:

```text
Downloading numpy-2.x.x.whl
Installing...
Done
```

### Benefits

- ✅ Fast installation
- ✅ No compiler needed
- ✅ Fewer installation errors

---

### Without Wheel Available

```bash
pip install numpy
```

Output:

```text
Downloading numpy-2.x.x.tar.gz
Building wheel for numpy...
```

Now `pip` must:
- Compile source code
- Build binaries
- Install package

### Drawbacks

- ❌ Slower installation
- ❌ Requires build tools
- ❌ More chances of failure

---

## What is a Pre-Compiled Wheel?

A **pre-compiled wheel** is a wheel that already contains machine code compiled by the package maintainer.

Your machine does **not** need to compile anything.

### Benefits

- Faster installs
- Consistent builds
- No compiler required
- Better developer experience

---

## Wheel Filename Breakdown

Example:

```text
numpy-2.3.1-cp313-cp313-manylinux_x86_64.whl
```

| Part | Meaning |
|--------|---------|
| numpy | Package name |
| 2.3.1 | Package version |
| cp313 | Python 3.13 |
| cp313 | CPython ABI |
| manylinux | Linux-compatible build |
| x86_64 | 64-bit Intel/AMD CPU |

---

## Why Wheels are Platform-Specific

Compiled code depends on:

### Operating System
- Linux
- macOS
- Windows

### CPU Architecture
- x86_64
- ARM64

### Python Version
- 3.10
- 3.11
- 3.12
- 3.13

A wheel built for:

```text
Linux + Python 3.13 + x86_64
```

cannot be used on:

```text
Windows + Python 3.13
```

---

## How pip Uses Wheels

When you run:

```bash
pip install package_name
```

`pip` follows this order:

1. Search for a compatible wheel
2. Download the wheel
3. Install the wheel

If no compatible wheel exists:

1. Download source package
2. Build wheel locally
3. Install generated wheel

---

## Common Wheel Extension

```text
.whl
```

Examples:

```text
numpy-2.3.1-cp313-manylinux_x86_64.whl

pandas-2.3.1-cp313-win_amd64.whl

cryptography-46.0.0-cp313-macosx_arm64.whl
```

---

## Interview Questions

### What is a Python Wheel?

A wheel (`.whl`) is a built distribution format for Python packages that allows `pip` to install packages without compiling source code.

### What is a Pre-Compiled Wheel?

A pre-compiled wheel contains already-built machine code created by the package maintainer, allowing fast installation without requiring compilers or build tools on the target system.

---

## Key Takeaways

> - `.whl` = Built Python package
> - Faster than installing from source
> - No compilation needed when a wheel is available
> - Platform-specific (OS, CPU, Python version)
> - `pip` prefers wheels before source packages
> - Pre-compiled wheels reduce installation time and build failures

---

## Quick Revision (30 Seconds)

- **Wheel** = Ready-to-install Python package (`.whl`)
- **Source package** = Must be compiled locally
- **Pre-compiled wheel** = Already compiled by maintainer
- **pip** always prefers wheels when available
- **Benefits:** Faster installs, fewer errors, no compiler required
- Wheels are specific to **OS + CPU + Python version**