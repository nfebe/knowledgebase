# ubTrace prod deployment knowledge base

Captured 2026-05-04 from the deployment session that took prod from chart 1.2.1 / app `1.3.0-dev.22` and recovered from the ALB outage on 2026-05-01.

The next user-visible upgrade target is `1.3.0-dev.23` (the only commit ahead of dev.22 on `develop` is the always-on metric builder feature flag, PR #1266).

This folder is private reference: account-specific identifiers, recovered secrets, post-mortems, and gap audits. The **generic upgrade process** is not duplicated here. It belongs in the ubtrace repo docs (currently being drafted into PR #1050 follow-ups).

Contents:

- [overview.md](overview.md): hostnames, AWS resources, account/profile, cluster, RDS, EFS, cert ARN.
- [credentials.md](credentials.md): Keycloak master admin, ubteam bootstrap user, OIDC client secret, RDS DB user/pass. **Sensitive: do not commit upstream.**
- [outage-2026-05-01.md](outage-2026-05-01.md): root cause and timeline of the ALB outage. The recurring lessons (chart quirks, immutability, ALB recreation, secret cutover) are listed here but belong in the chart docs / repo runbook, not duplicated as separate files.
- [deploy-gaps.md](deploy-gaps.md): the list of doc/chart/runbook gaps to close in PR #1050 follow-ups.
- [pr-1050-update-coverage.md](pr-1050-update-coverage.md): audit of whether PR #1050 docs cover routine upgrades (short answer: no).

## Quick facts

| Thing | Value |
| --- | --- |
| Prod AWS account | `809338888439` |
| Region | `eu-central-1` |
| AWS profile | `ubtrace-prod-admin` |
| EKS cluster | `ubtrace-prod` |
| K8s namespace | `ubtrace` |
| Helm release | `ubtrace` (current revision: 14, chart `ubtrace-1.2.1`, app `1.3.0-dev.22`) |
| Public hosts | `team.useblocks.com`, `api.team.useblocks.com`, `admin.team.useblocks.com`, `auth.team.useblocks.com` |
| ACM cert (in use, ISSUED) | `arn:aws:acm:eu-central-1:809338888439:certificate/aac8197b-13ad-4314-985a-d7a2941014dd` (4 SANs) |
| RDS app | `ubtrace-prod-app.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com:5432/ubtrace` (db.t3.small) |
| RDS keycloak | `ubtrace-prod-keycloak` (db.t3.small) |
| EFS | `fs-041fb5e6c6fe1a5e3` (StorageClass `efs-sc`, RWX) |
| Storage default | `gp3` for StatefulSets (Elasticsearch) via `global.storageClass` |
| Dev account (separate) | `610014107205` (`ubtrace-dev-admin` profile, cluster `ubtrace-internal`) |
