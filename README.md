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

---

# lecture 2

# Homework 1

```
(gdb) x/24x $esp
0x7bdc:	0x00007d7d	0x00000000	0x00000000	0x00000000
0x7bec:	0x00000000	0x00000000	0x00000000	0x00000000
0x7bfc:	0x00007c4d	0x8ec031fa	0x8ec08ed8	0xa864e4d0
0x7c0c:	0xb0fa7502	0xe464e6d1	0x7502a864	0xe6dfb0fa
0x7c1c:	0x16010f60	0x200f7c78	0xc88366c0	0xc0220f01
0x7c2c:	0x087c31ea	0x10b86600	0x8ed88e00	0x66d08ec0
```

After having entered the bootloader(0x7c00), The `$esp` is first set right before the calling instruction of `bootmain()`, at this point `$esp` was set to the start point of bootloader, yes it's `0x7c00`, having went through the following code till entering the kernel entry, there is no other instructions that reload `$esp`'s value. So till entering the kernel entry, `0x7c00` is the stack bottom.

So, the addresses above `0x7c00` are not parts of the stack, the valid stack parts printout are:

```
(gdb) x/24x $esp
0x7bdc:	0x00007d7d	0x00000000	0x00000000	0x00000000
0x7bec:	0x00000000	0x00000000	0x00000000	0x00000000
0x7bfc:	0x00007c4d
```

For easy explanation, let paste a snippet code of the bootloader:

```
 99   # Set up the stack pointer and call into C.
100   movl    $start, %esp
101     7c43:   bc 00 7c 00 00          mov    $0x7c00,%esp
102   call    bootmain
103     7c48:   e8 e6 00 00 00          call   7d33 <bootmain>
104
105   # If bootmain returns (it shouldn't), trigger a Bochs
106   # breakpoint if running under Bochs, then loop.
107   movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
108     7c4d:   66 b8 00 8a             mov    $0x8a00,%ax
109   movw    %ax, %dx
110     7c51:   66 89 c2                mov    %ax,%dx
```

The first cell of the stack(the four continous bytes leading by the address 0x7bfc) stores the address of the right instruction behind the `call bootmain` instruction: `0x00007c4d`.

Right after entering `bootmain()`, `ebp`, `edi`, `esi` and `ebx` are pushed to the stack afterwards which, at the time, all of their values are 0. So 4 consective 0 values are seen at the address `0x7bec`, `0x7bf0`, `0x7bf4` and `0x7bf8`.

```
284 00007d33 <bootmain>:
285 {
286     7d33:   55                      push   %ebp
287     7d34:   89 e5                   mov    %esp,%ebp
288     7d36:   57                      push   %edi
289     7d37:   56                      push   %esi
290     7d38:   53                      push   %ebx
291     7d39:   83 ec 10                sub    $0x10,%esp
292   readseg((uchar*)elf, 4096, 0);
293     7d3c:   6a 00                   push   $0x0
294     7d3e:   68 00 10 00 00          push   $0x1000
295     7d43:   68 00 00 01 00          push   $0x10000
296     7d48:   e8 a1 ff ff ff          call   7cee <readseg>
297   if(elf->magic != ELF_MAGIC)
298     7d4d:   83 c4 10                add    $0x10,%esp
299     7d50:   81 3d 00 00 01 00 7f    cmpl   $0x464c457f,0x10000
300     7d57:   45 4c 46
301     7d5a:   75 21                   jne    7d7d <bootmain+0x4a>
302   ph = (struct proghdr*)((uchar*)elf + elf->phoff);
303     7d5c:   a1 1c 00 01 00          mov    0x1001c,%eax
304     7d61:   8d 98 00 00 01 00       lea    0x10000(%eax),%ebx
305   eph = ph + elf->phnum;
306     7d67:   0f b7 35 2c 00 01 00    movzwl 0x1002c,%esi
307     7d6e:   c1 e6 05                shl    $0x5,%esi
308     7d71:   01 de                   add    %ebx,%esi
309   for(; ph < eph; ph++){
310     7d73:   39 f3                   cmp    %esi,%ebx
311     7d75:   72 15                   jb     7d8c <bootmain+0x59>
312   entry();
313     7d77:   ff 15 18 00 01 00       call   *0x10018
314 }
```

Till now the value of `esp` should be `0x7bec`, it is then, subtracted by `0x10`, so points to `0x7bdc`. The following instructions are related to calling to `readseg` which modifies esp, we don't need pay too much attention to it, we only need to pay attention to the value of `esp` after these instructions.

```
291     7d39:   83 ec 10                sub    $0x10,%esp
292   readseg((uchar*)elf, 4096, 0);
293     7d3c:   6a 00                   push   $0x0
294     7d3e:   68 00 10 00 00          push   $0x1000
295     7d43:   68 00 00 01 00          push   $0x10000
296     7d48:   e8 a1 ff ff ff          call   7cee <readseg>
```

Because I use gdb to track the excution, so I know that after the excution of the above instructions, the value of `esp` is `0x7bd0`. Note `esp` is then immediately added an offset of `0x10`, which, turns it to `0x7be0`.

There are other instructions before entering the kernel but it doesn't matter as well, we just set a breakpoint at the entering point of the kernel and you would see the value of `esp` is still `0x7be0`. Then comes the last instruction:

```
312   entry();
313     7d77:   ff 15 18 00 01 00       call   *0x10018
314 }
315     7d7d:   8d 65 f4                lea    -0xc(%ebp),%esp
```

After this `call` instruction, the address 7d7d is pushed into the stack addressed at `0x7bdc`. So the final notes:

```
0x7bdc:	0x00007d7d  // the right instruction behind the instruction the bootloader calling kernel entry, i.e. the ret address of the kernel entry
0x7be0 ~ 0x7be8: 0x00000000  // not use
0x7bec ~ 0x7bf8: 0x00000000  // `ebp`, `edi`, `esi` and `ebx` respectively from high address to low
0x7bfc: 0x00007c4d // the ret address of bootmain
0x7c00 ~ above: // not the stack space
```

---
# lecture 3

# Homework 2

```
void
runcmd(struct cmd *cmd)
{
  int p[2], r;
  struct execcmd *ecmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;

  if(cmd == 0)
    _exit(0);

  switch(cmd->type){
  default:
    fprintf(stderr, "unknown runcmd\n");
    _exit(-1);

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    if (execv(ecmd->argv[0], ecmd->argv) < 0) {
        perror("exec failed");
    }
    break;

  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;

    if (cmd->type == '>') close(1);
    else close(0);
    if (open(rcmd->file, rcmd->flags, 0666) < 0) {
        perror("open failed");
        break;
    }
    runcmd(rcmd->cmd);
    break;

  case '|':
    pcmd = (struct pipecmd*)cmd;

    int fd[2];
    if (pipe(fd) < 0) {
        perror("pipe failed");
        break;
    }
    if (fork1()) { // parent process closes the read end of the pipe and use
                   // the write end of the pipe as stdout
        close(fd[0]);
        close(1);
                dup(fd[1]);
        runcmd(pcmd->left);
    } else {       // child process closes the write end of the pipe and use
                   // the read end of the pipe as stdin
        close(fd[1]);
        close(0);
        dup(fd[0]);
        runcmd(pcmd->right);
    }
    break;
  }
  _exit(0);
}
```

# lab2

# how GDT is initialized?

The LGDT instruction loads a linear base address and limit value from a six-byte data operand in memory into the GDTR: https://pdos.csail.mit.edu/6.828/2018/readings/i386/LGDT.htm.

A bit clarify about whether the operand is 16-bit or 32-bit, normally, in read mode and a `.code16` section the operand is 16 bit, otherwise it's 32 bit. An instruction prefix can be used to change this behavior: http://web.mit.edu/rhel-doc/3/rhel-as-en-3/i386-prefixes.html.

In JOS source code, when the LGDT instruction is called to load the GDT, it's in 16 bit<sup>1</sup>, so the LGDT instruction will only use the first 5 bytes of the 6-byte data.

1. you can change the source code of `boot.S` to add an `addr32` prefix to the `lgdt    gdtdesc` instruction and compare the generated asm `obj/boot/boot.asm`
