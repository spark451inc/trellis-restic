# Trellis Restic Backups

A standalone Ansible role that installs restic and schedules backups of uploads
for remote [Roots Trellis](https://roots.io/trellis/) WordPress servers. It
creates one snapshot containing every available site's uploads directory and
stores the repository in an existing Amazon S3 bucket.

## Features

- Installs the official restic 0.19.1 Linux binary after verifying its pinned
  compressed-artifact SHA-256 checksum
- Supports Trellis on Ubuntu 24.04 for `x86_64` and `aarch64`
- Uses the host's EC2 instance profile; no static AWS keys or AWS CLI
- Creates one repository at `<stack>/<env>` and one snapshot per run across all
  selected sites
- Skips not-yet-created uploads paths while failing unreadable or invalid paths
- Supports site opt-outs, absolute path overrides, and site-scoped exclusions
- Runs four times daily through a hardened systemd service and timer
- Prevents overlapping runs with a nonblocking lock
- Optionally pushes success and failure status to Uptime Kuma
- Requires an explicit, one-time repository initialization action

Database backups are not included. Configure and verify a separate database
backup system.

## Requirements

- Trellis 1.31 or newer
- Ubuntu 24.04
- Ansible 2.10 or newer
- An existing S3 bucket and the EC2 instance-profile permissions described in
  [AWS contract](#aws-contract)
- One repository-writing host per stack/environment pair

The role consumes Trellis's existing `env`, `web_user`, `web_group`,
`www_root`, and `wordpress_sites` variables. Do not redefine them for this
role.

## Security model

This design deliberately uses restic's `--insecure-no-password` mode. Set
`restic_backup_insecure_no_password: true` only after accepting both
consequences:

1. **Any principal that can read the S3 repository objects can open the
   repository and read every backup.** There is no restic password or
   encryption key providing a second access boundary.
2. Uploads may contain private plugin data, protected media, or other
   non-public files. Passwordless operation does not make S3 objects public,
   but it makes S3 read authorization the only confidentiality control.

The systemd environment file is root-readable because an Uptime Kuma URL can
contain a monitor token. systemd injects that value into a process running as
`web_user`; it is therefore not isolated from that Unix account.

The EC2 instance profile is host-wide. Running restic as `web_user` limits
local filesystem access, not AWS credential access. A compromised PHP process
on the same instance can obtain and use the instance profile regardless of the
backup service's Unix user.

## Install in Trellis

### Add the private role

Add the role under `roles:` in `galaxy.yml`, pinned to a published semantic
release:

```yaml
roles:
  # Existing Trellis roles...
  - name: restic_backups
    src: git@github.com:spark451inc/trellis-restic-backups.git
    scm: git
    version: v1.0.0
```

The machine running Trellis must have SSH access to the private repository.
Update the pinned tag deliberately when adopting a later release.

### Add the server role

In `server.yml`, add the role immediately after `wordpress-setup`:

```yaml
roles:
  # Existing Trellis roles...
  - { role: wordpress-setup, tags: [wordpress, wordpress-setup, letsencrypt] }
  - { role: restic_backups, tags: [restic, backups] }
```

Do not add this role to `dev.yml`. Staging and production are the natural
targets. Preserve the `server.yml` entry when merging future Trellis upgrades.

### Configure each environment

Create a role variables file such as
`group_vars/production/restic_backups.yml`:

```yaml
restic_backup_s3_bucket: example-restic-backups
restic_backup_s3_region: us-east-1
restic_backup_stack_name: sparksites

# Required acknowledgment; read the security model above.
restic_backup_insecure_no_password: true
```

Configure staging independently. With the example values, production uses:

```text
Repository: s3:s3.us-east-1.amazonaws.com/example-restic-backups/sparksites/production
RESTIC_HOST: sparksites-production
S3 prefix:  sparksites/production
```

The `<stack>/<env>` prefix and `<stack>-<env>` host must remain stable to
preserve snapshot lineage. Only one host may write a given stack/environment
repository. Set `restic_backup_enabled: false` to stop and disable this role's
timer for a host without removing its configuration.

## Initialize once

Normal provisioning probes the repository as `web_user` with `restic cat
config --no-cache`. An existing repository is unchanged. A missing repository
stops provisioning with instructions rather than initializing implicitly.
Authentication, authorization, network, and other probe errors also stop
provisioning.

Initialize a missing repository with a one-time Trellis CLI extra variable:

```bash
trellis provision production \
  --tags restic \
  --extra-vars restic_backup_initialize=true
```

Never commit `restic_backup_initialize: true` to group variables. It is an
operator action, not persistent configuration. Initialization does not run a
backup. Subsequent provisions are harmless even if the one-time flag is
accidentally supplied again because the role only initializes after an exact
"repository absent" response.

Ansible check mode skips the remote repository probe and initialization. It
still validates configuration and reports a pending binary or unit
installation without trying to use files that check mode did not create.

After provisioning, start and watch the first backup manually:

```bash
sudo systemctl start restic-backup.service
sudo journalctl -fu restic-backup.service
```

The initial backup may take substantially longer than later incremental runs.

## Role variables

Defaults are in [`defaults/main.yml`](defaults/main.yml).

| Variable | Default | Purpose |
|---|---:|---|
| `restic_backup_version` | `"0.19.1"` | Pinned restic release |
| `restic_backup_checksums` | Architecture map | SHA-256 checksums for official `.bz2` artifacts |
| `restic_backup_enabled` | `true` | Configure and schedule backups; `false` stops/disables an existing timer |
| `restic_backup_s3_bucket` | `""` | Existing bucket name, without `s3://`; required |
| `restic_backup_s3_region` | `""` | AWS region; required |
| `restic_backup_stack_name` | `""` | Stable stack identifier used in repository prefix and host; required |
| `restic_backup_insecure_no_password` | `false` | Must be explicitly set to boolean `true` |
| `restic_backup_initialize` | `false` | One-time CLI-only permission to initialize an absent repository |
| `restic_backup_excluded_sites` | `[]` | `wordpress_sites` keys to omit |
| `restic_backup_uploads_paths` | `{}` | Site-keyed absolute source path overrides |
| `restic_backup_excludes` | `[]` | Relative restic exclude patterns applied within every selected source |
| `restic_backup_site_excludes` | `{}` | Site-keyed lists of relative exclude patterns |
| `restic_backup_on_calendar` | 00:00, 06:00, 12:00, 18:00 | systemd `OnCalendar` expressions, interpreted in server local time |
| `restic_backup_randomized_delay_sec` | `30m` | Maximum systemd randomized delay |
| `restic_backup_runtime_max_sec` | `5h` | Maximum service activation time |
| `restic_backup_nice` | `10` | CPU scheduling niceness |
| `restic_backup_io_scheduling_class` | `best-effort` | systemd I/O scheduling class |
| `restic_backup_io_scheduling_priority` | `7` | Lowest best-effort I/O priority |
| `restic_backup_memory_max` | `""` | Optional systemd `MemoryMax`; empty means unlimited |
| `restic_backup_limit_upload_kib` | `0` | Optional restic upload limit in KiB/s; `0` means unlimited |
| `restic_backup_s3_connections` | `0` | Optional S3 connection count; `0` keeps restic's default |
| `restic_backup_uptime_kuma_push_url` | `""` | Optional Kuma push URL; the role replaces its status/message query values |

### Site selection and paths

The default source for a site is:

```text
{{ www_root }}/<wordpress_sites key>/shared/uploads
```

Example overrides and opt-outs:

```yaml
restic_backup_excluded_sites:
  - retired.example.com

restic_backup_uploads_paths:
  media.example.com: /mnt/media/example-uploads
```

Overrides must name the real directory, not a symlink. The role never creates
or changes ownership of a source directory.

At runtime, a source that does not exist yet is skipped because Trellis creates
`shared/uploads` during the first deploy. The site and full missing path are
written prominently to journald and included in the Kuma message. An existing
non-directory, symlink, or unreadable source fails the run. If no usable source
remains, the run fails.

### Exclusions

There are no default exclusions. Patterns use restic exclude syntax and must
be relative to the uploads root:

```yaml
restic_backup_excludes:
  - cache/**
  - "*.tmp"

restic_backup_site_excludes:
  media.example.com:
    - generated/previews/**
```

The role expands every pattern to an absolute, source-specific pattern before
calling restic. A pattern intended for one site therefore cannot exclude a
same-named path in another site. Absolute or negated patterns, newlines, and
`..` path components are rejected.

## Runtime behavior

`restic-backup.timer` starts `restic-backup.service`; provisioning only
enables/starts or restarts the timer and never starts the backup service.
Schedule changes update the persistent timer stamp before restart so a past
newly-added schedule does not trigger a backup during provisioning.

The oneshot service runs as `web_user:web_group` with:

- `RESTIC_CACHE_DIR=/var/cache/restic-backups`, owned only by `web_user`
- low CPU priority and best-effort I/O priority 7
- read-only system and home views except for its cache and private temporary
  directory
- a five-hour `TimeoutStartSec` by default
- network-online ordering

The wrapper takes a nonblocking lock before source preflight. If another
backup holds it, the second invocation exits successfully without sending a
false Kuma failure. All usable paths are written to a temporary
`--files-from-verbatim` file and passed to one `restic backup` invocation.
Snapshots use stable `--group-by host`, `--skip-if-unchanged`, and
`stack:<name>`, `env:<env>`, and `uploads` tags.

Restic's status is returned unchanged, including partial-backup status 3.
Systemd termination is forwarded, and the TERM path sends Kuma a down status.
Kuma receives up after both a newly saved snapshot and a successful unchanged
run. A failed Kuma request is logged but never changes the backup result.

Do not invoke `/usr/local/sbin/restic-backup` directly. Manual operators should
use systemd so the root-only environment and service limits are present:

```bash
sudo systemctl start restic-backup.service
sudo systemctl status restic-backup.service
sudo journalctl -u restic-backup.service
sudo systemctl list-timers restic-backup.timer
```

For Kuma, set the heartbeat interval to:

```text
6 hours + configured random delay + observed maximum normal run time
```

Do not assume a fixed 7.5-hour heartbeat. Measure normal operation, and allow
extra time for the initial backup.

## AWS contract

AWS infrastructure is intentionally outside this role. A future
CloudFormation or OpenTofu implementation should supply the following.

### Shared bucket

- S3 Block Public Access enabled
- SSE-S3 default encryption
- Versioning enabled
- Noncurrent object versions expired after 30 days
- S3 Intelligent-Tiering, without optional asynchronous Archive Access or Deep
  Archive Access tiers
- Incomplete multipart uploads aborted
- Expired delete markers removed
- No lifecycle expiration of live restic objects
- No Object Lock initially

Every repository write in this role specifies
`s3.storage-class=INTELLIGENT_TIERING` and `s3.region=<region>`. External
write operations, including future `forget`, `prune`, and `repack` commands,
must specify both options as well.

### Per-stack/environment instance profile

Grant only:

- `s3:ListBucket` on the shared bucket, scoped by `s3:prefix` to
  `<stack>/<env>` and `<stack>/<env>/*`
- `s3:GetObject` and `s3:PutObject` on `<stack>/<env>/*`
- `s3:DeleteObject` only on `<stack>/<env>/locks/*`

Do not grant `s3:DeleteObjectVersion`. Do not install static AWS keys.

The limited delete permission allows restic's normal repository locking but
does not authorize retention/prune. `PutObject` can still overwrite a current
key; versioning provides a 30-day recovery window rather than preventing that
action. This residual overwrite/versioning risk is accepted.

## External maintenance and recovery

Retention, integrity checks, and restores are privileged operator workflows,
not unattended tasks in this role. Use an external maintenance identity with
the additional permissions needed for the operation.

### Monthly retention

At least monthly:

1. Stop `restic-backup.timer`.
2. Wait for any active `restic-backup.service` run to finish. Do not terminate
   a healthy backup merely to begin maintenance.
3. From the privileged maintenance context, dry-run and then apply the
   retention policy.
4. Prune only after `forget` succeeds.
5. Always restart `restic-backup.timer`, including after a failed maintenance
   command.

The default retention contract is 8 recent, 14 daily, 8 weekly, 12 monthly,
and 3 yearly snapshots, grouped by host:

```bash
export RESTIC_REPOSITORY='s3:s3.REGION.amazonaws.com/BUCKET/STACK/ENV'

restic_common=(
  /usr/local/bin/restic
  --insecure-no-password
  --option s3.region=REGION
  --option s3.storage-class=INTELLIGENT_TIERING
)

"${restic_common[@]}" forget \
  --group-by host \
  --keep-last 8 \
  --keep-daily 14 \
  --keep-weekly 8 \
  --keep-monthly 12 \
  --keep-yearly 3 \
  --dry-run

"${restic_common[@]}" forget \
  --group-by host \
  --keep-last 8 \
  --keep-daily 14 \
  --keep-weekly 8 \
  --keep-monthly 12 \
  --keep-yearly 3

"${restic_common[@]}" prune
```

Use a shell trap or equivalent operational control to ensure the timer is
restarted. Never delete live S3 repository objects directly.

### Integrity and restore

- Run external `restic check` at least weekly, including sampled data reads
  such as `--read-data-subset=5%`. Follow the same stop-timer, wait, run, and
  always-restart procedure used for retention so its exclusive lock cannot
  collide with a backup. Use the same passwordless, region, and storage-class
  options.
- Investigate failures quickly enough to remain within the 30-day noncurrent
  version recovery window.
- Perform a staged test restore at least quarterly.
- Restore to a temporary target, verify the result, and only then plan a
  separate production recovery. This role intentionally provides no in-place
  production restore helper.

Example staged restore:

```bash
restore_target="$(mktemp -d)"
"${restic_common[@]}" snapshots --host STACK-ENV
"${restic_common[@]}" restore latest \
  --host STACK-ENV \
  --target "$restore_target"
```

## Manual acceptance checklist

Complete this checklist against staging before production:

- [ ] Run Ansible check mode successfully.
- [ ] Confirm normal provisioning refuses an absent repository.
- [ ] Initialize once with the explicit CLI extra variable; confirm no backup
      runs during provisioning.
- [ ] Run a second provision and confirm idempotence.
- [ ] Manually start and observe the complete first backup in journald.
- [ ] Verify one snapshot contains every expected uploads root.
- [ ] Verify newly written S3 objects use `INTELLIGENT_TIERING`.
- [ ] Run again unchanged and confirm no snapshot is created; change a test
      file and confirm the next run creates a snapshot.
- [ ] Exercise a site opt-out, path override, global exclusion, and per-site
      exclusion without affecting another site's same-named path.
- [ ] Confirm a missing source is named and skipped.
- [ ] Confirm an unreadable source and zero usable sources each fail.
- [ ] Confirm timeout termination reports Kuma down and overlapping starts do
      not produce a false down notification.
- [ ] Confirm Kuma success, backup failure, and Kuma endpoint failure behavior.
- [ ] Confirm stack/environment host and repository lineage remain stable
      across source-set changes.
- [ ] Dry-run the documented retention policy using the maintenance identity.
- [ ] Complete and verify a staged restore to a temporary target.

## Releases and license

Publish immutable semantic tags such as `v1.0.0` and pin Trellis `galaxy.yml`
to a tested tag.

This role is licensed under [GPL-2.0-or-later](LICENSE).
