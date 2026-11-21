This repository holds reusable GitHub Actions workflows (`.github/workflows/`), called by
other repositories through `workflow_call` and pinned there by commit SHA. Its conventions
are in `AGENTS.md`; the README documents each workflow's inputs and security model.

Check in particular:

- Security of the workflows themselves: least-privilege `permissions:` per job, no secret or
  write token reaching a step that runs untrusted code or the model, no event or PR text
  interpolated with `${{ }}` into a `run:` script (it goes through `env:`), fork PRs never
  meeting a secret.
- Pinning: actions by full commit SHA with the tag in a comment, downloaded tools by version
  and sha256, no `curl | bash`.
- Compatibility: a changed or removed input, secret or output breaks callers. Say so, and
  whether the README's inputs table and usage example still match.
- CI runs actionlint (with shellcheck) and gitleaks; skip what they catch.
