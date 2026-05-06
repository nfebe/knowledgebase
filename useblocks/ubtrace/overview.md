# Prod environment overview

## AWS

| Resource | Value |
| --- | --- |
| Account | `809338888439` |
| Region | `eu-central-1` |
| CLI profile | `ubtrace-prod-admin` (auth via SSO, `aws sso login --profile ubtrace-prod-admin`) |
| EKS cluster | `ubtrace-prod` |
| Update kubeconfig | `AWS_PROFILE=ubtrace-prod-admin aws eks update-kubeconfig --name ubtrace-prod --region eu-central-1` |
| K8s context | `arn:aws:eks:eu-central-1:809338888439:cluster/ubtrace-prod` |

The dev account `610014107205` (cluster `ubtrace-internal`, profile `ubtrace-dev-admin`) is a separate environment behind `*.internal.useblocks.com`. Do not confuse them. Most existing docs (`docs/source/cloud-deployment/eks-internal.rst`, `values-internal.yaml`) target the dev one.

## Hostnames and ALBs

Four host-based ingresses, each fronting its own ALB:

| Host | Ingress (k8s) | Pod target | Healthcheck |
| --- | --- | --- | --- |
| `team.useblocks.com` | `ubtrace-frontend` | `ubtrace-frontend:3000` | `/` |
| `api.team.useblocks.com` | `ubtrace-api` | `ubtrace-api:3000` | `/api/v1/health/readiness` |
| `admin.team.useblocks.com` | `ubtrace-admin` | `ubtrace-admin:3000` | `/` |
| `auth.team.useblocks.com` | `ubtrace-keycloak` | `ubtrace-keycloak:8080` | `/realms/master` |

All four ingresses must have:

```yaml
annotations:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-central-1:809338888439:certificate/aac8197b-13ad-4314-985a-d7a2941014dd
  alb.ingress.kubernetes.io/healthcheck-path: <per table above>
```

`api.team.useblocks.com` healthcheck path is **`/api/v1/health/readiness`**, not `/health`. The chart default and existing docs say `/health`, which 404s on `1.3.0-dev.22+`.

## Certificate

Single ACM cert with four SANs covers all the hosts above. ARN ends `...aac8197b...`. Status: `ISSUED`. There are several `PENDING_VALIDATION` certs in the same account from earlier failed attempts (`f085c616-...`, `dce38907-...`, `cda95f97-...`) that can be cleaned up later.

## Route53

Hosted zone `team.useblocks.com.` (private + public). Each of the four hostnames is an A-ALIAS to its respective ALB DNS name. When ALBs are recreated (e.g. ingress recreated by Helm), the alias targets must be updated. See `outage-2026-05-01.md`.

## Data services (managed, outside the cluster)

| Service | Identifier | Notes |
| --- | --- | --- |
| RDS app DB | `ubtrace-prod-app.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com:5432/ubtrace` | db.t3.small, gp2 20GB, automated backups 7d |
| RDS keycloak | `keycloak.c7kw2gkyclmb.eu-central-1.rds.amazonaws.com:5432/keycloak` | db.t3.small, gp2 10GB |
| ElastiCache Redis | (single replication group, in eu-central-1, accessed via the SSM-published endpoint) | TLS on |
| EFS | `fs-041fb5e6c6fe1a5e3` | StorageClass `efs-sc`, RWX |

Both RDS instances require the Prisma TLS workaround `?sslmode=no-verify` in the `database-url` secret value. Without it the migrate job fails with a P1011 cert chain error. This is tracked in upstream issue #1122; the proper fix lands with PR #1198 / #1199 once they are released.

Keycloak users live in the keycloak RDS DB. Helm chart redeploys do **not** wipe them. Eighteen Keycloak users, including `ubteam`, were verified intact after the outage.

## Pod storage (PVCs)

The chart provisions four PVCs for app file storage. All four are RWX on EFS:

```yaml
persistence:
  inputSrc:      { accessMode: ReadWriteMany, storageClass: efs-sc, size: 10Gi }
  inputSrcBuild: { accessMode: ReadWriteMany, storageClass: efs-sc, size: 10Gi }
  inputBuild:    { accessMode: ReadWriteMany, storageClass: efs-sc, size: 10Gi }
  output:        { accessMode: ReadWriteMany, storageClass: efs-sc, size: 10Gi }
```

The chart's defaults are `ReadWriteOnce` and `storageClass: ""`, which is **immutable** and incompatible with the live PVCs. Any upgrade must keep these overrides. See `chart-quirks.md`.

The Elasticsearch StatefulSet uses `volumeClaimTemplates` with `storageClassName: gp3` (set via `global.storageClass: gp3`). That field is also immutable on a StatefulSet. Drop it from values and the upgrade fails.

## Resource summary

Roughly $320-$400/month baseline:

- 2x t3.large EKS nodes (~$120/mo)
- 2x db.t3.small RDS (~$70/mo)
- 1x ElastiCache cache.t3.micro (~$15/mo)
- 4x ALBs (~$100/mo, mostly LCU + idle hours)
- 1x NAT gateway + data (~$45/mo)
- EFS, EBS, ECR storage (~$20/mo)

The €1.5k/€900 forecast on the main account is almost certainly **not** this cluster. Check the main account separately.

Easy wins (~$150-$180/mo): drop one t3.large node, consolidate the four ALBs into one with host-based routing, consolidate the two db.t3.small RDS into one, add VPC endpoints to cut NAT data fees.
