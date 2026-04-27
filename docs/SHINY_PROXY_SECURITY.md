# Shiny proxy security plan

Current state:

- Users enter reports through PSIRA at `/psira/reports/:id`.
- The backend checks report status and role access before issuing an embed URL.
- The embed URL includes a short-lived signed `embed_token`.
- Shiny apps validate `embed_token` before loading sensitive data.
- The Shiny container no longer exposes host port `3838`.
- Caddy blocks direct `/shiny/...` browser access.
- Caddy proxies `/shiny-embed/<token>/<app>/...` internally after backend validation.

This is the hardened proxy path. The browser no longer needs direct `/shiny/...` access.

## Why not simply block `/shiny/...` in Caddy?

Shiny apps load additional assets and websocket-like connections under the same route tree. A naive Caddy rule such as "block every `/shiny/*` request without `embed_token`" can break app assets, sessions, or static resources.

Referer checks are also not strong security. They are useful as a soft signal, but they should not be the main protection.

## Robust target architecture

Use backend-assisted proxy authorization:

1. Browser opens `/psira/reports/:id`.
2. Backend verifies user permissions.
3. Backend issues a short-lived signed embed token.
4. Iframe loads `/shiny-embed/<token>/<app>/?embed_token=...`.
5. Caddy calls a backend auth endpoint before proxying Shiny traffic.
6. Shiny also validates `embed_token` before loading data.

This gives two layers:

- proxy-level gate before the request reaches Shiny;
- app-level gate before data is loaded.

## Proposed backend endpoint

Add an HTTP endpoint, not GraphQL, because Caddy `forward_auth` works naturally with HTTP:

```text
GET /api/report-embed/validate
```

Expected behavior:

- `2xx` if token is valid and unexpired;
- `401` or `403` if missing, invalid, expired, or revoked.

The endpoint validates the same token format used by `getReportEmbed`. Caddy passes the original URI in `X-Forwarded-Uri`, and the backend extracts the token from `/shiny-embed/<token>/...`.

## Proposed Caddy shape

Conceptual shape only:

```caddy
@shiny_embed path_regexp shiny_embed ^/shiny-embed/([^/]+)/(.*)$
route @shiny_embed {
    forward_auth psira-backend:3000 {
        uri /api/report-embed/validate
        header_up X-Forwarded-Uri {uri}
    }

    rewrite * /{http.regexp.shiny_embed.2}
    reverse_proxy shiny:3838
}
```

Direct `/shiny/...` is blocked with `403`. The response body is intentionally human-readable:

```text
Shiny apps must be opened from PSIRA.
```

Seeing that message confirms the direct route is blocked; it does not mean the Shiny app is usable outside PSIRA.

## Current recommendation

Test every app through `/psira/reports/:id`, then test direct `/shiny/<app>` and confirm it returns `403`. Also test token expiration by opening a report, waiting longer than `REPORT_EMBED_TOKEN_TTL`, and refreshing the iframe or page.
