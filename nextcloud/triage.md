# Nextcloud triage notes

From triaging Nextcloud Enterprise support tickets (the `nc-tickets` repo) and community-filed issues on `nextcloud/server`. Recurring findings worth keeping.

## Support tickets (nc-tickets repo)

Full per-ticket notes live in `nc-tickets`. The non-obvious bits:

- **Shared folder "kicks the user back to the parent" (#96100732).** A recursive server-side encryption keys path (`files_encryption/keys/files/<dir>/<dir>/...` self-nested) overflows the `oc_filecache.path` column (varchar 4000) during the background scan. On PostgreSQL the failed insert aborts the transaction, so `SharedStorage::init` then fails and the share falls back to a `FailedStorage`, which the UI can drop to the parent. Fix: back up, remove the recursive keys subtree, then `occ files:scan <user>`. The Files frontend has exactly one silent parent-redirect path (`apps/files/src/router/router.ts`, fired on `files:node:deleted` for the open folder); a failed directory load otherwise shows an error, not a silent redirect.

- **Out-of-band security patch, "patches over patches" (#96100741, sibling #96100737).** The June 2026 NC33 patch is the two-factor-authentication bypass via pre-2FA session token replay: CVE-2026-45690 (the token replayed via HTTP Basic auth) and CVE-2026-45691 (the pre-2FA session cookie reused as a Bearer token against DAV endpoints), both fixed in 33.0.3 and 32.0.9. Indicators of exploitation: authenticated requests to `/remote.php/dav/` using Basic or Bearer auth from accounts that have 2FA enforced, in the window before patching; the `admin_audit` log helps but writes at INFO, so entries are suppressed when the system `loglevel` is the default WARN. Applying a patch does not block the normal update path. The GPG "not certified with a trusted signature" warning is normal GnuPG output, not an error.

- **Slow "Remove activity entries of private events" repair (#96100713).** `apps/dav/lib/Migration/RemoveClassifiedEventActivity.php` issues one full-scan leading-wildcard `LIKE` DELETE per classified calendar event. There is no batch or resume: an interrupted run restarts from the first event, and the completion flag is set only after one full pass. Skip it with `occ config:app:set dav checked_for_classified_activity --value=1 --type=boolean`. The underlying leak was fixed in Nextcloud 16 (released 2019-04-24), so an instance that started after that and never migrated from ownCloud has little or nothing to delete; the step is kept for ownCloud migrations. Seeing DELETE statements does not mean rows are being deleted: it fires per event regardless, and the rows-removed total is the `Removed N activity entries` line it prints.

- **Background-job durations (#9697334).** Read per-job time from `oc_jobs.execution_duration` (seconds), not from gaps between `last_run` values (`last_run` is the start time, not the finish). The serverinfo `UpdateStorageStats` job is usually the heavy one (default interval 3 hours, app key `serverinfo job_interval_storage_stats`); the calendar reminder job runs in about 0 seconds. Cron is designed to spawn a fresh worker every 5 minutes with up to three overlapping, so a single long job does not block the every-5-minute reminder unless cron is configured to prevent overlap. Infrastructure and configuration questions are out of support scope.

## Community issue triage (nextcloud/server)

For a batch (filter, verify in code, label, comment):

- **Finding them.** The `community` label means "pull requests from community" and is also applied to the backport bot's PRs, so it is not a clean community-PR filter; for the human contributions, filter out `app/backportbot`. Issues have no community label, so filter by author association (anything other than MEMBER, OWNER, COLLABORATOR), excluding bots.

- **Labels.** Verify every suggested label against the real set before applying (the repo has about 145 labels). Component labels are `feature: <area>` (for example `feature: occ`, `feature: recommended apps`, `feature: dav`, `feature: contacts`, `feature: caldav`, `feature: apps management`, `feature: users and groups`). There is no `duplicate`, `invalid`, or `working-as-intended` label; those are close-reasons, not labels. Useful state labels: `33-feedback` / `34-feedback` / `35-feedback`, `regression`, `papercut`, `needs info`, `good first issue`, `design`.

- **NC34 appstore split.** App management moved out of the `settings` app into a new `appstore` app (commit `5b756ad8bcf`, "refactor: split appstore from settings"), removing the routes `settings.AppSettings.viewApps` and `settings/apps/list`. This caused regressions where old links or calls still point at them: the recommended-apps setup fetch (#61313, fix #61575) and the app-update notification link in `apps/updatenotification/lib/Notification/Notifier.php` (#61426). The new route is `appstore.Page.viewApps`.

- **Other confirmed server bugs from the 2026-06-13 to 2026-06-19 batch.** user_ldap `getDomainDNFromDN()` appends the integer `count` element from `ldap_explode_dn()` because the loop has no non-string guard, yielding an invalid DN (#61446, fix #61479). "Recently contacted" stores a person twice because a user share records the contact by user id only and a calendar attendee records it by email only, and the match cannot bridge them (#61347). The share dialog's "Type an email" placeholder shows even when no email-capable external provider is enabled, so Enter routes to whatever provider exists (#61364). DAV returns 500 instead of 4xx when a client `.ics` uses the non-conformant `VALUE=DATETIME` (RFC 5545 is `DATE-TIME`) (#61276).

- **Triage-comment rule.** Do not post a comment that only re-affirms what the reporter, or a maintainer earlier in the thread, already said. Comment only to add something: a root cause, a correction, a redirect to the right repo, a decision, or a question.
</content>
