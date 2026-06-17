# ubTrace PROD status monitoring (heartbeat)

Automated monitoring for ubTrace PROD (`team.useblocks.com`), built in ubinfra and
modeled on the existing `download_heartbeat` stack. Two-layer end state: an alerting
engine (shipped) and, later, a githubstatus.com-style public page.

Parent issue: ubinfra #169. Stack: `deploy/terraform/ubtrace_heartbeat/`.

## How it works

EventBridge invokes a Lambda every 5 minutes. The Lambda GETs each target and checks
the response against that target's expected status codes; redirects are not followed,
so the auth login redirect stays a `302`. Targets mirror `deploy/helm/ubtrace/deploy.sh
smoke`, so the monitor and the manual deploy check verify the same surfaces:

| Target | Healthy |
| --- | --- |
| team.useblocks.com | 200 |
| admin.team.useblocks.com | 200 |
| api.team.useblocks.com (root) | 404 |
| auth.team.useblocks.com | 302 |
| api.team.useblocks.com/api/v1/health/readiness | 200 |

A two-strike state machine in SSM emails via SNS: one ALARM after 2 consecutive
failures, a daily REMINDER while failing, one RECOVERY. This caps mailbox volume per
incident.

## Watch-the-watcher (two alarms)

- **Dead man's switch** (#170) on the Lambda's `Invocations`: fires if it has not run
  for 15 minutes (catches a deleted schedule or disabled rule).
- **Stale-run** (#171) on a `RunCompleted` metric the Lambda emits only after a fully
  successful run: fires if no successful run completes for 15 minutes. Catches a Lambda
  that is still invoked but crashes every run (broken IAM, code error, failing
  downstream call), which the invocation count alone misses.

## Account, state, cost

- Account: ubtrace-prod `809338888439`, region eu-central-1.
- Remote state bucket: `ubtrace-heartbeat-tfstate-809338888439` (bootstrap in
  `bootstrap_state/`).
- The Lambda only makes outbound HTTPS to public hostnames: no VPC, no cross-account.
- Cost ~$2/mo (versus ~$10/mo for an equivalent CloudWatch Synthetics canary).

## What shipped

- #170 reachability engine: PR #176 (base `main`).
- #171 stale-run hardening: PR #178 (stacked on the #170 branch).
- Both deployed to prod and live-verified: all 5 targets read correctly, the schedule
  runs autonomously, the forced-failure test alarmed after exactly 2 misses and
  recovered, and `RunCompleted` is flowing with all alarms in OK.

## Operational gotcha

SNS email subscriptions auto-delete after 72 hours if unconfirmed. The alert address
`security@useblocks.com` must click the AWS confirmation link or no alert delivers. A
`terraform apply` recreates a lapsed subscription, sending a fresh confirmation email.
This is the one manual step that makes the monitor actually page.

## Roadmap (sub-issues under #169)

- #172 functional/authenticated checks (would have caught the selector outage; needs a
  scoped Keycloak probe credential, coordinate with Laszlo).
- #173 public status page (S3 + CloudFront serving a static page that reads a
  Lambda-written `status.json`; needs a DNS record and an ACM cert in us-east-1).
- #174 incident history and uptime.
- #175 documentation and deploy-checklist wiring.

## Related: EKS cost reduction (#165, separate work)

The EKS 1.31 -> 1.35 upgrade on Jun 9 stopped the extended-support penalty. The account
daily run-rate dropped from ~$30/day to ~$15.6/day after the upgrade; the EKS line was
$446 in May and trending down through June. Close #165 once a full clean month shows the
extended-support charge at ~$0.
