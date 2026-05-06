# Prod credentials

**Sensitive.** This file lives in `~/work/knowledgebase` (not committed upstream). Move all entries to 1Password / Teams vault when convenient and reduce this file to vault references.

Recovered from the deployment session transcript on 2026-05-04 because the helm-rendered values were the original source and would have been overwritten on the next chart upgrade.

## Keycloak master realm admin

Used to log into the Keycloak admin console at `https://auth.team.useblocks.com/admin/master/console/` (the master realm, separate from the `ubtrace` realm).

| Field | Value |
| --- | --- |
| URL | https://auth.team.useblocks.com/admin/master/console/ |
| Username | `ubtrace-admin` |
| Password | `e16v5yDV7OBzOO2r94AtdWlj2EDt2J2` |
| Generated | 2026-04-22 (never rotated) |
| Source | values: `keycloak.adminUser` / `keycloak.adminPassword` (now should live in the `ubtrace` Secret under `keycloak-admin-password` once `global.existingSecret` adoption finishes) |

## ubteam (ubtrace realm bootstrap user)

The seeded admin user inside the application's `ubtrace` realm. Has the `ubtrace-admin` role, so it can sign into both the user app (`team.useblocks.com`) and the admin app (`admin.team.useblocks.com`).

| Field | Value |
| --- | --- |
| Username | `ubteam` |
| Password | `1Ypj1leOtksd4DIqfjSdpnk9JYwht2WP` |
| Realm | `ubtrace` |
| Realm role | `ubtrace-admin` |
| Source | values: `keycloak.bootstrapUserPassword` |

If the password is ever lost, reset via the master admin: log into the master console with `ubtrace-admin`, switch to the `ubtrace` realm, open the `ubteam` user, Credentials -> Reset password.

## OIDC client secret (nestjs-app)

The realm OIDC client used by the API for authn token exchange.

| Field | Value |
| --- | --- |
| Client ID | `nestjs-app` |
| Client secret | `FnZuseq02pwaNKqsvDxL3jq4HhzPey2b` |
| Realm | `ubtrace` |
| Token endpoint | `https://auth.team.useblocks.com/realms/ubtrace/protocol/openid-connect/token` |
| Stored in | Secret `ubtrace`, key `oidc-client-secret` |

## RDS app DB

| Field | Value |
| --- | --- |
| Host | `ubtrace-prod-app.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com` |
| Port | `5432` |
| Database | `ubtrace` |
| User | `ubtrace` |
| Password (raw) | `sqU+%NG:gW:}e17k2y!h1YM*` |
| Password (URL-encoded) | `sqU%2B%25NG%3AgW%3A%7De17k2y%21h1YM%2A` |
| Stored in | Secret `ubtrace`, key `database-url` |
| Required suffix | `?schema=public&sslmode=no-verify` (workaround for issue #1122) |

Full URL as currently stored:

```
postgresql://ubtrace:sqU%2B%25NG%3AgW%3A%7De17k2y%21h1YM%2A@ubtrace-prod-app.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com:5432/ubtrace?schema=public&sslmode=no-verify
```

## RDS Keycloak DB

Connection details are in the helm values under `keycloak.database.*` and the `ubtrace` Secret key `postgresql-keycloak-password`. Keycloak DB host: `keycloak.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com`.

## Other secret keys

The `ubtrace` Secret in the `ubtrace` namespace holds (under `global.existingSecret: ubtrace` adoption, post-upgrade revision 14):

```
database-url
oidc-client-secret
redis-password
postgresql-keycloak-password
keycloak-admin-password
```

Annotated with `helm.sh/resource-policy: keep` so future helm upgrades cannot delete it.

To dump current values:

```bash
AWS_PROFILE=ubtrace-prod-admin kubectl -n ubtrace get secret ubtrace -o json \
  | jq -r '.data | to_entries[] | "\(.key)\t\(.value | @base64d)"'
```

## AWS

`ubtrace-prod-admin` is configured in `~/.aws/config` as an SSO profile. `aws sso login --profile ubtrace-prod-admin` to refresh credentials when they expire.
