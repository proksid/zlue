## zLUe

zLUe is a Perl-based toolset for installing, upgrading, and managing the FreeBSD base system and Jail containers, leveraging ZFS volume features for efficient snapshotting, cloning, and storage management.
=======
# zLUe - ZFS Live Update Environment

**zLUe** (ZFS Live Update Environment) is a Perl-based toolkit for creating, updating, and managing a **Live Update** environment for the **FreeBSD** base system and **jails** on **ZFS**.

---

## Overview

zLUe uses ZFS snapshots, clones, and datasets to maintain versioned environments. You can prepare a new system or “soft” (userland) update in a separate tree, apply changes there, then switch the running system to that version or roll back to a previous one without rebooting into a separate environment. It supports:

- **Base (host) system** - the main FreeBSD installation
- **Master jail** - a template jail from which other jails can be cloned
- **Custom jails** - named jails with their own LUE layout (e.g. `jtest`)

Each environment is driven by a **scheme** that defines which filesystems participate in Live Update and how (system vs soft, clone-only, mount options).

---

## Requirements

- **FreeBSD** with **ZFS**
- **Perl** (script uses `strict`, no non-core modules)
- Standard tools: `zfs`, `zpool`, `mount`, `umount`, `chroot`, `tar`, etc.

---

## Project Layout

| Item        | Description |
|------------|-------------|
| `zlue`     | Main executable Perl script (CLI and all logic) |
| `zlue.conf`| Configuration: pool names, paths, and **filesystem schemes** for base, master, and jail types |

Configuration is loaded via `require 'zlue.conf';` and defines package `Conf` (pools, jail paths, `%fs_scheme`). The script uses packages `Val` (version, flags, LUE property names) and `Util` (paths to system commands).

---

## Configuration (`zlue.conf`)

- **Pools and layout**
  - `$zpool_base` - ZFS pool for the host (e.g. `zbase-nau`)
  - `$zpool_jail` / `$jbase_fs` - pool or dataset for jails (e.g. `zbase-nau/zjail`)
  - `$master_jfs` - master jail filesystem (e.g. `$jbase_fs/master`)
- **Paths**
  - `$tmp_fsh` - temporary mount hierarchy (e.g. `/tmp`)
  - `$jail_script` - jail RC script (e.g. `/etc/rc.d/jail`)
  - `$make_options` - e.g. `-j4` for build
- **Schemes** (`%fs_scheme`)
  - **base** - host system: `/`, `/usr`, `/usr/src`, `/usr/obj`, `/tmp`, `/usr/local`, `/usr/ports`, `/var` subdirs, etc., with `lue` (yes/no), `type` (system/soft/none), and ZFS properties for promote/mount (`zps_pm`, `zps_um`).
  - **master** - master jail: root, `/etc`, `/usr`, `/tmp`, `/usr/local`, `/usr/ports`, `/var`, `/var/db/*`, etc., including `clone_only` where applicable.
  - **jtest** (or other names) - custom jail scheme with its own mount and LUE settings.

Each entry can set `readonly`, `exec`, `setuid` and similar secure options for the promote and mount stages.

---

## Usage

General form:

```text
zlue [--showonly|-s] [--verbose|-v] [-d 1|2|3] <command> [options]
```

- **`--showonly` / `-s`** - print what would be done, do not change the system.
- **`--verbose` / `-v`** - extra output.
- **`-d 1|2|3`** - debug level.

Target is usually **base**, **master**, or a **jail name**; many commands also take a **version** (snapshot/version tag).

---

## Commands

### Update and rollback

- **`update`** *(base|master|jail_name)* *new_version*  
  Switch “current” to *new_version* (all: system + soft).
- **`update-system`** / **`update-soft`**  
  Same but only system or only soft datasets.

- **`rollback`** *(base|master|jail_name)* *outdate_version*  
  Switch “current” back to *outdate_version*.
- **`rollback-system`** / **`rollback-soft`**  
  Roll back only system or only soft.

### Create and init

- **`create-env`** / **`create-system-env`** / **`create-soft-env`**  
  *(base | master | jail_name)* *new_version*  
  Create a new LUE version (update side) for all, system, or soft.

- **`init-env`** / **`init-system-env`** / **`init-soft-env`**  
  *(base | master | jail_name)* *new_version*  
  Initialize that version (e.g. after you have upgraded inside the new tree).
- **`init-env-force`**  
  Same as `init-env` but force (e.g. overwrite) when applicable.

### Inspect and maintain

- **`check-env`** / **`check-system-env`** / **`check-soft-env`**  
  *(base | master | jail_name)* [*version*]  
  Check consistency and status of a given (or current) version.

- **`delete-env`** / **`delete-system-env`** / **`delete-soft-env`**  
  *(base | master | jail_name)* *version*  
  Remove a non-current version.

- **`clear-env`** / **`clear-system-env`** / **`clear-soft-env`**  
  *(base | master | jail_name)* *version*  
  Clear (e.g. inherit) properties for that version; or use *version* `current` to clear current for all types.

- **`set-env`** / **`set-system-env`** / **`set-soft-env`**  
  *(base | master | jail_name)* *version* *property=value*  
  Set a ZFS property on that version’s datasets.

### Mode (production / update)

- **`modify-env`** / **`modify-system-env`** / **`modify-soft-env`**  
  *jail_name* **(production | update)**  
  Switch **lue:mode** for the current version (production vs update).

### Mount and copy

- **`mount-env`** / **`mount-system-env`** / **`mount-soft-env`**  
  *(base | master | jail_name)* *version*  
  Mount that version (e.g. under `$tmp_fsh`).

- **`umount-env`**  
  *(base | master | jail_name)* *version*  
  Unmount that version.

- **`copy-env`** / **`copy-system-env`** / **`copy-soft-env`**  
  *src_jail_name* *dst_jail_name*  
  Copy environment from one jail to another.

### Show and snapshots

- **`show`** *(base | master | jail_name)* [*version*] [**brief | detail | status | versions**]  
  Show LUE properties and status for current or given version.

- **`create-snapshot`** *jail_name* *snapshot_version*  
  Create recursive snapshot of current version with tag *snapshot_version*.

- **`delete-snapshot`** *filesystem* *snapshot_version*  
  Destroy snapshots of *snapshot_version* under *filesystem*.

---

## Concepts

- **Version** - A snapshot/version tag (e.g. `00`, `01`) used to identify a LUE instance. “Current” is the active one; others are “update” or “outdate”.
- **System vs soft** - “System” datasets are core OS (e.g. `/`, `/usr`); “soft” are optional/userland (e.g. `/usr/local`, `/usr/ports`). Commands can act on all, system only, or soft only.
- **Scheme** - Defines which mount points and datasets participate in LUE and their options (e.g. `base`, `master`, `jtest`). Stored in `zlue.conf` as `%fs_scheme`.
- **Emergency** - Script can call an emergency rollback script (`$Val::emergency_script`, e.g. `zlue_rollback`) when needed.

---

## ZFS properties (LUE)

The script uses custom ZFS properties (e.g. under `lue:*`) to track:

- `lue:liveupdate` - yes/no  
- `lue:type` - system, soft, late, none  
- `lue:scheme` - base, master, jail name  
- `lue:jname` - base, master, jail name  
- `lue:version` - version list  
- `lue:parent` - parent version  
- `lue:status` - current, update, outdate  
- `lue:candel` - yes/no  
- `lue:clone_only` - yes/no  
- and others (see `%Val::lue_params_type` in `zlue`).

---

## Typical workflow (high level)

1. **Create** a new LUE version:  
   `zlue create-env base 01`
2. **Upgrade** inside that tree (e.g. chroot, then `make installworld`, or `freebsd-update`, or manual).
3. **Init** the new version:  
   `zlue init-env base 01`
4. **Update** (switch current to the new version):  
   `zlue update base 01`  
   Or **rollback** if needed:  
   `zlue rollback base 00`

For jails, use **master** or a custom jail name and the same create → upgrade → init → update/rollback pattern; copy-env can clone one jail’s environment to another.

---

## Version

- **zlue**: v.2.12.8 (in `$Val::version`)
- **zlue.conf**: v.2.12.8

---

## License

Apache License 2.0 (see `LICENSE`).
