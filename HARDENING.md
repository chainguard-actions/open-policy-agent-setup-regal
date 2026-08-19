<!-- markdownlint-disable -->

# Hardening Report: open-policy-agent--setup-regal/v0.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **open-policy-agent--setup-regal/v0.1.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'Get expected version' run: block in .github/workflows/test.yml directly interpolates ${{ matrix.version }} and ${{ secrets.GITHUB_TOKEN }} into shell commands (rule a). matrix.* and secrets.* values flow through YAML template substitution before the shell processes them, enabling script injection. Offending lines include: `if [ "${{ matrix.version }}" = "<0.9" ]`, `elif [ "${{ matrix.version }}" = "0.10.0" ]`, `echo "Expected version for ${{ matrix.version }}: $EXPECTED_VERSION"`, and `authorization: Bearer ${{ secrets.GITHUB_TOKEN }}`. These should be moved to env: variables and referenced as quoted shell variables instead.

Locations:

- `.github/workflows/test.yml:32`
- `.github/workflows/test.yml:34`
- `.github/workflows/test.yml:38`
- `.github/workflows/test.yml:40`

### github-env-injection (severity: high)

The 'Get expected version' step writes `echo "EXPECTED_VERSION=$EXPECTED_VERSION" >> $GITHUB_ENV` (line 41) where $EXPECTED_VERSION is derived from the workflow-controllable value ${{ matrix.version }} without the required sanitization step (`printf '%s' "$EXPECTED_VERSION" | tr -d '\n\r'`) before the write. A newline injected via matrix.version could allow an attacker to inject arbitrary environment variables.

Locations:

- `.github/workflows/test.yml:41`

### unpinned-uses (severity: high)

Two uses: references in test.yml pin to the mutable tag @v4 instead of a full 40-character SHA commit hash. If the tag is moved or the upstream repository is compromised, the action will silently execute different code. Affected references: `actions/checkout@v4` (line 23, job: test) and `actions/checkout@v4` (line 59, job: verify-dist).

Locations:

- `.github/workflows/test.yml:23`
- `.github/workflows/test.yml:59`

### missing-permissions (severity: medium)

The workflow file .github/workflows/test.yml has no top-level `permissions:` key, and neither the `test` job nor the `verify-dist` job defines a job-level `permissions:` block. Without explicit permissions, GitHub Actions grants the default token permissions (which may include write access to contents, packages, etc.), violating the principle of least privilege.

Locations:

- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings in hardened/action/.github/workflows/test.yml:
1. unpinned-uses: Pinned both `actions/checkout@v4` references to full SHA `34e114876b0b11c390a56381ad16ebd13914f8d5` with `# v4` comment.
2. missing-permissions: Added `permissions: {}` at the workflow top level, and `permissions: contents: read` at the job level for both `test` and `verify-dist` jobs (contents: read is needed for checkout).
3. script-injection: Moved `${{ matrix.version }}` to env var `MATRIX_VERSION` and `${{ secrets.GITHUB_TOKEN }}` to env var `GITHUB_TOKEN` in the 'Get expected version' step's `env:` block; all shell references now use plain `$MATRIX_VERSION` and `$GITHUB_TOKEN`.
4. github-env-injection: Added `safe=$(printf '%s' "$EXPECTED_VERSION" | tr -d '\n\r')` sanitization before writing to `$GITHUB_ENV`, and quoted `"$GITHUB_ENV"` in the redirect.

