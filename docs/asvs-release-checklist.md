# PSIRA ASVS Release Checklist

Use this checklist before a test-phase or production release. It is based on OWASP ASVS and focuses on the parts that matter most for PSIRA.

## Authentication And Session

- Login rejects invalid credentials without revealing whether the username exists.
- Password reset and password change flows require a valid authenticated or administrative path.
- Access tokens expire and are not logged.
- Logout clears local browser storage and server-side session material if introduced later.
- If refresh tokens move to cookies, cookies use `HttpOnly`, `Secure`, `SameSite` and narrow `Path`.

## Access Control

- Every GraphQL resolver that returns clinical, patient, user, notification, automation or report data has an explicit permission guard.
- Department-scoped users cannot access records outside their allowed departments.
- Assigned-scoped users cannot access records for unassigned patients or assessments.
- Shiny report embed URLs require a valid short-lived embed token.
- Direct `/shiny/*` access remains blocked.

## Input And Output Safety

- User-controlled input is validated at DTO/form boundaries.
- Rich text, report descriptions and email templates are sanitized or rendered only in trusted contexts.
- GraphQL query depth, pagination and large export paths have bounded limits.
- File/export endpoints do not allow path traversal or arbitrary file reads.

## Browser And Transport

- HTTPS is enforced for public environments.
- HSTS, `X-Content-Type-Options`, `Referrer-Policy` and `Permissions-Policy` are present.
- CSP report-only findings are reviewed before moving CSP to enforcement.
- Sensitive API responses use cache policy appropriate to clinical data.

## Data And Logging

- Logs do not include passwords, tokens, assessment answers or clinical notes.
- Audit logs capture clinically relevant administrative changes.
- Exports respect the same authorization rules as the UI.
- Backups are encrypted or stored in controlled infrastructure.

## Operations

- Dependency audit, CodeQL, Semgrep and Dependabot are active on frontend and backend.
- ZAP baseline and Observatory scans pass against staging.
- ZAP active/API scans run only against approved test data.
- Security findings have an owner and disposition before release.
