# 0169. Generations built on the host, their keyring in the manifest

Status: accepted

## Context

A production generation boots from `/nanos/gen/<n>/`: a signed manifest, the kernel image, its interface file and a genesis record, which the kernel writes into an empty store and compares with the running branch at every boot. The genesis record names the image's blob id, so the image cannot hold it; until now the kernel computed it at each start from its own code. `keys` runs from the `keyring` bundle the generation ships (0157), which nothing verified outside a development boot. NaN lays out a loaded image's modules after the loader has written the handoff record, so no module's address can enter that record. `nan run --keep-resets` reboots the sandbox's machine in place.

## Decision

- **A `genesis` package.** The base components' tables, their specifications and grants, the admission records of the base bundle and jobs and the genesis record move out of the kernel into a package targeting `any`. The kernel builds its records from it, and so does the host's `bundler`.
- **Built on the host.** `bundler generation` writes a generation's directory: the image, its interface file, the `keyring` bundle, the genesis record of the image, and the manifest naming them, signed. The owner's records are not part of the record; the kernel adds them at boot from the handoff record.
- **The keyring in the manifest.** The manifest names the `keyring` bundle's blob id. The loader verifies it as it verifies the image and the genesis record, measures it and passes it as the module `keyring.nbun`; the genesis record and the event log copy are the modules `genesis` and `eventlog`.
- **A sandbox vendor.** A development loader, and `boot` on its boots, accept manifests signed by `vendor.sandbox`, whose seed is in the repository. A production loader accepts no such key.
- **Real reboots.** The core reboots through the FADT's reset register, and the sandbox runs with `--keep-resets`, so reboots keep its boot volume, firmware variables and TPM.

## Consequences

- A generation's genesis record is reproducible from its image alone, by any build of the same sources.
- The `bundler` links the base components' packages, the drivers' among them.
- The handoff record's event log and final events addresses hold 0.
- A sandbox reboot restarts the machine instead of ending the run.
