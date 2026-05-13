# Workflow Patterns

Use these as compact starting points. Adapt commands to the repository's existing scripts and build files.

## Shared Header

```yaml
name: Build

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Node or Vite

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - uses: pnpm/action-setup@v4
        with:
          version: 10
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --if-present
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: dist/
          if-no-files-found: error
          retention-days: 14
```

If the repo uses npm or yarn, use the matching package manager and lockfile.

## Electron or Tauri Desktop

Use a matrix for OS-specific installers.

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [windows-latest, macos-latest, ubuntu-latest]
runs-on: ${{ matrix.os }}
```

For Tauri, install Rust and Node, then run the project's existing Tauri build command. Upload the platform-specific bundle folders. Signing secrets are optional for preview builds and required for trusted public releases.

## Python Package or CLI

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: python -m pip install --upgrade pip build
      - run: python -m pip install -e ".[dev]"
        if: hashFiles('pyproject.toml') != ''
      - run: pytest
        if: hashFiles('tests/**') != ''
      - run: python -m build
      - uses: actions/upload-artifact@v4
        with:
          name: python-dist
          path: dist/*
          if-no-files-found: error
```

## Java, Maven, Gradle, Spring Boot

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - run: ./mvnw -B test package
        if: hashFiles('mvnw') != ''
      - run: mvn -B test package
        if: hashFiles('mvnw') == '' && hashFiles('pom.xml') != ''
      - uses: actions/upload-artifact@v4
        with:
          name: java-package
          path: |
            target/*.jar
            target/*.war
          if-no-files-found: error
```

For Gradle, switch cache to `gradle`, use `./gradlew build`, and upload `build/libs/*`.

## Android APK

```yaml
jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
          cache: gradle
      - uses: android-actions/setup-android@v3
      - run: chmod +x ./gradlew
        if: hashFiles('gradlew') != ''
      - run: ./gradlew assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: android-debug-apk
          path: "**/build/outputs/apk/**/*.apk"
          if-no-files-found: error
```

Release signing needs GitHub Secrets for keystore material and passwords. Do not commit signing files.

## Flutter APK

```yaml
jobs:
  flutter:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true
      - run: flutter pub get
      - run: flutter test
        continue-on-error: false
      - run: flutter build apk --debug
      - uses: actions/upload-artifact@v4
        with:
          name: flutter-debug-apk
          path: build/app/outputs/flutter-apk/*.apk
          if-no-files-found: error
```

## Rust Binary

```yaml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --locked
      - run: cargo build --release --locked
      - uses: actions/upload-artifact@v4
        with:
          name: rust-${{ matrix.os }}
          path: |
            target/release/*
            !target/release/*.d
          if-no-files-found: error
```

## Release on Tag

```yaml
name: Release

on:
  workflow_dispatch:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      # Build steps go here.
      - uses: actions/upload-artifact@v4
        with:
          name: release-assets
          path: dist/*
          if-no-files-found: error
      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*
          generate_release_notes: true
```

Keep the release job's write permission scoped to the workflow that actually publishes.

## GitHub Pages

Use official Pages actions for static deployments. Build in one job, upload the Pages artifact, deploy in a separate job with `pages: write` and `id-token: write`.

## Docker Image

Use `docker/login-action`, `docker/setup-buildx-action`, and `docker/build-push-action`. Prefer GitHub Container Registry for GitHub-hosted projects. Use OIDC or scoped tokens for cloud registries.
