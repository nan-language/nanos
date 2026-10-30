# Charter

This chapter defines the scope of this specification, its reading conventions, the guarantees NaN OS gives, what it leaves out, conformance, the glossary, the chapter map and the index of code families.

## Scope

This specification defines NaN OS, an operating system that AI agents operate and humans own, for these layers:

- the boot loader;
- the kernel;
- the default utilities: the base instances the kernel image holds and the jobs a generation ships.

It defines nothing graphical. NaN OS is written in NaN, language 1 (NaN 00 §Scope), with the features [Requirements on NaN](22-nan.md) lists. It runs on the machines that [Machines](01-machines.md) defines.

## Reading conventions

### Normative wording

Every sentence is normative. A rule is a present-tense sentence with one meaning. These wordings carry a rule's force:

| Wording | Meaning |
| --- | --- |
| "X fails with `family.code`." | The request or operation doing X ends with that failure record. |
| "X is refused (`family.code`)." | A plan, admission or approval containing X is rejected before any effect. |
| "X abandons the instance's domain." | NaN 11 §Abandoning a domain applies to the instance's domain. |
| "The kernel halts (cause)." | NaN 24 §Kernel halts applies. |
| "must", "must not" | A requirement on an implementation, an instance, a bundle or a principal. |
| "may" | A permission. |
| "only", "never", "exactly" | A closed statement: what is not listed is excluded. |
| "by default N (`Record.field`)" | A default value, with its unit and the configuration field that changes it. |
| "is unspecified" | Any choice conforms, and no rule's outcome depends on it. |
| "on a machine with `c`" | The rule applies where the machine condition `c` of [Machines](01-machines.md) holds. |
| "(requires [Nn](22-nan.md))" | The rule depends on NaN feature Nn, listed in [Requirements on NaN](22-nan.md). |

### Notation

- Code font marks names, paths, monikers, codes, field names, commands and exact text.
- Terms are bold where their owning chapter defines them, and linked to that definition at their first use in a section of another chapter.
- "NaN nn §Section" cites section Section of chapter nn of the NaN specification, at <https://nanlang.org/spec/>. NaN terms keep NaN's meaning.
- `nanos::` paths name the std items that the NaN OS targets add ([N1](22-nan.md#n1)), such as `nanos::World`, `nanos::Port` and `nanos::ReplyId`; the chapter that defines an item owns it.
- Fenced blocks are tagged:
  - `nan`: valid language-1 NaN;
  - `ebnf`: grammars, in NaN's EBNF dialect (NaN 00 §Grammar notation);
  - `text`: exact text and link transcripts. In link transcripts, request lines begin with `> `.
- Byte formats are tables of fields with their offsets and sizes in bytes, little-endian unless the table states otherwise.

## Guarantees

NaN OS gives these guarantees on every conforming machine. The chapters named define the rules that establish each one.

### Authority

An instance acts outside its own values only through the capabilities its Env holds ([Authority](09-authority.md)). A session acts only through the capabilities minted from its principal's grants. No name confers authority: a path or a moniker resolves only inside a capability presented with it ([The tree](13-tree.md)). Capabilities never persist; after any restart or reboot, authority is exactly what the grant records in the store describe.

### Typed state

The system definition, configuration, installed code, grants, policy and principals' data are typed values in one store ([The store](11-store.md)). A record or stream entry is written only when it decodes under its declared type. These states lie outside the store's history:

- the state of `volatile` instances;
- [unversioned nodes](11-store.md#unversioned-nodes): the nodes below `/state`, the descendants of `ephemeral` dirs and stream entries;
- the observed state under `/live`;
- the keyring;
- the files of the boot volume.

### Recorded change

Every change to `/sys` and `/data` is a commit naming its principal, session, cause and plan ([The store](11-store.md)). Commits are never modified. History holds every commit its retention keeps. A revert writes a new commit restoring earlier values.

### Exact previews

A plan applies exactly as its preview showed, or not at all ([Changes](14-changes.md)):

- the committed delta is the previewed delta;
- a step that runs an instance replays the inputs the preview recorded, unless the instance holds a resource ([forked runs](14-changes.md#forked-runs));
- an irreversible effect is released only with the bytes the preview showed;
- a change in anything the preview read voids the plan.

### Irreversible effects

Every operation has a computed reversibility class ([Changes](14-changes.md)). An effect that no revert undoes (data leaving the machine, a physical effect, erasure, spending) is a point of no return. A point of no return is written to the outbox in the commit of its span, and is delivered only after that commit, its health window, any confirmation the plan requires and the approvals the policy requires ([Oversight](19-oversight.md)). Under the default policy, a human approves every point of no return before it happens.

### Quiescence

The runner touches an instance's state only between two handler calls. Snapshots, checkpoints, preview forks, updates, hibernation and reverts of an instance happen at those points ([Execution](06-execution.md)).

### Containment

A trap or a CPU exception in an instance's code abandons only that instance's domain ([Execution](06-execution.md), [The kernel](04-kernel.md)):

- its in-flight I/O is cancelled with acknowledgement;
- its memory is freed;
- the runner applies its restart policy.

A fault in a driver cuts its device off before the driver's DMA memory is freed ([Devices](15-devices.md)). Only a fault in the core halts the machine.

### Untrusted data

Data from outside a principal reaches a decision only through a validator ([Communication](07-communication.md)). The link renders every text the system did not produce inside an envelope naming its origin ([The link](18-link.md)). A session's taint records every origin whose content it received, and the policy decides what a tainted session may do ([Oversight](19-oversight.md)).

### Human authority

The owner's key signs the policy. Only `human.*` and `owner` principals approve, and an approval is a signature over the exact plan it covers ([Oversight](19-oversight.md)). The audit chain records every plan, grant, approval, admission and point of no return, and its signed anchors let a rewrite be detected off the machine ([Observability](17-observability.md)). The emergency controls revoke every agent session at once.

### Measured boot

On a machine with `tpm`, the loader measures the generation it boots. On every machine, a production loader boots only a generation whose manifest verifies and whose number is not below the anti-rollback floor ([Boot](03-boot.md)).

## Left out

This specification defines none of these:

- **Interfaces:** graphics, and every user interface beyond the console; natural-language commands.
- **Compatibility:** POSIX and any Linux binary interface; shells, terminals and line disciplines; process creation by fork and exec, signals, and environment variables as ambient state.
- **Access control:** users, groups, access control lists and set-user-ID.
- **Storage:** byte-stream file systems as primary storage, and a global namespace.
- **Memory:** swap, overcommit, an out-of-memory killer and demand paging.
- **Code:** kernel modules, shared libraries and just-in-time compilation; code not written in NaN.
- **Power:** system suspend.
- **Scale:** fleets of machines and distributed transactions.
- **Models:** any model inside a kernel mechanism.

## Conformance

An implementation conforms when it obeys every rule of this specification on each machine it supports. That covers:

- the rules that depend on a machine condition, wherever the condition holds;
- the rules that depend on a NaN requirement, as part of the definition.

Conformance is established by tests: NaN tests of each package, and link transcripts, whose responses must match the transcripts' records. Each transcript in this specification is a conformance test on a machine providing what it shows, and on the [sandbox](01-machines.md#the-sandbox) only when the sandbox provides its transport and mode.

## Glossary

| Term | Meaning | Defined in |
| --- | --- | --- |
| account | the unit CPU time, I/O and energy are charged to: one per instance, one per session | [08](08-scheduling.md) |
| audit anchor | a signed head of the audit chain | [17](17-observability.md) |
| base instance | an instance whose component is linked into the kernel image | [20](20-base.md) |
| blob | a store node holding bytes | [11](11-store.md) |
| boot volume | the EFI system partition holding the loader, the generations and the boot record | [03](03-boot.md) |
| branch | a movable name for a commit; `main` is the branch the machine runs | [11](11-store.md) |
| bundle | a lowered NaN final closure loaded as a unit | [10](10-code.md) |
| cause | a 64-bit identifier on every message and commit, leading to the request that caused it | [17](17-observability.md) |
| checkpoint | an instance's encoded state committed to `/state` | [06](06-execution.md) |
| commit | the result of one transaction on a branch | [11](11-store.md) |
| component | a NaN type implementing `sys::Component[Env, M]` (NaN 26 §Components) | NaN |
| connection | one transport stream of the link | [18](18-link.md) |
| core | the kernel code that is not a component | [04](04-kernel.md) |
| dir | a store node mapping names to nodes | [11](11-store.md) |
| domain | NaN's unit of fault and resource containment (NaN 11 §Domains) | NaN |
| effect set | what a plan's preview computes | [14](14-changes.md) |
| generation | a bootable system: a signed manifest naming a kernel image and a genesis record | [03](03-boot.md) |
| grant | a versioned record conferring authority on a holder | [09](09-authority.md) |
| history | the retained commits of `/sys` and `/data` | [11](11-store.md) |
| index flush | the store writing its index nodes and a flush record | [12](12-store-format.md) |
| instance | a running component, named by a moniker, in its own domain | [06](06-execution.md) |
| job | an instance that runs a program's `main` once and ends with a result | [06](06-execution.md) |
| kernel image | the core and the base components, one NaN final build | [04](04-kernel.md) |
| link | the base instance serving the link protocol | [18](18-link.md) |
| loader | the UEFI program that chooses, verifies, measures and starts a generation | [03](03-boot.md) |
| machine condition | a hardware feature that some rules depend on | [01](01-machines.md) |
| moniker | an instance's name: dotted segments, the prefix naming its supervisor | [02](02-identifiers.md) |
| origin label | the name of where untrusted content came from | [02](02-identifiers.md) |
| outbox | the irreversible effects of committed plans, with their delivery status | [14](14-changes.md) |
| plan | a list of steps previewed together and applied by its hash | [14](14-changes.md) |
| point of no return | an irreversible step of a plan | [14](14-changes.md) |
| port | a capability to send messages of one protocol to one instance | [07](07-communication.md) |
| primer | the self-description that the link's `primer` verb prints | [18](18-link.md) |
| principal | who acts and is accountable | [09](09-authority.md) |
| protocol | a NaN `protocol` item: named operations with heads, contracts and a hash | [07](07-communication.md) |
| provenance class | of a bundle: `vendor`, `reviewed` or `agent` | [09](09-authority.md) |
| record | a store node holding one typed value | [11](11-store.md) |
| request key | the idempotency key of a mutating request | [14](14-changes.md) |
| reversibility | the class of an operation: `none`, `recorded`, `restorable` or `irreversible` with its kinds | [14](14-changes.md) |
| runner | the core part that runs instances | [06](06-execution.md) |
| session | a principal's authenticated presence on a connection | [18](18-link.md) |
| span | the steps of a plan committed together, ending with at most one irreversible step | [14](14-changes.md) |
| space | an address space: the shared space or an isolated space | [04](04-kernel.md) |
| store | the versioned, transactional, content-addressed store of NaN OS | [11](11-store.md) |
| stream | a store node holding an append-only sequence of typed records | [11](11-store.md) |
| taint | the set of origin labels of untrusted content a session has received | [19](19-oversight.md) |
| transport | the medium of a connection: serial, vsock, TCP or port | [18](18-link.md) |
| tree | the namespace of the store's nodes and the live views | [13](13-tree.md) |

## Chapter map

| Chapter | Defines |
| --- | --- |
| [00 Charter](00-charter.md) | scope, conventions, guarantees, conformance, glossary |
| [01 Machines](01-machines.md) | architectures, CPU and firmware requirements, machine conditions, the sandbox |
| [02 Identifiers](02-identifiers.md) | hashes, names, identifiers, time, units, canonical rendering |
| [03 Boot](03-boot.md) | the boot volume, generations, the boot record, the loader, the handoff, modes, recovery |
| [04 The kernel](04-kernel.md) | the core, its start, interrupts, address spaces, kernel policies, faults, halts |
| [05 Memory](05-memory.md) | frames, mappings, accounting, pressure, stacks, transfers, hibernation memory |
| [06 Execution](06-execution.md) | instances, the runner, lifecycle, durability, restarts, jobs |
| [07 Communication](07-communication.md) | protocols, ports, sends, calls, replies, credits, transports between spaces |
| [08 Scheduling](08-scheduling.md) | CPU classes, shares, deadlines, accounting, clocks, timers, idle, energy, power off |
| [09 Authority](09-authority.md) | principals, keys, capability kinds, grants, delegation, provenance, placement |
| [10 Code](10-code.md) | bundles, admission, loading, registries, live update, generations of code |
| [11 The store](11-store.md) | nodes, transactions, history, branches, merges, retention, secrets, file views |
| [12 Store format](12-store-format.md) | the on-disk format, recovery, scrub, cleaning |
| [13 The tree](13-tree.md) | the namespace, the system definition, observed state, genesis |
| [14 Changes](14-changes.md) | reversibility, plans, previews, apply, the outbox, undo |
| [15 Devices](15-devices.md) | discovery, drivers, device protocols, DMA, interrupts, device power |
| [16 Network](16-network.md) | the network stack, the egress gate, network transports |
| [17 Observability](17-observability.md) | events, the journal, causes, metrics, traces, replay, the audit chain |
| [18 The link](18-link.md) | the agent protocol: sessions, requests, verbs, records, budgets, the primer |
| [19 Oversight](19-oversight.md) | the owner, policy, approvals, taint, extraction, emergency controls |
| [20 Base instances](20-base.md) | each base instance and its protocol, the shipped jobs |
| [21 Performance](21-performance.md) | the measurable requirements and how they are measured |
| [22 Requirements on NaN](22-nan.md) | the NaN features NaN OS depends on |

## Code families

A failure code is `family.name`. Each family belongs to one chapter, which lists its codes in its Codes section.

| Family | Chapter |
| --- | --- |
| `boot` | [03 Boot](03-boot.md) |
| `fault` | [04 The kernel](04-kernel.md) |
| `instance` | [06 Execution](06-execution.md) |
| `port` | [07 Communication](07-communication.md) |
| `auth`, `place` | [09 Authority](09-authority.md) |
| `code`, `update` | [10 Code](10-code.md) |
| `store`, `schema`, `conflict` | [11 The store](11-store.md) |
| `plan` | [14 Changes](14-changes.md) |
| `device` | [15 Devices](15-devices.md) |
| `net` | [16 Network](16-network.md) |
| `journal` | [17 Observability](17-observability.md) |
| `request`, `link`, `budget` | [18 The link](18-link.md) |
| `policy`, `taint` | [19 Oversight](19-oversight.md) |

Event kinds use the same form and families, and [Observability](17-observability.md) lists them all.
