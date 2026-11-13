# Getting started

> **Note:** valkey-large-object is in pre-release.
> Commands, arguments, configuration, and replies can change before the first stable release.

valkey-large-object is a Valkey module that stores large binary values (for example model data or cache blocks for machine learning inference) in DRAM or in files on a local disk.
Clients send and get the values over the normal Valkey connection, or with RDMA directly to and from client memory.

This page shows how to build the module, load it into Valkey, and run your first commands.
For the command reference, see [COMMANDS.md](COMMANDS.md).

## Contents

- [Terms](#terms)
- [Requirements](#requirements)
- [Build](#build)
- [Load the module](#load-the-module)
- [Verify the module](#verify-the-module)
- [First commands](#first-commands)
- [Behavior to know](#behavior-to-know)
- [Troubleshooting](#troubleshooting)
- [Next steps](#next-steps)

## Terms

| Term | Meaning |
|---|---|
| Segment | A large block of DRAM that the module reserves at one time. The module stores objects in segments. All segments have the size `segment-size`. |
| `Dram` mode | All objects are in DRAM segments. This is the default `operating-mode`. |
| `Tiered` mode | Each object is a file on disk, and DRAM segments are a read cache. |
| Promote | In `Tiered` mode, copy an object from disk into the DRAM cache. |
| Disk staging buffers | DRAM segments that the module uses for disk reads and writes in `Tiered` mode. |
| Fabric | The [libfabric](https://ofiwg.github.io/libfabric/) network layer that the RDMA commands use. A fabric domain is one network device as libfabric names it, for example `lo` (loopback) or an EFA device. The module starts one fabric service for each domain. |

## Requirements

- **Valkey 8.0 or later.** Get it from [valkey.io/download](https://valkey.io/download/), or build it from [source](https://github.com/valkey-io/valkey).
  The project builds and tests against the Valkey `unstable` branch.
- **Linux with io_uring.** In `Tiered` mode, the module uses io_uring for disk I/O.
- **Rust.** A recent stable toolchain with `cargo`, for example from [rustup](https://rustup.rs/).
- **Build tools.** `clang`, `libclang-dev` (for bindgen), `build-essential` (or `gcc` and `make`), and `pkg-config`.
- **libfabric 2.0 or later, with development files.** The build needs libfabric also when you use only the TCP commands.
  The module uses the libfabric 2.0 interface (symbol version `FABRIC_1.8`), and the build needs the header `rdma/fi_collective.h`.
  An older libfabric, for example 1.8.0, fails (see [Troubleshooting](#troubleshooting)).
  The project is tested with libfabric 2.1.0, and CI uses the libfabric from the [EFA installer](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start.html) 1.51.0.
  The build finds libfabric with `pkg-config`; check with `pkg-config --modversion libfabric`.
- **EFA hardware for production RDMA.** The default fabric provider, `EfaDirect`, uses [Elastic Fabric Adapter (EFA)](https://aws.amazon.com/hpc/efa/) hardware.
  Without RDMA hardware, use the TCP commands.
  To develop RDMA client code without RDMA hardware, use `fabric-provider Emulated` (see [COMMANDS.md](COMMANDS.md#rdma-overview)).
- **Locked-memory limit.** The module registers (pins) each segment one time for each fabric domain, and one more time for io_uring in `Tiered` mode.
  This counts against `RLIMIT_MEMLOCK`.
  Set the limit to `unlimited` (for example `ulimit -l unlimited`, or `LimitMEMLOCK=infinity` in a systemd unit).
  Else set it to at least the total segment bytes (DRAM pool and disk staging buffers) times the number of registrations.
- **Tiered mode only: a file system with user extended attributes.** `disk-dir` must be on a file system that supports `O_DIRECT` and user extended attributes (`user.*`), for example ext4 or XFS.
  The module stores a small header for each object file in the `user.lobj` extended attribute.
  The module does not check this at load time.
  On a file system without user extended attributes (for example tmpfs on older kernels), every write fails with `ERR disk write failed`.
- **Tiered mode only: open-file limit.** For all features, the open-file limit must be at least `2 x maxclients + 128 + max-cached-fds`.
  With the defaults (`maxclients` 10000, `max-cached-fds` 16384), this is 36512.
  Raise the limit with `ulimit -n` in the shell or `LimitNOFILE` in a systemd unit, or set a lower `maxclients`.
  Below `2 x maxclients + 128`, the module loads with the file descriptor cache off.
  Between that value and the full value, the cache is smaller.
  The server log shows the limit to set (see [Troubleshooting](#troubleshooting)).

## Build

```bash
git clone https://github.com/valkey-io/valkey-large-object
cd valkey-large-object
cargo build --release
```

The build writes the module to `target/release/libvalkey_large_object.so`.

If libfabric is not in a default location, or the default libfabric is older than 2.0, give the `pkg-config` directory of the newer one.
If bindgen cannot find the standard C headers (for example `stddef.h`), add the GCC include directory.
You can set both:

```bash
PKG_CONFIG_PATH=/opt/amazon/efa/lib/pkgconfig \
BINDGEN_EXTRA_CLANG_ARGS="-I$(gcc -print-file-name=include)" \
cargo build --release
```

`./build.sh build` runs the format and lint checks, the release build, and the unit tests.
`./build.sh` with no arguments also runs the integration tests.
See the [README](../README.md#test).

If you set `PKG_CONFIG_PATH`, also set `LD_LIBRARY_PATH` to the `lib` directory of the same libfabric.
You need it when you run `./build.sh`, `valkey-server`, or the test peer `fabric_target`.

## Load the module

Module arguments are pairs of configuration names and values.

- At load time, use the names without a prefix, for example `segment-size`.
- At run time, the names have the `largeobj.` prefix, for example `largeobj.segment-size`.
- **In module arguments, give memory sizes in bytes.**
  A value with a unit, such as `64mb`, stops the load.
  The server log then shows `invalid digit found in string`, and the server stops.
- `segment-size` must be at least twice `chunk-size` (default 1 MiB).
- Values such as `Dram`, `Tiered`, `EfaDirect`, and `Emulated` are case-sensitive.

### Dram mode

All objects are in DRAM.
This example uses 64 MiB (67108864 bytes) segments.
In `Dram` mode an object must fit in one segment, with 4 KiB of overhead for each `chunk-size` (default 1 MiB) part of the object, plus 4 KiB.
For 64 MiB segments, the largest `max-object-size` is 66842624.
This example uses 32 MiB (33554432 bytes).

```bash
valkey-server --port 6379 \
  --maxmemory 4gb \
  --loadmodule ./target/release/libvalkey_large_object.so \
    operating-mode Dram \
    segment-size 67108864 \
    max-object-size 33554432
```

The DRAM limit is the core Valkey `maxmemory` setting.
When a fabric is up, `max-rdma-registered-bytes` (default 380 GiB) also limits the DRAM pool.

### Tiered mode

Each object is stored in a file in `disk-dir`, and DRAM is a read cache.
A promoted object must fit in one segment, so `max-promote-size` (default 256 MiB) must be smaller than `segment-size`.
This example uses 64 MiB segments, a 32 MiB maximum object size and promotion size, a 10 GiB disk limit, and 64 MiB of disk staging buffers.
Change `disk-dir` to a directory on your disk.

```bash
valkey-server --port 6379 \
  --maxmemory 4gb \
  --loadmodule ./target/release/libvalkey_large_object.so \
    operating-mode Tiered \
    disk-dir /mnt/nvme/largeobj \
    segment-size 67108864 \
    max-object-size 33554432 \
    max-promote-size 33554432 \
    disk-maxmemory 10737418240 \
    disk-staging-size 67108864
```

> **Warning:** In `Tiered` mode, at startup and at shutdown, the module deletes the `disk-dir` directory and everything in it, then creates it again, empty.
> It deletes all files and subdirectories, not only object files.
> Use a dedicated subdirectory that is only for this module, for example `/mnt/nvme/largeobj`.
> Do not use a mount point.
> If `disk-dir` is a mount point, the module first deletes all contents of the mounted file system, then cannot remove the mount point (`EBUSY`), and the module does not load.
> The parent directory of `disk-dir` must be writable by the server, because the module removes and creates `disk-dir`.

The module makes `disk-dir-shards` subdirectories in `disk-dir` when it starts (default 1024, named `0000` to `03ff`).

### Load from valkey.conf

Add a `loadmodule` line to `valkey.conf`:

```
loadmodule /path/to/libvalkey_large_object.so operating-mode Dram segment-size 67108864 max-object-size 33554432
```

You can also put each configuration on its own line after the `loadmodule` line, with the `largeobj.` prefix.
On these lines, memory sizes can have units:

```
loadmodule /path/to/libvalkey_large_object.so
largeobj.segment-size 64mb
largeobj.max-object-size 32mb
```

### Fabric provider

By default the module tries to start the `EfaDirect` fabric provider.
On a host without EFA, the module still loads, and only the TCP commands work.
The server log then shows:

```
largeobj: fabric unavailable, RDMA path disabled: fabric error: fi_getinfo: No data available (-61). Only Large Object Commands of the TCP variant will be supported
```

To try the RDMA commands on a host without RDMA hardware, add `fabric-provider Emulated fabric-interfaces lo`.
See [COMMANDS.md](COMMANDS.md#try-rdma-without-rdma-hardware).

## Verify the module

The module name is `largeobj`.

```
127.0.0.1:6379> MODULE LIST
1) 1) "name"
   2) "largeobj"
   3) "ver"
   4) (integer) 1
   ...
```

All module commands are in the ACL category `largeobj`:

```
127.0.0.1:6379> COMMAND LIST FILTERBY ACLCAT largeobj
1) "BLOB.RDMA_HELLO"
2) "BLOB.TCP_GET"
3) "BLOB.TCP_SET"
4) "BLOB.INFO"
5) "BLOB.RDMA_GET"
6) "BLOB.RDMA_SET"
```

The order of the list can be different.

## First commands

Store a value, read it, and get its metadata:

```
127.0.0.1:6379> BLOB.TCP_SET greeting hello
OK
127.0.0.1:6379> BLOB.TCP_GET greeting
"hello"
127.0.0.1:6379> BLOB.INFO greeting
1) len
2) (integer) 5
3) crc
4) (integer) 2591144780
5) tier
6) dram
127.0.0.1:6379> BLOB.INFO greeting LEN
(integer) 5
127.0.0.1:6379> BLOB.TCP_GET missing
(nil)
```

`crc` is the CRC32C (Castagnoli) checksum of the bytes `hello`.
In `Tiered` mode, `tier` is `disk` until a read promotes the object to DRAM.
With the default `promote-min-hits` 1, the first read promotes it:

```
127.0.0.1:6379> BLOB.TCP_SET greeting hello
OK
127.0.0.1:6379> BLOB.INFO greeting TIER
disk
127.0.0.1:6379> BLOB.TCP_GET greeting
"hello"
127.0.0.1:6379> BLOB.INFO greeting TIER
dram
```

To store a file, use the `-x` option of `valkey-cli`.
It reads the last argument from standard input:

```bash
valkey-cli -x BLOB.TCP_SET model:embeddings:v1 < embeddings.bin
```

Delete an object with the core `DEL` command:

```
127.0.0.1:6379> DEL greeting
(integer) 1
127.0.0.1:6379> BLOB.INFO greeting
(nil)
```

See the module metrics with `INFO largeobj`:

```
127.0.0.1:6379> INFO largeobj
# largeobj_core_metrics
largeobj_num_objects:0
largeobj_inflight_requests:0

# largeobj_evictions
largeobj_evictions:0
largeobj_pending_evictions:0
...
```

`INFO largeobj` shows all fields, in sections for evictions, DRAM, disk, RDMA, and errors.
Some important fields:

| Field | Meaning |
|---|---|
| `largeobj_num_objects` | Number of large objects. |
| `largeobj_evictions` | Number of large object keys evicted. |
| `largeobj_dram_allocated_bytes`, `largeobj_dram_capacity_bytes` | DRAM in use by objects, and DRAM in segments. |
| `largeobj_disk_used_bytes` | `Tiered` mode: disk space in use. |
| `largeobj_disk_metadata_spills` | `Tiered` mode: object files that take disk blocks outside their inode, in addition to their data. For example, on ext4, the file of a key with a name longer than about 40 bytes uses one more 4 KiB block. These blocks count in `largeobj_disk_used_bytes`. |
| `largeobj_rdma_sessions` | Open RDMA sessions. |
| `largeobj_disk_write_errors`, `largeobj_rdma_read_errors`, `largeobj_crc_mismatch` | Error counters. |

## Behavior to know

### No persistence

The module does not keep object data across a restart or a reload, in `Dram` or `Tiered` mode.

- `SAVE`, `BGSAVE`, and `SHUTDOWN` with save points succeed, but the RDB file has only the key names, without the object data.
- `BGREWRITEAOF` keeps no object data.
  With `aof-use-rdb-preamble yes` (the default), the rewritten AOF has the key names, as with `SAVE`.
  With `aof-use-rdb-preamble no`, it has no large object keys.
- After the server loads an RDB file, each large object key reads as missing.
  The module starts to delete these keys within `scaling-poll-ms` (default 2.5 seconds) after the load.
  With many keys, this can take longer.
  Until then, `EXISTS` and `DBSIZE` still count them.
- `DUMP` returns a value without the object data.
  `RESTORE` of that value fails with `ERR Bad data format`.
  Thus `MIGRATE` of a large object key fails, and the key stays on the source server.
- The module does not replicate large objects.

Example after `SAVE`, `SHUTDOWN`, and a restart:

```
127.0.0.1:6379> EXISTS mykey
(integer) 1
127.0.0.1:6379> BLOB.TCP_GET mykey
(nil)
127.0.0.1:6379> EXISTS mykey
(integer) 1
```

A few seconds later:

```
127.0.0.1:6379> EXISTS mykey
(integer) 0
```

### Transactions and scripts

`BLOB.TCP_SET`, `BLOB.TCP_GET`, `BLOB.RDMA_SET`, and `BLOB.RDMA_GET` are not permitted in `MULTI`/`EXEC`, Lua scripts, functions, or module calls.
In a transaction, only that command fails, and the other commands run.
In this example, the queued `BLOB.TCP_SET` does not run, so `mykey` keeps its 11-byte value:

```
127.0.0.1:6379> BLOB.TCP_SET mykey "hello world"
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> BLOB.TCP_SET mykey "new value"
QUEUED
127.0.0.1:6379(TX)> BLOB.INFO mykey LEN
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (error) ERR command not allowed in MULTI, scripts or module calls
2) (integer) 11
127.0.0.1:6379> BLOB.TCP_GET mykey
"hello world"
```

`BLOB.INFO` and `BLOB.RDMA_HELLO` are permitted there.

### Turn off the module without a restart

You cannot unload the module with `MODULE UNLOAD`, because it registers a data type.
To stop clients from using the module, set `largeobj.enabled` to `no`.
Stored keys stay, and core commands such as `TYPE`, `EXISTS`, and `DEL` still work on them:

```
127.0.0.1:6379> BLOB.TCP_SET mykey "hello world"
OK
127.0.0.1:6379> CONFIG SET largeobj.enabled no
OK
127.0.0.1:6379> BLOB.TCP_GET mykey
(error) DISABLED largeobj module is disabled
127.0.0.1:6379> TYPE mykey
largeob-k
127.0.0.1:6379> CONFIG SET largeobj.enabled yes
OK
127.0.0.1:6379> BLOB.TCP_GET mykey
"hello world"
```

To remove the module, stop the server and start it without the `loadmodule` argument.

### Eviction

A write can evict other large objects to make space.
The core `maxmemory-policy` setting controls which objects the module can evict:

| Policy | Objects that the module can evict |
|---|---|
| `allkeys-*` | Any large object. |
| `volatile-*` | Only large object keys with a TTL. |
| `noeviction` | None. The write fails with an `OOM` error. |

The module evicts in least-recently-used (LRU) order for all policies.
A read and a TTL change count as a use.
For the `*-lfu`, `*-random`, and `volatile-ttl` policies, the module also uses LRU order, and the server log shows a warning.

- `Dram` mode: eviction occurs only when `maxmemory` is set.
- `Tiered` mode: eviction removes object files when `disk-maxmemory` is larger than 0.
  `maxmemory` is not necessary for disk eviction.
  When the DRAM cache is full, the module drops cached copies; the objects stay on disk.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Build fails. Log: `fatal error: fi_collective.h: No such file or directory` | The libfabric that `pkg-config` finds is older than 2.0. Install libfabric 2.0 or later, and set `PKG_CONFIG_PATH` to its `lib/pkgconfig` directory (see [Build](#build)). |
| Server stops at load, or `./build.sh` or `fabric_target` fails. Log: ``libfabric.so.1: version `FABRIC_1.8' not found`` | An older libfabric is loaded at run time. Set `LD_LIBRARY_PATH` to the `lib` directory of the libfabric that you built with. |
| Server stops at load. Log: `invalid digit found in string` | A module argument has a unit, for example `64mb`. Give the size in bytes. |
| Server stops at load. Log: `Enum 'dram' not exists` | Enum values are case-sensitive. Use `Dram`, `Tiered`, `EfaDirect`, or `Emulated`. |
| Server stops at load. Log: `largeobj.segment-size is insufficient for largeobj.max-object-size` | `Dram` mode: an object of `max-object-size` does not fit in one segment (default `max-object-size` is 512 MiB). Make `max-object-size` smaller or `segment-size` larger (see [Dram mode](#dram-mode)). |
| Server stops at load. Log: `largeobj.segment-size must be >= 2 * largeobj.chunk-size` | `segment-size` is less than twice `chunk-size` (default 1 MiB). Make `segment-size` larger, for example 2 MiB (2097152) or more. |
| Server stops at load. Log: `largeobj.segment-size is insufficient for largeobj.max-promote-size` | `Tiered` mode: `max-promote-size` (default 256 MiB) is not smaller than `segment-size`. Make `max-promote-size` smaller. |
| Server stops at load. Log: `IORING_REGISTER_BUFFERS failed: Cannot allocate memory (os error 12)` | `Tiered` mode: the locked-memory limit is too low. Raise `RLIMIT_MEMLOCK` (see [Requirements](#requirements)). |
| Server aborts at load or when the DRAM pool grows. Log: `largeobj: fatal panic — aborting server: ... EFA registration of segment at ... failed (fatal)` | A fabric is up, and the module cannot register a segment. Raise `RLIMIT_MEMLOCK` (see [Requirements](#requirements)), or set a lower `max-rdma-registered-bytes`. |
| Log: `fabric unavailable, RDMA path disabled`. `BLOB.RDMA_HELLO` returns `ERR RDMA provider unavailable on this instance` | The fabric provider did not start. Use the TCP commands, or check the EFA setup. For tests without RDMA hardware, use `fabric-provider Emulated`. |
| Log: `fd-reserve Lenient: open-file limit ... is too low, so fd placement and the fd cache are off; set LimitNOFILE to at least <n> to turn them on` | `Tiered` mode: raise the open-file limit (`ulimit -n` or `LimitNOFILE`) to the value in the message, or set a lower `maxclients`. With `fd-reserve Strict`, the module does not load in this case. |
| Log: `open-file limit ... is too low for max-cached-fds ...: the fd cache holds <m>; set LimitNOFILE to at least <n> to cache all` | `Tiered` mode: the file descriptor cache is smaller than `max-cached-fds`. Raise the open-file limit, or set a lower `max-cached-fds`. |
| Every `Tiered` write fails with `ERR disk write failed` | `disk-dir` does not support user extended attributes or `O_DIRECT`. Use ext4 or XFS. |
| `ERR command not allowed in MULTI, scripts or module calls` | See [Transactions and scripts](#transactions-and-scripts). |
| `DISABLED largeobj module is disabled` | `largeobj.enabled` is `no`. |

## Next steps

- [COMMANDS.md](COMMANDS.md): all commands, the RDMA flow, and the configurations that you are most likely to change.
- [README](../README.md): architecture, the full configuration table, tests, and benchmarks.
- [STORAGE_DESIGN.md](STORAGE_DESIGN.md) and [CACHE_POLICY_DESIGN.md](CACHE_POLICY_DESIGN.md): design details.
