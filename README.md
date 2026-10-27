# NaN OS

An operating system built for AI agents from the root, from the boot loader and kernel to drivers and user interfaces, rather than AI added on top of a system made for humans. It is written in [NaN](https://nanlang.org), a programming language made for agentic coding, and targets x86_64.

## Installation

NaN OS builds on Linux x86_64 and runs under QEMU 10.2 or later (`qemu-system-x86_64`) with OVMF. From the repository root:

```bash
tools/install
```

installs `nan`, the NaN toolchain release the project pins, as `~/.local/bin/nan`.

## Usage

```bash
tools/sandbox start
```

builds NaN OS and boots it under QEMU, its serial console on the terminal, until Ctrl-C. [docs/sandbox.md](docs/sandbox.md) describes the sandbox.

## Commands

Run from the repository root:

| Command | Purpose |
| --- | --- |
| `tools/install` | Install the pinned NaN toolchain |
| `nan check [<scope>]` | Report errors and incompleteness |
| `nan fmt [<scope>]` | Format |
| `nan test [<scope>]` | Run tests |
| `nan build [<scope>]` | Build images into `.nan/images/<target>/` |
| `tools/sandbox start [<instance>]` | Boot NaN OS under QEMU |
| `tools/sandbox reset [<instance>]` | Delete a sandbox instance |
