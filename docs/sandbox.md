# Sandbox

`tools/sandbox start [<instance>]` copies the working tree into `.sandbox/<instance>/`, `default` unless named, builds every program of the copy and boots `loader` under QEMU with `nan run`. The machine is the one NaN's `x86_64-uefi` descriptor fixes: q35, 4 processors, 512 MiB, an entropy device, OVMF, no display and no network. Its FAT boot volume holds the loader as `/EFI/BOOT/BOOTX64.EFI` and every other `x86_64-uefi` and `x86_64-bare` image at its root, the kernel as `kernel.elf`.

- Logs: the serial console is the command's standard output.
- Ports: none. Instances share only the toolchain's build cache, so several run side by side; a running instance refuses a second start and a reset.
- Data: `.sandbox/<instance>/.nan/images/<target>/` holds the images, and its `x86_64-uefi/loader.boot/` the boot volume with what the machine wrote to it, until the next start or reset.
- End: the machine ends the run by writing byte v to I/O port `0xF4`, the run's status being v mod 128; a termination request, Ctrl-C or `timeout`, stops it.

`tools/sandbox reset [<instance>]` deletes the instance.
