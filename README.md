# RY3T Nova — Integration Documentation

Integration documentation for the **RY3T Nova**, for building-automation and
energy-management systems that want to monitor and control it.

## What a RY3T Nova is

RY3T builds **Bitcoin heaters** — appliances that turn electricity into useful
heat *and* Bitcoin at the same time. The heat is used for hot water and space
heating; the mining makes the energy productive instead of simply burning it.

A **Nova** is the clustered model: one to three miners presented and controlled
as a **single heating appliance**. You give it one power target and it
distributes the load across its miners, reporting back the combined and
per-miner status.

## Why integrate

Exposing the Nova to your building automation or energy-management system lets
the building decide **when and how hard** it mines and heats — for example:

- mine more when there is surplus PV / solar production,
- throttle back when grid power is expensive or demand is high,
- follow actual heat demand (hot water, space heating),
- stop and start it as part of a wider energy strategy.

In return you get full visibility: combined and per-miner hashrate, power,
temperatures, fan state, pool connectivity, uptime, and error status.

## Integration surfaces

| Surface       | Status      | Documentation                  |
|---------------|-------------|--------------------------------|
| **Modbus TCP**| Available   | **[modbus.md](modbus.md)**     |
| MQTT          | Planned     | —                              |
| HTTP REST API | Planned     | —                              |

The Modbus TCP interface is the recommended integration path for Loxone,
PLCs, building EMS, and PV controllers today. See **[modbus.md](modbus.md)**
for the full register map, status and error codes, and control examples.

## Platform guides

| Platform      | Guide                                                  |
|---------------|--------------------------------------------------------|
| Solar Manager | [solar-manager-de.md](solar-manager-de.md) (Deutsch)   |

## What you can do over the interface

- **Read** the cluster's overall state and a per-miner breakdown: hashrate,
  power (target and actual), temperatures, fans, pool connection, uptime,
  firmware, and a detailed error status.
- **Control** the cluster: start it, stop it, and set a single power target
  that it distributes across its miners.

## Support

For integration support, contact **support@ry3t.com**.

---

This repository contains documentation only. It accompanies RY3T Nova devices
and describes their integration interfaces.
