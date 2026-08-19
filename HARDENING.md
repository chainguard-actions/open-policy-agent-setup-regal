<!-- markdownlint-disable -->

# Hardening Report: open-policy-agent--setup-regal/v1.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **open-policy-agent--setup-regal/v1.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### missing-permissions (severity: medium)

The workflow file .github/workflows/test.yml has no top-level `permissions:` key and neither of its jobs (test, verify-dist) defines a `permissions:` block. Without explicit permissions, the GITHUB_TOKEN is granted default (potentially write) permissions, violating least-privilege.

Locations:

- `.github/workflows/test.yml:1`

### unpinned-uses (severity: high)

Two `uses:` references in .github/workflows/test.yml pin to the mutable tag `@v4` instead of a full 40-character commit SHA. This exposes the workflow to supply-chain attacks if the tag is moved. Failing references: `actions/checkout@v4` (appears twice, in the `test` job and the `verify-dist` job).

Locations:

- `.github/workflows/test.yml:23`
- `.github/workflows/test.yml:50`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expressions are interpolated directly inside `run:` shell command strings in the 'Get expected version' step. The expression `${{ matrix.version }}` is injected into shell `if` comparisons and an `echo` statement without any quoting or env-var indirection, allowing an attacker who controls the matrix value to inject arbitrary shell commands. Additionally, `${{ secrets.GITHUB_TOKEN }}` is interpolated directly into a `curl` command header inside the same `run:` block. Any `${{ ... }}` inside a `run:` script is a script-injection risk regardless of the context it reads from. Offending lines:
- `if [ "${{ matrix.version }}" = "<0.9" ]; then`
- `elif [ "${{ matrix.version }}" = "0.10.0" ]; then`
- `export EXPECTED_VERSION=$(curl --header 'authorization: Bearer ${{ secrets.GITHUB_TOKEN }}' ...)`
- `echo "Expected version for ${{ matrix.version }}: $EXPECTED_VERSION"`

Locations:

- `.github/workflows/test.yml:31`
- `.github/workflows/test.yml:33`
- `.github/workflows/test.yml:36`
- `.github/workflows/test.yml:38`

### github-env-injection (severity: high)

The 'Get expected version' step writes `EXPECTED_VERSION` to `$GITHUB_ENV` without sanitization. The value of `EXPECTED_VERSION` is derived from `${{ matrix.version }}` (a workflow-controlled value interpolated directly into the shell) and from the output of a `curl` call to the GitHub API. Neither value is passed through `printf '%s' ... | tr -d '\n\r'` before being written to `$GITHUB_ENV`. A newline character in the value could allow an attacker to inject additional environment variable definitions. Offending line: `echo "EXPECTED_VERSION=$EXPECTED_VERSION" >> $GITHUB_ENV`

Locations:

- `.github/workflows/test.yml:39`

## Iteration Notes

### Iteration 1

**Fixes applied:** missing-permissions, unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all four findings in .github/workflows/test.yml:
1. missing-permissions: Added top-level `permissions: contents: read` block.
2. unpinned-uses: Pinned both `actions/checkout@v4` references to full SHA `34e114876b0b11c390a56381ad16ebd13914f8d5` with `# v4` comment.
3. script-injection: Moved `${{ matrix.version }}` into `MATRIX_VERSION` env var and `${{ secrets.GITHUB_TOKEN }}` into `GITHUB_TOKEN` env var in the step's `env:` block; all shell code now references plain env vars.
4. github-env-injection: Added `safe=$(printf '%s' "$EXPECTED_VERSION" | tr -d '\n\r')` to strip newlines before writing to `$GITHUB_ENV`, and quoted the `$GITHUB_ENV` path.

