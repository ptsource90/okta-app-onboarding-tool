# Scheduled Okta configuration sync (drift detection)

Resolves the "Drift detection" item that used to sit in
`08-open-questions.md`: if an Admin edits, deletes, deactivates,
reactivates, or changes assignments on an app directly in the Okta
console instead of through this tool, the local record silently
disagrees with reality until something notices. This job is that
"something."

## Confirmed decisions

- **Direction: Okta → local only.** The reverse direction (local changes
  reaching Okta) already happens synchronously, immediately, and only on
  explicit user confirmation via the create/edit/delete/deactivate/
  reactivate/assign/unassign flows — never batched or queued for a
  scheduled push. This job never writes to Okta.
- **Config or active/inactive-status drift: Okta always wins**, applied
  automatically, no human confirmation required. This covers both
  `ConfigurationJson` differences and `Application.IsActive` no longer
  matching Okta's own ACTIVE/INACTIVE status — both are corrected the
  same way, in the same pass, since both come from the same "read the
  app from Okta" call. This is different from every user-facing write in
  the app (create/edit/delete/deactivate/reactivate all require the user
  to review and submit) — the sync job is a background reconciliation
  step, not a user-initiated action, so the "always confirm" rule that
  governs those doesn't apply to it.
- **Deletion drift is handled, but differently from config/status drift.**
  If the job finds that an app tracked locally no longer exists in Okta:
  - The local record is soft-deleted (`DeletedAt` set, same column as
    every other delete), **but** `DeletionSource = DetectedInOkta`.
  - Unlike a normal delete, this app **stays visible** in the UI — the
    user/Admin can still open it and see its last-known configuration —
    but it's locked: no Edit button, no Delete button (there's nothing
    left in Okta to act on), and it's clearly labeled, e.g. a "Deleted in
    Okta" badge, distinct from an active app.
  - Contrast with a delete initiated *through this tool*
    (`DeletionSource = ToolInitiated`): that continues to behave exactly
    as already designed — the app simply disappears from the active
    list. The row is still soft-deleted underneath for audit-trail
    integrity, but there's no special locked "ghost" view for it. The
    ghost view is reserved for the Okta-direct case, because that's the
    one where the user might otherwise have no idea the app is gone.
- **No auto-discovery of new apps.** The job only re-checks apps this
  tool already tracks (`Application` rows not already soft-deleted). It
  never surfaces or adopts apps that exist in Okta but were never
  imported — that stays the existing, deliberate, Admin-only **Import**
  flow (`01-requirements.md` requirement 6). Running the two mechanisms
  side by side would create confusing overlap (is an unimported app
  something to "sync" or something to "import"?), so import stays
  entirely manual.
- **Per environment, not global.** Each `Environment` is a separate Okta
  org with its own API token and its own Okta rate limits, so the job
  runs once per environment rather than as a single global pass.
- **Scheduling mechanism: Hangfire, cron read from `appsettings.json`,
  defaulting to every 15 minutes.** One Hangfire recurring job per
  configured `Environment` (`RecurringJob.AddOrUpdate`), each
  independently invoking `SyncEnvironmentConfigurationCommand` for its
  own environment — so one environment's sync failing or running long
  doesn't block another's. The cron expression itself is not hard-coded:
  it's read via `IOptions<T>` from an `appsettings.json` setting (e.g.
  `DriftSync:CronExpression`), defaulting to `*/15 * * * *` if unset —
  same convention as the environment-seed settings in
  `04-data-model.md`. Hangfire uses the same MSSQL database as EF Core
  for its own job storage (`Hangfire.SqlServer`), giving retry-on-failure
  and an inspectable job history/dashboard for free rather than
  hand-rolling a `BackgroundService` + timer. See `06-tech-stack.md`.
- **Deletion drift also resolves any pending Production deletion
  approval for that app.** If an app has an open (`Status = Pending`)
  `ApprovalRequest` (a User's Production delete request awaiting Admin
  review) and the sync job separately discovers the app is already gone
  from Okta, the job auto-cancels that request: `Status = Cancelled`,
  `ReviewNote` set to a system-generated reason (e.g. "Auto-cancelled:
  app was deleted directly in Okta before this request was reviewed"),
  `ReviewedByUserId` left null (no human reviewed it). This is written as
  its own audit entry (`ApprovalRequestAutoCancelled`, `UserId = null`,
  `Details` including the `ApprovalRequest.Id` and the reason) in
  addition to the `AppDeletedInOkta` entry for the app itself — so the
  cancellation is independently visible in the audit trail, not just
  implied by the app's deletion.
- **System actor in the audit log.** Every change this job makes writes
  an `AuditLogEntry` with `UserId = null` and an action of
  `AppSyncedFromOkta` or `AppDeletedInOkta` (see `04-data-model.md`) — so
  it's clearly distinguishable in the audit trail from a human-initiated
  `AppEdited`/`AppDeleted`, without inventing a fake "system" user row.

## Flow

```
Hangfire recurring job (per Environment, independently, every 15 minutes)
     │
     ▼
1. Call IOktaAppService.ListAllAsync() for this environment's org
     │
     ▼
2. For each local Application in this environment where DeletedAt IS NULL:
     │
     ├─ Not found in Okta's list (by OktaAppId)?
     │     → soft-delete locally: DeletedAt = now, DeletionSource = DetectedInOkta
     │     → write AuditLogEntry (UserId = null, Action = AppDeletedInOkta,
     │        ApplicationId = this app, Result = Success)
     │     → if an ApprovalRequest with Status = Pending exists for this app:
     │          set Status = Cancelled, ReviewNote = system reason,
     │          ReviewedByUserId = null
     │          → write AuditLogEntry (UserId = null,
     │             Action = ApprovalRequestAutoCancelled, Result = Success)
     │     → invalidate this app's cache entry
     │
     └─ Found in Okta — compare Okta's current config AND active/inactive
          status to local ConfigurationJson / Application.IsActive:
           ├─ Same → update LastSyncedAt only, no audit entry (avoids audit-log
           │    noise on every no-op run)
           └─ Different (either config, status, or both) → overwrite local
                ConfigurationJson and/or IsActive to match Okta,
                set UpdatedAt and LastSyncedAt
                → write AuditLogEntry (UserId = null, Action = AppSyncedFromOkta,
                   ApplicationId = this app, Details = diff of changed fields,
                   Result = Success)
                → invalidate this app's cache entry
          Also, independently of the config/status comparison above:
          fetch this app's current assignments from Okta (the app
          resource's own _links.users.href / _links.groups.href) and
          diff against local ApplicationAssignment rows for this
          ApplicationId:
           ├─ Same set → update LastSyncedAt on each row only, no audit
           │    entry
           └─ Different → add/remove local ApplicationAssignment rows to
                match Okta (AssignedByUserId = null for anything added
                this way), write AuditLogEntry (UserId = null,
                Action = AssignmentSyncedFromOkta, ApplicationId = this
                app, Details = which principal(s) were added/removed),
                invalidate this app's assignment cache entry
     │
     ▼
3. On any per-app failure (e.g. one Okta call errors): log it, write an
   AuditLogEntry with Result = Failure for that app, and continue with the
   rest of the batch — one bad app shouldn't block reconciling the others.
```

## Open questions (flag if these need a firmer answer before building)

- **Fallback/validation behavior if the cron setting is missing or
  malformed** — confirmed the interval is read from `appsettings.json`
  and defaults to `*/15 * * * *`, but whether a malformed cron string
  should fail startup loudly vs. silently fall back to the default is not
  yet decided.
- **Environment domain re-sync after initial seed** — see
  `04-data-model.md`'s `Environment` entity and the matching item in
  `08-open-questions.md`.
