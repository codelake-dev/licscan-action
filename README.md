# licscan-action

> Official GitHub Action for [**licscan**](https://github.com/codelake-dev/licscan) — open-source license & compliance scanner for modern codebases.

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Marketplace](https://img.shields.io/badge/marketplace-licscan-blue?logo=github)](https://github.com/marketplace/actions/licscan)

Scan a PR for dependency licenses. Enforce a policy. Post a Markdown report as a PR comment. Optionally emit EU CRA evidence (PDF + JSON) as an artefact.

```yaml
- uses: codelake-dev/licscan-action@v1
```

---

## Quick start

### Minimum (scan + PR comment)

```yaml
name: License check
on: [pull_request]
jobs:
  licenses:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: codelake-dev/licscan-action@v1
```

By default the action:

- installs the latest stable licscan
- scans the repository root
- prints a markdown report into the workflow log
- posts the same report as a PR comment (requires `pull-requests: write`)
- uploads `licscan-report.md` as a workflow artefact (90-day retention)
- **fails the job** if your `.licscan.yml` policy denies any dependency

### Pin a specific licscan version

```yaml
- uses: codelake-dev/licscan-action@v1
  with:
    version: v0.11.0
```

### Scan a sub-directory

```yaml
- uses: codelake-dev/licscan-action@v1
  with:
    path: ./services/api
```

### Don't fail the build — report only

```yaml
- uses: codelake-dev/licscan-action@v1
  with:
    fail-on-violation: false
```

### Generate EU CRA evidence

```yaml
- uses: codelake-dev/licscan-action@v1
  with:
    cra: true
    cra-output: ./compliance/cra-evidence
```

Both `cra-evidence.pdf` and `cra-sbom.cdx.json` are written to `cra-output/` and uploaded with the report artefact. Set manufacturer + product details in `.licscan.yml` — see the [licscan README](https://github.com/codelake-dev/licscan#eu-cra-compliance-mode).

### Skip PR commenting (run on push)

```yaml
- uses: codelake-dev/licscan-action@v1
  with:
    pr-comment: false
```

---

## Inputs

| Name | Default | Description |
|---|---|---|
| `path` | `.` | Project directory to scan |
| `version` | `latest` | licscan version to install (e.g. `v0.11.0` or `latest`) |
| `format` | `table` | Format printed to the workflow log (`table`, `json`, `html`, `cyclonedx`, `spdx`, `markdown`). The markdown PR comment + artefact are produced regardless. |
| `fail-on-violation` | `true` | Exit non-zero on policy deny (CI mode) |
| `pr-comment` | `true` | Post the markdown report as a PR comment (only on `pull_request` events) |
| `cra` | `false` | Emit EU CRA evidence (PDF + CycloneDX JSON) into `cra-output` |
| `cra-output` | `licscan-cra-evidence` | Output directory for CRA artefacts |
| `upload-artifact` | `true` | Upload the markdown report (+ CRA artefacts) as a workflow artifact |
| `install-base-url` | `https://install.codelake.dev` | Override install endpoint — useful for self-hosted mirrors |

## Outputs

| Name | Description |
|---|---|
| `report-path` | Filesystem path to the generated markdown report |
| `total` | Total dependencies scanned |
| `denied` | Number of dependencies denied by policy |
| `warned` | Number of dependencies in the warn list |

---

## Permissions

The action needs different scopes depending on which features you use:

| Feature | Permission |
|---|---|
| Scan + log + artefact upload | `contents: read` |
| Post PR comment | `pull-requests: write` (`pr-comment: true` on a `pull_request` event) |

Recommended top-of-job block:

```yaml
permissions:
  contents: read
  pull-requests: write
```

---

## Recipes

### Block GPL-class licenses on PRs, ignore them on `main`

Create `.licscan.yml`:

```yaml
deny:
  - AGPL-3.0
  - GPL-3.0
  - GPL-2.0
  - SSPL-1.0
warn:
  - LGPL-3.0
  - LGPL-2.1
```

The default policy already covers strong-copyleft + viral — the file above just makes the rules explicit so they show up in code review.

### Generate a versioned CRA evidence archive on every release

```yaml
on:
  release:
    types: [published]

jobs:
  cra-evidence:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: codelake-dev/licscan-action@v1
        with:
          cra: true
          cra-output: ./cra-evidence
          pr-comment: false
      - uses: actions/upload-artifact@v4
        with:
          name: cra-evidence-${{ github.ref_name }}
          path: ./cra-evidence/
          retention-days: 3650   # CRA support-lifecycle: 10y
```

### Custom logic using the action outputs

```yaml
- id: licscan
  uses: codelake-dev/licscan-action@v1
  with:
    fail-on-violation: false

- name: Open issue if more than 5 warnings
  if: steps.licscan.outputs.warned > 5
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.create({
        owner: context.repo.owner,
        repo: context.repo.repo,
        title: `${process.env.WARNED} licence warnings on ${context.sha}`,
        body: require('fs').readFileSync(process.env.REPORT_PATH, 'utf8'),
      })
  env:
    WARNED:      ${{ steps.licscan.outputs.warned }}
    REPORT_PATH: ${{ steps.licscan.outputs.report-path }}
```

---

## How it works

The action is composite (pure YAML + shell, no JS runtime):

1. **Install** — downloads the requested licscan binary from `install.codelake.dev/licscan/` (Cloudflare R2 mirror) into `$RUNNER_TEMP/licscan-bin`. Cached implicitly per run.
2. **Scan** — three passes:
   - `--format markdown` → report file (used for PR comment + artefact)
   - `--format <user-chosen>` → workflow-log output
   - `--format json` + `jq` → numeric outputs (`total`, `denied`, `warned`)
   - Optional `--cra --output ...` when `cra: true`
3. **CI gate** — final `--ci` pass to derive the deterministic exit code from your `.licscan.yml`.
4. **PR comment** — `gh pr comment` posts the markdown report (only on `pull_request` events, requires `pull-requests: write`).
5. **Artefact upload** — `actions/upload-artifact@v4` with `if-no-files-found: ignore`.

The action makes no outbound calls beyond the binary download and the GitHub API.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE).

Copyright 2026 codelake Technologies LLC, an Akyros Labs brand.
