# SnapSBOM Action

Generate a **deterministic, byte-for-byte reproducible** [SPDX 2.3](https://spdx.github.io/spdx-spec/v2.3/) or [CycloneDX 1.6](https://cyclonedx.org/specification/overview/) Software Bill of Materials from `package-lock.json` — inside your build, on the runner you already have.

```yaml
- uses: kawacukennedy/snapsbom-action@v1
  with:
    format: cyclonedx
    output: sbom.cdx.json
```

### Why this one

Most SBOM generators stamp the current time, walk the filesystem in an arbitrary order and assign random IDs: run one twice and you get two different files, so every diff is noise. SnapSBOM pins timestamps, sorts every collection and assigns canonical identifiers, so **the same lockfile always produces the same bytes** — on Linux, macOS and Windows alike.

That is what makes the SBOM something you can commit, review with `git diff`, and assert on in CI:

```yaml
- uses: kawacukennedy/snapsbom-action@v1
  id: sbom
- run: test "${{ steps.sbom.outputs.sha256 }}" = "e8c6ab160b0a3fde859a6281da90b89673b1d9b796675dbbb1a17aa253996a95"
```

It reads the lockfile directly — no source scan, no container, no build, no cloud upload. The generation itself makes **zero network calls** (only fetching the pinned CLI from npm requires network).

## Inputs

| Name | Description | Default |
|---|---|---|
| `path` | Directory containing a `package-lock.json`, or a path to the lockfile itself. | `.` |
| `format` | `spdx` (SPDX 2.3) or `cyclonedx` (CycloneDX 1.6). | `spdx` |
| `output` | File to write the SBOM to. | `sbom.spdx.json` / `sbom.cdx.json` |
| `version` | SnapSBOM version to run. Pinned so a given workflow ref stays reproducible. | `0.2.1` |

Supported lockfiles: `package-lock.json` `lockfileVersion` 1, 2 and 3 (npm 5 through npm 11).

## Outputs

| Name | Description |
|---|---|
| `path` | Path of the generated SBOM file. |
| `sha256` | SHA-256 of the generated SBOM — identical across runs and across operating systems for identical input. |
| `components` | Number of components (CycloneDX) or packages (SPDX). |
| `format` | Format that was generated. |

## Usage

Generate an SPDX SBOM and commit it next to the lockfile:

```yaml
name: SBOM
on: [push]

permissions:
  contents: read

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: kawacukennedy/snapsbom-action@v1
        with:
          format: cyclonedx
          output: sbom.cdx.json

      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.cdx.json
```

Fail the build if dependencies changed but the committed SBOM did not:

```yaml
      - uses: kawacukennedy/snapsbom-action@v1
        with:
          format: cyclonedx
          output: sbom.cdx.json

      - name: SBOM must be up to date
        run: git diff --exit-code -- sbom.cdx.json
```

Gate on the digest instead of the file, so unrelated formatting churn can't break the build:

```yaml
      - uses: kawacukennedy/snapsbom-action@v1
        id: sbom
        with:
          format: cyclonedx
          output: sbom.cdx.json

      - name: SBOM must match the last reviewed digest
        env:
          EXPECTED: ${{ secrets.SBOM_SHA256 }}
        run: |
          set -euo pipefail
          test -n "${EXPECTED}" || { echo "::error::Set the SBOM_SHA256 secret."; exit 1; }
          test "${{ steps.sbom.outputs.sha256 }}" = "${EXPECTED}"
```

## Supported runners

Composite action, `shell: bash` — verified on `ubuntu-latest`, `macos-latest` and `windows-latest`. Requires Node.js on the runner (preinstalled on all GitHub-hosted runners).

The repository's own CI runs this action **twice per operating system per format**, asserts the two runs are byte-identical, and then compares the digests **across all three operating systems**.

## How the version pinning works

`version` is passed to `npx snapsbom@<version>`, so the action always runs an exact release. Bump it deliberately:

```yaml
- uses: kawacukennedy/snapsbom-action@v1
  with:
    version: 0.2.1   # upgrade intentionally, not implicitly
```

## License

[MIT](./LICENSE)
