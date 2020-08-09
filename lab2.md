# how GDT is initialized?

The LGDT instruction loads a linear base address and limit value from a six-byte data operand in memory into the GDTR: https://pdos.csail.mit.edu/6.828/2018/readings/i386/LGDT.htm.

A bit clarify about whether the operand is 16-bit or 32-bit, normally, in read mode and a `.code16` section the operand is 16 bit, otherwise it's 32 bit. An instruction prefix can be used to change this behavior: http://web.mit.edu/rhel-doc/3/rhel-as-en-3/i386-prefixes.html.

In JOS source code, when the LGDT instruction is called to load the GDT, it's in 16 bit<sup>1</sup>, so the LGDT instruction will only use the first 5 bytes of the 6-byte data. The 5 bytes is, as the code shown, pointed by the `gdtdesc` label:

```
 61   # Switch from real to protected mode, using a bootstrap GDT
 62   # and segment translation that makes virtual addresses
 63   # identical to their physical addresses, so that the
 64   # effective memory map does not change during the switch.
 65   lgdt    gdtdesc
 66     7c1e:   0f 01 16                lgdtl  (%esi)
 67     7c21:   64 7c 0f                fs jl  7c33 <protcseg+0x1>
 ...
 ...
112 00007c4c <gdt>:
113     ...
114     7c54:   ff                      (bad)
115     7c55:   ff 00                   incl   (%eax)
116     7c57:   00 00                   add    %al,(%eax)
117     7c59:   9a cf 00 ff ff 00 00    lcall  $0x0,$0xffff00cf
118     7c60:   00                      .byte 0x0
119     7c61:   92                      xchg   %eax,%edx
120     7c62:   cf                      iret
121     ...
122
123 00007c64 <gdtdesc>:
124     7c64:   17                      pop    %ss
125     7c65:   00 4c 7c 00             add    %cl,0x0(%esp,%edi,2)
 ```
 
So `0x0017` will be regarded as the limit part and `0x007c4c` will be regarded as the address of the GDT table, these two parts are then loaded into the GDTR register.

Note that `0x17` is the size of the gdt table minus 1, since it is to represent the end address of the table, so we know that the 24 bytes from `0x00007c4c` is the content of the GDT table, we should also know that each entry in GDT table is 8 bytes, so this table contains 3 entries, they are NULL, code and data entries respectively as the comments indicated in `boot.S`.

```
 75 # Bootstrap GDT
 76 .p2align 2                                # force 4 byte alignment
 77 gdt:
 78   SEG_NULL              # null seg
 79   SEG(STA_X|STA_R, 0x0, 0xffffffff) # code seg
 80   SEG(STA_W, 0x0, 0xffffffff)           # data seg
 ```
 
In lab1 we knew that the `ljmp    $0x8, $0x7c32` jumped the code in 32-bit mode, while if you paid attention to the value of the EIP register in gdb, you would see the value of the EIP register was `0x7c32`, no address offset was added to it. So does it mean the base address in the GDT table is 0, yes it is.

If we examine the exact content in the GDT table, we can check the content of address `0x7c4c` in GDB or QEMU console, we just need to check 24 bytes, you should get the following result:

```
(qemu) x/8x 0x7c4c
00007c4c: 0x00000000 0x00000000 0x0000ffff 0x00cf9a00
00007c5c: 0x0000ffff 0x00cf9200 0x7c4c0017 0xf7ba0000
```

the first 8 bytes is the NULL entry, the second 8 bytes is the code segmenet descriptor, if you put the value to a descriptor entry described in https://pdos.csail.mit.edu/6.828/2018/readings/i386/s05_01.htm#fig5-3, you will see this code segment's base address is 0. So this JOS just set a GDT table which maps the logical address to linear address without any changes.

1. you can change the source code of `boot.S` to add an `addr32` prefix to the `lgdt    gdtdesc` instruction and compare the generated asm `obj/boot/boot.asm`

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

> NOTE Actually the `kern/entrypgdir.c` also maps the 928th page, this is the page the KERNBASE exists, 928 is derived from KERNBASE>>PDXSHIFT, this part only checks the accessbility for the first page. As in lab2 our physical memory is just 128MB(the highest physical address is 0x7fffff), KERNBASE(0xf0000000), if regarded as physical adress, means nearly 4G space, the pages list certainly doesn't have that many pages to reach to 4G.

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

# Why we can access non-exist physical memory?

# Question 1 - 6

## 1

```
mystery_t x;
char* value = return_a_pointer();
*value = 10;
x = (mystery_t) value;
```

All pointers we get in C point to virtual address, so `value` is actually a uintptr_t type, hence x is uintptr_t type too.

## 2

here's the output of the `info pg` command of the patched QEMU:

```
VPN range     Entry         Flags        Physical page
[ef000-ef3ff]  PDE[3bc]     -------UWP
  [ef000-ef03f]  PTE[000-03f] -------U-P 0011a-00159
[ef400-ef7ff]  PDE[3bd]     -------U-P
  [ef7bc-ef7bc]  PTE[3bc]     -------UWP 003fd
  [ef7bd-ef7bd]  PTE[3bd]     -------U-P 00119
  [ef7bf-ef7bf]  PTE[3bf]     -------UWP 003fe
  [ef7c0-ef7c0]  PTE[3c0]     ----A--UWP 003ff
  [ef7c1-ef7ff]  PTE[3c1-3ff] -------UWP 003fc 003fb 003fa 003f9 003f8 003f7 ..
[efc00-effff]  PDE[3bf]     -------UWP
  [efff8-effff]  PTE[3f8-3ff] --------WP 0010e-00115
[f0000-f03ff]  PDE[3c0]     ----A--UWP
  [f0000-f0000]  PTE[000]     --------WP 00000
  [f0001-f009f]  PTE[001-09f] ---DA---WP 00001-0009f
  [f00a0-f00b7]  PTE[0a0-0b7] --------WP 000a0-000b7
  [f00b8-f00b8]  PTE[0b8]     ---DA---WP 000b8
  [f00b9-f00ff]  PTE[0b9-0ff] --------WP 000b9-000ff
  [f0100-f0100]  PTE[100]     ----A---WP 00100
  [f0101-f0101]  PTE[101]     --------WP 00101
  [f0102-f0104]  PTE[102-104] ----A---WP 00102-00104
  [f0105-f0114]  PTE[105-114] --------WP 00105-00114
  [f0115-f0115]  PTE[115]     ---DA---WP 00115
  [f0116-f0117]  PTE[116-117] --------WP 00116-00117
  [f0118-f0119]  PTE[118-119] ---DA---WP 00118-00119
  [f011a-f011a]  PTE[11a]     ----A---WP 0011a
  [f011b-f011b]  PTE[11b]     ---DA---WP 0011b
  [f011c-f0159]  PTE[11c-159] ----A---WP 0011c-00159
  [f015a-f03bd]  PTE[15a-3bd] ---DA---WP 0015a-003bd
  [f03be-f03ff]  PTE[3be-3ff] --------WP 003be-003ff
[f0400-fffff]  PDE[3c1-3ff] -------UWP
  [f0400-fffff]  PTE[000-3ff] --------WP 00400-0ffff
```

## 3

See above `info pg` output, the pages in kernel space (0xF0000000 above) are not with `U` permission bit, ensuring the user cannot access these pages.

For more detail https://pdos.csail.mit.edu/6.828/2018/readings/i386/s06_04.htm.

## 4

The maximum amount of physical memory this operating system can support is 4GB, this is the largest of the x86 addressing capability.

## 5

The current system in lab2 got a total of 128MB physical memory, this splits to 32768 pages so we need the same number of the `struct PageInfo` to hold them, according to its definition, a `struct PageInfo` will occupy 6 bytes, so this part costs `32768 * 6 / 1024 = 192 KB` (192KB is multiple of 4KB so no roundup needed here), this part is allocated by `boot_alloc(npages * sizeof *pages);` in the `mem_init()` function. 

In addition, we need a bunch of physical pages to hold the page directory and page tables, each of these are 4KB size. Page directory is allocated by `boot_alloc(PGSIZE)` as well at the beginning of `mem_init()`. The page tables are allocated during the `boot_map_region` calls afterwards.

```
boot_map_region(kern_pgdir, UPAGES, npages * sizeof *pages, PADDR(pages), PTE_U);
boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
boot_map_region(kern_pgdir, KERNBASE, (2^32)-KERNBASE, 0, PTE_W);
```

The first two `boot_map_region()` calls allocate 48(192KB/4KB) and 8 physical pages respectively, so add up to `56 * 4KB = 224KB`. 

It's easy to get that if we have a full 4GB physicall memory the overhead for this is `6MB`.
