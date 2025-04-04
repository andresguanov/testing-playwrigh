name: Playwright Tests

on:
push:
branches: [main, master]
pull_request:
branches: [main, master]

jobs:
unit:
timeout-minutes: 60
runs-on: ubuntu-latest
steps: - uses: actions/checkout@v4 - uses: actions/setup-node@v4
with:
node-version: lts/\* - name: Install dependencies
run: npm ci - name: Run unit tests
run: echo "No unit tests to run"

test:
runs-on: ubuntu-latest
timeout-minutes: 60
steps: - name: Checkout code
uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: |
          npx playwright test --reporter=html --output=playwright-report || echo "Tests failed" > .failed-tests

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Save Failed Tests
        if: failure()
        run: |
          grep -l "✖" playwright-report/*.json > .failed-tests || echo "No failed tests"

      - name: Upload Failed Tests for Manual Retry
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failed-tests
          path: .failed-tests
          retention-days: 30

deploy:
runs-on: ubuntu-latest
needs: test
steps: - uses: actions/checkout@v4 - name: Deploy to production
run: echo "Deploying to production"

retry-failed-tests:
runs-on: ubuntu-latest
needs: test # Esto asegura que se ejecute después de que los tests iniciales se hayan completado
steps: - name: Checkout code
uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Download Failed Tests
        uses: actions/download-artifact@v4
        with:
          name: failed-tests
          path: .

      - name: Run Failed Tests
        run: |
          cat .failed-tests | xargs -I {} npx playwright test {} --reporter=html --output=playwright-report

      - name: Upload Retry Report
        uses: actions/upload-artifact@v4
        with:
          name: retry-playwright-report
          path: playwright-report/
          retention-days: 30

      #     steps:
      # - uses: actions/checkout@v4
      # - uses: actions/setup-node@v4
      #   with:
      #     node-version: lts/*
      # - name: Run unit tests
      #   run: echo "unit tests to ${{ matrix.file }}"
      # - name: Install dependencies
      #   run: npm ci
      # - name: Install Playwright Browsers
      #   run: npx playwright install --with-deps
      # - name: Run Playwright tests
      #   run: npx playwright test ${{ matrix.file }}.spec.ts
      # - uses: actions/upload-artifact@v4
      #   if: ${{ !cancelled() }}
      #   with:
      #     name: playwright-report
      #     path: playwright-report/**/*
      #     retention-days: 30

    # steps:
    #   - uses: actions/checkout@v4
    #   - uses: actions/setup-node@v4
    #     with:
    #       node-version: lts/*
    #   - name: Install dependencies
    #     run: npm ci
    #   - name: Install Playwright Browsers
    #     run: npx playwright install --with-deps
    #   - name: Run Playwright tests
    #     run: npx playwright test ${{ matrix.file }}.spec.ts
    #   - uses: actions/upload-artifact@v4
    #     if: ${{ !cancelled() }}
    #     with:
    #       name: playwright-report
    #       path: playwright-report/**/*
    #       retention-days: 30

    - name: Cache node modules

uses: actions/cache@v3
with:
path: node_modules
key: ${{ runner.os }}-node-modules-${{ hashFiles('package-lock.json') }}
restore-keys: |
${{ runner.os }}-node-modules-
