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
