# Sandbox

`tools/sandbox start [<instance>]` copies the working tree into `.sandbox/<instance>/`, `default` unless named, builds every program of the copy and boots `loader` under QEMU with `nan run`. The machine is the one NaN's `x86_64-uefi` descriptor fixes: q35 with an Intel VT-d IOMMU, 4 processors, 512 MiB, an entropy device, a TPM that swtpm emulates, a watchdog, a blank 256 MiB disk, a NIC on QEMU's user network, OVMF, no display. Its FAT boot volume holds the loader as `/EFI/BOOT/BOOTX64.EFI` and every other `x86_64-uefi` and `x86_64-bare` image at its root, the kernel as `kernel.elf`.

- Logs: the serial console is the command's standard output, and the console port the socket `.sandbox/<instance>/.nan/machine/x86_64-uefi/console.sock`.
- Ports: none. Instances share only the toolchain's build cache, so several run side by side; a running instance refuses a second start and a reset.
- Data: `.sandbox/<instance>/.nan/images/<target>/` holds the images, its `x86_64-uefi/loader.boot/` the directory the boot volume is made from, and `.sandbox/<instance>/.nan/machine/x86_64-uefi/` the disk, the firmware variables and the TPM state, until the next start or reset. The machine's writes to the boot volume do not reliably reach that directory.
- End: the machine ends the run by writing byte v to I/O port `0xF4`, the run's status being v mod 128, and a reset ends it with status 70; a termination request, Ctrl-C or `timeout`, stops it.

`tools/sandbox reset [<instance>]` deletes the instance.
