# Multi-Report and Drive-Selftest — Linux fork

This is a fork of [JoeSchmuck/Multi-Report](https://github.com/JoeSchmuck/Multi-Report),
which was written for TrueNAS CORE and SCALE. The original scripts, report design and
documentation are **Joe Schmuck's** (joeschmuck), with contributions from Bidule0hm,
melp, toomuchdata, Basil Hendroff and Oxyde. This fork keeps that work intact and adds
support for running the same scripts on a plain Linux host.

- Upstream: https://github.com/JoeSchmuck/Multi-Report
- This fork: https://github.com/arrrghhh/Multi-Report

## What the scripts do

`multi_report.sh` builds a status report for your ZFS pools and drives — health, SMART
attributes, temperatures, test history — and emails it to you.

`drive_selftest.sh` is the companion scheduler that runs SMART short and long
self-tests on a routine basis. It can run standalone without Multi-Report.

## What this fork adds

- **Runs on generic Linux.** TrueNAS is detected at runtime (`/etc/version` plus
  `/data/freenas-v1.db`); on TrueNAS the original code paths run unchanged.
- **No hard TrueNAS dependencies on Linux.** `midclt`, `nvme` and `sqlite3` are used
  only when present. The config backup, `-tardrivedata` and the disk-layout editor
  remain TrueNAS-only.
- **Drive list from `smartctl --scan-open`**, with per-drive `-d` probing because the
  scan reports `-d scsi` for most SATA disks, which cannot read SMART.
- **Long tests stay out of the way of a scrub.** By default a colliding long test is
  replaced by a short test; `SCRUB_Minutes_Remaining="0"` suspends testing entirely
  for drives in a scrubbing pool.
- **Updates come from this fork**, fetching the `.sh` files directly instead of the
  `.txt` copies.
- **Three-part versions** (`MAJOR.MINOR.FORK`) so fork-only fixes reach installed
  copies.

Only Ubuntu 26.04 LTS has been tested. Other distributions should work, but that is a
design intent, not a claim.

## Install

```sh
mkdir -p /opt/Multi-Report
install -m 755 multi_report.sh drive_selftest.sh /opt/Multi-Report/
cd /opt/Multi-Report
./multi_report.sh -config     # create multi_report_config.txt
./multi_report.sh             # run a report
```

`drive_selftest.sh` can be run by hand:

```sh
./drive_selftest.sh -demo     # show what it would do, change nothing
./drive_selftest.sh -no_test  # full pass with tests skipped
./drive_selftest.sh           # normal run
./drive_selftest.sh -help     # list switches
```

Both scripts keep their config, logs and statistics database in their own directory, so
keep them together.

### Requirements

`bash`, `curl`, `jq`, `smartmontools`, `zfsutils-linux`, `util-linux` (`lsblk`), and a
local mail transport (sendmail, msmtp or equivalent). Run as root — `smartctl` needs it
for most attributes.

### Scheduling

```
# Multi-Report daily at 04:10
10 4 * * *  root  cd /opt/Multi-Report && ./multi_report.sh > /dev/null 2>&1

# Drive-Selftest daily at 03:10; it decides which drives are due
10 3 * * *  root  cd /opt/Multi-Report && ./drive_selftest.sh > /dev/null 2>&1
```

For long tests, one drive per week rotating through the array is a reasonable default:

```
Long_Test_Mode="1"
Long_Drives_Test_Period="Month"
Long_Drives_to_Test_Per_Day="1"
Long_Drives_Tested_Days_of_the_Week="1"
```

Pick a weekday that does not collide with your scrub. On Ubuntu, `zfsutils-linux`
scrubs on the second Sunday of the month, so Monday is a safe choice.

## Updating

```
./multi_report.sh -update           # Multi-Report
./multi_report.sh -update_selftest  # Drive-Selftest
./multi_report.sh -update_all       # everything
```

Third-party helpers (`smr-check.sh`, `sendemail.py`, `render_case.py`) still come from
their own upstream projects. `main` is the released branch; development happens on
`linux`.

## More documentation

- `LINUX.md` — porting notes, platform detection, merge workflow
- `Multi_Report_User_Guide.pdf`, `Multi_Report_Quick_Start_Guide.pdf`,
  `Drive_Selftest_User_Guide.pdf` — upstream documentation, still applies
- `multi_report.txt` and `drive_selftest.txt` are upstream's TrueNAS copies. Ignore
  them on Linux; use the `.sh` files.

If you are on TrueNAS, use upstream. This fork exists for people running the same
hardware and pools on a plain Linux distribution.
