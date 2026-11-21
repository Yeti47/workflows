# Agent Instructions

Baseline rules for AI coding agents working in this repository.

Sections 1–7 are the mandatory baseline from the yetibyte AGENTS.md template. This repository holds only workflows and their docs, so the template's coding-convention sections are left out.

---

## 1. Secrets in Agent Context

- **Never read `.env` files or `.env.*` variants** — not to debug configuration, verify a value, or confirm a variable name. Ask the user for the specific non-secret value instead. This covers `.env.local`, `.env.development`, `.env.production`, and every other real environment file, with no exceptions.
- The only exception is committed documentation examples — `.env.example`, `.env.<env>.example` (e.g. `.env.production.example`), and `example.env`. These hold placeholder values, not real credentials, and are safe to read. If one appears to contain a live credential instead of a placeholder, stop and tell the user.
- Never echo, quote, or summarize secret material (API keys, tokens, passwords, connection strings, private keys) into chat, commit messages, PR bodies, issues, or logs. Refer to a secret by name and location only.
- If a secret is exposed in chat — pasted by the user, printed by a tool, or leaked by your own action — stop the current task, tell the user exactly what was exposed and where, and recommend rotating it immediately. Do not repeat the value in your warning.
- Treat `secrets/`, key material, token files, and local auth files as sensitive. Reading them requires explicit user approval for that specific file.

---

## 2. Branching, Commits, and Documentation

### Branches

- Implement features on a feature branch, e.g. `feature/create-document-endpoint`.
- Implement bug fixes on a bug branch, e.g. `bug/fix-unresponsive-login-form`.
- If the project uses an issue tracker, prefix the branch description with the issue ID: `<type>/<issue-id>_<description>`, e.g. `feature/36836_create-order-endpoints` or `bug/41207_fix-unresponsive-login-form`. Use the ID exactly as the tracker exposes it — never invent one, and ask if the issue is not identifiable.
- The user may explicitly override these rules, e.g. to prototype on `main` or iterate on an experimental branch. An override covers only the work the user named — never treat one approval as a standing policy.

### Commits and pushes

- Never commit or push without the user's approval for that specific action.
- Blanket approvals ("commit as you go for this task") are valid only when the user states them. Never infer standing permission from an earlier one-off approval, from the task description, or from the absence of an objection.
- Before committing, run the relevant formatters, linters, and secret scan (see [§4](#4-secrets-handling)).

### Documentation

- Before committing, check whether documentation still matches the implementation. Fix discrepancies that fall within scope; tell the user before making substantial documentation changes.
- Update related documentation whenever behavior, structure, or conventions change.
- In Markdown docs, use embedded Mermaid diagrams to visualize design decisions, data flow, and structure when a diagram conveys them more clearly than prose:

  ````markdown
  ```mermaid
  flowchart LR
    Client -->|POST /orders| PlaceOrder
    PlaceOrder --> OrderService
    OrderService --> Db[(Database)]
  ```
  ````

---

## 3. Dependency Pinning

All external dependencies must be pinned to an **immutable, content-addressed identifier**. Tags, version ranges, and `latest` are forbidden.

| Ecosystem | Required pin |
| --- | --- |
| Docker images / Compose / Kubernetes | `@sha256:<digest>` |
| GitHub Actions | `@<commit-sha> # vX` |
| Git-sourced packages (npm, NuGet, pip, Go) | `#<commit-sha>` |
| npm / pnpm / yarn | Exact version + committed lockfile |
| NuGet | Exact version + `packages.lock.json` |
| pip | `==` + `--hash=sha256:<hash>` |

### Docker images
```dockerfile
# WRONG
FROM node:20
FROM alpine:latest
FROM mcr.microsoft.com/dotnet/aspnet:9.0

# CORRECT — pin to digest
FROM node@sha256:<digest>
FROM alpine@sha256:<digest>
FROM mcr.microsoft.com/dotnet/aspnet@sha256:<digest>
```
To get a digest: `docker pull <image>:<tag>` then `docker inspect --format='{{index .RepoDigests 0}}' <image>:<tag>`

### Docker Compose / Kubernetes
Same rule. Never use `image: mongo:7`, always `image: mongo@sha256:<digest>`.

### GitHub Actions
```yaml
# WRONG
uses: actions/checkout@v4
uses: actions/setup-node@v4

# CORRECT — pin to commit SHA, tag in comment for readability
uses: actions/checkout@<sha> # v4
uses: actions/setup-node@<sha> # v4
```
To get the SHA: `gh api /repos/<owner>/<repo>/git/ref/tags/<tag> --jq .object.sha`  
If the tag is annotated, dereference: `gh api /repos/<owner>/<repo>/git/tags/<sha> --jq .object.sha`

### npm / pnpm / yarn
```json
// WRONG — ranges are mutable
"express": "^4.18.0"

// CORRECT — exact versions + lockfile committed
"express": "4.18.2"
```
Always commit `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock`. Never run installs without `--frozen-lockfile` / `npm ci` in CI.

### NuGet
```xml
<!-- WRONG -->
<PackageReference Include="Newtonsoft.Json" Version="13.*" />

<!-- CORRECT — exact version + lock file -->
<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
```
Enable lock files in `Directory.Build.props`:
```xml
<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
```
Use `--locked-mode` in CI and Dockerfiles. Commit all `packages.lock.json` files.

### Python (pip)
```txt
# WRONG
requests>=2.28

# CORRECT — exact version + hash pinning
requests==2.31.0 --hash=sha256:<hash>
```
Generate with `pip-compile --generate-hashes`.

### Git and other source dependencies
Pin Git-sourced dependencies to a full commit SHA, never a branch or tag. Applies to `git+https://…#<sha>` in npm/pip, `RepositoryUrl`-style NuGet references, and `go get <module>@<sha>`.

---

## 4. Secrets Handling

- **Never** hardcode secrets, tokens, passwords, or API keys in source files, Dockerfiles, or CI config.
- **Never** pass secrets as environment variables when a secrets manager or mounted secret file is available (e.g. Docker secrets, Kubernetes secrets mounted as files).
- **Never** log secrets. Treat any header named `Authorization`, `X-*-Secret`, `X-*-Key`, `X-*-Token`, or `Password` as sensitive — mask or omit from logs.
- Secrets that change independently of code (e.g. SMTP credentials, API keys) must be stored outside the repository and injected at runtime. Rotating them must **not** require a code change or pipeline trigger.
- CI/CD secrets (e.g. GitHub Actions secrets) should be **scoped as narrowly as possible** and reviewed when pipeline steps change. Delete any secret that is no longer referenced.

### Secret scanning with gitleaks

Every project must have gitleaks enabled. Never skip or `--no-verify` past it, and never commit with `SKIP=gitleaks`.

- Local scan of staged changes: `gitleaks git --pre-commit --redact --staged --verbose`
- Local scan of history: `gitleaks git --redact --verbose`
- Local scan of a directory (no Git): `gitleaks dir --redact --verbose`

Always pass `--redact` so detected values are not printed into terminal output, logs, or agent transcripts.

`.pre-commit-config.yaml`, pinned to a commit SHA per [§3](#3-dependency-pinning):

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: 83d9cd684c87d95d656c1458ef04895a7f1cbd8e # v8.30.1
    hooks:
      - id: gitleaks
```

CI workflow — least-privilege, with the action pinned to a commit SHA:

```yaml
name: gitleaks

on:
  pull_request:
  push:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@e0c47f4f8be36e29cdc102c57e68cb5cbf0e8d1e # v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Required only when scanning repositories owned by an organization.
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

A detected secret is not resolved by deleting it in a later commit — it stays in history. Rotate the credential, then decide with the user whether to rewrite history.

---

## 5. CI/CD Pipeline Hardening

- Set **least-privilege permissions** on every workflow. Declare `permissions:` explicitly; default to `contents: read` and add only what each job actually needs.
- Do not use `runs-on: self-hosted` unless the runner is hardened and isolated.
- Never use `pull_request_target` without careful review — it runs with write permissions against external PRs.
- Avoid `workflow_dispatch` inputs that are interpolated directly into shell commands (injection risk).
- Do not `curl | bash` or pipe remote content into a shell in any pipeline step.

---

## 6. Container & Infrastructure Security

- Containers must run as a **non-root user**. Create a dedicated user in the Dockerfile; use `USER` before `CMD`/`ENTRYPOINT`.
- Do not bind container ports to `0.0.0.0` unless the port must be publicly reachable. Prefer `127.0.0.1:<host>:<container>`.
- Do not mount the Docker socket (`/var/run/docker.sock`) into containers unless absolutely required.
- Cron jobs and scheduled tasks must run as the **least-privileged user** that can perform the operation — never as `root`.
- Use multi-stage Docker builds to keep build tooling out of final images.

---

## 7. Updating Pinned Dependencies

When a dependency must be updated:
1. Obtain the new immutable identifier (digest/SHA/exact version).
2. Update the pin.
3. Regenerate lockfiles if applicable.
4. Commit pin + lockfile together in the same change.

Do **not** bump a pin without verifying the new version/digest against the official release (check release notes, compare digests from the official registry or package index).
