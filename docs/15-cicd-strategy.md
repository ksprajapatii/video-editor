# 15 — CI/CD Strategy

> **Document:** CI/CD Strategy v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The CI/CD pipeline is built on **GitHub Actions** with the following goals:
- Every PR is automatically tested
- Every merge to `main` builds release artifacts
- Releases are deployed automatically on tag push

---

## 2. Pipeline Overview

```
Developer pushes code
    |
    v
Pull Request Opened
    |
    +---> [PR Pipeline]
    |       - Lint (ESLint, Ruff, mypy)
    |       - Unit tests (Vitest + pytest)
    |       - Integration tests
    |       - Build check (no artifacts stored)
    |       Result: All must pass to merge
    |
    v
Merge to main
    |
    +---> [Main Pipeline]
    |       - All PR checks
    |       - E2E tests (Playwright)
    |       - Build packages (3 platforms)
    |       - Smoke test packages
    |       - Upload to dev CDN
    |
    v
Tag push (v1.x.x)
    |
    +---> [Release Pipeline]
            - Full test suite
            - Build signed packages (3 platforms)
            - Code signing
            - macOS notarization
            - GitHub Release creation
            - Upload to production CDN
            - Update server push
```

---

## 3. GitHub Actions Workflows

### 3.1 PR Checks (`.github/workflows/pr-checks.yml`)

```yaml
name: PR Checks

on:
  pull_request:
    branches: [main]

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  lint-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  lint-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install ruff black mypy
      - run: ruff check backend/
      - run: black --check backend/
      - run: mypy backend/ --strict

  test-frontend:
    runs-on: ubuntu-latest
    needs: [lint-frontend]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: frontend-coverage
          path: coverage/

  test-backend:
    runs-on: ubuntu-latest
    needs: [lint-backend]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/unit tests/integration -v --cov=backend --cov-report=xml
      - uses: codecov/codecov-action@v4

  build-check:
    runs-on: ubuntu-latest
    needs: [test-frontend, test-backend]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
        env:
          SKIP_PACKAGE: 'true'   # Just compile, don't bundle Electron
```

### 3.2 Main Branch Pipeline (`.github/workflows/main.yml`)

```yaml
name: Main Branch Pipeline

on:
  push:
    branches: [main]

jobs:
  all-tests:
    uses: ./.github/workflows/pr-checks.yml

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [all-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-screenshots
          path: playwright-report/

  build-dev:
    needs: [e2e-tests]
    strategy:
      matrix:
        os: [windows-2022, macos-14, ubuntu-22.04]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: npm ci
      - run: npm run build:electron
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CSC_LINK: ''      # No signing for dev builds
      - uses: actions/upload-artifact@v4
        with:
          name: dev-build-${{ matrix.os }}
          path: dist/
          retention-days: 7
```

### 3.3 Release Pipeline (`.github/workflows/release.yml`)

```yaml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  build-and-release:
    strategy:
      matrix:
        include:
          - os: windows-2022
            platform: windows
          - os: macos-14
            platform: macos
          - os: ubuntu-22.04
            platform: linux
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build and sign (Windows)
        if: matrix.platform == 'windows'
        run: npm run dist
        env:
          CSC_LINK: ${{ secrets.WIN_CERT_P12_BASE64 }}
          CSC_KEY_PASSWORD: ${{ secrets.WIN_CERT_PASSWORD }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build, sign, and notarize (macOS)
        if: matrix.platform == 'macos'
        run: npm run dist
        env:
          CSC_LINK: ${{ secrets.MAC_CERT_P12_BASE64 }}
          CSC_KEY_PASSWORD: ${{ secrets.MAC_CERT_PASSWORD }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_PASSWORD }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and sign (Linux)
        if: matrix.platform == 'linux'
        run: |
          npm run dist
          # GPG sign packages
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --import
          gpg --detach-sign dist/*.deb
          gpg --detach-sign dist/*.rpm
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Upload to S3 CDN
        run: |
          aws s3 sync dist/ s3://aivideoedit-releases/${{ github.ref_name }}/
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref_name, 'beta') || contains(github.ref_name, 'alpha') }}
```

---

## 4. Secret Management

| Secret | Used For | Storage |
|--------|---------|---------|
| `WIN_CERT_P12_BASE64` | Windows EV signing | GitHub Actions Secrets |
| `MAC_CERT_P12_BASE64` | macOS signing | GitHub Actions Secrets |
| `APPLE_ID` | macOS notarization | GitHub Actions Secrets |
| `APPLE_APP_PASSWORD` | macOS notarization | GitHub Actions Secrets |
| `GPG_PRIVATE_KEY` | Linux package signing | GitHub Actions Secrets |
| `AWS_ACCESS_KEY_ID` | CDN uploads | GitHub Actions Secrets |
| `AWS_SECRET_ACCESS_KEY` | CDN uploads | GitHub Actions Secrets |
| `CODECOV_TOKEN` | Coverage reporting | GitHub Actions Secrets |

---

## 5. Quality Gates

All of these must pass before code merges to `main`:

| Gate | Threshold | Blocking |
|------|---------|---------|
| ESLint errors | 0 | Yes |
| TypeScript errors | 0 | Yes |
| Ruff errors | 0 | Yes |
| mypy errors | 0 | Yes |
| Frontend unit tests | 100% pass | Yes |
| Backend unit tests | 100% pass | Yes |
| Frontend coverage | >= 70% | Yes |
| Backend coverage | >= 80% | Yes |
| Build succeeds | Yes | Yes |

---

## 6. CI Performance Optimization

| Optimization | Impact |
|-------------|--------|
| `concurrency: cancel-in-progress` | Cancel outdated PR runs |
| `npm ci` + cache | Fast npm install |
| Test sharding (Vitest) | Parallel test execution |
| Matrix builds (3 OS) | Parallel platform builds |
| Artifact caching | Reuse Python venv |
| Only run E2E on main | Fewer slow tests on PR |

---

## 7. Monitoring & Alerts

- Build status badge in README
- Slack/Discord notifications on:
  - CI failure on `main`
  - Release deployment complete
  - Security vulnerability detected (Dependabot)
- Weekly dependency updates via Dependabot
