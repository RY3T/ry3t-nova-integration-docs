# RY3T Nova — Modbus TCP Integration Guide

This document describes the Modbus TCP interface of a **RY3T Nova** so an
external automation system (Loxone, a PLC, a building EMS, a PV controller,
etc.) can read its status and control it.

## What a Nova is, on the wire

A Nova is a **cluster of up to four miners presented as a single device**.
You talk to one Modbus endpoint; it reports the combined state of the cluster
plus a per-miner breakdown, and accepts one power target that it distributes
across its miners.

```text
        ┌─────────────────────────────────┐
        │   Your automation system        │
        │  (Loxone · PLC · EMS · PV …)    │
        └────────────────┬────────────────┘
                         │ Modbus TCP (port 502)
                         ▼
        ┌─────────────────────────────────┐
        │            RY3T Nova             │
        │   one device · 1–4 miners        │
        └─────────────────────────────────┘
```

A single-miner installation behaves identically — it is just a cluster with
`MinerCount = 1`.

You set one cluster-wide power target (`SetTargetPower_W`); the Nova decides
how many of its miners run and at what level to meet that target, and reports
the result back in the per-miner registers so you can see which miner is doing
what.

## Contents

- [Transport](#transport)
- [Controlling the Nova](#controlling-the-nova)
- [Value encoding](#value-encoding)
- [Address map at a glance](#address-map-at-a-glance)
- [Block 0 — Cluster status](#block-0--cluster-status)
- [Block 1 — Per-miner data](#block-1--per-miner-data)
- [Block 2 — Sensors & fans](#block-2--sensors--fans)
- [Block 3 — Pool & shares](#block-3--pool--shares)
- [Block 4 — Device identification](#block-4--device-identification)
- [Block 5 — Control](#block-5--control)
- [Block 6 — Per-hashboard detail](#block-6--per-hashboard-detail)
- [Status codes](#status-codes)
- [Error codes](#error-codes)
- [Versioning](#versioning)
- [Security](#security)

## Transport

| Setting        | Value                                                              |
|----------------|-------------------------------------------------------------------|
| Role           | Modbus TCP **Server** (Slave)                                      |
| Port           | `502`                                                              |
| IP address     | the IP of the Nova device                                         |
| Unit ID        | `1`                                                                |
| Register type  | Holding registers only                                            |
| Function codes | `FC03` read, `FC06` write single, `FC16` write multiple           |
| Base addressing| `0` (protocol address `0` = register `40001`)                     |

Register numbers in this document are **decimal protocol addresses**. If your
client counts registers from `40001`, add `40001`: register `10` is `40011`.

> **Don't write register numbers with a leading zero in code.** C, Java, Go,
> PHP and JavaScript treat `010` as an octal number, so a read of `010`
> actually fetches register `8`, and `015` fetches register `13`. Registers `0 – 7`
> and everything from `100` up are unaffected, so the mistake is easy to miss.

## Controlling the Nova

This is the short version — everything you need to start, stop, and set the
power of a Nova. The register reference below covers the rest (reading status,
sensors, errors, per-miner detail).

Control uses two **write** registers:

| Reg   | Name               | Write                                                                |
|-------|--------------------|----------------------------------------------------------------------|
| `500` | `MiningCommand`    | `1` = stop the cluster, `2` = start the cluster (`0` = ignore).       |
| `501` | `SetTargetPower_W` | Cluster power target, in watts.                                       |

The pattern is **set the target, then start** — write the power target to
`501`, then write the start command to `500`. After that, changing the power
target alone takes effect without another start command.

### Examples

```text
Start the cluster at 7000 W
  1. Write 7000  → register 501   (SetTargetPower_W)
  2. Write 2     → register 500   (MiningCommand = start)

Change the power target while running
  1. Write 9000  → register 501
  (no start command needed)

Stop the cluster
  1. Write 1     → register 500   (MiningCommand = stop)
  (the power target is remembered for the next start)
```

### Notes

- Writing `0` to `MiningCommand` (register `500`) is **ignored**, so an
  automation program can safely clear its command word between cycles without
  toggling the miner.
- `start` while already running, and `stop` while already stopped, are no-ops.
- A command is **accepted immediately**, but the physical change can take up
  to about **two minutes per miner** (miners ramp up/down gradually). Poll
  `SystemStatus` (register `0`) and the per-miner `MinerStatus` to confirm the
  cluster has reached the commanded state.
- The target you set is cluster-wide. The Nova distributes it across its
  miners and reports the resulting per-miner allocation in
  `Miner*.TargetPower_W`, and the combined draw in `ActualPower_W`. Targets
  between `NameplateMinPower_W` and `NameplateMaxPower_W` (registers `15`/`14`)
  are honoured. The Nova runs as few miners as it needs, so a low target runs
  one miner and idles the rest; only targets below `NameplateMinPower_W` idle
  the whole cluster.

To read what the cluster is doing, start with `SystemStatus` (register `0`),
`ActualPower_W` (`6`), and `TotalHashrate_THs` (`7 – 8`). The full register map
follows.

## Value encoding

| Type      | Width       | Notes                                                             |
|-----------|-------------|-------------------------------------------------------------------|
| `UINT16`  | 1 register  | Big-endian. Used for status codes, counts, watts, percentages.   |
| `UINT32`  | 2 registers | Big-endian, high word at the lower address.                       |
| `FLOAT32` | 2 registers | IEEE 754, big-endian, high word at the lower address.            |
| `STRING`  | N registers | ASCII, 2 characters per register, null-padded.                   |

Multi-register values always put the **high word at the lower address**
(commonly called big-endian word order / `ABCD`).

**"Not available" sentinels.** When a value isn't currently available (a
sensor that hasn't reported, an offline miner, a metric this hardware can't
measure), the register reads a sentinel: `0xFFFF` for a `UINT16`, and a quiet
NaN (`0x7FC00000`) for a `FLOAT32`. Treat these as "no reading" rather than a
real value of 65535 or NaN.

All power values are integer watts. Some power readings are **estimates**
where the hardware cannot measure draw directly; they are still reported in
the same registers.

## Address map at a glance

| Range         | Block                                        | Access |
|---------------|----------------------------------------------|--------|
| `0 – 31`      | [Cluster status](#block-0--cluster-status)   | read   |
| `100 – 227`   | [Per-miner data (4 × 32)](#block-1--per-miner-data) | read |
| `300 – 319`   | [Sensors & fans](#block-2--sensors--fans)    | read   |
| `400 – 419`   | [Pool & shares](#block-3--pool--shares)      | read   |
| `500 – 501`   | [Control](#block-5--control)                 | write  |
| `1000 – 1019` | [Device identification](#block-4--device-identification) | read |
| `2000 – 2511` | [Per-hashboard detail (optional)](#block-6--per-hashboard-detail) | read |

Any address not listed is reserved, reads `0x0000`, and has no side effects.

## Block 0 — Cluster status

The cluster as a whole — the view most integrations need.

| Reg     | Name                  | Type      | Description                                                                 |
|---------|-----------------------|-----------|-----------------------------------------------------------------------------|
| `0`     | `SystemStatus`        | `UINT16`  | Overall cluster state. See [Status codes](#status-codes).                   |
| `1 – 3` | `ErrorCodes`          | 3×`UINT16`| 48-bit error bitmask across the cluster. See [Error codes](#error-codes).   |
| `4`     | `MiningCommanded`     | `UINT16`  | What you last commanded: `0` = stopped, `1` = running.                      |
| `5`     | `TargetPower_W`       | `UINT16`  | The cluster power target currently in effect (W).                          |
| `6`     | `ActualPower_W`       | `UINT16`  | Best-known combined power draw of the cluster (W). May be estimated.        |
| `7 – 8` | `TotalHashrate_THs`   | `FLOAT32` | Combined hashrate of the cluster (TH/s).                                    |
| `9`     | `UptimeMinutes`       | `UINT16`  | Minutes since the cluster first started mining. Saturates at `0xFFFE`.      |
| `10`    | `MinerCount`          | `UINT16`  | Number of miners in this cluster (`1 – 4`).                                 |
| `11`    | `MinersOnline`        | `UINT16`  | Miners currently reachable and healthy.                                     |
| `12`    | `MinersMining`        | `UINT16`  | Miners currently producing shares.                                         |
| `13`    | `PoolConnected`       | `UINT16`  | `1` = every online miner is connected to its pool, `0` = at least one is not.|
| `14`    | `NameplateMaxPower_W` | `UINT16`  | Highest power target the cluster can accept (sum of online miners' max).    |
| `15`    | `NameplateMinPower_W` | `UINT16`  | Lowest non-zero power target the cluster will run at: the smallest minimum of any online miner (one miner running, the rest idle). Targets below this idle the whole cluster. |
| `16`    | `PrimaryMinerIndex`   | `UINT16`  | Index of the miner that serves this Modbus endpoint (`0`).                  |
| `17 – 31`| reserved             | —         |                                                                             |

## Block 1 — Per-miner data

Each miner occupies a 32-register block. Miner `n` starts at `100 + 32·n`:

| Miner | Range       |
|-------|-------------|
| 0     | `100 – 131` |
| 1     | `132 – 163` |
| 2     | `164 – 195` |
| 3     | `196 – 227` |

Slots for miners that don't exist (`n ≥ MinerCount`) still respond: their
`OnlineStatus` is `NotConfigured` and numeric fields read the not-available
sentinels. You can poll `100 – 227` unconditionally and learn the topology
from the data.

Offsets below are relative to the miner's block start.

| Offset | Name                  | Type      | Description                                                                |
|--------|-----------------------|-----------|----------------------------------------------------------------------------|
| `+0`   | `OnlineStatus`        | `UINT16`  | Reachability of this miner. See [Status codes](#status-codes).             |
| `+1`   | `MinerStatus`         | `UINT16`  | What this miner is doing (same codes as `SystemStatus`).                   |
| `+2 – +3` | `Hashrate_THs`     | `FLOAT32` | This miner's hashrate (TH/s).                                              |
| `+4`   | `TargetPower_W`       | `UINT16`  | Power the cluster has allocated to this miner (W).                        |
| `+5`   | `ActualPower_W`       | `UINT16`  | This miner's power draw (W). May be estimated.                            |
| `+6`   | `NameplateMaxPower_W` | `UINT16`  | This miner's maximum power.                                               |
| `+7`   | `NameplateMinPower_W` | `UINT16`  | This miner's minimum running power; below this it idles.                  |
| `+8 – +9` | `HotSpotTemp_C`    | `FLOAT32` | Hottest temperature on this miner (°C).                                   |
| `+10`  | `ChipsResponding`     | `UINT16`  | Mining chips currently responding on this miner.                          |
| `+11`  | `ChipsExpected`       | `UINT16`  | Total mining chips on this miner.                                         |
| `+12`  | `HashboardCount`      | `UINT16`  | Hashboards in this miner (`1 – 3`).                                       |
| `+13`  | `HashboardsActive`    | `UINT16`  | Hashboards currently producing shares.                                   |
| `+14`  | `UptimeMinutes`       | `UINT16`  | Minutes this miner has been mining. Saturates at `0xFFFE`.               |
| `+15`  | `LastContactSeconds`  | `UINT16`  | Seconds since the cluster last reached this miner (`0` for the primary). Saturates at `0xFFFE`. |
| `+16 – +17` | `MinerFirmware` | `UINT32`  | This miner's firmware version, packed as `major×1_000_000 + minor×1_000 + patch` (e.g. `1.4.7` → `1004007`). |
| `+18 – +23` | `SerialNumber`  | `STRING`  | This miner's serial (12 chars).                                          |
| `+24`  | `Model`               | `UINT16`  | Hardware model. See [Model codes](#model-codes).                         |
| `+25`  | `PoolStatus`          | `UINT16`  | This miner's pool-connection state. See [Pool status codes](#pool-status-codes). |
| `+26 – +27`| `PoolDifficulty` | `FLOAT32` | Share difficulty this miner's pool has set.                             |
| `+28 – +31`| reserved         | —         |                                                                          |

## Block 2 — Sensors & fans

Cluster-wide environmental aggregates.

| Reg         | Name              | Type      | Description                                          |
|-------------|-------------------|-----------|------------------------------------------------------|
| `300 – 301` | `HottestSensor_C` | `FLOAT32` | Hottest temperature anywhere in the cluster (°C).   |
| `302 – 303` | `CoolestSensor_C` | `FLOAT32` | Coolest temperature anywhere in the cluster (°C).   |
| `304`       | `AverageFanPct`   | `UINT16`  | Average fan duty across the cluster (`0 – 100`).    |
| `305`       | `FansRunning`     | `UINT16`  | Fans currently spinning.                            |
| `306`       | `FansTotal`       | `UINT16`  | Fans in the cluster.                                |
| `307 – 308` | `AmbientTemp_C`   | `FLOAT32` | Best estimate of intake/ambient temperature (°C).   |
| `309 – 319` | reserved          | —         |                                                      |

## Block 3 — Pool & shares

Mining-pool telemetry, aggregated across the cluster.

| Reg         | Name                  | Type      | Description                                                          |
|-------------|-----------------------|-----------|---------------------------------------------------------------------|
| `400 – 401` | `TotalSharesAccepted` | `UINT32`  | Accepted shares across the cluster.                                 |
| `402 – 403` | `TotalSharesRejected` | `UINT32`  | Rejected / stale shares across the cluster.                         |
| `404`       | `PoolStatus`          | `UINT16`  | Worst pool-connection state across online miners. See [Pool status codes](#pool-status-codes). |
| `405`       | `PoolLatency_ms`      | `UINT16`  | Typical share-submit round-trip (ms). `0xFFFF` if no shares yet.    |
| `406 – 407` | `BestDifficulty`      | `FLOAT32` | Best share difficulty the cluster has found.                       |
| `408 – 409` | `PoolDifficultyMin`   | `FLOAT32` | Lowest share difficulty set across miners.                         |
| `410 – 411` | `PoolDifficultyMax`   | `FLOAT32` | Highest share difficulty set across miners.                        |
| `412 – 419` | reserved              | —         |                                                                     |

## Block 4 — Device identification

| Reg           | Name              | Type      | Description                                          |
|---------------|-------------------|-----------|------------------------------------------------------|
| `1000 – 1009` | `ClusterName`     | `STRING`  | Operator-assigned name for this Nova (20 chars).    |
| `1010 – 1015` | `ClusterId`       | `STRING`  | Stable identifier for this Nova (12 chars).          |
| `1016`        | `FirmwareMajor`   | `UINT16`  | Controller firmware version.                        |
| `1017`        | `FirmwareMinor`   | `UINT16`  |                                                      |
| `1018`        | `FirmwarePatch`   | `UINT16`  |                                                      |
| `1019`        | `ApiVersion`      | `UINT16`  | Modbus map version. See [Versioning](#versioning).  |

Per-miner serials and firmware versions are in [Block 1](#block-1--per-miner-data).

## Block 5 — Control

Two write registers. Reading them returns `0`; the *effect* of a write is
visible in the read registers (`MiningCommanded`, `TargetPower_W`, and the
per-miner blocks).

| Reg   | Name              | Type     | Description                                                                 |
|-------|-------------------|----------|-----------------------------------------------------------------------------|
| `500` | `MiningCommand`   | `UINT16` | `0` = ignore, `1` = stop the cluster, `2` = start the cluster. See [Controlling the Nova](#controlling-the-nova). |
| `501` | `SetTargetPower_W`| `UINT16` | Cluster power target in watts (`0` to `NameplateMaxPower_W`). The Nova distributes it across its miners. |
| `502 – 559` | reserved    | —        | Reserved for future commands.                                              |

Fan and temperature control are managed automatically by the device and are
not exposed over Modbus.

## Block 6 — Per-hashboard detail

Optional, for diagnostics. Most integrations can ignore it — Block 1 already
aggregates hashboards into per-miner totals.

Address of miner `m`, hashboard `h`:

```
address = 2000 + (m × 128) + (h × 32)
```

So miner 0 hashboard 0 is at `2000`, miner 0 hashboard 2 at `2064`, miner 1
hashboard 0 at `2128`, and so on. Cells outside the actual hardware read as
`SlotStatus = Absent` with not-available sentinels.

| Offset      | Name              | Type      | Description                                              |
|-------------|-------------------|-----------|---------------------------------------------------------|
| `+0`        | `SlotStatus`      | `UINT16`  | Hashboard state. See [Slot status codes](#slot-status-codes). |
| `+1 – +2`   | `Hashrate_THs`    | `FLOAT32` | This hashboard's hashrate (TH/s).                       |
| `+3`        | `Power_W`         | `UINT16`  | This hashboard's power (W). May be estimated.           |
| `+4 – +5`   | `HotSpot_C`       | `FLOAT32` | Hottest temperature on this hashboard (°C).             |
| `+6`        | `ChipsResponding` | `UINT16`  | Chips currently responding on this hashboard.           |
| `+7`        | `ChipsExpected`   | `UINT16`  | Total chips on this hashboard.                          |
| `+8`        | `SharesLastHour`  | `UINT16`  | Shares from this hashboard in the last 60 min.          |
| `+9 – +10`  | `ChainVoltage_V`  | `FLOAT32` | Supply voltage to this hashboard (V).                   |
| `+11 – +12` | `CurrentMhz`      | `FLOAT32` | Current chip frequency (MHz).                           |
| `+13 – +15` | reserved          | —         |                                                         |
| `+16 – +21` | `SerialNumber`    | `STRING`  | This hashboard's serial (12 chars).                    |
| `+22 – +31` | reserved          | —         |                                                         |

## Status codes

### System / miner status

Used by `SystemStatus` (reg `0`) and per-miner `MinerStatus` (offset `+1`).

| Code | Status   | Meaning                                                        |
|------|----------|----------------------------------------------------------------|
| 1    | Init     | Starting up.                                                   |
| 2    | Ready    | Up, not currently mining.                                      |
| 3    | Ramping  | Starting to mine (warming up to full speed).                  |
| 4    | Mining   | Mining and producing shares.                                  |
| 5    | Paused   | Stopped on command.                                           |
| 6    | Error    | An error requires attention — see [Error codes](#error-codes).|
| 7    | Updating | Firmware update in progress.                                  |
| 8    | Offline  | Not reachable.                                                |

### Miner online status

Used by per-miner `OnlineStatus` (offset `+0`).

| Code | Status        | Meaning                                              |
|------|---------------|------------------------------------------------------|
| 0    | NotConfigured | This slot is not part of the cluster.                |
| 1    | Offline       | Configured but not currently reachable.              |
| 2    | Online_Idle   | Reachable and healthy, not currently mining.         |
| 3    | Online_Mining | Reachable and mining.                                |
| 4    | Online_Error  | Reachable but reporting an error.                    |

### Pool status codes

Used by `PoolStatus` (reg `404` and per-miner offset `+25`).

| Code | Status        |
|------|---------------|
| 0    | Disconnected  |
| 1    | Connecting    |
| 2    | Subscribed    |
| 3    | Authenticated |
| 4    | Error         |

### Slot status codes

Used by Block 6 `SlotStatus`.

| Code | Status        | Meaning                                         |
|------|---------------|-------------------------------------------------|
| 0    | Absent        | No hashboard in this slot.                       |
| 1    | Present_Idle  | Hashboard present, not mining.                  |
| 2    | Mining        | Hashboard mining.                               |
| 3    | Error         | Hashboard error.                                |

### Model codes

Used by per-miner `Model` (offset `+24`).

| Code | Model             |
|------|-------------------|
| 0    | Unknown           |
| 1    | S19k Pro          |
| 2    | S19j Pro          |
| 3    | S19j Pro (variant)|
| 4    | EmberOne          |

## Error codes

`ErrorCodes` (registers `1 – 3`) is a 48-bit bitmask. Register `1` holds bits
`0 – 15`, register `2` bits `16 – 31`, register `3` bits `32 – 47`; within
each register, bit `0` is the least-significant bit. Each set bit corresponds
to a three-digit error code:

| Bit | Code | Meaning                                          |
|----:|-----:|--------------------------------------------------|
| 0   | 101  | Pool not connected                               |
| 1   | 102  | Pool authentication failed                       |
| 2   | 103  | Pool difficulty too high for the hashrate        |
| 3   | 104  | A miner is unreachable from the controller       |
| 4   | 201  | Hashboard 0 missing                              |
| 5   | 202  | Hashboard 1 missing                              |
| 6   | 203  | Hashboard 2 missing                              |
| 7   | 204  | Hashboard identity check failed                  |
| 8   | 301  | Hashboard 0 failed to start                      |
| 9   | 302  | Hashboard 1 failed to start                      |
| 10  | 303  | Hashboard 2 failed to start                      |
| 11  | 304  | Mining start-up aborted                          |
| 12  | 401  | Over-temperature — mining stopped for safety     |
| 13  | 402  | Temperature sensor not reporting — fans forced high |
| 14  | 403  | Fan control fault                                |
| 15  | 404  | Fan speed fault                                  |
| 16  | 501  | Power-supply communication error                 |
| 17  | 502  | Power-supply voltage out of range                |
| 18  | 503  | Power supply disabled for safety                 |
| 19  | 601  | Pool subscription failed                          |
| 20  | 602  | Pool subscription failed                          |
| 21  | 603  | No shares accepted recently while mining         |
| 22  | 701  | Power target out of range                        |
| 24  | 801  | Internal controller error                        |
| 25  | 999  | Other error                                      |
| 28  | 901  | A miner in the cluster is unreachable            |
| 29  | 902  | Cluster cannot reach the requested power target  |

Bits clear automatically when the underlying condition resolves. Any set bit
also raises `SystemStatus` to `Error` (6).

## Versioning

`ApiVersion` (register `1019`) identifies the version of this register map.
This document describes **version 1**. The version is increased only on a
breaking change (a register removed, moved, or changed in type/meaning);
backward-compatible additions in reserved ranges do not change it. Read
`ApiVersion` at start-up if you need to confirm compatibility.

## Security

Modbus TCP has no authentication. Anyone who can reach port `502` can read
status and start/stop the device and change its power target. Treat the Nova
as a local control device:

- Keep it on a trusted local network / management VLAN.
- Restrict port `502` to the specific automation hosts that need it.
- Never expose it directly to the internet.

---

_For support, contact support@ry3t.com._
