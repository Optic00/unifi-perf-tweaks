# UniFi OS 6.0.12 Compatibility Verification

**Result: compatible.** Static check of the UCG-Fiber image, and live checks on two UCG-Fiber gateways on 6.0.12, 2026-10-11: one without SSD (10 + 15, read-only SSH) and one with SSD (08 + 09 + 10 + 15, Network Optimizer status page).

| | |
|---|---|
| Build | `UCGF.ipq9574.v6.0.12.257a726.261007.1700` |
| `.bin` md5 | `b945969262d6c83c7ac855093ebfe067` (953,202,655 bytes) |
| Rootfs | zstd squashfs at offset `16090774`, 52,429 inodes |
| Kernel | `#5.4.213 SMP PREEMPT Wed Oct 7 17:02:07 CST 2026` (rebuilt), GCC 14.2.0-19 |
| vermagic | `5.4.213-ui-ipq9574 SMP preempt mod_unload aarch64`, unchanged. No module rebuild. |
| Baseline | 6.0.11 UCG-Fiber image |

| Tweak | Static (UCG-Fiber image) |
|---|---|
| 19 + 20 SGMII+ | ✓ `qca-ssdk.ko` `.text`-identical to 6.0.11 |
| 06 + 07 MongoDB | ✓ `mongod`, `unifi-mongodb.service`, `/etc/default/unifi` identical to 6.0.11 |
| 08 + 09 PostgreSQL | ✓ `/etc/default/postgresql/14-apps`, `pg-cluster-setup`, PostgreSQL 14 binaries and units identical to 6.0.11 |
| `mongodb-ssd-decommission.sh` | ✓ Its dependencies are covered by the 06 + 07 and 08 + 09 rows |
| 10 journald | ✓ `journald.conf` and `syslog-ng/` identical to 6.0.11 |
| 15 fan | ✓ `ufcd` and `ufcd.service` unchanged. `ustd` 6.0.9 → 6.0.10 changed code in `sdb_client.so`; the live run below shows the API works |

## Kernel

The decompressed kernel `Image` is the same size as 6.0.11 (13,395,976 bytes). Only 51 bytes differ: the two build-timestamp banners, the build-id, and the initramfs cpio mtimes. The device tree is byte-identical to 6.0.11. This is a rebuild with no code change.

## qca-ssdk

md5 `1906a5af6bf1b0a3f8dfeeebe9ba4f58`, 3,446,432 bytes. 42 bytes differ from 6.0.11, all provenance: the build-id (20), the embedded build timestamp, and the Jenkins job path (`49-40dc` → `76-28CO`). `.text` is byte-identical and the section layout is unchanged, so the symbols and the `0x690`/`0x6d0` cache offsets are intact by construction.

## Other modules

Same 362-module set as 6.0.11. 27 changed md5. 26 are `.text`-identical to 6.0.11 and differ only in build-id and Jenkins job path.

`qca-nss-dp.ko` has a code change (`.text` 71,876 → 71,860 bytes). Per-function disassembly comparison finds 3 changed functions, all in the EDMA loopback ring code that 6.0.10 added:

- `edma_cfg_rx_loopback_mapping`: now reads the queue-map byte per ring index and computes each ring's start from that index. 6.0.11 read one fixed byte for all rings.
- `edma_cfg_rx_loopback_rings` and `edma_cfg_rx_loopback_rings_alloc`: `__LINE__` constants in log calls shift by +5. No logic change.

The SGMII+ module does not use `qca-nss-dp`.

## Userland

15 firmware packages changed version against 6.0.11. None of `python3`, `systemd`, `syslog-ng`, `mongod` changed. Notable: `unifi-core` 6.0.123 → 6.0.125, `ustd` 6.0.9 → 6.0.10, `qca-nss-dp-ipq9574` (above), new revisions of `unifi-hal`, `udapi-server`/`libudapi`, `libubnt` and `mcagent`. Debian security update: `libpng16-16t64` deb13u5 → u6.

`ustd` 6.0.10 changed md5 on every `.so`. `ufcd`, `uhwd` and the `statusdb` modules checked differ by build-id only, except `sdb_client.so`: its `.text` grew 143,036 → 143,176 bytes. Its exported symbols and string set are unchanged, and `SDBClient.run`, `get` and `update` (the three calls script 15 makes) are present. The internal change cannot be read statically (Cython). A live run of script 15 confirms the API.

No package was added or removed. The unit set and `/etc/cron.d` are identical to 6.0.11.

## PostgreSQL (08 + 09)

PostgreSQL packages are unchanged: `postgresql-14` 14.24, `postgresql-common` 293, `libpq5` 18.6. These files are identical to 6.0.11:

- `/etc/default/postgresql/14-apps`. It still defines `PG_VERSION=14`, `DIR=/data/postgresql/14/apps`, `CLUSTER_NAME="apps"`, `CLUSTER_PORT=5434`, which is the layout 08 and 09 require.
- `/sbin/pg-cluster-setup` (the `rm -rf` / `initdb` path that 08 avoids by binding only `data/`).
- `usr/lib/postgresql/14/`, `usr/share/postgresql-common/` (`pg_wrapper`, `pg_ctlcluster`), `/etc/postgresql/`, `/etc/postgresql-common/`.
- All systemd unit trees, including `postgresql@.service`, `postgresql-cluster@.service` and the `14-apps` `limits.conf` drop-ins.
- `runuser`, `flock`, `timeout`, `mountpoint`, `findmnt`.

`ubnt-device-info` is a symlink to `ubios-udapi-server`, which changed with `udapi-server`. The `model_short` subcommand string is still present. The `/proc/ubnthal/system.info` fallback comes from `ubnthal.ko`, which is `.text`-identical.

No firmware component other than the env file and the drop-ins references the `14-apps` cluster, in 6.0.11 or 6.0.12. So the firmware adds no database that would trip the shared-cluster refusal in 08.

## Live

### Gateway A: UCG-Fiber without SSD (read-only)

On 6.0.12 (kernel banner matches the image), checked on the first boot after the upgrade. No SSD, so 06 to 09 are not deployed. The Network app runs in PostgreSQL mode. Deployed 10 and 15 are byte-identical to repo HEAD. `udm-boot` `ExecMainStatus=0`.

| Tweak | Live |
|---|---|
| 10 journald | ✓ `Storage=volatile`, `ForwardToSyslog=no`, 19M in `/run/log/journal`. Boot log: local log routes disabled in 10 `conf.d` files, `syslog-ng` restarted. |
| 15 fan | ✓ API, write and reload. Actuator inconclusive (see below). |

Script 15 on this boot, against `ustd` 6.0.10:

- **API:** `BEFORE` stock (100/68/109/103), `AFTER` tuned (65/55/85/90), 0 `ERROR` lines. `SDBClient` `run`/`get`/`update` work on the changed `sdb_client.so`.
- **Write:** SDB `config.fan` read back hours later holds 65/55/85/90.
- **Reload:** `ufcd` `ActiveEnterTimestamp` is the same second as the script's SDB write, after the stop the script issued. `NRestarts=0`. `fan_ctrl_sm` mapped in `ufcd` (4).
- **Actuator:** CPU 57.1 °C (tuned 65), rtl8372 78 °C (tuned 85), rtl8261 60 °C (tuned 90). Every component is below its tuned setpoint, so pwm 38 / ~1,845 rpm (3 samples) is the correct floor under both tuned and stock config. This observation cannot tell them apart. The gateway was not perturbed.

### Gateway B: UCG-Fiber with SSD

On 6.0.12, read from the Network Optimizer deployment status page (no SSH session). Network app in PostgreSQL mode.

| Tweak | Live |
|---|---|
| 08 PostgreSQL offload | ✓ Bind mount active from `/dev/md3[/postgresql-14-apps]`, authority marker present, PostgreSQL running |
| 09 PostgreSQL backup | ✓ Active, a daily backup completed on 6.0.12 |
| 10 journald | ✓ `Storage` volatile, `ForwardToSyslog` disabled, all eMMC log routes disabled, IDS/IPS threat pipeline active |
| 15 fan | ✓ `ufcd` running, last run OK. **Actuator:** CPU die 69 °C, above the tuned 65 °C and below the stock 100 °C. The fan runs at PWM 68 / 3,081 rpm, above the PWM 38 floor that stock setpoints would hold. The tuned setpoints are driving the fan loop on 6.0.12. |

This closes the actuator link that gateway A could not show.

## Outstanding

- 6.0.11 and earlier compat docs did not check 08 + 09. The 6.0.10 → 6.0.11 upgrade is field-verified with 08 offloaded (see README).

Reference `.ko`: `research/qca-ssdk-compare/qca-ssdk-6.0.12.ko`. Working files: `~/fw-6012/` on the NAS.
