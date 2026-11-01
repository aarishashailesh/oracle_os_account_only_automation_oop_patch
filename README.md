[README.md](https://github.com/user-attachments/files/34247285/README.md)
[README.md](https://github.com/user-attachments/files/34247114/README.md)
# Oracle 19c OOP Patch Automation Framework
## Single Node CRS - single OS owner (`oracle` only, no `grid` user)

Tested layout: lnx103.home.aarisha.com, 19.3 to 19.33, Grid and database both owned by `oracle`.

---

## Environment this version targets

| Item | Value |
|---|---|# Oracle 19c OOP Patch Automation Framework
## Single Node CRS - single OS owner (`oracle` only, no `grid` user)

Tested layout: lnx103.home.aarisha.com, 19.3 to 19.33, Grid and database both owned by `oracle`.

---

## Environment this version targets

| Item | Value |
|---|---|
| OS user owning Grid Infrastructure and the database | `oracle` (there is no `grid` user) |
| Primary group | `oinstall` |
| ASM groups (OSDBA / OSOPER / OSASM) | all `dba` (take the real values from your original Grid response file) |
| Grid environment | defined in `/home/oracle/.bashrc` (`ORACLE_HOME`, `ORACLE_SID=+ASM1`, `LD_LIBRARY_PATH`) |
| Database environment | `. oraenv` |
| Grid base (`GRID_BASE`) | `/u01/app/oracle` |
| Database base (`ORACLE_BASE`) | `/u01/app/odaorabase/oracle` |

The orchestrator is started as **root**. Phases that need the software owner switch to `oracle`
internally. Nothing in this framework refers to a `grid` OS account.

---

## Directory Layout

```
/u01/patching/automation/            <- SCRIPTS_DIR (must be readable by oracle, never under /root)
|-- patch.conf                       <- EDIT THIS FOR EACH PATCH CYCLE
|-- patch_orchestrator.sh            <- Master orchestrator (run as root)
|-- phase1_grid_prep.sh              <- Build + patch new Grid home      (runs as oracle)
|-- phase2_db_prep.sh                <- Build + patch new DB home        (runs as oracle)
|-- phase3_switch.sh                 <- Home switch + AutoUpgrade deploy (root, su - oracle)
|-- phase4_post_install.sh           <- Post-patch validation + utlrp    (root, su - oracle)
|-- phase5_cleanup.sh                <- Deinstall old homes              (manual)
|-- phase6_security.sh               <- FIPS file + extproc disable      (manual)
|-- phase7_aide.sh                   <- AIDE baseline reset              (manual)
`-- README.md                        <- This file

/u01/patching/logs/logs_1933/        <- LOG_DIR, one directory per patch cycle
|-- phaseN_*.log
`-- .phase_status/phaseN.done        <- success markers that enforce phase order

/opt/oracle/patches/                 <- PATCH_LOC (staging)
|-- p6880880_190000_Linux-x86-64.zip <- Latest OPatch
|-- p<RU_COMBO>_190000_Linux-x86-64.zip   <- RU combo (Phase 1 extracts it here)
|-- p<DB_STANDALONE>_*.zip           <- Standalone DB patch (Phase 2 extracts it here)
|-- autoupgrade.jar                  <- Latest AutoUpgrade
|-- orcl_auto_patch.cfg              <- AutoUpgrade config (source/target homes, SID)
|-- grid_install.rsp                 <- Grid response file template
`-- db_install.rsp                   <- DB response file template

/u01/shared/software/db/base_releases/   <- BASE_SW_DIR: LINUX.X64_193000_grid_home.zip, LINUX.X64_193000_db_home.zip
```

Extracted patch directories (`<RU_COMBO>/...`, `<DB_STANDALONE>/`) appear under `/opt/oracle/patches`
only after Phase 1 and Phase 2 run. They are not expected beforehand.

---

## For Each New Patch Cycle

**Only edit `patch.conf`.** Update:

### Patch identity (always update)
| Variable | Description |
|---|---|
| `NEW_PATCH_LEVEL` / `OLD_PATCH_LEVEL` | Labels used in logs (for example `19.33` / `19.3`) |
| `RU_COMBO_PATCH` | Outer RU combo patch number |
| `GI_RU_PATCH` | GI RU bundle directory inside the combo (contains `bundle.xml`) |
| `GI_APPLY_RU_PATCH` | `gridSetup.sh -applyRU` sub-patch (OCW) |
| `GI_ONEOFF_PATCHES` | `gridSetup.sh -applyOneOffs`, comma-separated IDs |
| `DB_APPLY_RU_PATCH` | `runInstaller -applyRU` sub-patch (DB RU) |
| `DB_ONEOFF_PATCHES` | `runInstaller -applyOneOffs`, comma-separated IDs |
| `RU_COMBO_TOPLEVEL_PATCHES` | Patches at the top level of the combo (for example OJVM) |
| `DB_RU_STANDALONE_PATCH` / `DB_RU_STANDALONE_ZIP` | Standalone DB patch number and zip name |

### Home paths (explicit, never derived from the patch level)
| Variable | Example |
|---|---|
| `OLD_GRID_HOME` | `/u01/app/19.0.0.0/oracle` |
| `NEW_GRID_HOME` | `/u01/app/19.33.0.0/oracle` |
| `OLD_DB_HOME` | `/u01/app/odaorahome/oracle/product/19.0.0.0/dbhome_1930` |
| `NEW_DB_HOME` | `/u01/app/odaorahome/oracle/product/19.0.0.0/dbhome_1933` |

> **No trailing slash on any home path.** Phases 1 and 3 run `sed s|OLD|NEW|` over helper scripts and
> profile files, and Phase 3 compares `NEW_GRID_HOME` with `crs_home` in `/etc/oracle/olr.loc`.
> A trailing `/` makes both of those miss.

### Users, groups and bases (set once per host)
```bash
GRID_USER="oracle"          # same account as ORACLE_USER
ORACLE_USER="oracle"
INSTALL_GROUP="oinstall"
GI_OSDBA_GROUP="dba"        # use the values in the ORIGINAL grid response file
GI_OSOPER_GROUP="dba"
GI_OSASM_GROUP="dba"
GRID_BASE="/u01/app/oracle"                  # Grid ORACLE_BASE (from .bashrc / grid rsp)
ORACLE_BASE="/u01/app/odaorabase/oracle"     # database base (what oraenv reports)
DB_NAME="maxstage"                           # CRS database name (srvctl config database)
```
Verify the groups with:
`grep -i "osdba\|osoper\|osasm" /opt/oracle/patches/grid_install.rsp`

### Response files
| Variable | Value |
|---|---|
| `GRID_RSP` | `/opt/oracle/patches/grid_install.rsp` |
| `DB_RSP` | `/opt/oracle/patches/db_install.rsp` |

These are templates only. Each run works on a temporary copy, so the originals are never modified.

- Phase 1 rewrites `ORACLE_HOME`, forces `oracle.install.option=CRS_SWONLY` and sets `clusterNodes` to the short hostname.
- Phase 2 rewrites only `ORACLE_HOME` in the DB response file. `ORACLE_BASE` is taken from the file as is.
- Phase 3 rewrites OSDBA / OSOPER / OSASM from `patch.conf` and forces `CRS_SWONLY`. It keeps `clusterNodes` from the template, so the template must read `<short-host>:<short-host>-vip.<domain>`, for example `lnx103:lnx103-vip.home.aarisha.com`.
- Plain-text ASM passwords are not needed for a software-only run. Remove them from the staged template.

---

## Running the Orchestrator (as root)

### Validate configuration first
```bash
./patch_orchestrator.sh --phase 1 --dry-run
```
A dry run checks `patch.conf`, `PATCH_LOC` and script readability by `oracle`. It does not
check that the patch zips exist.

### Phase by phase (recommended while testing)
```bash
./patch_orchestrator.sh --phase 1    # Grid home prep (online)
./patch_orchestrator.sh --phase 2    # DB home prep + AutoUpgrade analyze (online)
./patch_orchestrator.sh --phase 3    # Switch (OUTAGE)
./patch_orchestrator.sh --phase 4    # Post-patch validation
```

### Full run, phases 1-4
```bash
chmod 755 /u01/patching/automation/*.sh
nohup /u01/patching/automation/patch_orchestrator.sh > /dev/null 2>&1 &
tail -f /u01/patching/logs/logs_1933/master_patch.log
```

### Hardened hosts (session killed after 600 s idle, `KillUserProcesses=yes`, `TMOUT=600`)
`nohup` and `&` do not survive these settings because logind kills the whole login session scope.
Run the orchestrator as a transient systemd service instead. It lives outside your session, so
neither logout nor the idle timeout can kill it. No phase script prompts for input, so no tty is needed.
```bash
systemd-run --unit=oppatch-p1 --remain-after-exit \
  --working-directory=/u01/patching/automation \
  /u01/patching/automation/patch_orchestrator.sh --phase 1

tail -f /u01/patching/logs/logs_1933/master_patch.log     # progress
systemctl status oppatch-p1                                # running / exited
systemctl show oppatch-p1 -p Result -p ExecMainStatus      # Result=success, ExecMainStatus=0
systemctl stop oppatch-p1                                  # clear the unit when done
```
Use a different `--unit` name per run. `TMOUT` only logs out your interactive shell, so reconnect
and keep watching the log.

### Other options
```bash
./patch_orchestrator.sh --from-phase 2     # resume after fixing a failure
./patch_orchestrator.sh --cleanup          # also run phase 5
./patch_orchestrator.sh --full             # phases 1-7
./patch_orchestrator.sh --reset-status     # clear markers after a snapshot restore
```

---

## Phase Summary

| Phase | Script | Runs as | Window | Description |
|---|---|---|---|---|
| 1 | phase1_grid_prep.sh | oracle | Online | Extract base Grid, OPatch, RU combo; `gridSetup.sh -applyRU/-applyOneOffs` (CRS_SWONLY) |
| 2 | phase2_db_prep.sh | oracle | Online | Extract base DB, OPatch, standalone patch; `runInstaller -applyRU/-applyOneOffs`; AutoUpgrade analyze |
| 3 | phase3_switch.sh | root, then `su - oracle` | **OUTAGE** | Stop listener, `gridSetup.sh -switchGridHome`, `root.sh`, AutoUpgrade deploy (DB home switch + datapatch), profile updates, registration check |
| 4 | phase4_post_install.sh | root, then `su - oracle` | Online | `utlrp`, registry validation |
| 5 | phase5_cleanup.sh | root | Manual | Deinstall old Grid and DB homes (only runs with `--cleanup`, `--full` or `--phase 5`) |
| 6 | phase6_security.sh | root | Manual | FIPS file, disable `extproc` |
| 7 | phase7_aide.sh | root | Manual | Reset AIDE baseline (must run last) |

Phase order is enforced by markers in `.phase_status`. Phase 7 runs last so the AIDE baseline matches
the final filesystem state after the old homes are gone.

---

## Single OS owner: how the scripts handle it

1. **Login-shell environment.** `su - oracle` sources `.bashrc`, which exports a Grid `ORACLE_HOME`,
   `ORACLE_SID=+ASM1` and `LD_LIBRARY_PATH=$ORACLE_HOME/grid/lib`. Every `su -` and `sudo -i` call therefore starts with
   `unset LD_LIBRARY_PATH`, and the installer calls set their own `ORACLE_HOME`.
2. **Installers.** Run `gridSetup.sh` and `runInstaller` with `ORACLE_HOME`, `ORACLE_SID` and `LD_LIBRARY_PATH` unset, otherwise the
   installer picks up the running Grid environment.
3. **Listener.** Stopped with `srvctl stop listener -l LISTENER` from the old Grid home. `lsnrctl` is not used. A listener that is already down is tolerated on re-run.
4. **Response file permissions.** The Phase 3 temporary response file is created by root and then handed to `oracle` with mode 600, because `gridSetup.sh` runs as `oracle`. Passwords are never written to the log.
5. **`/etc/oratab`.** Maintained by the CRS agent (`# line added by Agent`). The scripts do not edit it. Phase 3 ends with a read-only check of the CRS database home, `oratab` and `crs_home` in `/etc/oracle/olr.loc`.
6. **Profiles.** Phase 3 updates `ORACLE_HOME` paths in the `oracle` profile files with `sed`. Home paths must be exact, with no trailing slash.
7. **Database name.** `phase5_cleanup.sh` uses `ora.${DB_NAME}.db`, not a hardcoded name.
8. **`runInstaller` exit code 6** (success with optional prerequisite warnings) is accepted. If the new DB home is already built, Phase 2 resumes at Step 8. Force a step with `PHASE2_FROM_STEP=<n>`.

---

## Known Issues and Notes

- `gridSetup.sh -switchGridHome` is used for the Grid home switch on this clustered (CRS) install. It cannot be re-run once CRS is already running from the new home (`INS-45102`). Phase 3 detects that case and skips to the AutoUpgrade deploy step.
- `srvctl modify database -o` must run with `ORACLE_HOME` set to the old home. Phase 3 handles this.
- `opatch lspatches` on a new home can return errors before the switch. Use the `.patch_storage` listing for pre-switch checks.
- Do not shut the database down before Phase 3 Step 7. AutoUpgrade deploy needs it running to connect to the source home.
- `chopt` options apply to non-CDB databases only. Skip them for CDBs.
- `CV_ASSUME_DISTID=OEL7.8` is exported from `patch.conf`. Without it, `dbca`, `gridSetup.sh` and `deinstall` can fail with `INS-08101` or a `NullPointerException`.
- A stale `PDB_PLUG_IN_VIOLATIONS` row can block the AutoUpgrade analyze step. Closing and reopening the PDB refreshes it.

---

## Troubleshooting quick reference

| Symptom | Likely cause | Action |
|---|---|---|
| `INS-10105 ... (Permission denied)` in Phase 3 | Temp response file owned by root | Use the current `phase3_switch.sh` (chowns the file to `oracle`) |
| `.bashrc` still shows the old Grid home after Phase 3 | Trailing `/` on `OLD_GRID_HOME` | Remove the slash in `patch.conf` and fix the profile by hand |
| `INS-41885` on `switchGridHome` | Group values wrong or passed on the command line | Set `GI_OS*_GROUP` to match the original Grid response file |
| `CLSRSC-901` in `root.sh` | Old home path left in `root.sh` or `rootmacro.sh` | Phase 3 Step 3 corrects it, re-run the phase |
| `runInstaller` exit code 6 | Optional prerequisite warnings | Accepted by Phase 2 |
| `INS-08101` / `NullPointerException` | `CV_ASSUME_DISTID` not set | Set it in `patch.conf` |
| `TNS-01190` from `lsnrctl stop` | Listener is CRS-managed | Use `srvctl stop listener` |
| `kfod` Error 49802 in AutoUpgrade analyze | ADR directory owned by wrong user | `chown -R oracle:oinstall $ORACLE_BASE/diag` |
| "Phase N has not completed" | Missing success marker | Run earlier phases, or `--reset-status` after a snapshot restore |

| OS user owning Grid Infrastructure and the database | `oracle` (there is no `grid` user) |
| Primary group | `oinstall` |
| ASM groups (OSDBA / OSOPER / OSASM) | all `dba` (take the real values from your original Grid response file) |
| Grid environment | defined in `/home/oracle/.bashrc` (`ORACLE_HOME`, `ORACLE_SID=+ASM1`, `LD_LIBRARY_PATH`) |
| Database environment | `. oraenv` |
| Grid base (`GRID_BASE`) | `/u01/app/oracle` |
| Database base (`ORACLE_BASE`) | `/u01/app/odaorabase/oracle` |

The orchestrator is started as **root**. Phases that need the software owner switch to `oracle`
internally. Nothing in this framework refers to a `grid` OS account.

---

## Directory Layout

```
/u01/patching/automation/            <- SCRIPTS_DIR (must be readable by oracle, never under /root)
|-- patch.conf                       <- EDIT THIS FOR EACH PATCH CYCLE
|-- patch_orchestrator.sh            <- Master orchestrator (run as root)
|-- phase1_grid_prep.sh              <- Build + patch new Grid home      (runs as oracle)
|-- phase2_db_prep.sh                <- Build + patch new DB home        (runs as oracle)
|-- phase3_switch.sh                 <- Home switch + AutoUpgrade deploy (root, su - oracle)
|-- phase4_post_install.sh           <- Post-patch validation + utlrp    (root, su - oracle)
|-- phase5_cleanup.sh                <- Deinstall old homes              (manual)
|-- phase6_security.sh               <- FIPS file + extproc disable      (manual)
|-- phase7_aide.sh                   <- AIDE baseline reset              (manual)
`-- README.md                        <- This file

/u01/patching/logs/logs_1933/        <- LOG_DIR, one directory per patch cycle
|-- phaseN_*.log
`-- .phase_status/phaseN.done        <- success markers that enforce phase order

/opt/oracle/patches/                 <- PATCH_LOC (staging)
|-- p6880880_190000_Linux-x86-64.zip <- Latest OPatch
|-- p<RU_COMBO>_190000_Linux-x86-64.zip   <- RU combo (Phase 1 extracts it here)
|-- p<DB_STANDALONE>_*.zip           <- Standalone DB patch (Phase 2 extracts it here)
|-- autoupgrade.jar                  <- Latest AutoUpgrade
|-- orcl_auto_patch.cfg              <- AutoUpgrade config (source/target homes, SID)
|-- grid_install.rsp                 <- Grid response file template
`-- db_install.rsp                   <- DB response file template

/u01/shared/software/db/base_releases/   <- BASE_SW_DIR: LINUX.X64_193000_grid_home.zip, LINUX.X64_193000_db_home.zip
```

Extracted patch directories (`<RU_COMBO>/...`, `<DB_STANDALONE>/`) appear under `/opt/oracle/patches`
only after Phase 1 and Phase 2 run. They are not expected beforehand.

---

## For Each New Patch Cycle

**Only edit `patch.conf`.** Update:

### Patch identity (always update)
| Variable | Description |
|---|---|
| `NEW_PATCH_LEVEL` / `OLD_PATCH_LEVEL` | Labels used in logs (for example `19.33` / `19.3`) |
| `RU_COMBO_PATCH` | Outer RU combo patch number |
| `GI_RU_PATCH` | GI RU bundle directory inside the combo (contains `bundle.xml`) |
| `GI_APPLY_RU_PATCH` | `gridSetup.sh -applyRU` sub-patch (OCW) |
| `GI_ONEOFF_PATCHES` | `gridSetup.sh -applyOneOffs`, comma-separated IDs |
| `DB_APPLY_RU_PATCH` | `runInstaller -applyRU` sub-patch (DB RU) |
| `DB_ONEOFF_PATCHES` | `runInstaller -applyOneOffs`, comma-separated IDs |
| `RU_COMBO_TOPLEVEL_PATCHES` | Patches at the top level of the combo (for example OJVM) |
| `DB_RU_STANDALONE_PATCH` / `DB_RU_STANDALONE_ZIP` | Standalone DB patch number and zip name |

### Home paths (explicit, never derived from the patch level)
| Variable | Example |
|---|---|
| `OLD_GRID_HOME` | `/u01/app/19.0.0.0/oracle` |
| `NEW_GRID_HOME` | `/u01/app/19.33.0.0/oracle` |
| `OLD_DB_HOME` | `/u01/app/odaorahome/oracle/product/19.0.0.0/dbhome_1930` |
| `NEW_DB_HOME` | `/u01/app/odaorahome/oracle/product/19.0.0.0/dbhome_1933` |

> **No trailing slash on any home path.** Phases 1 and 3 run `sed s|OLD|NEW|` over helper scripts and
> profile files, and Phase 3 compares `NEW_GRID_HOME` with `crs_home` in `/etc/oracle/olr.loc`.
> A trailing `/` makes both of those miss.

### Users, groups and bases (set once per host)
```bash
GRID_USER="oracle"          # same account as ORACLE_USER
ORACLE_USER="oracle"
INSTALL_GROUP="oinstall"
GI_OSDBA_GROUP="dba"        # use the values in the ORIGINAL grid response file
GI_OSOPER_GROUP="dba"
GI_OSASM_GROUP="dba"
GRID_BASE="/u01/app/oracle"                  # Grid ORACLE_BASE (from .bashrc / grid rsp)
ORACLE_BASE="/u01/app/odaorabase/oracle"     # database base (what oraenv reports)
DB_NAME="maxstage"                           # CRS database name (srvctl config database)
```
Verify the groups with:
`grep -i "osdba\|osoper\|osasm" /opt/oracle/patches/grid_install.rsp`

### Response files
| Variable | Value |
|---|---|
| `GRID_RSP` | `/opt/oracle/patches/grid_install.rsp` |
| `DB_RSP` | `/opt/oracle/patches/db_install.rsp` |

These are templates only. Each run works on a temporary copy, so the originals are never modified.

- Phase 1 rewrites `ORACLE_HOME`, forces `oracle.install.option=CRS_SWONLY` and sets `clusterNodes` to the short hostname.
- Phase 2 rewrites only `ORACLE_HOME` in the DB response file. `ORACLE_BASE` is taken from the file as is.
- Phase 3 rewrites OSDBA / OSOPER / OSASM from `patch.conf` and forces `CRS_SWONLY`. It keeps `clusterNodes` from the template, so the template must read `<short-host>:<short-host>-vip.<domain>`, for example `lnx103:lnx103-vip.home.aarisha.com`.
- Plain-text ASM passwords are not needed for a software-only run. Remove them from the staged template.

---

## Running the Orchestrator (as root)

### Validate configuration first
```bash
./patch_orchestrator.sh --phase 1 --dry-run
```
A dry run checks `patch.conf`, `PATCH_LOC` and script readability by `oracle`. It does not
check that the patch zips exist.

### Phase by phase (recommended while testing)
```bash
./patch_orchestrator.sh --phase 1    # Grid home prep (online)
./patch_orchestrator.sh --phase 2    # DB home prep + AutoUpgrade analyze (online)
./patch_orchestrator.sh --phase 3    # Switch (OUTAGE)
./patch_orchestrator.sh --phase 4    # Post-patch validation
```

### Full run, phases 1-4
```bash
chmod 755 /u01/patching/automation/*.sh
nohup /u01/patching/automation/patch_orchestrator.sh > /dev/null 2>&1 &
tail -f /u01/patching/logs/logs_1933/master_patch.log
```

### Other options
```bash
./patch_orchestrator.sh --from-phase 2     # resume after fixing a failure
./patch_orchestrator.sh --cleanup          # also run phase 5
./patch_orchestrator.sh --full             # phases 1-7
./patch_orchestrator.sh --reset-status     # clear markers after a snapshot restore
```

---

## Phase Summary

| Phase | Script | Runs as | Window | Description |
|---|---|---|---|---|
| 1 | phase1_grid_prep.sh | oracle | Online | Extract base Grid, OPatch, RU combo; `gridSetup.sh -applyRU/-applyOneOffs` (CRS_SWONLY) |
| 2 | phase2_db_prep.sh | oracle | Online | Extract base DB, OPatch, standalone patch; `runInstaller -applyRU/-applyOneOffs`; AutoUpgrade analyze |
| 3 | phase3_switch.sh | root, then `su - oracle` | **OUTAGE** | Stop listener, `gridSetup.sh -switchGridHome`, `root.sh`, AutoUpgrade deploy (DB home switch + datapatch), profile updates, registration check |
| 4 | phase4_post_install.sh | root, then `su - oracle` | Online | `utlrp`, registry validation |
| 5 | phase5_cleanup.sh | root | Manual | Deinstall old Grid and DB homes (requires typing `YES`) |
| 6 | phase6_security.sh | root | Manual | FIPS file, disable `extproc` |
| 7 | phase7_aide.sh | root | Manual | Reset AIDE baseline (must run last) |

Phase order is enforced by markers in `.phase_status`. Phase 7 runs last so the AIDE baseline matches
the final filesystem state after the old homes are gone.

---

## Single OS owner: how the scripts handle it

1. **Login-shell environment.** `su - oracle` sources `.bashrc`, which exports a Grid `ORACLE_HOME`,
   `ORACLE_SID=+ASM1` and `LD_LIBRARY_PATH=$ORACLE_HOME/grid/lib`. Every `su -` and `sudo -i` call therefore starts with
   `unset LD_LIBRARY_PATH`, and the installer calls set their own `ORACLE_HOME`.
2. **Installers.** Run `gridSetup.sh` and `runInstaller` with `ORACLE_HOME`, `ORACLE_SID` and `LD_LIBRARY_PATH` unset, otherwise the
   installer picks up the running Grid environment.
3. **Listener.** Stopped with `srvctl stop listener -l LISTENER` from the old Grid home. `lsnrctl` is not used. A listener that is already down is tolerated on re-run.
4. **Response file permissions.** The Phase 3 temporary response file is created by root and then handed to `oracle` with mode 600, because `gridSetup.sh` runs as `oracle`. Passwords are never written to the log.
5. **`/etc/oratab`.** Maintained by the CRS agent (`# line added by Agent`). The scripts do not edit it. Phase 3 ends with a read-only check of the CRS database home, `oratab` and `crs_home` in `/etc/oracle/olr.loc`.
6. **Profiles.** Phase 3 updates `ORACLE_HOME` paths in the `oracle` profile files with `sed`. Home paths must be exact, with no trailing slash.
7. **Database name.** `phase5_cleanup.sh` uses `ora.${DB_NAME}.db`, not a hardcoded name.
8. **`runInstaller` exit code 6** (success with optional prerequisite warnings) is accepted. If the new DB home is already built, Phase 2 resumes at Step 8. Force a step with `PHASE2_FROM_STEP=<n>`.

---

## Known Issues and Notes

- `gridSetup.sh -switchGridHome` is used for the Grid home switch on this clustered (CRS) install. It cannot be re-run once CRS is already running from the new home (`INS-45102`). Phase 3 detects that case and skips to the AutoUpgrade deploy step.
- `srvctl modify database -o` must run with `ORACLE_HOME` set to the old home. Phase 3 handles this.
- `opatch lspatches` on a new home can return errors before the switch. Use the `.patch_storage` listing for pre-switch checks.
- Do not shut the database down before Phase 3 Step 7. AutoUpgrade deploy needs it running to connect to the source home.
- `chopt` options apply to non-CDB databases only. Skip them for CDBs.
- `CV_ASSUME_DISTID=OEL7.8` is exported from `patch.conf`. Without it, `dbca`, `gridSetup.sh` and `deinstall` can fail with `INS-08101` or a `NullPointerException`.
- A stale `PDB_PLUG_IN_VIOLATIONS` row can block the AutoUpgrade analyze step. Closing and reopening the PDB refreshes it.

---

## Troubleshooting quick reference

| Symptom | Likely cause | Action |
|---|---|---|
| `INS-10105 ... (Permission denied)` in Phase 3 | Temp response file owned by root | Use the current `phase3_switch.sh` (chowns the file to `oracle`) |
| `.bashrc` still shows the old Grid home after Phase 3 | Trailing `/` on `OLD_GRID_HOME` | Remove the slash in `patch.conf` and fix the profile by hand |
| `INS-41885` on `switchGridHome` | Group values wrong or passed on the command line | Set `GI_OS*_GROUP` to match the original Grid response file |
| `CLSRSC-901` in `root.sh` | Old home path left in `root.sh` or `rootmacro.sh` | Phase 3 Step 3 corrects it, re-run the phase |
| `runInstaller` exit code 6 | Optional prerequisite warnings | Accepted by Phase 2 |
| `INS-08101` / `NullPointerException` | `CV_ASSUME_DISTID` not set | Set it in `patch.conf` |
| `TNS-01190` from `lsnrctl stop` | Listener is CRS-managed | Use `srvctl stop listener` |
| `kfod` Error 49802 in AutoUpgrade analyze | ADR directory owned by wrong user | `chown -R oracle:oinstall $ORACLE_BASE/diag` |
| "Phase N has not completed" | Missing success marker | Run earlier phases, or `--reset-status` after a snapshot restore |
