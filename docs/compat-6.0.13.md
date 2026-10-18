# UniFi OS 6.0.13 Compatibility Verification

**Result: compatible.** Static check of the UCG-Fiber image on 2026-10-18. No live check.

| | |
|---|---|
| Build | `UCGF.ipq9574.v6.0.13.7a9bdc5.261013.0803` |
| `.bin` md5 | `4af24999198f17dc66347e9c1c167019` (953,240,647 bytes) |
| Rootfs | zstd squashfs at offset `16083710`, 52,429 inodes |
| Kernel | `#5.4.213 SMP PREEMPT Tue Oct 13 08:05:11 CST 2026` (rebuilt), GCC 14.2.0-19 |
| vermagic | `5.4.213-ui-ipq9574 SMP preempt mod_unload aarch64`, unchanged. No module rebuild. |
| Baseline | 6.0.12 UCG-Fiber image |

| Tweak | Static (UCG-Fiber image) |
|---|---|
| 19 + 20 SGMII+ | ✓ `qca-ssdk.ko` byte-identical to 6.0.12 |
| 05 JVM heap | ✓ `/etc/default/unifi` and `unifi.service` identical to 6.0.12 (`UNIFI_NATIVE_OPTS`, `UNIFI_NATIVE_OVERRIDE_OPTS` unchanged) |
| 06 + 07 MongoDB | ✓ `mongod`, `mongodump`, `unifi-mongodb.service`, `/etc/default/unifi` identical to 6.0.12 |
| 08 + 09 PostgreSQL | ✓ `/etc/default/postgresql/14-apps`, `pg-cluster-setup`, PostgreSQL 14 binaries, configs and units identical to 6.0.12. No new `14-apps` reference |
| `mongodb-ssd-decommission.sh` | ✓ Its dependencies are covered by the 06 + 07 and 08 + 09 rows |
| 10 journald | ✓ `journald.conf` and `syslog-ng/` identical to 6.0.12 |
| 15 fan | ✓ `ufcd.service` identical, `ufcd`/`uhwd` `.text`-identical to 6.0.12. `sdb_client.so` `.text` is byte-identical to 6.0.11 (see below) |

## Kernel

The decompressed kernel `Image` is the same size as 6.0.12 (13,395,976 bytes). Only 99 bytes differ: the two build-timestamp banners, the build-id, the initramfs cpio mtimes, and one pointer that moves by 1 because the new banner string is one character longer. The device tree is byte-identical to 6.0.12. This is a rebuild with no code change.

## Modules

Same 362-module set as 6.0.12. All 362 are byte-identical to 6.0.12, including `qca-ssdk.ko` (md5 `1906a5af6bf1b0a3f8dfeeebe9ba4f58`). The symbols and the `0x690`/`0x6d0` cache offsets are intact by construction.

## Userland

10 firmware packages changed version against 6.0.12. None of `python3`, `systemd`, `syslog-ng`, `mongod`, PostgreSQL changed. Notable: `unifi-core` 6.0.125 → 6.0.127, `ulp-go` 1.14.7 → 1.14.8, `uos-discovery-client` 6.0.6 → 6.0.7, new revisions of `udapi-server`/`libudapi`, `libubnt`, `mcagent` and `ustd` (version still 6.0.10). Debian security update: `xz-utils`/`liblzma5` deb13u1 → u2.

The `ustd` rebuild changed one module's code: `sdb_client.so` `.text` 143,176 → 143,036 bytes. That `.text` is byte-identical to the 6.0.11 `sdb_client.so` (`ustd` 6.0.9), so 6.0.13 rolls back the 6.0.12 change. Script 15 ran live against that code on 6.0.11, so the `SDBClient` `run`/`get`/`update` API is intact by construction. All other `statusdb` modules are `.text`-identical to 6.0.12.

`ubnt-device-info` still links to `ubios-udapi-server`, which changed with `udapi-server`. The `model_short` string is present. `ubnthal.ko` is byte-identical.

No package was added or removed. The unit set and `/etc/cron.d` are identical to 6.0.12.

## Outstanding

- 6.0.12 and earlier compat docs did not check 05.

No new reference `.ko` (byte-identical to `research/qca-ssdk-compare/qca-ssdk-6.0.12.ko`). Working files: `~/fw-6013/` on the NAS.
