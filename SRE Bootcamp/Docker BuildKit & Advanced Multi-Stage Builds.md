
> Source: Docker Blog – Advanced Dockerfiles: Faster Builds and Smaller Images Using BuildKit and Multistage Builds
>
> Goal: Build faster, smaller, more secure, and more maintainable Docker images.

---

# Why This Matters

Traditional Dockerfiles often:

- Produce large images
- Repeat common setup steps
- Have slower build times
- Contain unnecessary build tools in production images

Using **BuildKit + Multi-Stage Builds** helps:

- Faster builds
- Better caching
- Smaller images
- Lower attack surface
- Cleaner Dockerfiles

---

# BuildKit

## What is BuildKit?

BuildKit is Docker's modern build engine.

It improves Docker builds through:

- Parallel execution
- Smarter caching
- Dependency graph optimization
- Skipping unused stages

---

## Key Benefits

### 1. Parallel Stage Execution

If stages are independent:

```dockerfile
FROM base AS frontend
RUN npm run build

FROM base AS backend
RUN go build
```

BuildKit can build them simultaneously.

Traditional Builder:

```text
frontend -> backend
```

BuildKit:

```text
frontend
    \
     --> parallel
    /
backend
```

### Benefit

- Faster CI/CD pipelines
- Better CPU utilization

---

### 2. Skip Unused Stages

Example:

```dockerfile
FROM base AS build

FROM build AS test

FROM build AS production
```

Building:

```bash
docker build --target production .
```

BuildKit skips unnecessary stages.

### Benefit

- Reduced build time
- Less resource usage

---

### 3. Better Caching

BuildKit understands stage dependencies and reuses cache more efficiently.

### Benefit

- Faster rebuilds
- Reduced CI costs

---

# Multi-Stage Builds

## What are Multi-Stage Builds?

A Dockerfile can contain multiple `FROM` instructions.

Example:

```dockerfile
FROM golang AS builder

RUN go build

FROM alpine

COPY --from=builder /app .
```

---

## Why Use Multi-Stage Builds?

Without Multi-Stage Builds:

Final image contains:

- Source code
- Compiler
- Build tools
- Runtime files

With Multi-Stage Builds:

Final image contains:

- Only required runtime artifacts

---

## Benefits

### Smaller Images

Before:

```text
Application
+ Source Code
+ GCC
+ Make
+ Package Managers
```

After:

```text
Application Binary Only
```

---

### Faster Deployments

Smaller images mean:

- Faster downloads
- Faster uploads
- Faster deployments

---

### Better Security

Removes unnecessary tools:

- gcc
- make
- curl
- apt
- package managers

Result:

- Smaller attack surface
- Fewer vulnerabilities

---

# Using Stages as Base Images

Most engineers know:

```dockerfile
COPY --from=builder /app .
```

Less known:

Stages can be reused in `FROM`.

---

## Example

```dockerfile
FROM ubuntu AS base

RUN apt-get update

FROM base AS build
RUN make

FROM base AS test
RUN pytest
```

---

## Benefits

Avoid duplication:

```dockerfile
RUN apt-get update
RUN apt-get install ...
```

Write once, reuse everywhere.

---

# Reusable Stages

Treat stages like reusable building blocks.

---

## Example

```dockerfile
FROM golang AS base

WORKDIR /src

COPY . .

FROM base AS build-app1
RUN go build app1

FROM base AS build-app2
RUN go build app2
```

---

## Benefits

- Cleaner Dockerfiles
- Shared cache
- Easier maintenance
- Less duplication

---

# Stage Inheritance

Stages can inherit from previous stages.

```dockerfile
FROM ubuntu AS base

FROM base AS build

FROM build AS test
```

Dependency chain:

```text
base
 └── build
      └── test
```

---

# Build Targets

A single Dockerfile can produce multiple images.

---

## Example

```dockerfile
FROM base AS development

FROM base AS testing

FROM base AS production
```

Build only the required target:

```bash
docker build --target development .
```

```bash
docker build --target testing .
```

```bash
docker build --target production .
```

---

## Benefits

One Dockerfile for:

- Development
- Testing
- Production

No need for:

```text
Dockerfile.dev
Dockerfile.test
Dockerfile.prod
```

---

# External Images as Build Stages

`COPY --from` can use external images.

---

## Example

```dockerfile
COPY --from=nginx:latest \
     /etc/nginx/nginx.conf \
     .
```

Docker:

1. Pulls nginx image
2. Extracts file
3. Copies file

---

## Benefits

Reuse files from existing images:

- Configurations
- Certificates
- Binaries
- Runtime dependencies

---

# Using Images as Dependency Providers

Instead of:

```dockerfile
RUN curl -L https://example.com/tool
```

Use:

```dockerfile
COPY --from=tool-image \
     /usr/local/bin/tool \
     /usr/local/bin/tool
```

---

## Benefits

- Faster builds
- Reproducible builds
- No network dependency
- Better security

---

# Think of Stages as Functions

Most beginners think:

```text
Dockerfile = List of Commands
```

Advanced mindset:

```text
Dockerfile = Collection of Reusable Stages
```

---

## Example

```dockerfile
FROM base AS deps

FROM deps AS frontend

FROM deps AS backend

FROM backend AS production
```

Visualized:

```text
deps
├── frontend
└── backend
     └── production
```

Each stage acts like a reusable function.

---

# Best Practices

## Keep a Common Base Stage

```dockerfile
FROM ubuntu AS base
```

Install shared dependencies once.

---

## Use Multi-Stage Builds

Separate:

- Build environment
- Runtime environment

---

## Use Named Stages

Prefer:

```dockerfile
FROM golang AS builder
```

Instead of:

```dockerfile
COPY --from=0
```

Named stages improve readability.

---

## Build Only Required Targets

```bash
docker build --target production .
```

Avoid building unnecessary stages.

---

## Keep Runtime Images Minimal

Production image should contain:

- Application
- Required libraries

Avoid including:

- Source code
- Build tools
- Package managers

---

# Common Interview Questions

## What is BuildKit?

BuildKit is Docker's modern build engine that provides:

- Parallel builds
- Better caching
- Dependency optimization
- Faster build performance

---

## What are Multi-Stage Builds?

A technique using multiple `FROM` statements to separate build and runtime environments.

---

## Why Use Multi-Stage Builds?

- Smaller images
- Faster deployments
- Better security
- Cleaner Dockerfiles

---

## Can a Stage Be Used as a Base Image?

Yes.

```dockerfile
FROM base AS build

FROM build AS test
```

---

## Can COPY --from Use External Images?

Yes.

```dockerfile
COPY --from=nginx:latest /etc/nginx/nginx.conf .
```

---

## What is the Benefit of BuildKit?

- Parallel execution
- Better caching
- Faster builds
- Skipping unused stages

---

# Quick Revision (30 Seconds)

- BuildKit = Modern Docker builder.
- BuildKit supports parallel stage execution.
- BuildKit skips unused stages.
- Multi-stage builds reduce image size.
- Smaller images = faster deployments + better security.
- Stages can inherit from other stages using `FROM`.
- `COPY --from` works with both stages and external images.
- Use `--target` to build specific stages.
- Treat stages as reusable building blocks, not just sequential commands.
- Final production image should contain only runtime artifacts.

---

# Visual Summary

```text
                    BuildKit
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   Parallel        Better Cache    Skip Unused
    Builds                          Stages

                        │

                 Multi-Stage Builds

                        │

        ┌───────────────┼───────────────┐
        │               │               │
   Smaller Images  Better Security  Faster Deployments

                        │

                 Reusable Stages

                        │

      base
      ├── frontend
      ├── backend
      └── production
```