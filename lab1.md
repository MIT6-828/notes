---
# lab1

This file records the notes of lab1.

# Setting up toolchain on macOS

Basically two things need setting up: QEMU and the compiler toolchain. Setup for QEMU can refer to [6.828 official tools page](https://pdos.csail.mit.edu/6.828/2018/tools.html) and you should not get into trouble.

6.828 official doesn't provide the ready-to-use compiler toolchain but forturnately, many users made out the toolchain and published it to a brew repo, there are several brew repos we can fetch it, such as:

* https://github.com/lizebang/jos-xv6-macOS
* https://blog.zzhou612.com/2019/03/09/mit-6-828-toolchains-setup-on-macos/

choose either of the above mentioned repo by running `brew tap ...` and then install the toolchain.

```
brew tap zzhou612/jos
brew install i386-jos-elf-binutils i386-jos-elf-gcc i386-jos-elf-gdb
```

(PS: this toolchain we installed is actually just a toolchain of i386 arch)

# gdb not installed?

I once encountered such an error when running `make gdb`, after having checked the file `GNUmakefile`, I found `make gdb` attemptted to invoke the `gdb` command which is not installed on my host.

```
$ make gdb
gdb -n -x .gdbinit
make: gdb: No such file or directory
make: *** [gdb] Error 1
```

Note that `GNUmakefile` does call `i386-jos-elf-` cross-compiled version gcc while compiling but doesn't have to also use the prefixed version of gdb to do debugging, it can use the local version gdb.

But since I don't have a local gdb installed and I dont want to, and since I have already had the cross-compiled version installed, so I just have had the `GNUmakefile` changed to invoke `i386-jos-elf-gdb`.

# Who print the Booting info?

Once we have start the system(by running `make qemu`), a virtual VGA window is displayed and the following info is printed:

```
SeaBIOS (version rel-1.8.1-0-g4adadbd-20150316_085822-nilsson.home.kraxel.org)


iPXE (http://ipxe.org) 00:03.0 C980 PCI2.10 PnP PMM+07F93BE0+07EF3BE0 C980



Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

where the first three lines are printed by BIOS program integrated in QEMU, you can find these two lines in these two BIOS images `pc-bios/bios.bin` and `pc-bios/pxe-*.rom` respectively under the QEMU source code downloaded from MIT repo.

```
SeaBIOS (version rel-1.8.1-0-g4adadbd-20150316_085822-nilsson.home.kraxel.org)


iPXE (http://ipxe.org) 00:03.0 C980 PCI2.10 PnP PMM+07F93BE0+07EF3BE0 C980



Booting from Hard Disk...
```

The remaining lines are printed by the kernel itself, you can dive into the `kern/` sub directory to check the source code.

# At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

After BIOS finishes its work it jump to 0x7c00 to transfer its control to the bootloader which is loaded from the first sector of the hard disk.

The bootloader turns the processor into 32-bit mode by setting the 1 bit of `cr0` to 1<sup>1</sup>.

```
boot/boot.S

 48   lgdt    gdtdesc
 49   movl    %cr0, %eax
 50   orl     $CR0_PE_ON, %eax
 51   movl    %eax, %cr0
```

## How to check the control registers state?

In gdb we can get the state of general purpose registers, eip, eflags, and the segment selectors. For a much more thorough dump of the machine register state, see QEMU's own info registers command<sup>2</sup>. This need you to be in QEMU window and hit `ctrl-a c` to switch back to QEMU's own command interface, and then enter `info registers` command<sup>3</sup>.

# Booting process

1. BIOS starts executing at 0xffff0 in real mode. CS is 0xf000.
2. BIOS transfers control to bootloader at 0x7c00.
3. Bootloader starts executing at 0x7c00 in real mode. CS is 0x0.
4. Bootloader setups segment descriptor table(GDT) and turns on protected mode.(from now on, the value of CS will be representing an offset to the GDT)
5. Bootloader `ljump` to run in 32-bit mode and call the `bootmain()` function defined in `boot/main.c`. (`ljmp` is the AT&T notation of far jump, it takes the form of `ljmp 0x8, 0x7c32` in our code, besides jumping, this also reloads CS with 0x8 and EIP with 0x7c32)

# Why bootloader doesn't work after changing its link address?

In exercise 5 of lab1, we are asked to change the bootloader's link address to other value than 0x7c00 and see whether it still works. Surely it not works anymore, the interesting thing on this is, as the exercise hinted, it doesn't fail from the starting, but fail at a certain instruction, the `ljmp` instruction right after the GDT setting-up instructions. Is this accidental or invetable?

If you dig more into this you will know that this is invetable, the bootloader's first failing instruction must be the above mentioned `ljmp` instruction, because `ljmp` is a long jump instruction that was attempting to access an absolute address inferred from the GDT, while all of the previous bootloader code lines are short jump and using relative address.

When the processor was trying to access the abosulte address(link address), the bootloader, however, was loaded into the different address(load address), so the error occurs.

# Exercise 9?

> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

Kernel initializes the stack by the `movl    $(bootstacktop),%esp` command at `kern/entry.S:77`, this makes ESP point to the address marked by `bootstacktop` label, the definition of this label is also in `kern/entry.S`.

```
 86 .data
 87 ###################################################################
 88 # boot stack
 89 ###################################################################
 90     .p2align    PGSHIFT     # force page alignment
 91     .globl      bootstack
 92 bootstack:
 93     .space      KSTKSIZE
 94     .globl      bootstacktop
 95 bootstacktop:
 96
 ```
 
 where the `KSTKSIZE` is defined to 8 pages(`8*4096`), and from the `kern/kernel.ld` and the objdump output of kernel we can infer that the `.data` section's link address is `0xf0108000`, so the stack space for kernel is `0xf0108000 ~ 0xf010ffff` and the `bootstacktop`, as shown in above code, right after this area, points to `0xf0110000`.
 
# Why can't the backtrace code detect how many arguments there actually are? How could this limitation be fixed?

This is a thinking question after the exercise 10. The reason we can't detect how many arguments is because the compiled assembly code doesn't explicitly include these info, there are some clues we can use to implicitly guess the number of arguments but it's not reliable. As explained [here](https://stackoverflow.com/a/9554597/1436289), there is no general way to deduce this.

But since we are doing the labratory assignment, we can think out ways to achieve this. The straightest way I can think out is to make a calling convention that arguments count always being passed as the first argument, this can be done implicitly by compiler.

# References

1. https://pdos.csail.mit.edu/6.828/2018/readings/i386/s14_04.htm
2. GDB section of https://pdos.csail.mit.edu/6.828/2018/labguide.html
3. QEMU section of https://pdos.csail.mit.edu/6.828/2018/labguide.html
