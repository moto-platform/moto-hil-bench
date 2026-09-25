# CLAUDE.md — moto-hil-bench

## What this repo is

A vehicle-independent HIL (Hardware-in-the-Loop) test bench. Two parts:
- `simulator/` — STM32F4 firmware (restbus simulation, multiple virtual ECUs + a simple vehicle dynamics model)
- `host/` — Python scenario engine, evaluator, report generator, control panel
- `scenarios/` — YAML test scenarios (vehicle-independent)
- `vehicles/cl250/` — CL250-specific signal/parameter definitions (CL250 is only the FIRST APPLICATION, the core never knows about CL250)

## Critical architectural rule

`simulator/` and `host/` **never** reference a file under `vehicles/*` directly — they only work through definition files loaded at runtime from the `vehicles/<vehicle>/` folder. Adding a new vehicle = a new folder, the core code doesn't change. Verify this on every PR.

## Two-bus simulation (D-009)

The DUT sees two CAN buses: the **vehicle bus** (restbus with the `vehicles/<vehicle>/` DBC — mimicking the CL250 ECU) and the **platform bus** (`platform.dbc` — mimicking the platform nodes not under test; e.g. when testing safety-node, rt-core's EKF messages + heartbeat, including E2E). E2E fault injection (corrupted CRC, frozen counter, timeout) is a core scenario class. This is why the STM32F4's two bxCANs are used.

## Independence principle (why STM32F4, not the DUT's chip)

The simulator must **not be the same chip** as the device under test (DUT — `moto-rt-core`/`moto-connectivity-node`). Otherwise both share the same library/blind spot, and the test can't catch the bug. The simulator is STM32F4, the main vehicle MCU is STM32H7/ESP32-S3 — a different class, same family (keeps the vehicle chain familiar). If a suggestion comes in to change this decision, ask the user.

## Dependencies

Reads CAN schema/VSS definitions from `moto-vehicle-defs` (submodule: `external/moto-vehicle-defs`, pinned to a tag; generated code at `external/moto-vehicle-defs/gen/c/<node>/`). Watch the version pin — when `moto-vehicle-defs` is updated, the submodule reference here does not advance automatically; it requires a deliberate update.

## Test/build

- `simulator/`: CMake + STM32CubeMX + arm-none-eabi-gcc (D-007) — update the commands here as they get documented
- `host/`: Python, `uv` + `ruff` + `pytest` (D-007), `python-can` (SocketCAN) + `cantools`; unit tests for the scenario engine; the simulator firmware can also be tested with **Renode** without real hardware — this is what hardware-less development uses

## The host side's three modules (must stay separate, must not leak into each other)

1. Scenario engine (logic — reads the scenario, executes the steps)
2. Control panel / visualization (web dashboard or Foxglove — display only)
3. Evaluator (expected-vs-actual comparison, report generation)

## Out of scope (does not belong in this repo)

- The vehicle's main MCU firmware (that's in `moto-rt-core`/`moto-connectivity-node`)
- ECU flash writing/stage mapping — never, under any circumstances

## Context

Full HIL hardware architecture: `../moto-vehicle-defs/docs/ARCHITECTURE.md` (summary) · detail: `../moto-vehicle-defs/docs/hardware-architecture.md` (section index in `docs/README.md`) section 9b. For the scenario format and test catalog, also see the vehicle work plan (`../moto-vehicle-defs/docs/vehicle-work-plan.md`, section 7).
