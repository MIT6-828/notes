# How page table is initialized?

Page table is initialized right after entering kernel, in `kern/entry.S`, it loads the address of `entry_dir` defined in `kern/entrypgdir.c` into cr3 and then enables the PG bit of cr0.

In `kern/entrypgdir.c`, `entry_pgdir` and `entry_pgtable` are defined, they are the page directory and page table, each of them has up to 1024 entries. They adds up to map the 4MB of virtual space to physical space.
