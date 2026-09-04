# Trellis Restic Backups

A standalone Ansible role that installs restic and schedules backups of uploads
for remote [Roots Trellis](https://roots.io/trellis/) WordPress servers. Every
six hours it creates one snapshot containing every site's `shared/uploads`
directory and stores it in an existing Amazon S3 bucket under `<stack>/<env>`.

Database backups are not included. Configure and verify a separate database
backup system.

## Requirements

- Trellis 1.31 or newer on Ubuntu 24.04 (`x86_64` or `aarch64`); the role
  relies on Trellis's `env`, `web_user`, `web_group`, `www_root`,
  `wordpress_sites`, `apt_cache_valid_time`, and the `curl` package installed
  by the `common` role
- Ansible 2.10 or newer
- An existing S3 bucket, an initialized restic repository, and the EC2
  instance-profile permissions described in [AWS contract](#aws-contract); no
  static AWS keys or AWS CLI
- One repository-writing host per stack/environment pair

## Security model

The repository is intentionally passwordless (restic `--insecure-no-password`
on every command). **Any principal that can read the S3 repository objects can
read every backup.** S3 read authorization is the only confidentiality control
for uploads that may contain private plugin data or protected media.

The Uptime Kuma push URL can contain a monitor token, so the systemd
environment file is root-only. systemd still injects it into a process running
as `web_user`, and PHP-FPM runs as that same account, so it is not isolated from
`web_user`. The EC2 instance profile is host-wide; running restic as `web_user`
limits local filesystem access, not AWS credential access.

## Creating a stack/environment

Do this once per `<stack>/<env>` pair, from your workstation, before the first
provision (the acceptance checklist deliberately provisions once before step 3
to observe the documented failure):

1. Confirm the shared bucket exists and matches the [AWS contract](#aws-contract).
2. Create the instance profile for `<stack>/<env>` with the host permissions
   from the AWS contract.
3. Initialize the repository using the maintenance identity. Provisioning never
   does this; the repository must already exist:

   ```bash
   AWS_DEFAULT_REGION=REGION restic \
     -r s3:s3.REGION.amazonaws.com/BUCKET/STACK/ENV \
     --insecure-no-password \
     --option s3.storage-class=INTELLIGENT_TIERING \
     init
   ```

4. Attach the instance profile to the EC2 host.
5. Configure the environment in Trellis (see [Install in Trellis](#install-in-trellis)).
6. Provision. The role fails the provision if the repository is absent or
   unreachable; fix the cause and re-run.

## Install in Trellis

Add the role to `galaxy.yml`, pinned to a published tag:

```yaml
roles:
  # Existing Trellis roles...
  - name: restic_backups
    src: git@github.com:spark451inc/trellis-restic-backups.git
    scm: git
    version: v1.0.0
```

Add it to `server.yml` immediately after `wordpress-setup`; do not add it to
`dev.yml`. Preserve the entry when merging future Trellis upgrades:

```yaml
roles:
  # Existing Trellis roles...
  - { role: wordpress-setup, tags: [wordpress, wordpress-setup, letsencrypt] }
  - { role: restic_backups, tags: [restic, backups] }
```

Configure each environment, for example in
`group_vars/production/restic_backups.yml`:

```yaml
restic_backup_s3_bucket: example-restic-backups
restic_backup_s3_region: us-east-1
restic_backup_stack_name: sparksites
```

With these values production uses repository
`s3:s3.us-east-1.amazonaws.com/example-restic-backups/sparksites/production`
and restic host `sparksites-production`. Both must remain stable to preserve
snapshot lineage, and only one host may write a given stack/environment
repository.

To stop backups on a host, remove or condition the role in `server.yml`; to
suspend them temporarily, run `sudo systemctl disable --now
restic-backup.timer` (a later provision re-enables it).

## Provisioning

Every provision runs `restic cat config` as `web_user` against the repository.
Any error (absent repository, authentication, network) stops provisioning. The
role never initializes a repository and never runs a backup during
provisioning.

After the first provision, start and watch the first backup manually. It may
take much longer than later incremental runs; if it exceeds the five-hour
timeout, already uploaded data stays in the repository and the next scheduled
run resumes from it:

```bash
sudo systemctl start restic-backup.service
sudo journalctl -fu restic-backup.service
```

## Role variables

Defaults are in [`defaults/main.yml`](defaults/main.yml).

| Variable | Default | Purpose |
|---|---:|---|
| `restic_backup_version` | `"0.19.1"` | Pinned restic release |
| `restic_backup_checksums` | Architecture map | SHA-256 checksums for the official `.bz2` artifacts |
| `restic_backup_s3_bucket` | `""` | Existing bucket name, without `s3://`; required |
| `restic_backup_s3_region` | `""` | AWS region; required |
| `restic_backup_stack_name` | `""` | Stable stack identifier used in the repository prefix and host; required |
| `restic_backup_excluded_sites` | `[]` | `wordpress_sites` keys to omit |
| `restic_backup_excludes` | `[]` | restic exclude patterns relative to each site's uploads directory |
| `restic_backup_uptime_kuma_push_url` | `""` | Optional Kuma push URL; the role replaces its status/message query values |

The derived `restic_backup_repository`, `restic_backup_host`, and
`restic_backup_environment` values are the single source for the repository
URL, host, cache directory, and `AWS_DEFAULT_REGION` used by provisioning and
the service. Do not override them.

### Sites and exclusions

Each selected site is backed up from `{{ www_root }}/<wordpress_sites key>/shared/uploads`.

```yaml
restic_backup_excluded_sites:
  - retired.example.com

restic_backup_excludes:
  - cache/**
  - "**/*.tmp"
```

Exclusion patterns use restic syntax. The same list applies to every selected
site; each pattern is anchored at that site's uploads root, so `cache/**`
matches only a top-level `cache` directory in each site. `*` never crosses a
`/`; use `**/` to match at any depth. Per-site exclusions are not supported.
Excluded site keys must exist in `wordpress_sites`, and at least one site must
remain selected. The role never creates or changes ownership of a source
directory.

Until a site's first deploy its uploads path does not exist; restic skips it
with a warning, exits 3, and Kuma shows down. Deploy the site.

## Runtime behavior

`restic-backup.timer` starts `restic-backup.service` at 00:00, 06:00, 12:00,
and 18:00 server local time with `Persistent=true` and a five-hour
`TimeoutStartSec`. Provisioning enables or restarts the timer; because of
`Persistent=true`, that starts the service immediately only if a scheduled run
was missed while the timer was stopped.

The oneshot service runs as `web_user:web_group` with `Nice=10`, best-effort
I/O priority 7, `RESTIC_CACHE_DIR=/var/cache/restic-backups`, a private
temporary directory, and a read-only system view except for its cache. systemd
prevents overlapping runs of the service, and restic locks the repository
against concurrent maintenance.

All selected uploads directories are passed to one `restic backup` with
`--group-by host` and `--skip-if-unchanged`. Restic's exit status is returned
unchanged and any nonzero status, including partial-backup status 3, reports
`down` to Kuma. On timeout, systemd terminates restic and the wrapper reports
the failure to Kuma before exiting. Kuma receives `up` after both a new
snapshot and a successful unchanged run; a failed Kuma request is logged but
never changes the result.

Always run backups through systemd so the root-only environment and service
limits apply; do not invoke `/usr/local/sbin/restic-backup` directly:

```bash
sudo systemctl start restic-backup.service
sudo systemctl status restic-backup.service
sudo journalctl -u restic-backup.service
sudo systemctl list-timers restic-backup.timer
```

Set the Kuma heartbeat interval to six hours plus the observed maximum normal
run time; allow extra time for the initial backup.

## AWS contract

AWS infrastructure is intentionally outside this role. A future CloudFormation
or OpenTofu implementation should supply the following.

Shared bucket:

- S3 Block Public Access, SSE-S3 default encryption, and versioning enabled
- Noncurrent object versions expired after 180 days. restic never overwrites a
  live object, so noncurrent versions come only from lock churn, `prune`, or
  an accident; this window is the only undo for a bad retention policy or an
  overwritten pack
- S3 Intelligent-Tiering without the optional Archive Access tiers
- Incomplete multipart uploads aborted and expired delete markers removed
- No lifecycle expiration of live restic objects; no Object Lock initially

Every repository write in this role specifies
`s3.storage-class=INTELLIGENT_TIERING` and supplies the region via
`AWS_DEFAULT_REGION`. External write operations, including `init`, `forget`,
`prune`, and `repack`, must do the same.

Per-stack/environment instance profile (the host), granting only:

- `s3:ListBucket` on the shared bucket, scoped by `s3:prefix` to
  `<stack>/<env>` and `<stack>/<env>/*`
- `s3:GetObject` and `s3:PutObject` on `<stack>/<env>/*`
- `s3:DeleteObject` only on `<stack>/<env>/locks/*`

Do not grant `s3:DeleteObjectVersion` or install static AWS keys. The limited
delete permission allows restic's normal locking but not retention/prune.
`PutObject` can still overwrite a current key; versioning provides the
recovery window rather than preventing that, and this residual risk is
accepted.

Maintenance identity (the operator, from a workstation), granting:

- `s3:ListBucket` and `s3:ListBucketVersions` on the shared bucket, scoped by
  `s3:prefix` to `<stack>/<env>` and `<stack>/<env>/*`
- `s3:GetObject`, `s3:GetObjectVersion`, `s3:PutObject`, and
  `s3:DeleteObject` on `<stack>/<env>/*`

It is used for `init`, retention, integrity checks, restores, and recovery. To
recover from an overwritten or corrupted object, restore the noncurrent
versions under `<stack>/<env>/` with this identity, then run `restic check`.

## External maintenance and recovery

Retention, integrity checks, and restores are operator workflows using the
maintenance identity. For every maintenance command:

1. Stop `restic-backup.timer`.
2. Wait for any active `restic-backup.service` run to finish; do not terminate
   a healthy backup.
3. Run `restic unlock` to remove stale locks left by an interrupted run. It
   only removes locks that are stale; a live run refreshes its lock every few
   minutes.
4. Run the maintenance command.
5. Always restart `restic-backup.timer`, even after a failure (use a shell
   trap or equivalent).

Never delete live S3 repository objects directly.

```bash
export RESTIC_REPOSITORY='s3:s3.REGION.amazonaws.com/BUCKET/STACK/ENV'
export AWS_DEFAULT_REGION=REGION

restic_common=(
  restic
  --insecure-no-password
  --option s3.storage-class=INTELLIGENT_TIERING
)

"${restic_common[@]}" unlock
```

### Monthly retention

Dry-run, apply, then prune only after `forget` succeeds. The default retention
contract is 8 recent, 14 daily, 8 weekly, 12 monthly, and 3 yearly snapshots,
grouped by host:

```bash
"${restic_common[@]}" forget --group-by host \
  --keep-last 8 --keep-daily 14 --keep-weekly 8 --keep-monthly 12 --keep-yearly 3 --dry-run

"${restic_common[@]}" forget --group-by host \
  --keep-last 8 --keep-daily 14 --keep-weekly 8 --keep-monthly 12 --keep-yearly 3

"${restic_common[@]}" prune
```

### Weekly integrity check

Run `restic check` at least weekly with sampled data reads, using the same
stop-timer, wait, unlock, run, always-restart procedure so its exclusive lock
cannot collide with a backup:

```bash
"${restic_common[@]}" check --read-data-subset=5%
```

Investigate failures quickly enough to stay within the 180-day noncurrent
version recovery window.

### Quarterly staged restore

Restore one site to a temporary target, verify the result, and only then plan
a separate production recovery. This role intentionally provides no in-place
production restore helper.

```bash
restore_target="$(mktemp -d)"
"${restic_common[@]}" snapshots --host STACK-ENV
"${restic_common[@]}" ls latest --host STACK-ENV /srv/www/SITE/shared/uploads
"${restic_common[@]}" restore latest --host STACK-ENV \
  --include /srv/www/SITE/shared/uploads --target "$restore_target"
```

Omit `--include` to restore every site.

## Future work

Maintenance is manual today. A scheduled off-host job (for example an ECS
Fargate task) running the maintenance identity could automate `unlock`,
`check`, `forget`, and `prune`. Before building it:

- It must run `restic unlock` first and use `--retry-lock`, and the backup
  wrapper must gain the same, so a killed job cannot leave an exclusive lock
  that blocks every backup and a running backup cannot fail the job. That
  change also retires the stop-timer procedure above.
- Its restic version must be pinned together with `restic_backup_version`.
- The 180-day noncurrent retention is a prerequisite for any automated `prune`.

## Manual acceptance checklist

Complete this checklist against staging before production:

- [ ] Provision a host against an absent repository; confirm provisioning
      fails at the repository check with a clear message and no backup runs.
- [ ] Initialize the repository from the workstation, provision again, and
      confirm success; run a third provision and confirm idempotence.
- [ ] Manually start and observe the complete first backup in journald.
- [ ] Verify one snapshot contains every expected uploads root and that the
      snapshot's data objects and the newest lock version written by
      provisioning use `INTELLIGENT_TIERING`.
- [ ] Run again unchanged and confirm no snapshot is created; change a test
      file and confirm the next run creates a snapshot.
- [ ] Exercise a site opt-out and a global exclusion; confirm the opted-out
      site is absent, the excluded path is absent from every selected site, and
      a similarly named path at a different depth is still present.
- [ ] Confirm a not-yet-deployed site produces a restic warning, status 3, and
      Kuma down, and that the run succeeds after the site's first deploy.
- [ ] Confirm timeout termination reports Kuma down.
- [ ] Confirm Kuma success, backup failure, and Kuma endpoint failure behavior.
- [ ] Dry-run the documented retention policy using the maintenance identity.
- [ ] Complete and verify a staged single-site restore to a temporary target.

## Releases

Publish immutable semantic tags such as `v1.0.0` only after the acceptance
checklist has passed on staging, and pin Trellis `galaxy.yml` to that tag.
