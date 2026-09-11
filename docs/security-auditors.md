# PSIRA Security Auditors

This repository contains the runtime security scans for PSIRA deployments.

## Automated Checks

- OWASP ZAP baseline scan checks the public web surface with passive rules.
- OWASP ZAP GraphQL scan can check the GraphQL API when `PSIRA_GRAPHQL_SCHEMA_URL` is configured.
- Mozilla HTTP Observatory checks HTTP security headers and HTTPS posture.

The staging workflow can be run manually with `target_url`, or on schedule with repository secrets:

- `PSIRA_STAGING_URL`: public staging URL, for example `https://staging.psira.net`.
- `PSIRA_GRAPHQL_SCHEMA_URL`: GraphQL schema URL or file endpoint for ZAP API scan.

## Manual Release Review

Use OWASP ASVS 5.0 as the checklist before production releases. For PSIRA, start with these areas:

- Authentication and session handling.
- Access control by role, department and assigned patient.
- Input validation and output encoding.
- GraphQL authorization on every resolver.
- Security headers, TLS and cache policy for sensitive data.
- Logging without clinical data leakage.

## Scan Policy

Run passive scans against shared staging. Run active scans only against disposable test data and an environment approved for destructive testing.
