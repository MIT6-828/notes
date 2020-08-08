# How page table is initialized?

Page table is initialized right after entering kernel, in `kern/entry.S`, it loads the address of `entry_dir` defined in `kern/entrypgdir.c` into cr3 and then enables the PG bit of cr0.

In `kern/entrypgdir.c`, `entry_pgdir` and `entry_pgtable` are defined, they are the page directory and page table, each of them has up to 1024 entries. They adds up to map the 4MB of virtual space to physical space.

# Can segment translation and segment-based protection be turned off?

as said in lab2 exercise 2:

> segment translation and segment-based protection cannot be disabled on the x86 cannot be disabled on the x86

but segment translation can be

>installed a Global Descriptor Table (GDT) that effectively disabled segment translation by setting all segment base addresses to 0 and limits to 0xffffffff. Hence the "selector" has no effect and the linear address always equals the offset of the virtual address.
