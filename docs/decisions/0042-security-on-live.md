# 0042. Security on /live

Status: accepted

## Context

Chapter 13 has `/live/security` report the machine's protections, which the core, `keys`, `boot` and `journal` provide, and `/live/status` list as exposures the fields away from their protected value. The core knows from the handoff record and the measured facts whether the loader keeps a floor, its build, secure boot, measurement and DMA confinement; `keys` and `journal` do not run yet.

## Decision

- The board holds `/live/security` from the start, as the core knows it: the loader's floor and build, secure boot and measurement from the handoff record, unconfined DMA, no isolated space, `keys` unconfined with its key storage following `tpm`, no link or audit key, an intact audit chain and a plain volume.
- `/live/status` lists the exposures that record holds when it is read.

## Consequences

- The fields of `keys`, `boot` and `journal` keep their start values until those instances publish them.
