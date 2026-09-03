# Trellis Restic Backups

A standalone Ansible role that installs restic and schedules backups of uploads
for remote [Roots Trellis](https://roots.io/trellis/) WordPress servers. Each
run creates one snapshot containing every available site's uploads directory
and stores it in an existing Amazon S3 bucket under `<stack>/<env>`.

Database backups are not included. Configure and verify a separate database
backup system.

## Requirements

- Trellis 1.31 or newer on Ubuntu 24.04 (`x86_64` or `aarch64`); the role
  relies on Trellis's `env`, `web_user`, `web_group`, `www_root`,
  `wordpress_sites`, `apt_cache_valid_time`, and the `curl` package installed
  by the `common` role
- Ansible 2.10 or newer
- An existing S3 bucket and the EC2 instance-profile permissions described in
  [AWS contract](#aws-contract); no static AWS keys or AWS CLI
- One repository-writing host per stack/environment pair

## Security model

This design deliberately uses restic's `--insecure-no-password` mode. Set
`restic_backup_insecure_no_password: true` only after accepting that **any
principal that can read the S3 repository objects can read every backup**.
There is no restic password providing a second boundary, so S3 read
authorization is the only confidentiality control for uploads that may contain
private plugin data or protected media.

The Uptime Kuma push URL can contain a monitor token, so the systemd
environment file is root-only. systemd still injects it into a process running
as `web_user`, and PHP-FPM runs as that same account, so it is not isolated from
`web_user`. The EC2 instance profile is host-wide; running restic as `web_user`
limits local filesystem access, not AWS credential access.

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

# Required acknowledgment; read the security model above.
restic_backup_insecure_no_password: true
```

With these values production uses repository
`s3:s3.us-east-1.amazonaws.com/example-restic-backups/sparksites/production`
and restic host `sparksites-production`. Both must remain stable to preserve
snapshot lineage, and only one host may write a given stack/environment
repository. Set `restic_backup_enabled: false` to stop and disable the timer on
a host without removing its configuration.

## Initialize once

Every provision probes the repository as `web_user` with `restic cat config`.
An existing repository is left unchanged. A missing repository stops
provisioning with instructions instead of initializing implicitly; other probe
errors (authentication, network) also stop provisioning.

Initialize a missing repository with a one-time extra variable:

```bash
trellis provision production --tags restic --extra-vars restic_backup_initialize=true
```

Never commit `restic_backup_initialize: true` to group variables.
Initialization does not run a backup, and re-supplying the flag later is
harmless because the role only initializes after an exact "repository absent"
response.

Ansible check mode skips the remote probe and initialization. It still
validates configuration and reports a pending binary or unit installation.

After provisioning, start and watch the first backup manually. It may take
much longer than later incremental runs; if it exceeds the five-hour timeout,
already uploaded data stays in the repository and the next scheduled run
resumes from it:

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
| `restic_backup_enabled` | `true` | `false` stops and disables an existing timer |
| `restic_backup_s3_bucket` | `""` | Existing bucket name, without `s3://`; required |
| `restic_backup_s3_region` | `""` | AWS region; required |
| `restic_backup_stack_name` | `""` | Stable stack identifier used in the repository prefix and host; required |
| `restic_backup_insecure_no_password` | `false` | Must be explicitly set to `true` |
| `restic_backup_initialize` | `false` | One-time CLI-only permission to initialize an absent repository |
| `restic_backup_excluded_sites` | `[]` | `wordpress_sites` keys to omit |
| `restic_backup_uploads_paths` | `{}` | Site-keyed absolute source path overrides |
| `restic_backup_excludes` | `[]` | Relative restic exclude patterns applied within every selected source |
| `restic_backup_site_excludes` | `{}` | Site-keyed lists of relative exclude patterns |
| `restic_backup_on_calendar` | `["*-*-* 00/6:00:00"]` | systemd `OnCalendar` expressions (server local time) |
| `restic_backup_randomized_delay_sec` | `30m` | Maximum systemd randomized delay |
| `restic_backup_runtime_max_sec` | `5h` | Maximum service activation time |
| `restic_backup_nice` | `10` | CPU scheduling niceness |
| `restic_backup_io_scheduling_class` | `best-effort` | systemd I/O scheduling class |
| `restic_backup_io_scheduling_priority` | `7` | Lowest best-effort I/O priority |
| `restic_backup_memory_max` | `""` | Optional systemd `MemoryMax`; empty means unlimited |
| `restic_backup_limit_upload_kib` | `0` | Optional restic upload limit in KiB/s; `0` means unlimited |
| `restic_backup_s3_connections` | `0` | Optional S3 connection count; `0` keeps restic's default |
| `restic_backup_uptime_kuma_push_url` | `""` | Optional Kuma push URL; the role replaces its status/message query values |

The derived `restic_backup_repository`, `restic_backup_host`, and
`restic_backup_environment` values are the single source for the repository
URL, host, cache directory, and `AWS_DEFAULT_REGION` used by provisioning and
the service. Do not override them.

### Sites, paths, and exclusions

The default source for a site is `{{ www_root }}/<wordpress_sites key>/shared/uploads`.

```yaml
restic_backup_excluded_sites:
  - retired.example.com

restic_backup_uploads_paths:
  media.example.com: /mnt/media/example-uploads

restic_backup_excludes:
  - cache/**
  - "*.tmp"

restic_backup_site_excludes:
  media.example.com:
    - generated/previews/**
```

Overrides must name the real directory, not a symlink; the role never creates
or changes ownership of a source directory. Exclusion patterns use restic
syntax relative to the uploads root and are expanded to absolute,
source-specific patterns, so a pattern for one site cannot exclude a same-named
path in another site. Site keys must exist in `wordpress_sites`, and at least
one site must remain enabled.

At runtime a source that does not exist yet is skipped, because Trellis creates
`shared/uploads` during the first deploy. Skipped sites are logged with their
full path in journald and listed by name in the Kuma message. An existing
non-directory or symlink source fails the run, an unreadable source fails
through restic's own status 3, and if no usable source remains the run fails.

## Runtime behavior

`restic-backup.timer` starts `restic-backup.service` every six hours with up to
30 minutes of random delay, `Persistent=true`, and a five-hour
`TimeoutStartSec`. Provisioning only enables, starts, or restarts the timer.
Because the timer is persistent, changing the schedule may trigger one
catch-up backup shortly after provisioning; this is harmless.

The oneshot service runs as `web_user:web_group` with low CPU priority,
best-effort I/O priority 7, `RESTIC_CACHE_DIR=/var/cache/restic-backups`, and
read-only system and home views except for its cache and private temporary
directory.

The wrapper takes a nonblocking lock, so a second invocation exits successfully
without sending a false Kuma failure. Sites, paths, and exclusions are rendered
into the wrapper at provision time; all usable paths are passed to one
`restic backup` with `--group-by host` and `--skip-if-unchanged`. Restic's exit
status is returned unchanged, including partial-backup status 3. On timeout,
systemd terminates restic, and the wrapper reports the failure to Kuma before
exiting. Kuma receives `up` after both a new snapshot and a successful
unchanged run; a failed Kuma request is logged but never changes the result.

Use systemd rather than invoking `/usr/local/sbin/restic-backup` directly so
the root-only environment and service limits apply:

```bash
sudo systemctl start restic-backup.service
sudo systemctl status restic-backup.service
sudo journalctl -u restic-backup.service
sudo systemctl list-timers restic-backup.timer
```

Set the Kuma heartbeat interval to six hours plus the random delay plus the
observed maximum normal run time; allow extra time for the initial backup.

## AWS contract

AWS infrastructure is intentionally outside this role. A future CloudFormation
or OpenTofu implementation should supply the following.

Shared bucket:

- S3 Block Public Access, SSE-S3 default encryption, and versioning enabled
- Noncurrent object versions expired after 30 days
- S3 Intelligent-Tiering without the optional Archive Access tiers
- Incomplete multipart uploads aborted and expired delete markers removed
- No lifecycle expiration of live restic objects; no Object Lock initially

Every repository write in this role specifies
`s3.storage-class=INTELLIGENT_TIERING`, and the region is always supplied via
`AWS_DEFAULT_REGION` because `s3:GetBucketLocation` is not granted. External
write operations, including `forget`, `prune`, and `repack`, must do the same.

Per-stack/environment instance profile, granting only:

- `s3:ListBucket` on the shared bucket, scoped by `s3:prefix` to
  `<stack>/<env>` and `<stack>/<env>/*`
- `s3:GetObject` and `s3:PutObject` on `<stack>/<env>/*`
- `s3:DeleteObject` only on `<stack>/<env>/locks/*`

Do not grant `s3:DeleteObjectVersion` or install static AWS keys. The limited
delete permission allows restic's normal locking but not retention/prune.
`PutObject` can still overwrite a current key; versioning provides a 30-day
recovery window rather than preventing that, and this residual risk is
accepted.

## External maintenance and recovery

Retention, integrity checks, and restores are privileged operator workflows
using an external maintenance identity with the additional permissions each
operation needs. For every maintenance command:

1. Stop `restic-backup.timer`.
2. Wait for any active `restic-backup.service` run to finish; do not terminate
   a healthy backup.
3. Run the maintenance command from the privileged context.
4. Always restart `restic-backup.timer`, even after a failure (use a shell
   trap or equivalent).

Never delete live S3 repository objects directly.

```bash
export RESTIC_REPOSITORY='s3:s3.REGION.amazonaws.com/BUCKET/STACK/ENV'
export AWS_DEFAULT_REGION=REGION

restic_common=(
  /usr/local/bin/restic
  --insecure-no-password
  --option s3.storage-class=INTELLIGENT_TIERING
)
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
stop-timer, wait, run, always-restart procedure so its exclusive lock cannot
collide with a backup:

```bash
"${restic_common[@]}" check --read-data-subset=5%
```

Investigate failures quickly enough to stay within the 30-day noncurrent
version recovery window.

### Quarterly staged restore

Restore to a temporary target, verify the result, and only then plan a
separate production recovery. This role intentionally provides no in-place
production restore helper.

```bash
restore_target="$(mktemp -d)"
"${restic_common[@]}" snapshots --host STACK-ENV
"${restic_common[@]}" restore latest --host STACK-ENV --target "$restore_target"
```

## Manual acceptance checklist

Complete this checklist against staging before production:

- [ ] Run Ansible check mode successfully on a fresh and on a provisioned host.
- [ ] Confirm normal provisioning refuses an absent repository.
- [ ] Initialize once with the explicit CLI extra variable; confirm no backup
      runs during provisioning.
- [ ] Run a second provision and confirm idempotence.
- [ ] Manually start and observe the complete first backup in journald.
- [ ] Verify one snapshot contains every expected uploads root and that new S3
      objects use `INTELLIGENT_TIERING`.
- [ ] Run again unchanged and confirm no snapshot is created; change a test
      file and confirm the next run creates a snapshot.
- [ ] Exercise a site opt-out, path override, global exclusion, and per-site
      exclusion without affecting another site's same-named path.
- [ ] Confirm a missing source is named and skipped, and that an unreadable
      source and zero usable sources each fail.
- [ ] Confirm timeout termination reports Kuma down and overlapping starts do
      not produce a false down notification.
- [ ] Confirm Kuma success, backup failure, and Kuma endpoint failure behavior.
- [ ] Confirm `restic_backup_enabled: false` stops and disables the timer.
- [ ] Dry-run the documented retention policy using the maintenance identity.
- [ ] Complete and verify a staged restore to a temporary target.

## Releases

Publish immutable semantic tags such as `v1.0.0` and pin Trellis `galaxy.yml`
to a tested tag.
