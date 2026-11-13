# 0072. Kernel images in a file heap

Status: accepted

## Context

The loader reads `kernel.elf` whole to hash and validate it before placing its segments. A UEFI program's boot heap is NaN's fixed 16 MiB, and the loader also draws the memory map, the pages the image fills and the boot records from it. The kernel image passed 15 MiB, after which the loader could no longer read the memory map and stopped with `boot.placement`. The firmware gives free pages at a chosen address, and `mem::Heap.over` makes a heap of them.

## Decision

The loader reads each kernel image into a 64 MiB file heap of firmware pages, at the top of the highest range of free memory below 4 GiB that holds it. The boot heap keeps everything else. No free range holding the heap, or a file beyond it, is `boot.too-large`.

Alternatives rejected:

- **Reading the image in pieces into its placed segments:** the blob id and the validation need the whole file before placement.
- **A larger boot heap in NaN:** NaN's target descriptors fix it, and the firmware already gives the loader memory on request.

## Consequences

- The file heap's pages are loader data, which the hand-off's memory map gives the kernel as usable memory.
- A kernel image may grow to just under 64 MiB.
