# PSIRA Security Audit - 2026-09-10

## Scope

First local security audit before the PSIRA test phase.

Repositories checked:

- `psira-frontend`
- `psira-backend`
- `psira-docker`
- `psira-shiny`

## Auditors Executed

### Dependency audit

Command executed through the local backend Docker image so the host did not need a local Node.js installation:

```bash
npm audit --json --package-lock-only --omit=dev --audit-level=moderate
```

Results were saved under `/tmp/psira-security-audit/` during the local run.

#### Frontend result

`npm audit` failed the threshold.

Summary from npm metadata:

- Critical: 6
- High: 66
- Moderate: 47
- Low: 5

Highest priority packages/advisories include:

- `crypto-js < 4.2.0`: weak PBKDF2 behavior.
- `xlsx < 0.20.2`: prototype pollution and ReDoS advisories.
- Angular packages around version 10: multiple newer advisories affecting Angular packages.
- Transitive build/runtime packages such as `minimist`, `minimatch`, `json5`, `lodash`, `postcss` and `@babel/traverse`.

#### Backend result

`npm audit` failed the threshold.

Summary from npm metadata:

- Critical: 53
- High: 249
- Moderate: 169
- Low: 44

Highest priority packages/advisories include:

- `class-validator < 0.14.0`: SQL injection and XSS advisory.
- `otp-generator < 3.0.0`: insecure OTP generation.
- `typeorm < 0.3.0`: SQL injection advisory.
- `mongoose < 6.13.6`: prototype pollution/search injection advisories.
- `handlebars <= 4.7.8`: JavaScript injection advisory.
- `crypto-js < 4.2.0`: weak PBKDF2 behavior.
- `vm2`: multiple sandbox escape advisories through transitive dependencies.
- Transitive packages including `form-data`, `json-schema`, `tar`, `minimist` and Apollo Federation packages.

### Runtime header checks

Checked the running local Docker deployment from inside the Docker network using HTTPS and `Host: localhost`.

Observed good controls:

- HTTP redirects to HTTPS with 308.
- `Strict-Transport-Security` is present.
- `X-Content-Type-Options: nosniff` is present.
- `X-Frame-Options: SAMEORIGIN` is present.
- `Referrer-Policy: strict-origin-when-cross-origin` is present.
- `Permissions-Policy` blocks camera, microphone and geolocation.
- `Cross-Origin-Resource-Policy: same-origin` is present.
- CSP is present in report-only mode.
- `/shiny/` direct access returns 403.
- `/shiny-embed/...` without an embed token returns 401.
- `/graphql` uses `Cache-Control: no-store`.

Findings fixed during this audit:

- Backend disclosed `X-Powered-By: Express`; the backend now disables this header.
- GraphQL returned broad CORS headers; the GraphQL module now receives the same explicit CORS policy as the Nest app.

These backend header fixes require rebuilding/restarting the backend container before they appear in runtime scans.

### Static secret scan

A lightweight ripgrep scan looked for common hardcoded secret patterns while excluding local database data, `.git`, `node_modules`, `dist`, `coverage` and logs.

Findings fixed during this audit:

- Removed hardcoded test credentials from Shiny test scripts.
- Shiny test scripts now read `PSIRA_TEST_USERNAME` and `PSIRA_TEST_PASSWORD`.
- Removed the insecure backend fallback superadmin credentials.
- Backend superadmin seeding now requires `SUPERADMIN_USERNAME` and `SUPERADMIN_PASSWORD`.
- Backend logging no longer prints the seeded superadmin password.

Remaining matches are documentation/template tokens such as `{{password}}`, translation labels and setting names, not real secrets.

### Workflow configuration validation

YAML parsing passed for:

- `psira-frontend/.github/workflows/security.yml`
- `psira-backend/.github/workflows/security.yml`
- `psira-docker/.github/workflows/staging-security.yml`
- `psira-frontend/.github/dependabot.yml`
- `psira-backend/.github/dependabot.yml`
- `psira-docker/.github/dependabot.yml`

## Auditors Not Fully Executed Locally

- CodeQL: configured for GitHub Actions; not practical to run locally in this environment.
- Semgrep: configured for GitHub Actions; no local `semgrep` binary or image is currently available.
- OWASP ZAP baseline/API scan: configured for GitHub Actions. The local environment does not have a ZAP image installed. Run against public staging with `target_url` or `PSIRA_STAGING_URL`.
- Mozilla HTTP Observatory: configured for public hosts. It should be run against the public staging domain, not the local Docker host.

## Priority Remediation Plan

1. Upgrade or replace backend critical direct dependencies first: `class-validator`, `otp-generator`, `typeorm`, `mongoose`, `handlebars` and `crypto-js`.
2. Upgrade or replace frontend critical/high direct dependencies: `crypto-js`, `xlsx`, Angular packages and Tailwind/PostCSS chain.
3. Rebuild and restart backend/Caddy, then rerun header checks to confirm `X-Powered-By` and GraphQL CORS are corrected at runtime.
4. Run GitHub Actions security workflows after these changes are pushed.
5. Run ZAP and HTTP Observatory against the real staging URL before opening the test phase.

## Remediation pass - dependency fixes

Applied on 2026-09-10 after the first audit.

### Frontend corrections

- Updated `crypto-js` to `^4.2.0`.
- Updated `moment` to `^2.30.1`.
- Updated `moment-timezone` to `^0.5.48`.
- Updated `@ngneat/until-destroy` to `^8.1.4`, keeping Angular 10 compatibility.
- Moved `tailwindcss` from production dependencies to dev dependencies.
- Removed vulnerable `xlsx` and unused `FileSaver` dependencies.
- Replaced the XLSX export implementation with a dependency-free `.xls` export generated in the browser.
- Added npm `overrides` for vulnerable transitive packages that can be patched safely without framework migration.

Final production audit with npm 8 after this pass:

- Critical: 0
- High: 26
- Moderate: 2
- Low: 1

Remaining frontend findings are mostly tied to Angular 10 packages and require an Angular framework migration rather than a patch-level dependency update.

Frontend production build passed in a temporary Docker-based build directory. It completed with warnings about bundle size/CommonJS dependencies and a Moment locale warning, but no build error.

### Backend corrections

- Updated `crypto-js` to `^4.2.0`.
- Updated `handlebars` to `^4.7.9`.
- Updated `moment` to `^2.30.1`.
- Updated `moment-timezone` to `^0.5.48`.
- Updated `passport` to `^0.7.0`.
- Updated `yaml` to `^1.10.3`.
- Updated `node-xlsx` to `^0.24.0`, which uses the patched SheetJS tarball instead of the vulnerable npm `xlsx` package.
- Updated `mongoose` within the Nest 7 compatible major line to `^5.13.23`.
- Updated `class-validator` to `0.14.0` and pinned compatible `libphonenumber-js` / `@types/validator` versions so the TypeScript 3.7 backend compiles.
- Declared `@nestjs-query/core` explicitly because the code imports it directly.
- Removed unused direct `otp-generator` dependency.
- Moved `commitizen` from production dependencies to dev dependencies.
- Replaced `bipsms` with an internal Infobip HTTP client using Node `http`/`https`, removing the deprecated `request`/`form-data` chain.
- Replaced `@nestjs-modules/mailer` with an internal Nodemailer-backed `MailerService`, reducing unnecessary mailer preview/template dependencies.
- Added npm `overrides` for vulnerable transitives that can be patched safely.

Final production audit with npm 8 after this pass:

- Critical: 5
- High: 45
- Moderate: 16
- Low: 5

Remaining backend critical findings require framework-level migrations:

- `typeorm >=0.3.0`: current code and `@nestjs/typeorm@7` are on the TypeORM 0.2 API.
- `mongoose >=6.13.6`: current `@nestjs/mongoose@7` declares peer compatibility with Mongoose 5, not 6.
- `@apollo/gateway` / `@apollo/query-planner` / `tar`: these are pulled by the Nest 7 GraphQL/Apollo 2 stack and require a Nest GraphQL/Apollo migration to remove properly.

Backend build passed in a temporary Docker-based build directory after the dependency fixes. The final npm 8 installation in Docker was also aligned by updating Dockerfiles/workflows to use npm 8 and legacy peer dependency resolution where this Nest 7 stack requires it.
