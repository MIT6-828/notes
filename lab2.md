# How page table is initialized?

Page table is initialized right after entering kernel, in `kern/entry.S`, it loads the address of `entry_dir` defined in `kern/entrypgdir.c` into cr3 and then enables the PG bit of cr0.

In `kern/entrypgdir.c`, `entry_pgdir` and `entry_pgtable` are defined, they are the page directory and page table, each of them has up to 1024 entries. They adds up to map the 4MB of virtual space to physical space.

# Can segment translation and segment-based protection be turned off?

as said in lab2 exercise 2:

> segment translation and segment-based protection cannot be disabled on the x86 cannot be disabled on the x86

but segment translation can be

>installed a Global Descriptor Table (GDT) that effectively disabled segment translation by setting all segment base addresses to 0 and limits to 0xffffffff. Hence the "selector" has no effect and the linear address always equals the offset of the virtual address.

# What the memory layout looks like after page_init()?

```
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| 0xf03ff000      --+
 *                     |                              |                   |
 *                     |                              |                   |
 *                     |                              |                   |
 *                     +------------------------------+                   |
 *                     |  list of all physical pages  |                   |
 *                     +------------------------------+                   |
 *                     |      kern_pgdir (4KB)        |                   |
 *                     +------------------------------+              The 4MB space mapped
 *                     |        Kernel bss            |              to 0x0 in kern/entry.S
 *                     +------------------------------+                   |
 *                     |    Kernel data - contains    |                   |
 *                     | entry_pgdir & entry_pgtable  |                   |
 *                     +------------------------------+                   |
 *                     |        Kernel code           |                   |
 *                     +------------------------------+ 0xf0100000        |
 *                     |                              |                   |
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |                              |                   |
 *                     +------------------------------+                   |
 *                     | kern_pgdir actual area (4KB) |                   |
 *                     +------------------------------+                   |
 *                     |  Kernel actual loaded area   |                   |
 *                     +------------------------------+ 0x00100000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
 */
 ```
 
# Why setting `only_low_memory` to true at the first check_page_free_list()?

In check_page_free_list(bool only_low_memory), if we pass only_low_memory==true, then we will run into a part that reorder the free pages list, look at below code snippet.

This part could be hard to read at first, so it's better to draw a picture to help understand it. After that I got known that this code snippet is to fetch the lowest addressed pages, here also the directory 0 pages(there should be 1024 pages in directory 0) and make them ahead of others.

```
if (only_low_memory) {
    // Move pages with lower addresses first in the free
    // list, since entry_pgdir does not map all pages.
    struct PageInfo *pp1, *pp2;
    struct PageInfo **tp[2] = { &pp1, &pp2 };
    for (pp = page_free_list; pp; pp = pp->pp_link) {
        int pagetype = PDX(page2pa(pp)) >= pdx_limit;
        *tp[pagetype] = pp;
        tp[pagetype] = &pp->pp_link;
    }
    *tp[1] = 0;
    *tp[0] = pp2;
    page_free_list = pp1;
}
```

The reason we need to set the directory 0 pages as page_free_list head is unclear to me, we actually needn't to do this to make the check passed, but we do need to ensure the `pdx_limit` is 1 in the following part to prohibit the checking of the pages in directory >= 1, because in `kern/entrypgdir.c` we only map the 1st directory, i.e. the MMU only able to map the vritual address that corresponds to the first 1024 batch of physical pages now. The `memset(page2kva(pp), 0x97, 128)` call is to cause a memory access instruction and test whether MMU can succesfully translate the virtual address and access the physical memory, for other addresses MMU are unable to recongize and will cause sigtrap fault.

> NOTE Actually the `kern/entrypgdir.c` also maps the 58th page, this is the page the KERNBASE exists, 58 is derived from KERNBASE>>PDXSHIFT, this part only checks the accessbility for the first page. As in lab2 our physical memory is just 128MB(the highest physical address is 0x7fffff), KERNBASE(0xf0000000), if regarded as physical adress, means nearly 4G space, the pages list certainly doesn't have that many pages to reach to 4G.

```
    // if there's a page that shouldn't be on the free list,
	// try to make sure it eventually causes trouble.
	for (pp = page_free_list; pp; pp = pp->pp_link)
		if (PDX(page2pa(pp)) < pdx_limit)
			memset(page2kva(pp), 0x97, 128);
```

```
// Initial page directory setting in kern/entrypgdir.c

__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```

Step back to the first question, what the purpose of passing true to `only_low_memory` and make the directory 0 pages ahead and reset the `page_free_list` head if we don't need it to have the check_page_free_list() passed? The answer is this is for the check_page_alloc() check. In check_page_alloc() the `page_alloc()` method will beb called to allocate some physical pages(definitely won't exceed 1024 pages), if the `page_free_list` points to a higher addressed page and our `page_alloc()` is implemented without looking up the lowest page first, then the new allocated page will probably not in the directory 0 and then cause the MMU to trap.

# Why keep getting sig trap fault?

I encountered many times of trap fault during setting up the page table, I made these faults because I'm not very clear about the page table entry at first. I spent some time to understand the mechanism and finally resolved all the faults, and I also find a single tip which can help us avoid the faults, always keep in mind of this tip:

1. Each page directory entry is the `or` result of a physical page's adress and the privilege bits, thus the final entry value is not page aligned.
2. So does the page table entry.
3. When we are getting an address to a page table entry(as in `pgdir_walk()`), we need to strip the privilege bits and convert it to virtual address.
