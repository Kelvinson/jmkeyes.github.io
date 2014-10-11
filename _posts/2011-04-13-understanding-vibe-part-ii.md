---
layout: post
title: ! 'Understanding ViBE: Part II'
date: 2011-04-13 00:08:37.000000000 -07:00
status: published
type: post
---
*In the [last part][1], I began reverse engineering the ViBE binary
blob by analyzing its public API and the design strategies employed
by the author. In this part I will analyze the ViBE object code
distributed by the author.*

Looking into the headers of the binary blob reveal some interesting 
facts. In the symbol table are the names of the exposed functions 
described in the API. There are also undefined symbols that are bound 
at link time: `printf`/`puts`, `malloc`/`calloc`/`free`, 
`srand`/`rand`, `time` and `__assert_fail`. The last symbol is special. 

After failing an assertion coded using the `assert` macro, the program 
emits a message containing the scalar expression of the assertion and 
the file, function, and line number of where the failed assertion 
triggered. Because its arguments are constants, the compiler wrote them 
into the read-only data section. This can help uncover the internals of 
the function around where the assertion exists. Checking the text 
section confirms that there are many relocations that the linker will 
fix during the linking phase. Now the fun part: disassembly.

Functions on Linux/i686 usually follow a very common sequence of 
operations. The first part of the sequence involves setting up a new 
stack frame, followed by allocating memory on the stack for local 
variables and occasionally some function arguments. The function 
executes its body of code. Finally, the function loads the return value 
into a specific register, the stack is de-incremented, and the function 
returns to the caller.

The `libvibeModelPrintParameters` symbol starts at offset 0. It begins 
with a new stack frame and some extra stack space for local variables. 
It also saves the contents of `%esi` and `%ebx` to the stack. The 
object's relocations have not been applied yet.

{% highlight objdump %}
00000000 <libvibeModelPrintParameters>:
       0:       55                      push   %ebp
       1:       89 e5                   mov    %esp,%ebp
       3:       56                      push   %esi
       4:       53                      push   %ebx
       5:       83 ec 20                sub    $0x20,%esp
       8:       8b 45 08                mov    0x8(%ebp),%eax
       b:       8b 70 1c                mov    0x1c(%eax),%esi
       e:       8b 45 08                mov    0x8(%ebp),%eax
      11:       8b 58 14                mov    0x14(%eax),%ebx
      14:       8b 45 08                mov    0x8(%ebp),%eax
      17:       8b 48 18                mov    0x18(%eax),%ecx
      1a:       8b 45 08                mov    0x8(%ebp),%eax
      1d:       8b 50 10                mov    0x10(%eax),%edx
      20:       b8 00 00 00 00          mov    $0x0,%eax 21: R_386_32    .rodata
      25:       89 74 24 10             mov    %esi,0x10(%esp)
      29:       89 5c 24 0c             mov    %ebx,0xc(%esp)
      2d:       89 4c 24 08             mov    %ecx,0x8(%esp)
      31:       89 54 24 04             mov    %edx,0x4(%esp)
      35:       89 04 24                mov    %eax,(%esp)
      38:       e8 fc ff ff ff          call   39 <libvibeModelPrintParameters+0x39> 39: R_386_PC32  printf
      3d:       b8 00 00 00 00          mov    $0x0,%eax
      42:       83 c4 20                add    $0x20,%esp
      45:       5b                      pop    %ebx
      46:       5e                      pop    %esi
      47:       5d                      pop    %ebp
      48:       c3                      ret
{% endhighlight %}

The disassembler indicates the function call in the middle of this 
routine is a call to `printf`. Since `printf` requires a format string 
to display its arguments, it's possible to find the number and type of 
each argument given. At offset `libvibeModelPrintParameters+0x21`, the 
disassembler identified a relocation to the read-only data section at 
`(rodata+0x0)`. The contents of that location are:

{% highlight text %}
0000 5573696e 67205669 42652062 61636b67  Using ViBe backg
0010 726f756e 64207375 62747261 6374696f  round subtractio
0020 6e20616c 676f7269 74686d0a 20202d20  n algorithm.  -
0030 4e756d62 6572206f 66207361 6d706c65  Number of sample
0040 73207065 72207069 78656c3a 20202020  s per pixel:
0050 20202025 3033640a 20202d20 4e756d62     %03d.  - Numb
0060 6572206f 66206d61 74636865 73206e65  er of matches ne
0070 65646564 3a202020 20202020 20202025  eded:          %
0080 3033640a 20202d20 4d617463 68696e67  03d.  - Matching
0090 20746872 6573686f 6c643a20 20202020   threshold:
00a0 20202020 20202020 20202025 3033640a             %03d.
00b0 20202d20 4d6f6465 6c207570 64617465    - Model update
00c0 20737562 73616d70 6c696e67 20666163   subsampling fac
00d0 746f723a 20202025 3033640a 00766962  tor:   %03d..vib
{% endhighlight %}

From offset 0x0 to 0xd in the read-only data is the format string. This 
shows that the call to printf has four arguments that are all either 
casted-to or are really 32 bit integers that have a field width of 
three characters, padded on the left with zeros. 

Going back to the disassembly of libvibeModelPrintParameters, at offset 
0x8 the function loads a 32 bit value from 0x8 (8) bytes into the base 
pointer into the accumulator. This is likely the function's argument: 
the pointer to a vibeModel_t. The next operation dereferences 0x1c (28) 
bytes into the accumulator and loads the source index with a 32 bit 
value from the accumulator. This continues, with different destination 
registers, three more times with offsets 0x14 (20), 0x18 (24), and 0x10 
(16) into the accumulator. Each of the arguments are copied from their 
registers and onto the stack, along with a pointer to the format 
string. This segment of code corroborates the format string: there are 
four 32 bit arguments loaded onto the stack. Then the function finally 
calls printf, loads the accumulator with zero, restores the registers 
it saved, cleans up the stack and returns to the caller. 

{% highlight c %}
int libvibeModelPrintParameters(struct vibeModel_t *model) {
    int32_t num_samples, num_matches, match_threshold, subsample_factor;

    num_samples = *(uint32_t *) ((uint8_t *) model + 0x1c);
    num_matches = *(uint32_t *) ((uint8_t *) model + 0x14);
    match_threshold = *(uint32_t *) ((uint8_t *) model + 0x18);
    subsample_factor = *(uint32_t *) ((uint8_t *) model + 0x10);

    printf(
        "Using ViBE background subtraction algorithm\n"
        "  - Number of samples per pixel:       %03d\n"
        "  - Number of matches needed:          %03d\n"
        "  - Matching threshold:                %03d\n"
        "  - Model update subsampling factor:   %03d\n"
        num_samples, num_matches, match_threshold, subsample_factor
    );

    return 0;
}
{% endhighlight %}

It's now almost certain that there are at least four distinct 32 bit 
integers as members within the vibeModel_t structure. I refer to the 
possible structure members as "almost" certain because there is no 
absolute measurement here: it really is just a very educated guess. 
Without access to the original source code used to compile the blob, 
trying to reason about its original contents will only be as accurate 
as what its contents are most likely to be. The comp.lang.c news group 
calls this the cow-from-hamburger problem. This is where most 
disassemblers and pedantic people throw up their hands and/or boil 
over. Reverse engineering, like compiling efficiently, is not a perfect 
mathematical process but a series of educated guesses based on insider 
knowledge. It's definitely far from impossible but it requires 
knowledge outside of the domain of pure C programming.

Immediately after the return opcode is another at offset 0x49 that 
generates a new stack frame. It's likely that this is a function that 
has had its name removed from the symbol table. We can still 
disassemble and analyze the function, but for clarity I'll name it 
`lambda00` for now. This naming scheme comes from Scheme where 'lambda' 
denotes an anonymous function. This function is also bigger than the 
previous one. 

{% highlight objdump %}
00000049 <lambda00>:
      49:       55                      push   %ebp
      4a:       89 e5                   mov    %esp,%ebp
      4c:       53                      push   %ebx
      4d:       83 ec 24                sub    $0x24,%esp
      50:       a1 00 00 00 00          mov    0x0,%eax 51: R_386_32    .bss
      55:       85 c0                   test   %eax,%eax
      57:       75 69                   jne    c2 <libvibeModelPrintParameters+0xc2>
      59:       c7 04 24 00 00 04 00    movl   $0x40000,(%esp)
      60:       e8 fc ff ff ff          call   61 <libvibeModelPrintParameters+0x61> 61: R_386_PC32  malloc
      65:       a3 04 00 00 00          mov    %eax,0x4 66: R_386_32    .bss
      6a:       a1 04 00 00 00          mov    0x4,%eax 6b: R_386_32    .bss
      6f:       85 c0                   test   %eax,%eax
      71:       75 24                   jne    97 <libvibeModelPrintParameters+0x97>
      73:       c7 44 24 0c f3 05 00 00 movl   $0x5f3,0xc(%esp) 77: R_386_32    .rodata
      7b:       c7 44 24 08 39 00 00 00 movl   $0x39,0x8(%esp)
      83:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp) 87: R_386_32    .rodata
      8b:       c7 04 24 ef 00 00 00    movl   $0xef,(%esp) 8e: R_386_32    .rodata
      92:       e8 fc ff ff ff          call   93 <libvibeModelPrintParameters+0x93> 93: R_386_PC32  __assert_fail
      97:       c7 45 f4 00 00 00 00    movl   $0x0,-0xc(%ebp)
      9e:       eb 19                   jmp    b9 <libvibeModelPrintParameters+0xb9>
      a0:       a1 04 00 00 00          mov    0x4,%eax a1: R_386_32    .bss
      a5:       8b 55 f4                mov    -0xc(%ebp),%edx
      a8:       c1 e2 02                shl    $0x2,%edx
      ab:       8d 1c 10                lea    (%eax,%edx,1),%ebx
      ae:       e8 fc ff ff ff          call   af <libvibeModelPrintParameters+0xaf> af: R_386_PC32  rand
      b3:       89 03                   mov    %eax,(%ebx)
      b5:       83 45 f4 01             addl   $0x1,-0xc(%ebp)
      b9:       81 7d f4 ff ff 00 00    cmpl   $0xffff,-0xc(%ebp)
      c0:       7e de                   jle    a0 <libvibeModelPrintParameters+0xa0>
      c2:       a1 00 00 00 00          mov    0x0,%eax c3: R_386_32    .bss
      c7:       83 c0 01                add    $0x1,%eax
      ca:       a3 00 00 00 00          mov    %eax,0x0 cb: R_386_32    .bss
      cf:       a1 04 00 00 00          mov    0x4,%eax d0: R_386_32    .bss
      d4:       8b 15 00 00 00 00       mov    0x0,%edx d6: R_386_32    .bss
      da:       81 e2 ff ff 00 00       and    $0xffff,%edx
      e0:       c1 e2 02                shl    $0x2,%edx
      e3:       01 d0                   add    %edx,%eax
      e5:       8b 00                   mov    (%eax),%eax
      e7:       83 c4 24                add    $0x24,%esp
      ea:       5b                      pop    %ebx
      eb:       5d                      pop    %ebp
      ec:       c3                      ret
{% endhighlight %}

Like the previous routine this also has a number of relocations. The 
offsets are not always correct because the disassembler believes this 
is still a part of the previous function. Some of the relocations the 
disassembler has indicated are to known symbols and so they have been 
filled in already. The call to `__assert_fail` stands out when skimming 
quickly through the relocations. As predicted, the routine pushes four 
arguments onto the stack before calling the function. Those four 
arguments are in right-to-left in order: the assertion function, the 
line number, the filename, and the text of the scalar `assert` argument 
as a string. Referencing the read-only data section again for the three 
strings (`0xdd - 0xef`, `0xef - 0x108`, `0x5f3 - 0x601`) shows:

{% highlight text %}
00d0 746f723a 20202025 3033640a 00766962  tor:   %03d..vib
00e0 652d6261 636b6772 6f756e64 2e630072  e-background.c.r
00f0 616e6454 61626c65 20213d20 2828766f  andTable != ((vo
0100 6964202a 29302900 6d6f6465 6c20213d  id *)0).model !=
    ...
05f0 65770063 69726375 6c61725f 72616e64  ew.circular_rand
0600 00                                   .
{% endhighlight %}

So our function was originally called "circular_rand", in a file called 
"vibe-background.c". The assertion was on line 57, and the assertion 
text was "randTable != ((void *)0)". There's now four pieces of data 
that we didn't have before. Additionally, because the name of an 
variable is known, it's a little easier to understand the intent in the 
original code: the assertion checks if a pointer called "randTable" was 
NULL. We know the function is called `circular_rand`. We know there is 
a call to allocate memory and one or more calls to `rand`. From the 
disassembly we can see a lot of comparison and binary arithmetic and we 
know an important variable name refers to something called a 
"randTable". This function performs binary arithmetic on a lot of 
integral values that relate to a portmanteau of the "rand" function 
name and a "table". The function name has "circular" within in and we 
know the function uses a variable from common section where mutable 
global variables exist. Near the end of the program code is a call that 
dereferences a 32 bit integral value from the accumulator and stores it 
back into the accumulator for a return value. Combining these facts 
leads me to believe this function may seed and/or emit random numbers 
from a buffer of them. 

{% highlight c %}
extern uint32_t *global_bss_0x0;
extern uint32_t *global_bss_0x4;

uint32_t lambda00(void) {
/*   50:       a1 00 00 00 00          mov    0x0,%eax 51: R_386_32    .bss */
    uint32_t *eax = global_bss_0x0;
    uint32_t ebp_12;

/*   55:       85 c0                   test   %eax,%eax */
    if (zf_set(eax)) {
/*   57:       75 69                   jne    c2 <libvibeModelPrintParameters+0xc2> */
        goto label_0xc2;
    } else {
/*   59:       c7 04 24 00 00 04 00    movl   $0x40000,(%esp) */
/*   60:       e8 fc ff ff ff          call   61 <libvibeModelPrintParameters+0x61> 61: R_386_PC32  malloc */
        eax = malloc (0x40000);
/*   65:       a3 04 00 00 00          mov    %eax,0x4 66: R_386_32    .bss */
        global_bss_0x4 = eax;
/*   6a:       a1 04 00 00 00          mov    0x4,%eax 6b: R_386_32    .bss */
        eax = global_bss_0x4;
/*   6f:       85 c0                   test   %eax,%eax */
       if (zf_set(eax)) {
/*   71:       75 24                   jne    97 <libvibeModelPrintParameters+0x97> */
           goto label_0x97;
       } else {
/*   73:       c7 44 24 0c f3 05 00 00 movl   $0x5f3,0xc(%esp) 77: R_386_32    .rodata */
/*   7b:       c7 44 24 08 39 00 00 00 movl   $0x39,0x8(%esp) */
/*   83:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp) 87: R_386_32    .rodata */
/*   8b:       c7 04 24 ef 00 00 00    movl   $0xef,(%esp) 8e: R_386_32    .rodata */
/*   92:       e8 fc ff ff ff          call   93 <libvibeModelPrintParameters+0x93> 93: R_386_PC32  __assert_fail */
        __assert_fail("randTable != ((void *)0)", "vibe-background.c", 0x39, "circular_rand");
       }
    }

/*   97:       c7 45 f4 00 00 00 00    movl   $0x0,-0xc(%ebp) */
label_0x97:
    ebp_12 = 0x0;
    goto label_0xb9;
/*   9e:       eb 19                   jmp    b9 <libvibeModelPrintParameters+0xb9> */
label_0xa0:
/*   a0:       a1 04 00 00 00          mov    0x4,%eax a1: R_386_32    .bss */
    eax = global_bss_0x4;
/*   a5:       8b 55 f4                mov    -0xc(%ebp),%edx */
    eax = ebp_12;
/*   a8:       c1 e2 02                shl    $0x2,%edx */
    edx <<= 0x2;
/*   ab:       8d 1c 10                lea    (%eax,%edx,1),%ebx */
    ebx = (eax + edx * 1);
/*   ae:       e8 fc ff ff ff          call   af <libvibeModelPrintParameters+0xaf> af: R_386_PC32  rand */
    eax = rand();
/*   b3:       89 03                   mov    %eax,(%ebx) */
    *ebx = eax;
/*   b5:       83 45 f4 01             addl   $0x1,-0xc(%ebp) */
     ebp_12++;
label_0xb9:
/*   b9:       81 7d f4 ff ff 00 00    cmpl   $0xffff,-0xc(%ebp) */
    if (ebp_12 <= 0xffff) {
/*   c0:       7e de                   jle    a0 <libvibeModelPrintParameters+0xa0> */
            goto label_0xa0;
     } else {
/*   c2:       a1 00 00 00 00          mov    0x0,%eax c3: R_386_32    .bss */
label_0xc2:
        eax = global_bss_0x0;
/*   c7:       83 c0 01                add    $0x1,%eax */
        eax++;
/*   ca:       a3 00 00 00 00          mov    %eax,0x0 cb: R_386_32    .bss */
        global_bss_0x0 = eax;
/*    cf:       a1 04 00 00 00          mov    0x4,%eax d0: R_386_32   .bss */
        edx = global_bss_0x4;
/*    d4:       8b 15 00 00 00 00       mov    0x0,%edx d6: R_386_32   .bss */
        edx = global_bss_0x0;
/*    da:       81 e2 ff ff 00 00       and    $0xffff,%edx */
        edx &amp;= 0xffff;
/*    e0:       c1 e2 02                shl    $0x2,%edx */
        edx <<= 0x2;
/*    e3:       01 d0                   add    %edx,%eax */
        eax += edx;
/*   e5:       8b 00                   mov    (%eax),%eax */
    }
    return *eax;
}
{% endhighlight %}

You won't hear any argument from me: that is one ugly mess. However, 
because the object code was compiled with some extra information that 
wasn't stripped out, it's easy to pick out which variables the compiler 
stored in registers for the life of the function call. All references 
to bss+0x0 and bss+0x4 are global0 and global1 respectively. All local 
variables are named in incremental order. All registers are implicitly 
32 bit integral pointers. All labels follow the format label_0xXX, 
where X is a hexadecimal offset.

{% highlight c %}
uint32_t circular_rand() {
    extern uint32_t global0, global1;
    uint32_t local0;

    eax = global0;
    if (eax == NULL) {
        eax = malloc(0x40000);
        global1 = eax;
        eax = global1;
        if (eax == NULL) {
            __assert_fail("randTable != ((void*)0)", "vibe-background.c", 57, "circular_rand");
            /* Unreachable by abort(). */
        } else {
            goto label_0xc2;
        }
    } else {
        goto label_0x97;
    }

label_0x97:
    local0 = 0;
    goto label_0xb9;

label_0xa0:
    eax = global1;
    edx = local0;
    edx <<= 0x2;
    ebx = eax + edx * 1;
    eax = rand();
    *ebx = eax;

label_0xb9:
    if (local0 <= 0xffff)
        goto label_0xa0;

label_0xc2:
    eax = global0;
    eax += 1;
    global0 = eax;
    eax = global1;
    edx = global0;
    edx &amp;= 0xffff;
    edx <<= 0x2;
    eax += edx;

    return *eax;
}
{% endhighlight %}

That's a bit better, but it's still not that great. Ideally there 
shouldn't be any implicit registers in the code and the arithmetic 
should be readable.

{% highlight c %}
uint32_t circular_rand() {
    static uint32_t *randTable = NULL;
    static uint32_t index = 0;
    int i;

    if (index == 0) {
        randTable = malloc(0x40000);

        assert(randTable != NULL);

        for (i = 0; i <= 0xffff; i++) {
            randTable[i] = rand();
        }
    }  

    index++;

    return randTable[(index & 0xffff)];
}
{% endhighlight %}

All of the function's uninitialized and zero-initialized variables, in 
both the global scope and the function-static scope, have references 
into the .bss section. This means that any of the static variables 
could in fact be global variables, but we'll leave them in the 
function-static scope since we don't have any other references to them. 
There's a memory leak in this function. Do you see it?

(Continued in [part III][2])

  [1]: {{ site.baseurl }}/post/understanding-vibe-part-i
  [2]: {{ site.baseurl }}/post/understanding-vibe-part-iii

