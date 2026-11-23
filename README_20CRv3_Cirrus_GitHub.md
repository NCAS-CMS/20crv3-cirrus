# 20CRv3 on Cirrus — Quick Start

**Four-cycle reference workflow · 23 November 2026 · Version 2 (final audit pending)**

This page reproduces the operational instructions and all **11 copy-and-paste command blocks** from the Version 2 Quick Start. It is intended for Ed Hawkins and the project team. Use a **Bash terminal on Cirrus**. Run one block at a time, in order; **stop at any FAIL**.

> **Safety:** Step 8 submits a real **80-node exclusive Slurm job**. Do not run it merely to test this guide. The commands are tailored to the **October 2026 reference case**, not arbitrary dates. Do not modify shared executables, science, stochastic seeds or inputs.

## Reference configuration

| Item | Reference value |
|---|---|
| Experiment | `ensda_602_2026` |
| Members / nodes | 80 / 80 |
| Assimilation interval | 6 hours |
| Initial analysis date | `2026101000` |
| Four-cycle next date | `2026101100` |
| Initial restart | 886 files, approximately 40.55 GB |
| Free space required | 290 GiB before staging; 250 GiB before submission |

The validated shared installation is:

```text
/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1
```

**Files:** The [PDF guide](docs/20CRv3_Cirrus_Quick_Start_V2_20261123.pdf), [LaTeX source](docs/20CRv3_Cirrus_Quick_Start_V2_20261123.tex), and [plain-text commands](commands/20CRv3_Cirrus_Quick_Start_Commands_V2_20261123.txt) are companion formats **once uploaded to those repository paths**. The GitHub code blocks below are the easiest way to copy commands. If using a new terminal, **repeat Step 1**.

## Step 1 — Define your own paths (B0)

Defines private paths using your Cirrus login. Use **Bash**, and choose unused directories. `DATADIR/` must be no longer than 40 characters.

**Copy, paste and run this entire block:**

```bash
SHARED=/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1
SRC_WF=$SHARED/run/coriII_home/20CRv3_scripts/ensda_602_2026
ME=$(id -un)
WORK=/epccfs/n02/n02-NEZ000203/$ME/20crv3_work/run1/ensda_602_2026
DATADIR=/epccfs/n02/n02-NEZ000203/$ME/L
DRIVER=gfsenkf_20crV3_sing_archer2_V47_cirrus_test4.csh
WF_FILES="dateutils3.py get_biascorr3.py $DRIVER incdate rewrite2.py"
WF_FILES+=" run_enkf_ps_h5_biascor_sing_archer2"
WF_FILES+=" run_fg_gfs_ens_sing_archer2 run_gefs_sa2"
DP=$DATADIR/gfsenkf_20crV3_archer2/ensda_602_2026
die() { echo "FAIL: $*" >&2; exit 1; } # called only inside parenthesised blocks
echo "PASS B0: variables defined for $ME"
```

**Expected:** `PASS B0: variables defined for <your login>`  
**PASS:** Continue to Step 2.

**FAIL:** If variables are missing, repeat Step 1 in the same Bash session.

## Step 2 — Check access, paths and free space (B1)

Verifies project membership, unused directories, observations, SST/ice files, and at least **290 GiB** available before staging.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  case "$WORK" in *[[:space:]]*|*\#*) die "WORK must not contain whitespace or #";; esac
  case "$WORK/" in "/epccfs/n02/n02-NEZ000203/$(id -un)/"*) ;; *) die "WORK must be under /epccfs/n02/n02-NEZ000203/$(id -un)/";; esac
  case "$DATADIR/" in "/epccfs/n02/n02-NEZ000203/$(id -un)/"*) ;; *) die "DATADIR must be under /epccfs/n02/n02-NEZ000203/$(id -un)/";; esac
  id -nG | tr ' ' '\n' | grep -qx n02-nez000203 || die "not in group n02-nez000203"
  [ "$(basename "$WORK")" = ensda_602_2026 ] || die "workflow directory must end in ensda_602_2026"
  [ ! -e "$WORK" ] || die "$WORK already exists"
  [ ! -e "$DATADIR" ] || die "$DATADIR already exists"
  case "$DATADIR" in *[[:space:]]*|*\#*) die "DATADIR must not contain whitespace or #";; esac
  n=$(printf '%s' "$DATADIR/" | wc -c)
  [ "$n" -le 40 ] || die "datadir is $n characters including the slash; limit is 40"
  [ "$n" -eq 37 ] || echo "NOTE: datadir is $n characters; the validated length is 37"
  [ -r "$SHARED/MANIFEST_OPERATIONAL.sha256" ] || die "cannot read the shared installation"
  [ -r "$SHARED/inputs/restart/ensda_602_2026/analdate.csh" ] || die "cannot read the start state"
  for c in 2026101000 2026101006 2026101012 2026101018; do
    [ -r "$SHARED/inputs/obs/NRT/$c/psobfile" ] || die "observations missing for $c"
  done
  [ "$(ls "$SHARED/inputs/sst/SSTsNRT_CF/2026" | wc -l)" -eq 160 ] || die "SST/ice files incomplete"
  avail=$(df -Pk "$(dirname "$DATADIR")" | awk 'NR==2{print $4}')
  [ "$avail" -ge 304087040 ] || die "less than 290 GiB free before staging under $(dirname "$DATADIR")"
  echo "PASS B1 (datadir $n characters)"
)
```

**Expected:** `PASS B1 (datadir 37 characters)` for the default layout.  
**PASS:** Continue to Step 3.

**FAIL:** Stop. Investigate access, paths, missing inputs, or insufficient free space. Existing directories are deliberately rejected.

## Step 3 — Verify shared executables and scripts (B2)

Checks SHA-256 hashes of 17 essential files in the shared installation.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  cd "$SHARED"
  RE='  \./(run/coriII_home/20CRv3_scripts/ensda_602_2026/[^/]+'
  RE+='|build/bin/(getsfcensmeanp\.x|getsigensmeanp\.x|global_postgpp|gribmeanp\.x|psop_gfs_biascor2be\.x)'
  RE+='|build/enkf/trunk/src/global_enkf_new'
  RE+='|build/ext_New2_bin/global_sighdr'
  RE+='|build/gfs/global_fcst\.fd\.v14\.0\.1/global_fcst_cmip5'
  RE+='|tree_B/gfs/global_cycle\.fd/global_cyclep)$'
  [ "$(grep -cE "$RE" MANIFEST_OPERATIONAL.sha256)" -eq 17 ] || die "manifest selection is not 17 files"
  grep -E "$RE" MANIFEST_OPERATIONAL.sha256 | sha256sum --quiet -c - || die "a shared runtime file has changed: stop and contact the installation owner"
  echo "PASS B2"
)
```

**Expected:** `PASS B2`  
**PASS:** Continue to Step 4.

**FAIL:** Stop and contact the installation owner if files are missing or hashes differ.

## Step 4 — Copy eight workflow files (B3)

Creates your private workflow and copies exactly eight small scripts. The shared installation remains untouched.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ ! -e "$WORK" ] || die "$WORK already exists"
  mkdir -p "$(dirname "$WORK")"
  mkdir "$WORK"
  for f in $WF_FILES; do cp -p "$SRC_WF/$f" "$WORK/$f"; done
  [ "$(ls -A "$WORK" | wc -l)" -eq 8 ] || die "expected exactly 8 files in $WORK"
  for f in $WF_FILES; do cmp -s "$SRC_WF/$f" "$WORK/$f" || die "copy differs: $f"; done
  echo "PASS B3"
)
```

**Expected:** `PASS B3`  
**PASS:** Continue to Step 5.

**FAIL:** Stop and inspect the partially created private directory. Do not rerun blindly.

## Step 5 — Configure the driver (B4)

Patches **only line 205** of the private driver, verifies unchanged lines and the corrected surface-cycle executable.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  cd "$WORK"
  [ "$(sha256sum < "$DRIVER" | cut -d' ' -f1)" = 7e5d9bef4c61f9d69af80ffddb128ecdca5b65c7b5d5f13fed5f71320ec85bd2 ] \
    || die "driver is not the validated original (already edited?): redo B3"
  LC_ALL=C sed -i "s#^set datadir=/epccfs/n02/n02-NEZ000203/shared/edr/\$#set datadir=$DATADIR/#" "$DRIVER"
  [ "$(sed -n 205p "$DRIVER")" = "set datadir=$DATADIR/" ] || die "line 205 is not the new datadir"
  diff <(sed 205d "$SRC_WF/$DRIVER") <(sed 205d "$DRIVER") >/dev/null || die "a line other than 205 changed"
  if [ "$DATADIR" = /epccfs/n02/n02-NEZ000203/ehawkins/L ]; then
    [ "$(sha256sum < "$DRIVER" | cut -d' ' -f1)" = f24a4fd7eba6155079a4811ed7e1b894d2b69691a602e6e5439cf8dbbc14f513 ] || die "edited driver hash differs from the audited value"
  fi
  for f in $WF_FILES; do [ "$f" = "$DRIVER" ] || cmp -s "$SRC_WF/$f" "$f" || die "$f differs from the shared copy"; done
  csh -n "$DRIVER" || die "csh syntax: $DRIVER"
  csh -n run_enkf_ps_h5_biascor_sing_archer2 || die "csh syntax: run_enkf_ps_h5_biascor_sing_archer2"
  csh -n run_fg_gfs_ens_sing_archer2 || die "csh syntax: run_fg_gfs_ens_sing_archer2"
  sh -n run_gefs_sa2 || die "sh syntax: run_gefs_sa2"
  hits=$(grep -nE 'j_bilbao|shared/edr|shared/x2c|shared/x2d|shared/edh' $WF_FILES | grep -vE '^[^:]+:[0-9]+:[[:space:]]*#' | grep -vxF "$DRIVER:205:set datadir=$DATADIR/" || true)
  [ -z "$hits" ] || die "active reference to a private or evidence path: $hits"
  grep -qx 'setenv CYCLEXEC /epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1/tree_B/gfs/global_cycle.fd/global_cyclep' "$DRIVER" \
    || die "CYCLEXEC is not the corrected X5c-B binary"
  echo "PASS B4"
)
```

**Expected:** `PASS B4`  
**PASS:** Continue to Step 6.

**FAIL:** Stop. Never patch the shared driver.

## Step 6 — Copy and verify the initial restart (B5)

Stages the **886-file, ~40.55 GB** initial restart into your private data directory and verifies hashes.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ ! -e "$DATADIR" ] || die "$DATADIR already exists"
  [ "$(sed -n 205p "$WORK/$DRIVER")" = "set datadir=$DATADIR/" ] || die "driver not pointed at $DATADIR: complete B4 first"
  mkdir "$DATADIR"
  mkdir "$DATADIR/gfsenkf_20crV3_archer2" "$DATADIR/logs"
  rsync -rlpt "$SHARED"/inputs/restart/ensda_602_2026 "$DATADIR"/gfsenkf_20crV3_archer2/
  [ "$(ls -A "$DATADIR/gfsenkf_20crV3_archer2")" = ensda_602_2026 ] || die "unexpected layout under gfsenkf_20crV3_archer2"
  [ "$(find "$DP" -type f | wc -l)" -eq 886 ] || die "expected 886 staged files in $DP"
  [ "$(find "$DP" -type l | wc -l)" -eq 0 ] || die "unexpected symlinks in $DP"
  [ "$(grep -c '  \./inputs/restart/ensda_602_2026/' "$SHARED/MANIFEST_OPERATIONAL.sha256")" -eq 886 ] || die "manifest selection is not 886"
  ( cd "$DATADIR/gfsenkf_20crV3_archer2" &&
    grep '  \./inputs/restart/ensda_602_2026/' "$SHARED/MANIFEST_OPERATIONAL.sha256" |
    sed 's#  \./inputs/restart/#  ./#' |
    sha256sum --quiet -c -
  ) || die "start state differs from the manifest"
  [ "$(cat "$DP/analdate.csh")" = "setenv analdate 2026101000" ] || die "analdate.csh is not 2026101000"
  [ "$(cat "$DP/fg_only.csh")" = "setenv fg_only false" ] || die "fg_only.csh is not false"
  [ ! -L "$DATADIR" ] || die "$DATADIR is a symlink"
  echo "PASS B5"
)
```

**Expected:** `PASS B5`  
**PASS:** Continue to Step 7.

**FAIL:** Stop and inspect partial staging. Do not repeat blindly or modify the shared restart.

## Step 7 — Check the login environment (C1)

Checks that the clean login module environment matches the validated configuration. The driver loads its own modules.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  for f in ~/.cshrc ~/.tcshrc; do [ ! -e "$f" ] || die "$f exists: show it to the reviewer before submitting"; done
  ref='craype-x86-turin:libfabric/2.3.1:craype-network-ofi:'
  ref+='perftools-base/25.03.0:xpmem/1.0.1-1.5_1_gfb6998056825:'
  ref+='cce/19.0.0:cse_env/0.2:craype/2.7.34:cray-dsmml/0.3.1:'
  ref+='cray-mpich/8.1.32:cray-libsci/25.03.0:'
  ref+='PrgEnv-cray/8.6.0:epcc-setup-env:load-epcc-module'
  got=$(env -i HOME=$HOME USER=$USER LOGNAME=$USER TERM=dumb bash --login -c 'echo "$LOADEDMODULES"' | tail -n 1)
  [ "$got" = "$ref" ] || die "clean-login modules differ from the validated account: $got"
  echo "PASS C1"
)
```

**Expected:** `PASS C1`  
**PASS:** Continue to Step 8.

**FAIL:** Stop and investigate the module environment; Ed’s own login configuration has not been independently tested.

## Step 8 — Submit the 80-node job (D1)

**SUBMITS A REAL 80-NODE JOB.** Obtain allocation approval and confirm free space first. Execute only once.

**Copy, paste and run this entire block:**

```bash
JOB=
if ( set -euo pipefail
  queue=$(squeue -h -u "$USER" -n 2026_602_gfsenkfs) || die "squeue failed: do not submit"
  [ -z "$queue" ] || die "a 20CRv3 job is already queued or running"
  [ "$(sed -n 205p "$WORK/$DRIVER")" = "set datadir=$DATADIR/" ] || die "driver datadir is not $DATADIR/"
  [ "$(cat "$DP/analdate.csh")" = "setenv analdate 2026101000" ] || die "not a fresh start state: restage (B5)"
  [ ! -e "$DP/2026101006" ] || die "$DATADIR has already been used: use a new datadir"
  [ -z "$(ls -A "$DATADIR/logs")" ] || die "$DATADIR/logs is not empty"
  [ ! -e "$WORK/walltime.out" ] || die "$WORK has already been used: use a new workflow directory"
  avail=$(df -Pk "$(dirname "$DATADIR")" | awk 'NR==2{print $4}')
  [ "$avail" -ge 262144000 ] || die "less than 250 GiB free: do not submit"
  echo "PASS D1-precheck"
); then
  submit="cd $WORK && sbatch --parsable"
  submit+=" -o gfsenkf_cirrus_%j.out"
  submit+=" -e gfsenkf_cirrus_%j.err $DRIVER"
  if JOB=$(env -i HOME="$HOME" USER="$USER" LOGNAME="$USER" TERM=dumb bash --login -c "$submit"); then
    JOB=${JOB%%;*}
    case "$JOB" in
      ''|*[!0-9]*) echo "FAIL: invalid job ID. Check 'squeue -u $USER' before retrying"; JOB=; false ;;
      *) echo "JOB=$JOB" ;;
    esac
  else
    JOB=
    echo "FAIL: sbatch failed. Check 'squeue -u $USER' before retrying" >&2
    false
  fi
else
  echo "FAIL: D1 prechecks failed; no submission attempted" >&2
  false
fi
```

**Expected:** `PASS D1-precheck`, followed by `JOB=<numeric Slurm ID>`  
**PASS:** Continue to Step 9.

**FAIL:** Do not retry until checking the queue and logs. A precheck failure means no submission was attempted.

## Step 9 — Monitor the job (D2)

Monitor the Slurm job. If reconnecting, first rerun Step 1 and set `JOB=<actual ID>`. Use **Ctrl-C** to leave `tail -F` (this does not cancel the job).

**Copy, paste and run this entire block:**

```bash
squeue -u $USER
sacct -j $JOB --format=JobID,JobName%24,Elapsed,State,ExitCode
tail -F "$WORK"/gfsenkf_cirrus_$JOB.out           # Ctrl-C to stop
ls "$DATADIR"/logs/ensda_602_2026/                 # one ensda_out_<date> per cycle started
```

**Expected:** While running, `squeue` shows the job. On completion, `sacct` should show `COMPLETED` and exit code `0:0`.  
**PASS:** Continue to Step 10.

**FAIL:** An output or log file may not exist immediately. Wait and check the job state before concluding failure.

## Step 10 — Basic completion check

Run only after the job finishes. Checks Slurm exit state, final analysis date, 116 MPI steps and nonempty stdout.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ -n "${JOB:-}" ] || die "JOB is unset"
  state=$(sacct -j "$JOB" -X -n -P --format=State,ExitCode)
  [ "$state" = 'COMPLETED|0:0' ] || die "job state: $state"
  [ "$(cat "$DP/analdate.csh")" = 'setenv analdate 2026101100' ] || die "unexpected final date"
  [ "$(sacct -j "$JOB" -n -P --format=JobID | grep -cE '^[0-9]+\.[0-9]+$')" -eq 116 ] || die "MPI step count differs"
  [ -s "$WORK/gfsenkf_cirrus_$JOB.out" ] || die "stdout missing"
  echo "PASS basic completion checks"
)
```

**Expected:** `PASS basic completion checks`  
**PASS:** Continue to Step 11.

**FAIL:** Stop and inspect `sacct`, stdout/stderr and `analdate.csh`. **This alone is not scientific acceptance.**

## Step 11 — Check 81 deterministic surface analyses (X2)

**Reference case only:** checks 81 deterministic cycle-one surface analyses against the corrected reference hashes; other stochastic outputs need not be bitwise identical.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ -n "${JOB:-}" ] || die "JOB is unset"
  C1SUM=$SHARED/docs/validation_record/C1_610169/L_SHA256SUMS
  [ -r "$C1SUM" ] || die "reference checksum list missing"
  [ "$(grep -c 'sfcanl_2026101000_fhr03' "$C1SUM")" -eq 81 ] || die "expected 81 surface-analysis hashes"
  cd "$DATADIR"
  grep 'sfcanl_2026101000_fhr03' "$C1SUM" |
    sed 's#  L/#  ./#' |
    sha256sum --quiet -c - || die "X2: surface analyses differ from corrected reference"
  echo "PASS X2: 81 deterministic surface analyses verified"
)
```

**Expected:** `PASS X2: 81 deterministic surface analyses verified`  
**PASS:** The reference checks in this Quick Start have passed.

**FAIL:** Stop: do not accept the run as matching the corrected reference. Consult the shared operational manual for full E/X checks.

## Adapting the example to another experiment

**The above commands are validated for the October 2026 reference only.** The following is a change map, **not** a validated recipe for other cases. New dates require matching restarts, observations, SST/sea-ice inputs and revised validation.

| Change | Where / caution |
|---|---|
| Private directories | Step 1 `WORK`, `DATADIR`; keep `DATADIR/` at most 40 characters |
| Experiment identity | Workflow directory name (driver derives year from name), Slurm job name and output paths |
| Initial date | Matching restart and `analdate.csh`; do not relabel the October restart |
| Cycles | Driver line 72 `set ncycles=4`; six-hour interval `ANALINC=6` |
| Runtime limit | Driver line 73 `timemax=1` hour; set it below Slurm walltime with **at least one cycle of margin** (roughly 3 minutes at reference pace, preferably more), then validate |
| Slurm resources | Account and walltime/time-min; **keep 80 nodes for the validated 80-member layout** |
| Observations and SST | Driver lines 282 and 296; verify the entire requested date range |
| Checks | Update date-specific and output-specific expectations; do not demand bitwise equality of independently seeded outputs |

**Restart caution:** The driver reads `analdate.csh` on startup and overwrites it with the **next** analysis date after each completed cycle. The reference job has a 50-minute Slurm walltime, shorter than the default one-hour internal limit; Slurm can therefore kill the job first. Increasing `ncycles` alone is not sufficient, and automatic Slurm resubmission or safe multi-job continuation has **not** been established. Use fresh directories for a fresh run.

## If a step fails

Stop and record the failing command and output. Do **not** modify anything under `shared/` or resubmit into a used data directory. For full E/X checks and troubleshooting, consult the project-readable manual on Cirrus:

```text
/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1/docs/20CRv3_CIRRUS_USER_GUIDE_FINAL.md
```

## Verification status

The reference workflow and earlier operational checks were reviewed against completed Cirrus job `629598`. The Version 2 document and its new Step 11 still require final targeted audit; **no new 80-node job was submitted to validate this Markdown conversion**. The reported filesystem free space was approximately 89 GiB on 23 November 2026, below the guide’s 290 GiB staging requirement; recheck before running.
