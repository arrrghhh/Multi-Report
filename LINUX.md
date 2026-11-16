# Linux port notes

This file documents how the Linux support in this fork works, what it replaces, and
where the sharp edges are. It is for anyone maintaining the fork or debugging it on a
new machine. Day-to-day usage is in `README.md`.

Everything here is scoped to `multi_report.sh` and `drive_selftest.sh`. Upstream's
PDFs still describe the configuration options and the report itself.

## Platform detection

Both scripts use the same test, so they always agree on what "TrueNAS" means:

```sh
if [[ -f /etc/version ]] && [[ -f /data/freenas-v1.db ]]; then
  is_truenas="true"
else
  is_truenas="false"
fi
```

`/etc/version` alone is not enough — plenty of Linux boxes have unrelated files with
that name. `/data/freenas-v1.db` is the middleware configuration database, and only
TrueNAS (CORE or SCALE) has it.

The result is stored in `is_truenas` and every Linux-specific branch is guarded by it.
Where the upstream code already handled both platforms, it is left alone.

## What the Linux path replaces

| Purpose | TrueNAS path (unchanged) | Linux path (this fork) |
| --- | --- | --- |
| OS version string | `/etc/version` | `PRETTY_NAME` from `/etc/os-release` |
| OS version number | parsed from `/etc/version` | `VERSION_ID` from `/etc/os-release` |
| Memory | `midclt call system.info` → `.physmem` | `free -b` |
| Uptime | `midclt call system.info` → `.uptime` | `/proc/uptime` |
| Drive list | `midclt call disk.query` | `smartctl --scan-open` + `lsblk` |
| Pool list | `midclt call pool.query` | `zpool status -LP` |
| Drive SMART interface | value reported by middleware/scan | probed per drive, cached |
| NVMe low-level tests | `nvme` CLI on old SCALE | `smartctl` only |
| Config backup | `/data/freenas-v1.db` | not available |
| `-tardrivedata` | `midclt` drive query | not available |

`midclt`, `nvme` and `sqlite3` are therefore optional. If they exist they are used; if
not, the Linux path runs. `-tardrivedata` and the disk-layout editor remain
TrueNAS-only features and exit cleanly on Linux.

The Linux drive and pool builders emit the same JSON shape that `disk.query` and
`pool.query` return, so the rest of the scripts — which consume that JSON — needed no
changes. In `drive_selftest.sh` they are `build_drive_disk_query_linux` and
`build_pool_query_linux`. `multi_report.sh` has no equivalent builder; its Linux disk
list is read from `lsblk` in place of the old `fdisk -l` parsing.

## Drive interface detection

`smartctl --scan-open` on Linux frequently reports `-d scsi` for SATA disks attached to
a SATA controller. `-d scsi` cannot read SMART attributes, so trusting the scan output
produces a drive list that looks right and then fails every SMART call.

`resolve_drive_interface` handles this. For each device it tries a short list of
`-d` types in order until one returns real SMART data (`power_on_time.hours` or
`smart_status.passed` in the JSON output), then caches the winner in
`drive_iface_cache`:

- NVMe devices: `nvme, auto, atacam, sat, scsi`
- everything else: `sat, auto, atacam, scsi, nvme`

The fallback is `auto`. On this fork's test machine the answer is `-d sat` for all
SATA drives.

`smart_iface` is the single accessor used by the rest of the script. On TrueNAS it
returns the interface the middleware reported, preserving upstream behaviour; on Linux
it returns the probed value.

Note that `--scan-open` is still used as the *device list*. It is authoritative about
which devices exist and can be opened; it is only its `-d` suggestion that is
unreliable.

## Scrub gate

`SCRUB_Minutes_Remaining` is a threshold, not a switch. While a pool is scrubbing or
resilvering, `check_scrub` compares the remaining time against that threshold:

- Default (`60`): if the scrub has more than 60 minutes left, a pending long test is
  replaced by a short test on those drives.
- `0`: all SMART testing is disabled for drives in the pool for the duration of the
  scrub or resilver.
- `SCRUB_RESILVER_OVERRIDE="enable"`: ignores the scrub state entirely.

The Linux pool builder parses `zpool status -LP` and emits the scrub function plus
`total_secs_left` under the same field names `pool.query` uses, so `check_scrub` works
untouched. If the builder returns `null` for `total_secs_left`, the script treats the
pool as idle and tests normally.

If you change the pool builder, keep those field names. That is the contract with the
rest of the script.

## Version comparison

Versions are `MAJOR.MINOR.FORK`. Upstream compared only major and minor, which would
silently ignore a fork-only revision and never deliver it. Both scripts now call
`version_is_newer <remote> <local>`, which compares three numeric components and
treats missing components as zero.

Consequences:

- `3.34.1` → `3.34.2` is an update.
- `3.34.1` → `3.35.0` is an update.
- `3.34.1` → `3.34.0` is not.

`multi_report.sh` still strips a `_date` suffix when it reads the version out of a
filename, so `CurrentFilename` must keep the form
`multi_report_vMAJOR.MINOR.FORK_YYYY_MM_DD.sh`.

## Update source

Both scripts fetch from this fork's `main` branch:

```
https://raw.githubusercontent.com/arrrghhh/Multi-Report/refs/heads/main/multi_report.sh
https://raw.githubusercontent.com/arrrghhh/Multi-Report/refs/heads/main/drive_selftest.sh
```

Using `refs/heads/main` rather than a branch alias means renaming the development
branch cannot break already-installed copies.

The download is written to a temporary `.txt` filename (`multi_report.txt`,
`drive_selftest.txt`) because the surrounding upstream code expects those names during
the update, then copied over the `.sh` file. Fetching the `.sh` source directly is the
point: there is no second copy to drift.

Third-party downloads are unchanged and still come from their own projects:
`smr-check.sh`, `sendemail.py`, `render_case.py`.

## Configuration notes for Linux

The config file is `multi_report_config.txt`, created with `./multi_report.sh -config`.
It lives next to the scripts, as do `DS_Logs/` and the statistics database.

For long tests, one drive per week rotating through the array is a reasonable default
for a small pool:

```
Long_Test_Mode="1"
Long_Drives_Test_Period="Month"
Long_Drives_to_Test_Per_Day="1"
Long_Drives_Tested_Days_of_the_Week="1"
Maximum_Catchup_Drive_Count="1"
```

`Month` plus a single weekday means each drive gets its turn once a month, with one
drive starting per week. Choose a weekday that does not overlap the scrub window. On
Ubuntu, `/etc/cron.d/zfsutils-linux` scrubs on the second Sunday of the month, so
Monday is a safe pick.

Short tests are cheap and can stay frequent (`Short_Test_Mode="2"` with a short delay).

Keep `Automatic_*_Update="disable"` unless you want scripts to change themselves
unattended. The report will still tell you when an update exists.

## Porting conventions

Linux changes are wrapped so they are easy to find and easy to merge:

```sh
# >>> LINUX PORT
...
# <<< LINUX PORT
```

Rules followed in this fork:

1. **Guard, do not delete.** TrueNAS code stays in place behind an `is_truenas` test.
   Deleting it guarantees a conflict on the next upstream release.
2. **Keep upstream variable names.** The Linux path produces the same JSON field names
   and the same shell variables, so downstream logic does not branch.
3. **No fake platform files.** Do not write a synthetic `/etc/version`. Read the real
   machine.
4. **Leave upstream `.txt` files alone.** They are TrueNAS artifacts; editing or
   deleting them creates conflicts and serves no purpose on Linux.
5. **Test with `bash -n` before anything else.** Both scripts are large and a syntax
   error is easy to introduce in a heredoc or an `if`.

## Merging an upstream release

```sh
git fetch upstream
git log --oneline HEAD..upstream/main
git merge upstream/main
```

Then:

1. Resolve conflicts. Most should land in the marked blocks; if a conflict lands
   outside them, check whether upstream restructured something the Linux path depends
   on.
2. Bump `FORK` in the affected script and update its `CurrentFilename` date.
3. `bash -n multi_report.sh drive_selftest.sh`
4. Test on Linux: `drive_selftest.sh -demo`, `drive_selftest.sh -no_test`, then a full
   `multi_report.sh` run and an email.
5. Fast-forward `main` to the tested commit.

If upstream renames a field the Linux builder emits, fix the builder, not the
consumer — that keeps the merge surface small.

## Testing checklist

Run as root, since `smartctl` needs it for most attributes:

```sh
./drive_selftest.sh -demo      # prints what it would do, changes nothing
./drive_selftest.sh -no_test   # full pass with tests skipped
./drive_selftest.sh -help      # switches
./multi_report.sh              # full report + email
./multi_report.sh -check_for_updates
```

Expected report header on the test machine:

```
Multi-Report v3.34.1 dtd:2026-11-16 (Ubuntu 26.04.1 LTS)
```

## Known limitations

- **Only Ubuntu has been tested** (26.04 LTS). Other distributions should work; the
  likely failure points are `/etc/os-release` parsing and drive-name resolution.
- **No TrueNAS config backup on Linux.** There is no equivalent file to attach.
- **`-tardrivedata` exits 1 on Linux.** It depends on `midclt`. Use `-dump` instead if
  you want drive data collected.
- **`-d` probing costs one `smartctl` call per drive**, once per run. With very large
  drive counts that is measurable, though small next to the SMART tests themselves.
- **`jq` is required** on Linux (it is also required on TrueNAS SCALE). The scripts
  build and read JSON throughout.
- **Serial numbers may be blank** for USB bridges and some controllers. The Linux
  builder falls back from `lsblk` to `smartctl -i`, but neither is guaranteed.
