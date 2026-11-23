# 0150. Base jobs, the nan job and admission through it

Status: accepted

## Context

Chapter 20 has a generation ship its jobs as admitted bundles of provenance `vendor`. Admission refuses a bundle whose trust escapes carry no seal by a sealer the policy names, and a seal attests a human review. `selftest`, `scrub` and `replay` decode records holding text and bytes, whose decoders are trust escapes. The genesis record carries the base bundle's admission record, which the running kernel builds (0122).

The `nan` job is the NaN toolchain over a store tree, on machines with at least 4 GiB of RAM. The toolchain is a host-compatible program. Built from its release's source archive, with its manifest declaring a nanos target, against a kernel's interface file, it runs through `nanos::run_isolated` in an isolated space: its world reaches files through `sys::Fs`, it reads its tree hash from `image.ntree`, keeps its user directory in `home`, and draws a boot heap of up to 768 MiB from the frames it is given. The runner runs instances in domains of the shared space only. An image loads only into the kernel whose interface file it was built against, and that file records the code roots of the kernel's whole closure. Since v0.26.2 the toolchain built for a nanos target has the toolchain hash of the release that builds the kernel, so it builds bundles against the running kernel's interface file. NaN checks every release from its source, and an artefact only identifies and attests its release. A plan's preview changes nothing of the running system. The development loader boots `/kernel.elf` from the boot volume, which `nan run` fills with every image of the workspace. A 4 GiB machine has less than 4 GiB of usable RAM.

## Decision

- **In the kernel image.** `selftest`, `scrub` and `replay` are library packages the kernel image links, each exporting the `main` of its job.
- **The toolchain beside it.** `tools/toolchain` builds the toolchain from the pinned release's source archive, its manifest declaring `x86_64-nanos`, against the workspace's kernel interface file, keeping the build of the latest interface in `.nan/toolchain/`; `tools/sandbox start` runs it, so the boot volume holds `nan.nbun` and `nan.ntree`. The development loader passes them and `kernel.nlif` as boot modules named by their files.
- **Code hash.** A base job's code hash is that of the `CodeId` rooted at its name whose code roots are the base bundle's code hash and, for `nan`, the toolchain's tree hash.
- **Admitted at genesis.** The genesis record holds each base job's admission record, which the kernel builds as the base bundle's: provenance `vendor`, shared, the kernel image as its image, and its `main`'s input and output schemas and class. Genesis writes them, and a later boot writes those `main` lacks, the `nan` job's of a new toolchain included.
- **Run from the kernel.** The runner asks its components whether a code hash is a base job's, and runs such a job's `main` from the kernel image in the job's domain, decoding its input and encoding its result as a loaded bundle's. The `nan` job's `main` has a core task run the toolchain.
- **Isolated at start.** On a machine with at least 4 GiB of RAM, the kernel holds the toolchain's modules back from the frame allocator until it has isolated the toolchain through `Loader.isolate_unverified`, a trusted call whose premise is that the generation built it against its kernel, then gives their memory to the frame allocator. It sets aside 896 MiB of frames, the space's boot heap and 128 MiB for its images, stacks and tables, counted as reserved in the memory summary. The core task runs one run at a time, each in a space of its own on those frames, drawing its own records through NaN's boot heap, since the core pool cannot hold a run's.
- **A run's world.** Its file system is the dir the request names in the job's file view, served by a task of the job, holding in memory `home`, `image.ntree` and `kernel.nlif`; its standard streams are kept by the core, up to 1 MiB, to make the report; its clock, log and random source are the job's.
- **Files in pieces.** Every order between the core and a domain about a file system carries at most 64 KiB of a file, each side drawing what it holds through its own memory: a job's file view through the job's cell, the toolchain's file system through the core's heap.
- **Bundles carry their sources.** A bundle file holds the source and machine files of its closure's packages outside std at their paths in the workspace, which `bundler` and the `nan` job gather from the packages its artefacts name.
- **Builds make bundle files.** A `build` of the `nan` job that passed writes `<package>.bundle` of what it built beside NaN's images, so what an agent builds on the machine installs as it is.
- **Admission checks with the job's toolchain.** On a machine running the `nan` job, admission serves a workspace in its memory holding a bundle's sources, over which the core runs the job's toolchain: its release gate, then a bundle build of the root against the running kernel. The build must give the bundle's code hash; its image serves step 13 unless a trusted image of the machine's level does. Admission keeps the results of its latest four checks by root and sources. There, a bundle carrying no sources is refused.
- **Sandbox.** The sandbox's machine has 5 GiB.

Alternatives rejected:

- **The toolchain as a bundle the runner loads:** a host-compatible program runs only in an isolated space.
- **Frames lent for each run:** a run would wait on memory pressure, or fail under it, for frames a machine of 4 GiB can set aside.
- **The release's tree hash as `image.ntree`:** the toolchain would claim the hash of another toolchain, which NaN reserves for toolchains giving equal outputs from equal inputs.
- **Admission starting the `nan` job:** a preview would start an instance and write the sources into the store.
- **The core holding a check's workspace:** a bundle's sources and the toolchain's cache would draw on the kernel's heap, whose exhaustion halts the machine.
- **Certificates on a machine running the job:** the machine would rely on a builder's release gate where it can run its own.

## Consequences

- A base job changes only with a new generation.
- A change to the kernel's closure rebuilds the toolchain at the next sandbox start, in about 40 s.
- A machine running the `nan` job holds 896 MiB for it, and runs one command at a time, admission's checks included.
- Every bundle installed on a machine running the `nan` job carries its sources, and its first preview takes a release gate and a build.
- Whoever serves a run's workspace holds the toolchain's cache in its memory, about 17 MiB for a build of a small closure: a `build` in the job needs more than the default budget of 16 MiB, and `admission` has a limit of 256 MiB.
