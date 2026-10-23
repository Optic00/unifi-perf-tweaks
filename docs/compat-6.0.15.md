# UniFi OS 6.0.15 Compatibility Verification

**Result: compatible.** Static check of the UCG-Fiber image on 2026-10-23. No live check.

| | |
|---|---|
| Build | `UCGF.ipq9574.v6.0.15.404d052.261022.1754` |
| `.bin` md5 | `b1dd90eb126e9d8fbbd31cd1bcf781d0` (954,858,664 bytes) |
| Rootfs | zstd squashfs at offset `16083866`, 52,425 inodes |
| Kernel | `#5.4.213 SMP PREEMPT Thu Oct 22 17:59:40 CST 2026` (rebuilt), GCC 14.2.0-19 |
| vermagic | `5.4.213-ui-ipq9574 SMP preempt mod_unload aarch64`, unchanged. No module rebuild. |
| Baseline | 6.0.13 UCG-Fiber image |

| Tweak | Static (UCG-Fiber image) |
|---|---|
| 19 + 20 SGMII+ | ✓ `qca-ssdk.ko` code, data and symbol table identical to 6.0.13 |
| 06 + 07 MongoDB | ✓ `mongod`, `mongodump`, `unifi-mongodb.service`, `/etc/default/unifi` identical to 6.0.13 |
| 08 + 09 PostgreSQL | ✓ `/etc/default/postgresql/14-apps`, `pg-cluster-setup`, PostgreSQL 14 binaries, configs and units identical to 6.0.13. No new `14-apps` reference |
| `mongodb-ssd-decommission.sh` | ✓ Its dependencies are covered by the 06 + 07 and 08 + 09 rows |
| 10 journald | ✓ `journald.conf` and `syslog-ng/` identical to 6.0.13 |
| 15 fan | ✓ `ufcd.service` identical. `ufcd`, `uhwd`, `usdbd`, `sdb_client.so`, `fan_ctrl.so` and `fan_ctrl_sm.so` `.text`-identical to 6.0.13. `statusdb/config.so` changed (see below) |

## Kernel

The decompressed kernel `Image` is the same size as 6.0.13 (13,395,976 bytes), but unlike 6.0.11 through 6.0.13 this rebuild changes kernel code. The device tree is byte-identical to 6.0.13.

A per-function comparison of all 36,164 kallsyms entries, with PC-relative immediates masked, finds one changed function: `skb_recycler_alloc` (the QSDK skb recycler) grows from 608 to 620 bytes. The 3 new instructions set `skb->truesize = SKB_TRUESIZE(skb_end_offset(skb))` on a recycled skb. In 6.0.13 this function does not write `truesize`.

The 12-byte growth shifts all later `.text` by 12 bytes, up to the alignment padding after `__arch_clear_user`. That shift accounts for the 1.5M differing bytes: branch and `adrp`/`:lo12:` immediates, and 17,755 kallsyms offsets that grow by exactly 12. The only other differences are the build banners, the build-id and the initramfs cpio mtimes.

The change is in the core network stack, outside the SGMII+ path. No tweak touches skb allocation.

## Modules

Same 362-module set as 6.0.13. 29 modules changed md5, and all 29 are `.text`-identical to 6.0.13. `qca-ssdk.ko` is md5 `e4fd1d4e360a0c4ceaf74598dbda3114`. Its 477 differing bytes are the build timestamp, the Jenkins workspace paths, `.modinfo` and the build-id. `.text`, `.rodata`, `.data`, `.bss`, `__versions` and the symbol table are identical, so the symbols and the `0x690`/`0x6d0` cache offsets are intact by construction.

## Userland

Same 583 packages as 6.0.13, none added or removed. None of `python3`, `systemd`, `syslog-ng`, `mongod` or PostgreSQL changed. Notable: `unifi-core` 6.0.127 → 6.0.130, `ubnt-tools` 6.0.20 → 6.0.21, `ustd` 6.0.10 → 6.0.11, the identity stack (`ucs-agent`, `unifi-credential-server`, `unifi-directory`, `unifi-identity-update`), `udapi-bridge` 0.34 → 0.35, `node24` 24.15.0 → 24.22.1. Debian security updates: `bind9` 9.20.29 → 9.20.30, `curl`/`libcurl` deb13u5 → u7, `libexpat1` 2.8.3 → 2.9.0, `ca-certificates` deb13u1.

`ustd` 6.0.11 changed code in four Python modules: `statusdb/config.so`, `hwmon/disk_power_event_config.so`, `hwmon/disk_power_event_sm.so` and `storage/disk_layout.so`. `config.so` `.text` grows 169,508 → 175,816 bytes and adds one function, `read_default_config`, next to the existing `get_default_fan_config`. Script 15 does not import `config`. It uses `SDBClient` `run`/`get`/`update`, and `sdb_client.so` is `.text`-identical to 6.0.13 (the 6.0.11 code, where script 15 ran live). The API is intact by construction. If the new function re-reads default fan tables at runtime, SDB `config.fan` could revert to stock between boots. The first live check on 6.0.15 must confirm that SDB `config.fan` still holds the tuned values.

`ubnt-device-info` changed with `ubnt-tools`. The `model_short` string is present. `ubnthal.ko` is byte-identical.

The unit set and `/etc/cron.d` are identical to 6.0.13.

Reference `.ko`: `research/qca-ssdk-compare/qca-ssdk-6.0.15.ko`. Working files: `~/fw-6015/` on the NAS. The kallsyms parser for this kernel and the per-function compare are in `~/tools-6015/`.
