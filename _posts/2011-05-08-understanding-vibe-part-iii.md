---
layout: post
title: 'Understanding ViBE: Part III'
date: 2011-05-08 20:51:04.000000000 -07:00
status: published
type: post
---
*In the [last part][1], I took apart the first few functions of the ViBE
binary blob and looked at the design strategies employed by the author.
In this part I continue doing so.*

So far I've taken apart and reconstructed the source code for the first 
two functions successfully. The next function is the constructor for a 
ViBE model followed by eight getters/setters for tweaking the 
algorithm. Because this function deals with constructing a new instance 
of the algorithm, I expect to find similar offsets into the model's 
internal structure members. Cross-referencing them with the information 
we gained in `libvibeModelPrintParameters` may yield some new 
information about the algorithm itself.

{% highlight objdump %}
000000ed <libvibeModelNew>:
      ed:       55                      push   %ebp
      ee:       89 e5                   mov    %esp,%ebp
      f0:       83 ec 28                sub    $0x28,%esp
      f3:       c7 45 f4 00 00 00 00    movl   $0x0,-0xc(%ebp)
      fa:       c7 44 24 04 20 00 00 00 movl   $0x20,0x4(%esp)
     102:       c7 04 24 01 00 00 00    movl   $0x1,(%esp)
     109:       e8 fc ff ff ff          call   10a <libvibeModelNew+0x1d>       10a: R_386_PC32 calloc
     10e:       89 45 f4                mov    %eax,-0xc(%ebp)
     111:       83 7d f4 00             cmpl   $0x0,-0xc(%ebp)
     115:       75 24                   jne    13b <libvibeModelNew+0x4e>
     117:       c7 44 24 0c e3 05 00 00 movl   $0x5e3,0xc(%esp) 11b: R_386_32   .rodata
     11f:       c7 44 24 08 48 00 00 00 movl   $0x48,0x8(%esp)
     127:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  12b: R_386_32   .rodata
     12f:       c7 04 24 08 01 00 00    movl   $0x108,(%esp)    132: R_386_32   .rodata
     136:       e8 fc ff ff ff          call   137 <libvibeModelNew+0x4a>       137: R_386_PC32 __assert_fail
     13b:       8b 45 f4                mov    -0xc(%ebp),%eax
     13e:       c7 40 10 14 00 00 00    movl   $0x14,0x10(%eax)
     145:       8b 45 f4                mov    -0xc(%ebp),%eax
     148:       c7 40 14 14 00 00 00    movl   $0x14,0x14(%eax)
     14f:       8b 45 f4                mov    -0xc(%ebp),%eax
     152:       c7 40 18 02 00 00 00    movl   $0x2,0x18(%eax)
     159:       8b 45 f4                mov    -0xc(%ebp),%eax
     15c:       c7 40 1c 10 00 00 00    movl   $0x10,0x1c(%eax)
     163:       c7 04 24 00 00 00 00    movl   $0x0,(%esp)
     16a:       e8 fc ff ff ff          call   16b <libvibeModelNew+0x7e>       16b: R_386_PC32 time
     16f:       89 04 24                mov    %eax,(%esp)
     172:       e8 fc ff ff ff          call   173 <libvibeModelNew+0x86>       173: R_386_PC32 srand
     177:       8b 45 f4                mov    -0xc(%ebp),%eax
     17a:       c9                      leave
     17b:       c3                      ret
{% endhighlight %}

There are some typical compiler idioms and the another similar pattern 
emerges: generation of a stack frame, stack expansion for local 
storage, program code, and then cleanup code. We know this function has 
no arguments so it's likely that all uses of the base and stack pointer 
registers are references to the local storage area. When reading the 
disassembled object code it's these patterns that stand out. With them, 
generating a hypothesis about the surrounding code and the context it 
is in can corroborate already gathered evidence and point to which 
constructs generated the code. Instead of finding an absolute result, 
we can look at which evidence supports which belief and expand on that. 

This code is trivial. The use of the `(base pointer - 12)` indicates a 
local 32 bit variable initialized to zero. The function copies two 
32-bit integer arguments onto the stack from right to left: 32 and 1 at 
offsets above the stack pointer. Then the function calls `calloc` and 
stores its result at the local variable's storage area. There is a 
really simple pattern: an allocation using calloc of one element of 32 
bytes in size, stored into the only local variable in scope. If that 
variable is NULL an assertion fires and aborts program flow. After 
that, the compiler stores a number of bytes into the allocated memory 
at various offsets. The program pushes a 32 bit integer zero onto the 
stack, calls `time`, replaces the zero on the stack with the result and 
then calls `srand`. This is a typical idiom in C which seeds the 
pseudo-random number generator with the current time in seconds since 
the epoch. Finally, the function stores the local variable in the 
accumulator and the function returns to the caller. We know `calloc` 
returns a pointer so it's very likely that the local variable is also a 
pointer. It's almost guaranteed that this pointer is of the type 
vibeModel_t, because the function prototype returns a pointer of that 
type.

{% highlight c %}
vibeModel_t * libvibeModelNew(void) {
    vibeModel_t *model = NULL;

    model = calloc(1, 32);

    assert(model != NULL);

    /*
     * Assign the 32 bit integers 0x14, 0x14, 0x02, and 0x10 at 0x10,
     * 0x14, 0x18, and 0x1c respectively. If we actually knew the
     * structure members' names they would be used instead. Use
     * "fake" members instead labeled "xFF" where FF is the offset
     * into the structure in bytes.
     */

    model->x10 = 0x14;
    model->x14 = 0x14;
    model->x18 = 0x02;
    model->x1c = 0x10;

    return model;
}
{% endhighlight %}

We've learned that the vibeModel_t structure has an in-memory size of 
32 bytes. We also know there are at least four members that are 32 but 
integral values starting at 16 bytes offset into the structure. 
Including only this function's used members leaves 16 bytes that are 
otherwise unused. 

The next eight functions are all very similar to each other. They are 
the four pairs of getters and setters I mentioned earlier. They each 
tweak the algorithm's public state and change values at various offsets 
within the vibeModel_t structure. I've grouped the next two functions 
together to speed things up.

{% highlight objdump %}
0000017c <libvibeModelSetNumberOfSamples>:
     17c:       55                      push   %ebp
     17d:       89 e5                   mov    %esp,%ebp
     17f:       83 ec 18                sub    $0x18,%esp
     182:       83 7d 08 00             cmpl   $0x0,0x8(%ebp)
     186:       75 24                   jne    1ac <libvibeModelSetNumberOfSamples+0x30>
     188:       c7 44 24 0c c4 05 00 00 movl   $0x5c4,0xc(%esp) 18c: R_386_32   .rodata
     190:       c7 44 24 08 5a 00 00 00 movl   $0x5a,0x8(%esp)
     198:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  19c: R_386_32   .rodata
     1a0:       c7 04 24 08 01 00 00    movl   $0x108,(%esp)    1a3: R_386_32   .rodata
     1a7:       e8 fc ff ff ff          call   1a8 <libvibeModelSetNumberOfSamples+0x2c>        1a8: R_386_PC32 __assert_fail
     1ac:       83 7d 0c 00             cmpl   $0x0,0xc(%ebp)
     1b0:       75 24                   jne    1d6 <libvibeModelSetNumberOfSamples+0x5a>
     1b2:       c7 44 24 0c c4 05 00 00 movl   $0x5c4,0xc(%esp) 1b6: R_386_32   .rodata
     1ba:       c7 44 24 08 5b 00 00 00 movl   $0x5b,0x8(%esp)
     1c2:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  1c6: R_386_32   .rodata
     1ca:       c7 04 24 1d 01 00 00    movl   $0x11d,(%esp)    1cd: R_386_32   .rodata
     1d1:       e8 fc ff ff ff          call   1d2 <libvibeModelSetNumberOfSamples+0x56>        1d2: R_386_PC32 __assert_fail
     1d6:       8b 45 08                mov    0x8(%ebp),%eax
     1d9:       8b 55 0c                mov    0xc(%ebp),%edx
     1dc:       89 50 10                mov    %edx,0x10(%eax)
     1df:       b8 00 00 00 00          mov    $0x0,%eax
     1e4:       c9                      leave
     1e5:       c3                      ret    

000001e6 <libvibeModelGetNumberOfSamples>:
     1e6:       55                      push   %ebp
     1e7:       89 e5                   mov    %esp,%ebp
     1e9:       83 ec 18                sub    $0x18,%esp
     1ec:       83 7d 08 00             cmpl   $0x0,0x8(%ebp)
     1f0:       75 24                   jne    216 <libvibeModelGetNumberOfSamples+0x30>
     1f2:       c7 44 24 0c a4 05 00 00 movl   $0x5a4,0xc(%esp) 1f6: R_386_32   .rodata
     1fa:       c7 44 24 08 65 00 00 00 movl   $0x65,0x8(%esp)
     202:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  206: R_386_32   .rodata
     20a:       c7 04 24 08 01 00 00    movl   $0x108,(%esp)    20d: R_386_32   .rodata
     211:       e8 fc ff ff ff          call   212 <libvibeModelGetNumberOfSamples+0x2c>        212: R_386_PC32 __assert_fail
     216:       8b 45 08                mov    0x8(%ebp),%eax
     219:       8b 40 10                mov    0x10(%eax),%eax
     21c:       c9                      leave
     21d:       c3                      ret
{% endhighlight %}

Each of the setters performs the typical stack frame setup and pulls 
out a single argument at eight bytes above the base pointer. This is 
the pointer-to-vibeModel_t structure argument, "model", given by the 
public API. The second argument is a constant 32 bit integer that the 
function stores in the structure. There are two assertions made: one to 
make sure the "model" argument is non-null, and one to make sure the 
constant integer argument is above zero. These corroborate the strings 
stored in the read-only data section for each assertion. The function 
loads the model structure pointer into the accumulator and the integer 
argument into another register. The function stores the value at a 
specific offset in the structure and always returns zero. 

Each getter is similar to the setter described above, but there is no 
mutation of state. There are the typical assertion checks and then the 
function loads the contents of the structure member at a specific 
offset into the accumulator and returns to the caller. 

{% highlight c %}
int32_t libvibeModelSetNumberOfSamples(vibeModel_t *model, const uint32_t numberOfSamples) {
    assert(model != NULL);
    assert(numberOfSamples > 0);

    model->x10 = numberOfSamples;
    return 0;
}

uint32_t libvibeModelGetNumberOfSamples(const vibeModel_t *model) {
    assert(model != NULL);
    return model->x10;
}
{% endhighlight %}

It's interesting to note the author chose to implement the setter by 
making it always return a signed 32 bit integer zero but everything 
else as an unsigned 32 bit integer. Not only is it backward and 
inefficient to return a constant like this, but it may also be a source 
of bugs in the future due to the way integers work in C. These 
functions are trivial; there really isn't any reason for this added 
complexity. Thanks to the public API, we can now we can map each of the 
prior structure offsets from the getter/setter pairs (0x10, 0x14, 0x18, 
and 0x1c) to the labels "numberOfSamples", "matchingThreshold", 
"matchingNumber" and "updateFactor". Note that these are the same 
offsets we assigned to in the constructor.

Immediately following the return instruction of the last of the getters 
and setters is another stripped function at offset 0x404. This function 
is significantly longer than the others. We'll name it `lambda01` for 
now. 

{% highlight objdump %}
00000404 <lambda01>:
     404:       55                      push   %ebp
     405:       89 e5                   mov    %esp,%ebp
     407:       83 ec 28                sub    $0x28,%esp
     40a:       8b 45 14                mov    0x14(%ebp),%eax
     40d:       0f af 45 1c             imul   0x1c(%ebp),%eax
     411:       03 45 18                add    0x18(%ebp),%eax
     414:       89 45 f4                mov    %eax,-0xc(%ebp)
     417:       8b 45 18                mov    0x18(%ebp),%eax
     41a:       3b 45 0c                cmp    0xc(%ebp),%eax
     41d:       7c 24                   jl     443 <libvibeModelGetUpdateFactor+0x77>
     41f:       c7 44 24 0c a0 04 00 00 movl   $0x4a0,0xc(%esp) 423: R_386_32   .rodata
     427:       c7 44 24 08 af 00 00 00 movl   $0xaf,0x8(%esp)
     42f:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  433: R_386_32   .rodata
     437:       c7 04 24 6b 01 00 00    movl   $0x16b,(%esp)    43a: R_386_32   .rodata
     43e:       e8 fc ff ff ff          call   43f <libvibeModelGetUpdateFactor+0x73>   43f: R_386_PC32 __assert_fail
     443:       8b 45 1c                mov    0x1c(%ebp),%eax
     446:       3b 45 10                cmp    0x10(%ebp),%eax
     449:       7c 24                   jl     46f <libvibeModelGetUpdateFactor+0xa3>
     44b:       c7 44 24 0c a0 04 00 00 movl   $0x4a0,0xc(%esp) 44f: R_386_32   .rodata
     453:       c7 44 24 08 b0 00 00 00 movl   $0xb0,0x8(%esp)
     45b:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  45f: R_386_32   .rodata
     463:       c7 04 24 75 01 00 00    movl   $0x175,(%esp)    466: R_386_32   .rodata
     46a:       e8 fc ff ff ff          call   46b <libvibeModelGetUpdateFactor+0x9f>   46b: R_386_PC32 __assert_fail
     46f:       e8 d5 fb ff ff          call   49 <libvibeModelPrintParameters+0x49>
     474:       83 e0 07                and    $0x7,%eax
     477:       83 f8 07                cmp    $0x7,%eax
     47a:       0f 87 d0 00 00 00       ja     550 <libvibeModelGetUpdateFactor+0x184>
     480:       8b 04 85 a4 01 00 00    mov    0x1a4(,%eax,4),%eax      483: R_386_32   .rodata
     487:       ff e0                   jmp    *%eax
     489:       8b 45 0c                mov    0xc(%ebp),%eax
     48c:       83 e8 01                sub    $0x1,%eax
     48f:       3b 45 18                cmp    0x18(%ebp),%eax
     492:       0f 8e c6 00 00 00       jle    55e <libvibeModelGetUpdateFactor+0x192>
     498:       83 45 f4 01             addl   $0x1,-0xc(%ebp)
     49c:       e9 d3 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     4a1:       8b 45 0c                mov    0xc(%ebp),%eax
     4a4:       83 e8 01                sub    $0x1,%eax
     4a7:       3b 45 18                cmp    0x18(%ebp),%eax
     4aa:       7e 04                   jle    4b0 <libvibeModelGetUpdateFactor+0xe4>
     4ac:       83 45 f4 01             addl   $0x1,-0xc(%ebp)
     4b0:       8b 45 10                mov    0x10(%ebp),%eax
     4b3:       83 e8 01                sub    $0x1,%eax
     4b6:       3b 45 1c                cmp    0x1c(%ebp),%eax
     4b9:       0f 8e a2 00 00 00       jle    561 <libvibeModelGetUpdateFactor+0x195>
     4bf:       8b 45 14                mov    0x14(%ebp),%eax
     4c2:       01 45 f4                add    %eax,-0xc(%ebp)
     4c5:       e9 aa 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     4ca:       8b 45 10                mov    0x10(%ebp),%eax
     4cd:       83 e8 01                sub    $0x1,%eax
     4d0:       3b 45 1c                cmp    0x1c(%ebp),%eax
     4d3:       0f 8e 8b 00 00 00       jle    564 <libvibeModelGetUpdateFactor+0x198>
     4d9:       8b 45 14                mov    0x14(%ebp),%eax
     4dc:       01 45 f4                add    %eax,-0xc(%ebp)
     4df:       e9 90 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     4e4:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     4e8:       7e 04                   jle    4ee <libvibeModelGetUpdateFactor+0x122>
     4ea:       83 6d f4 01             subl   $0x1,-0xc(%ebp)
     4ee:       8b 45 10                mov    0x10(%ebp),%eax
     4f1:       83 e8 01                sub    $0x1,%eax
     4f4:       3b 45 1c                cmp    0x1c(%ebp),%eax
     4f7:       7e 6e                   jle    567 <libvibeModelGetUpdateFactor+0x19b>
     4f9:       8b 45 14                mov    0x14(%ebp),%eax
     4fc:       01 45 f4                add    %eax,-0xc(%ebp)
     4ff:       eb 73                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     501:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     505:       7e 63                   jle    56a <libvibeModelGetUpdateFactor+0x19e>
     507:       83 6d f4 01             subl   $0x1,-0xc(%ebp)
     50b:       eb 67                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     50d:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     511:       7e 04                   jle    517 <libvibeModelGetUpdateFactor+0x14b>
     513:       83 6d f4 01             subl   $0x1,-0xc(%ebp)
     517:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     51b:       7e 50                   jle    56d <libvibeModelGetUpdateFactor+0x1a1>
     51d:       8b 45 14                mov    0x14(%ebp),%eax
     520:       29 45 f4                sub    %eax,-0xc(%ebp)
     523:       eb 4f                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     525:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     529:       7e 45                   jle    570 <libvibeModelGetUpdateFactor+0x1a4>
     52b:       8b 45 14                mov    0x14(%ebp),%eax
     52e:       29 45 f4                sub    %eax,-0xc(%ebp)
     531:       eb 41                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     533:       8b 45 0c                mov    0xc(%ebp),%eax
     536:       83 e8 01                sub    $0x1,%eax
     539:       3b 45 18                cmp    0x18(%ebp),%eax
     53c:       7e 04                   jle    542 <libvibeModelGetUpdateFactor+0x176>
     53e:       83 45 f4 01             addl   $0x1,-0xc(%ebp)
     542:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     546:       7e 2b                   jle    573 <libvibeModelGetUpdateFactor+0x1a7>
     548:       8b 45 14                mov    0x14(%ebp),%eax
     54b:       29 45 f4                sub    %eax,-0xc(%ebp)
     54e:       eb 24                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     550:       c7 04 24 80 01 00 00    movl   $0x180,(%esp)    553: R_386_32   .rodata
     557:       e8 fc ff ff ff          call   558 <libvibeModelGetUpdateFactor+0x18c>  558: R_386_PC32 puts
     55c:       eb 16                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     55e:       90                      nop
     55f:       eb 13                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     561:       90                      nop
     562:       eb 10                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     564:       90                      nop
     565:       eb 0d                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     567:       90                      nop
     568:       eb 0a                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     56a:       90                      nop
     56b:       eb 07                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     56d:       90                      nop
     56e:       eb 04                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     570:       90                      nop
     571:       eb 01                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8>
     573:       90                      nop
     574:       8b 45 f4                mov    -0xc(%ebp),%eax
     577:       8b 55 08                mov    0x8(%ebp),%edx
     57a:       8d 04 02                lea    (%edx,%eax,1),%eax
     57d:       0f b6 00                movzbl (%eax),%eax
     580:       c9                      leave
     581:       c3                      ret
{% endhighlight %}

There is a lot to notice about this function. First, there are a lot of 
arguments and a number of local variables. there are also some familiar 
and unfamiliar idioms that may take some time to decode. This function 
has had its symbols stripped so all of the symbol+offset notations 
incorrectly reference the previous function. Ignoring those symbols and 
using the raw jump addresses is sufficient.

First, look at the last instruction before the function epilogue. It's 
a move-zero-extended-byte-to-long instruction. It copies a byte at the 
location specified in the accumulator and stores that value in the back 
into accumulator. This means that no matter what the function does, it 
returns a single byte by dereferencing a value in a register. This is 
analogous to dereferencing a pointer-to-byte value in C. We can imply 
then that this function returns a possible value in the range from zero 
to 256 inclusively (keep in mind the zero extension). That means our 
return type must be an eight bit unsigned value. We also know that 
there are a lot of arguments to this function so a basic prototype 
might be `uint8_t lambda01(type0 param0, type1 param1, ... typeN 
paramN);`

There are a lot of unconditional and conditional jump instructions. 
This likely means the compiler generated a jump table, which it 
commonly uses to compile switch statements. We then can infer that 
there is at least one switch statement in this function. There are also 
the typical calls to `__assert_fail` which as discussed before mean 
that program flow is aborted on error. Because assertions compile down 
to strings, we can gain some more information about the internals of 
this function. Looking into the read-only data section shows this 
function was originally named `getRandomNeighbourPixelValue_8u_C1`. The 
documentation has a similarly named function ending with "_8u_C1R" 
which is used when dealing with grey scale input. We can also guess 
that "8u" refers to eight bit unsigned pixel values. If these 
inferences hold, this function likely uses eight byte unsigned values 
with a stride for the pixels. It's also likely that the pixels are an 
array passed to the function (including its height and width) and that 
if we want a "neighboring" value in an array we're going to want to 
know what pixel to start from; this should be a pair of x-y 
coordinates.  This x-y coordinate pair can't exceed the width and 
height, respectively, of the pixel array (that would cause a crash) so 
we can also infer that constraint.

{% highlight c %}
uint8_t getRandomNeighbourPixelValue_8u_C1() {
    /* 407:       83 ec 28                sub    $0x28,%esp */
    esp -= 0x28;

    /* 40a:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 40d:       0f af 45 1c             imul   0x1c(%ebp),%eax */
    /* 411:       03 45 18                add    0x18(%ebp),%eax */
    eax = ebp[0x14];
    eax *= ebp[0x1c];
    eax += ebp[0x18];

    /* 414:       89 45 f4                mov    %eax,-0xc(%ebp) */
    /* 417:       8b 45 18                mov    0x18(%ebp),%eax */
    ebp[-0xc] = eax;
    eax = ebp[0x18];

    /* 41a:       3b 45 0c                cmp    0xc(%ebp),%eax */
    /* 41d:       7c 24                   jl     443 <libvibeModelGetUpdateFactor+0x77> */
    /* 41f:       c7 44 24 0c a0 04 00 00 movl   $0x4a0,0xc(%esp) 423: R_386_32   .rodata "getRandomNeighbourPixelValue_8u_C1" */
    /* 427:       c7 44 24 08 af 00 00 00 movl   $0xaf,0x8(%esp)  "175" */
    /* 42f:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  433: R_386_32   .rodata "vibe-background.c" */
    /* 437:       c7 04 24 6b 01 00 00    movl   $0x16b,(%esp)    43a: R_386_32   .rodata "x < width" */
    /* 43e:       e8 fc ff ff ff          call   43f <libvibeModelGetUpdateFactor+0x73>   43f: R_386_PC32 __assert_fail */
    if (ebp[0xc] >= eax) {
        __assert_fail("x < width", "vibe-background.c", 175, "getRandomNeighbourPixelValue_8u_C1");
    }

    /* 443:       8b 45 1c                mov    0x1c(%ebp),%eax */
    /* 446:       3b 45 10                cmp    0x10(%ebp),%eax */
    /* 449:       7c 24                   jl     46f <libvibeModelGetUpdateFactor+0xa3> */
    /* 44b:       c7 44 24 0c a0 04 00 00 movl   $0x4a0,0xc(%esp) 44f: R_386_32   .rodata "getRandomNeighbourPixelValue_8u_C1" */
    /* 453:       c7 44 24 08 b0 00 00 00 movl   $0xb0,0x8(%esp) "176" */
    /* 45b:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  45f: R_386_32   .rodata "vibe-background.c" */
    /* 463:       c7 04 24 75 01 00 00    movl   $0x175,(%esp)    466: R_386_32   .rodata "y < height" */
    /* 46a:       e8 fc ff ff ff          call   46b <libvibeModelGetUpdateFactor+0x9f>   46b: R_386_PC32 __assert_fail */
    if (ebp[0x10] >= eax) {
        __assert_fail("y < height", "vibe-background.c", 176, "getRandomNeighbourPixelValue_8u_C1");
    }

    /* 46f:       e8 d5 fb ff ff          call   49 <libvibeModelPrintParameters+0x49> */
    /* 474:       83 e0 07                and    $0x7,%eax */
    /* 477:       83 f8 07                cmp    $0x7,%eax */
    eax = circular_rand();
    eax &= 0x7;

    /* 47a:       0f 87 d0 00 00 00       ja     550 <libvibeModelGetUpdateFactor+0x184> */
    /* 480:       8b 04 85 a4 01 00 00    mov    0x1a4(,%eax,4),%eax      483: R_386_32   .rodata */
    /* 487:       ff e0                   jmp    *%eax */

    /* 489:       8b 45 0c                mov    0xc(%ebp),%eax */
    /* 48c:       83 e8 01                sub    $0x1,%eax */
    /* 48f:       3b 45 18                cmp    0x18(%ebp),%eax */
    /* 492:       0f 8e c6 00 00 00       jle    55e <libvibeModelGetUpdateFactor+0x192> */
    /* 498:       83 45 f4 01             addl   $0x1,-0xc(%ebp) */
    /* 49c:       e9 d3 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */

    /* 4a1:       8b 45 0c                mov    0xc(%ebp),%eax */
    /* 4a4:       83 e8 01                sub    $0x1,%eax */
    /* 4a7:       3b 45 18                cmp    0x18(%ebp),%eax */
    /* 4aa:       7e 04                   jle    4b0 <libvibeModelGetUpdateFactor+0xe4> */
    /* 4ac:       83 45 f4 01             addl   $0x1,-0xc(%ebp) */

    /* 4b0:       8b 45 10                mov    0x10(%ebp),%eax */
    /* 4b3:       83 e8 01                sub    $0x1,%eax */
    /* 4b6:       3b 45 1c                cmp    0x1c(%ebp),%eax */
    /* 4b9:       0f 8e a2 00 00 00       jle    561 <libvibeModelGetUpdateFactor+0x195> */

    /* 4bf:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 4c2:       01 45 f4                add    %eax,-0xc(%ebp) */
    /* 4c5:       e9 aa 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */

    /* 4ca:       8b 45 10                mov    0x10(%ebp),%eax */
    /* 4cd:       83 e8 01                sub    $0x1,%eax */
    /* 4d0:       3b 45 1c                cmp    0x1c(%ebp),%eax */
    /* 4d3:       0f 8e 8b 00 00 00       jle    564 <libvibeModelGetUpdateFactor+0x198> */

    /* 4d9:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 4dc:       01 45 f4                add    %eax,-0xc(%ebp) */
    /* 4df:       e9 90 00 00 00          jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 4e4:       83 7d 18 00             cmpl   $0x0,0x18(%ebp) */
    /* 4e8:       7e 04                   jle    4ee <libvibeModelGetUpdateFactor+0x122> */
    /* 4ea:       83 6d f4 01             subl   $0x1,-0xc(%ebp) */

    /* 4ee:       8b 45 10                mov    0x10(%ebp),%eax */
    /* 4f1:       83 e8 01                sub    $0x1,%eax */
    /* 4f4:       3b 45 1c                cmp    0x1c(%ebp),%eax */
    /* 4f7:       7e 6e                   jle    567 <libvibeModelGetUpdateFactor+0x19b> */

    /* 4f9:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 4fc:       01 45 f4                add    %eax,-0xc(%ebp) */
    /* 4ff:       eb 73                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 501:       83 7d 18 00             cmpl   $0x0,0x18(%ebp) */
    /* 505:       7e 63                   jle    56a <libvibeModelGetUpdateFactor+0x19e> */
    /* 507:       83 6d f4 01             subl   $0x1,-0xc(%ebp) */
    /* 50b:       eb 67                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 50d:       83 7d 18 00             cmpl   $0x0,0x18(%ebp) */
    /* 511:       7e 04                   jle    517 <libvibeModelGetUpdateFactor+0x14b> */
    /* 513:       83 6d f4 01             subl   $0x1,-0xc(%ebp) */
    /* 517:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp) */
    /* 51b:       7e 50                   jle    56d <libvibeModelGetUpdateFactor+0x1a1> */

    /* 51d:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 520:       29 45 f4                sub    %eax,-0xc(%ebp) */
    /* 523:       eb 4f                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 525:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp) */
    /* 529:       7e 45                   jle    570 <libvibeModelGetUpdateFactor+0x1a4> */

    /* 52b:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 52e:       29 45 f4                sub    %eax,-0xc(%ebp) */
    /* 531:       eb 41                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */

    /* 533:       8b 45 0c                mov    0xc(%ebp),%eax */
    /* 536:       83 e8 01                sub    $0x1,%eax */
    /* 539:       3b 45 18                cmp    0x18(%ebp),%eax */
    /* 53c:       7e 04                   jle    542 <libvibeModelGetUpdateFactor+0x176> */
    /* 53e:       83 45 f4 01             addl   $0x1,-0xc(%ebp) */
    /* 542:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp) */
    /* 546:       7e 2b                   jle    573 <libvibeModelGetUpdateFactor+0x1a7> */

    /* 548:       8b 45 14                mov    0x14(%ebp),%eax */
    /* 54b:       29 45 f4                sub    %eax,-0xc(%ebp) */
    /* 54e:       eb 24                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */

    /* 550:       c7 04 24 80 01 00 00    movl   $0x180,(%esp)    553: R_386_32   .rodata "You should not see this message!!!"*/
    /* 557:       e8 fc ff ff ff          call   558 <libvibeModelGetUpdateFactor+0x18c>  558: R_386_PC32 puts */

    /* 55c:       eb 16                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 55e:       90                      nop */
    /* 55f:       eb 13                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 561:       90                      nop */
    /* 562:       eb 10                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 564:       90                      nop */
    /* 565:       eb 0d                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 567:       90                      nop */
    /* 568:       eb 0a                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 56a:       90                      nop */
    /* 56b:       eb 07                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 56d:       90                      nop */
    /* 56e:       eb 04                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 570:       90                      nop */
    /* 571:       eb 01                   jmp    574 <libvibeModelGetUpdateFactor+0x1a8> */
    /* 573:       90                      nop */

    /* 574:       8b 45 f4                mov    -0xc(%ebp),%eax */
    /* 577:       8b 55 08                mov    0x8(%ebp),%edx */
    /* 57a:       8d 04 02                lea    (%edx,%eax,1),%eax */
    /* 57d:       0f b6 00                movzbl (%eax),%eax */
    eax = ebp[-0xc];
    edx = ebp[0x8];
    eax = &(edx+eax);
    eax = *(uint8_t*)(eax);

    return (uint8_t) eax;
}
{% endhighlight %}

That is some nasty code. There's also a jump table that the compiler 
built from a switch statement in there. Am I insane for trying to 
reverse engineer it? Yeah! Bring it on! That reference to the read-only 
data section at 0x480 to 0x1a4 is a compiled jump table. Its contents 
are:

{% highlight text %}
0000 0489 0000 04a1 0000 04ca 0000 04e4
0000 0501 0000 050d 0000 0525 0000 0533
0000 0611 0000 0629 0000 0652 0000 066c
0000 0689 0000 0695 0000 06ad 0000 06bb
{% endhighlight %}

This is an array of sixteen words representing jump table targets. The 
program calls `circular_rand`, masks the value to three bits wide and 
then directs control flow depending on that lookup target. There are 
sixteen words in the table but we can only use eight of them (each of 
the eight neighbouring pixels). See how the jump targets in the higher 
areas of the table point to areas outside of this function's scope? 
That's because there is a compliment to this function for color pixel 
values that probably uses a similar jump table approach. 

A funny quirk in the jump table is between 0x46f and 0x47a. It checks 
that the result of evaluating the result of a call to `circular_rand`, 
masked to three bits, is greater than seven and if so, jumps to a 
diagnostic message. That evaluation will always evaluate to false. Even 
so, the author has added a statement saying "You should not see this 
message!!!" whenever it might happen to evaluate to true. Thumbs up for 
bulletproofing code against sudden changes to the fundamental laws of 
the physics.

{% highlight c %}
uint8_t getRandomNeighbourPixelValue_8u_C1(uint8_t *pixels, int32_t width, int32_t height, int32_t stride, int32_t x, int32_t y) {
    uint32_t n = (stride * y + x);

    assert(x < width);
    assert(y < height);

    switch(circular_rand() & 0x7) {
        case 0:
	    if ((width - 1) <= x) {
	    } else {
		n++;
	    }
	    break;
	case 1:
	    if ((width - 1) <= x) {
	    } else {
	        n++;
            }
	    if ((height - 1) <= y) {
            } else {
                n += stride;
	    }
            break;
	case 2:
	    if ((height - 1) <= y) {
	    } else {
		n += stride;
	    }
            break;
	case 3:
	    if (x <= 0) {
	    } else {
                n--;
	    }
	    if ((height - 1) <= y) {
	    } else {
                n += stride;
	    }
	    break;
	case 4:
	    if (x <= 0) {
	    } else {
                n--;
	    }
	    break;
	case 5:
	    if (x <= 0) {
	    } else {
                n--;
	    }
	    if (y <= 0) {
	    } else {
		n -= stride;
	    }
            break;
	case 6:
	    if (y <= 0) {
	    } else {
                n -= stride;
	    }
            break;
    case 7:
	    if ((width - 1) <= x) {
	    } else {
		n++;
	    }
	    if (y <= 0) {
	    } else {
		n -= stride;
            }
            break;
	default:
	    puts("You should not see this message!!!");
	    break;
    }

    return *(uint8_t *) &pixels[n];
}
{% endhighlight %}

The color version of this function is very similar. The prologue, 
storage area, arguments and epilogue are all the same. The function 
returns void, but modifies its arguments to output eight bit values. 
Considering that this is the color version, it should be a lot more 
complex.

{% highlight objdump %}
00000582 <lambda02>:
     582:       55                      push   %ebp
     583:       89 e5                   mov    %esp,%ebp
     585:       83 ec 28                sub    $0x28,%esp
     588:       8b 55 18                mov    0x18(%ebp),%edx
     58b:       89 d0                   mov    %edx,%eax
     58d:       01 c0                   add    %eax,%eax
     58f:       8d 14 10                lea    (%eax,%edx,1),%edx
     592:       8b 45 14                mov    0x14(%ebp),%eax
     595:       0f af 45 1c             imul   0x1c(%ebp),%eax
     599:       8d 04 02                lea    (%edx,%eax,1),%eax
     59c:       89 45 f4                mov    %eax,-0xc(%ebp)
     59f:       8b 45 18                mov    0x18(%ebp),%eax
     5a2:       3b 45 0c                cmp    0xc(%ebp),%eax
     5a5:       7c 24                   jl     5cb <libvibeModelGetUpdateFactor+0x1ff>
     5a7:       c7 44 24 0c 60 04 00 00 movl   $0x460,0xc(%esp) 5ab: R_386_32   .rodata
     5af:       c7 44 24 08 ef 00 00 00 movl   $0xef,0x8(%esp)
     5b7:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  5bb: R_386_32   .rodata
     5bf:       c7 04 24 6b 01 00 00    movl   $0x16b,(%esp)    5c2: R_386_32   .rodata
     5c6:       e8 fc ff ff ff          call   5c7 <libvibeModelGetUpdateFactor+0x1fb>  5c7: R_386_PC32 __assert_fail
     5cb:       8b 45 1c                mov    0x1c(%ebp),%eax
     5ce:       3b 45 10                cmp    0x10(%ebp),%eax
     5d1:       7c 24                   jl     5f7 <libvibeModelGetUpdateFactor+0x22b>
     5d3:       c7 44 24 0c 60 04 00 00 movl   $0x460,0xc(%esp) 5d7: R_386_32   .rodata
     5db:       c7 44 24 08 f0 00 00 00 movl   $0xf0,0x8(%esp)
     5e3:       c7 44 24 04 dd 00 00 00 movl   $0xdd,0x4(%esp)  5e7: R_386_32   .rodata
     5eb:       c7 04 24 75 01 00 00    movl   $0x175,(%esp)    5ee: R_386_32   .rodata
     5f2:       e8 fc ff ff ff          call   5f3 <libvibeModelGetUpdateFactor+0x227>  5f3: R_386_PC32 __assert_fail
     5f7:       e8 4d fa ff ff          call   49 <libvibeModelPrintParameters+0x49>
     5fc:       83 e0 07                and    $0x7,%eax
     5ff:       83 f8 07                cmp    $0x7,%eax
     602:       0f 87 d0 00 00 00       ja     6d8 <libvibeModelGetUpdateFactor+0x30c>
     608:       8b 04 85 c4 01 00 00    mov    0x1c4(,%eax,4),%eax      60b: R_386_32   .rodata
     60f:       ff e0                   jmp    *%eax
     611:       8b 45 0c                mov    0xc(%ebp),%eax
     614:       83 e8 01                sub    $0x1,%eax
     617:       3b 45 18                cmp    0x18(%ebp),%eax
     61a:       0f 8e c6 00 00 00       jle    6e6 <libvibeModelGetUpdateFactor+0x31a>
     620:       83 45 f4 03             addl   $0x3,-0xc(%ebp)
     624:       e9 d3 00 00 00          jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     629:       8b 45 0c                mov    0xc(%ebp),%eax
     62c:       83 e8 01                sub    $0x1,%eax
     62f:       3b 45 18                cmp    0x18(%ebp),%eax
     632:       7e 04                   jle    638 <libvibeModelGetUpdateFactor+0x26c>
     634:       83 45 f4 03             addl   $0x3,-0xc(%ebp)
     638:       8b 45 10                mov    0x10(%ebp),%eax
     63b:       83 e8 01                sub    $0x1,%eax
     63e:       3b 45 1c                cmp    0x1c(%ebp),%eax
     641:       0f 8e a2 00 00 00       jle    6e9 <libvibeModelGetUpdateFactor+0x31d>
     647:       8b 45 14                mov    0x14(%ebp),%eax
     64a:       01 45 f4                add    %eax,-0xc(%ebp)
     64d:       e9 aa 00 00 00          jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     652:       8b 45 10                mov    0x10(%ebp),%eax
     655:       83 e8 01                sub    $0x1,%eax
     658:       3b 45 1c                cmp    0x1c(%ebp),%eax
     65b:       0f 8e 8b 00 00 00       jle    6ec <libvibeModelGetUpdateFactor+0x320>
     661:       8b 45 14                mov    0x14(%ebp),%eax
     664:       01 45 f4                add    %eax,-0xc(%ebp)
     667:       e9 90 00 00 00          jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     66c:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     670:       7e 04                   jle    676 <libvibeModelGetUpdateFactor+0x2aa>
     672:       83 6d f4 03             subl   $0x3,-0xc(%ebp)
     676:       8b 45 10                mov    0x10(%ebp),%eax
     679:       83 e8 01                sub    $0x1,%eax
     67c:       3b 45 1c                cmp    0x1c(%ebp),%eax
     67f:       7e 6e                   jle    6ef <libvibeModelGetUpdateFactor+0x323>
     681:       8b 45 14                mov    0x14(%ebp),%eax
     684:       01 45 f4                add    %eax,-0xc(%ebp)
     687:       eb 73                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     689:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     68d:       7e 63                   jle    6f2 <libvibeModelGetUpdateFactor+0x326>
     68f:       83 6d f4 03             subl   $0x3,-0xc(%ebp)
     693:       eb 67                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     695:       83 7d 18 00             cmpl   $0x0,0x18(%ebp)
     699:       7e 04                   jle    69f <libvibeModelGetUpdateFactor+0x2d3>
     69b:       83 6d f4 03             subl   $0x3,-0xc(%ebp)
     69f:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     6a3:       7e 50                   jle    6f5 <libvibeModelGetUpdateFactor+0x329>
     6a5:       8b 45 14                mov    0x14(%ebp),%eax
     6a8:       29 45 f4                sub    %eax,-0xc(%ebp)
     6ab:       eb 4f                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6ad:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     6b1:       7e 45                   jle    6f8 <libvibeModelGetUpdateFactor+0x32c>
     6b3:       8b 45 14                mov    0x14(%ebp),%eax
     6b6:       29 45 f4                sub    %eax,-0xc(%ebp)
     6b9:       eb 41                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6bb:       8b 45 0c                mov    0xc(%ebp),%eax
     6be:       83 e8 01                sub    $0x1,%eax
     6c1:       3b 45 18                cmp    0x18(%ebp),%eax
     6c4:       7e 04                   jle    6ca <libvibeModelGetUpdateFactor+0x2fe>
     6c6:       83 45 f4 03             addl   $0x3,-0xc(%ebp)
     6ca:       83 7d 1c 00             cmpl   $0x0,0x1c(%ebp)
     6ce:       7e 2b                   jle    6fb <libvibeModelGetUpdateFactor+0x32f>
     6d0:       8b 45 14                mov    0x14(%ebp),%eax
     6d3:       29 45 f4                sub    %eax,-0xc(%ebp)
     6d6:       eb 24                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6d8:       c7 04 24 80 01 00 00    movl   $0x180,(%esp)    6db: R_386_32   .rodata
     6df:       e8 fc ff ff ff          call   6e0 <libvibeModelGetUpdateFactor+0x314>  6e0: R_386_PC32 puts
     6e4:       eb 16                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6e6:       90                      nop
     6e7:       eb 13                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6e9:       90                      nop
     6ea:       eb 10                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6ec:       90                      nop
     6ed:       eb 0d                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6ef:       90                      nop
     6f0:       eb 0a                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6f2:       90                      nop
     6f3:       eb 07                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6f5:       90                      nop
     6f6:       eb 04                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6f8:       90                      nop
     6f9:       eb 01                   jmp    6fc <libvibeModelGetUpdateFactor+0x330>
     6fb:       90                      nop
     6fc:       8b 45 f4                mov    -0xc(%ebp),%eax
     6ff:       8b 55 08                mov    0x8(%ebp),%edx
     702:       8d 04 02                lea    (%edx,%eax,1),%eax
     705:       0f b6 10                movzbl (%eax),%edx
     708:       8b 45 20                mov    0x20(%ebp),%eax
     70b:       88 10                   mov    %dl,(%eax)
     70d:       8b 45 f4                mov    -0xc(%ebp),%eax
     710:       83 c0 01                add    $0x1,%eax
     713:       03 45 08                add    0x8(%ebp),%eax
     716:       0f b6 10                movzbl (%eax),%edx
     719:       8b 45 24                mov    0x24(%ebp),%eax
     71c:       88 10                   mov    %dl,(%eax)
     71e:       8b 45 f4                mov    -0xc(%ebp),%eax
     721:       83 c0 02                add    $0x2,%eax
     724:       03 45 08                add    0x8(%ebp),%eax
     727:       0f b6 10                movzbl (%eax),%edx
     72a:       8b 45 28                mov    0x28(%ebp),%eax
     72d:       88 10                   mov    %dl,(%eax)
     72f:       c9                      leave
     730:       c3                      ret
{% endhighlight %}

And the reconstructed code:

{% highlight c %}
void getRandomNeighbourPixelValue_8u_C3(uint8_t *pixels, int32_t width, 
int32_t height, int32_t stride, int32_t x, int32_t y, uint8_t *r, 
uint8_t *g, uint8_t *b) {
    uint32_t n = ((3 * x) + (stride * y));

    assert(x < width);
    assert(y < height);

    switch(circular_rand() & 0x7) {
        case 0:
	    if ((width - 1) <= x) {

	    } else {
	        n += 3;
	    }
	    break;
	case 1:
	    if ((width - 1) <= x) {

	    } else {
		n += 3;
	    }
	    if ((height - 1) <= y) {

	    } else {
		n += stride;
	    }
	    break;
	case 2:
	    if ((height - 1) <= y) {

	    } else {
		n += stride;
	    }
	    break;
	case 3:
	    if (x <= 0) {

	    } else {
		n -= 0x3;
	    }
	    if ((height - 1) <= y) {

	    } else {
		n += stride;
	    }
	    break;
	case 4:
	    if (x <= 0) {

	    } else {
		n -= 3;
	    }
	    break;
	case 5:
	    if (x <= 0) {

	    } else {
		n -= 3;
	    }
	    if (y <= 0) {

	    } else {
		n -= stride;
	    }
	    break;
	case 6:
	    if (y <= 0) {

	    } else {
		n -= stride;
	    }
	    break;
	case 7:
	    if ((width - 1) <= x) {

	    } else {
	        n += 3;
	    }
	    if (y <= 0) {

	    } else {
		n -= stride;
	    }
	    break;
	default:
	    puts("You should not see this message!!!");
	    break;

    }

    *r = pixels[n];
    *g = pixels[n+1];
    *b = pixels[n+2];

    return;
{% endhighlight %}

Well that wraps up those two functions. There's a lot more to go so 
I'll save that for another post later on in part IV.

  [1]: {{ site.baseurl }}/post/understanding-vibe-part-ii

