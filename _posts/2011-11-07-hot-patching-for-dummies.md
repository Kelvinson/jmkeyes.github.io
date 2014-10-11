---
layout: post
title: 'Hot Patching For Dummies'
date: 2011-11-07 18:52:41.000000000 -08:00
status: published
type: post
---
This morning when I checked Reddit's programming board, I noticed an
article that stood out because it had linked to [one of my earlier
posts][1] about hot patching. [As one Reddit user brought up][2] in
the comments (referring to the linked article):

> Unfortunately, this article doesn't discuss how to do the hard
> part -- possibly calling back the original function after you
> modified it. This involves writing a trampoline with a modified
> version of the function preamble that was overwritten, fixed up
> so PC-relative instructions are still correct in the (relocated)
> trampoline. This is a popular method on jailbroken iOS devices
> that MobileSubstrate performs.

I'd wanted to write an entry about that exact process and the problems
that appear when trying to complete it, so I thought tonight I would do
exactly that. The same platform caveats of the previous article still
apply. I should also add that I'll be focusing on only the ELF format
of binary, though there are others in use by different operating 
systems.

To make a long story short, the reason this part is relatively complex
(and part of the reason I left it out of the previous article) is that
it involves reliably decoding the x86 and/or x86-64 instruction set.

To put it lightly, decoding the instruction set is an extremely complex
task itself (and rife with its own problems) so I'd normally rely on a
disassembler library like [udis86][3] or [diStorm64][4]. Each has its
own API and its own way of doing things, so I'll leave using them as an
exercise for the reader.

The first case is the simplest: we're about to place an unconditional
jump at the prologue to a function, just as we were going to before. We
need to calculate the total number of bytes that our replacement opcodes
will use.

For a direct unconditional jump, it's the size of the direct jump 
opcode (direct jumps are single-byte opcodes) plus it's displacement 
address (four bytes, to a maximum offset of 2GB in either direction.) 
For the indirect unconditional jump it's the size of the encoded 
`movabs` opcode, it's target register, the 64 bit offset we want 
to jump to, and the encoded indirect-unconditional jump opcode, of 
which all add up to 13 bytes in total.

Now we need to start up our disassembler at the function entry point 
(or wherever we are placing the detour patch). Point it at the right 
offset and begin disassembling whole instructions until we've reached a 
point *equal to or greater than* the total number of bytes we need to
overwrite. We want to find the least number of whole instructions that
we can overwrite so that we don't leave half-written opcodes dangling.

Next, allocate a chunk of memory in the process equal to the total size 
of the disassembled instructions and copy the source's bytes directly 
into it. This is our backup location, storing an exact copy of the bytes
we are going to overwrite. Now we can overwrite the target as usual.
Remember to write NOP (0x90) instructions immediately after the detour,
so that we don't leave dangling opcodes behind. Now our detour is in
place. If we want to restore the default functionality, we copy the
source bytes we backed up over-top of the detour.

Now the hard part: how to transfer control flow back to the original
function, without overwriting our detour. This is where things get very
complex, very fast. The path taken forks at the architecture we're using.
I'll start with the x86 and move onto it's newer cousin in a bit.

Short and sweet; we have to disassemble the previously backed-up bytes, 
one at a time, so that all hard-coded addresses and offsets (including 
implicit ones) to adjust for their new location. Since we don't want to 
directly overwrite our backup instructions, we have to make an extra 
copy of them somewhere else, piece by piece. (Hint: state machines are 
great for this task.) Because the x86 doesn't have a way to get the 
current value of the instruction pointer without modifying the stack 
(not one that I know of anyway), we have to make some educated guesses 
about what is going to happen "next" from the point-of-view of the 
executing code, similarly to how we computed the offset-displacement 
for the direct jump. Any instructions that attempt to get access to 
addresses or offsets that are outside of the small block of 
instructions we backed-up need a rewrite.

This includes any control transfer instructions to other areas of code, 
any calls to other functions, the global offset table (GOT) or the 
procedure linkage table (PLT) for the ELF executable, any calls to 
`__i686.get_pc_thunk.bx` (which loads the `%ebx` register with the 
current instruction pointer, post-instruction) for code compiled on the 
x86 as position-independent (many shared libraries and executables 
are), and the occasional changes to the stack that would use or 
manipulate hard-coded offsets or addresses. This list also changes 
depending on the architecture that you're hot-patching for.

As for the x86-64, many of the ABI changes alter the instructions you 
need to rewrite. Some are simpler, and some are not. Because of the 
expanded address space, you need to account for the offsets that the 
backed-up instructions reside at, and either deliberately place the 
executable ones below the 2GB maximum offset, or use 
indirect-unconditional jumps to move control flow from one to the 
other. The former is safer for many things because the latter has to 
alter registers in most cases. Beware the issues at the boundary of the 
2GB offset. It can trip up a lot of automated rewriting on x86-64. 
Additionally, unlike the horrid mess that is `__i686.get_pc_thunk.bx`
on x86, the x86-64 uses "%rip-relative" addressing which means 
that instead of using stack pushing and popping tricks to get the 
instruction pointer, you can just use the instruction pointer directly.

These lists include only the basics. Depending on the code you're 
reverse engineering, it's possible to have many other combinations of 
instructions that would need rewriting. *Any rewritten instructions must
fulfill their contracts*. This means that, to the executing process, the
control transfers and voodoo magic we're doing are entirely transparent
and that it would function as normal if the detours didn't exist. If it
isn't, you'll get some weird bugs that will take ages to track down.

At the end of the instruction rewriting, place the proper jump to jump 
back to the address of the NOP instructions or the instructions 
following them. When you're ready to give it a shot, import the address 
of the beginning of the rewritten instructions into your C program as a 
function pointer of a type matching the prototype of the function you 
want to detour. Also make sure that you're using a similar compiler and 
compiler options. If you can't get either of those, either write some 
glue logic to get the same result with a few more instructions, or try 
harder. Creating more code won't make up for the lack of an attempt.

If you're wondering why this isn't done more often, it's usually 
because there are a lot of x86 disassemblers and not many x86 
assemblers that have a usable API. It's likely you'll have to tack that 
onto a new library. If I get some time in the future, I may write a 
library to do that, but for the mean time the best opcode-research tool 
is the debugger and disassembler.

As you can see, once you understand the code from the point-of-view of 
the processor, rewriting the code while "hot" is fairly easy. If 
there's interest, I'll write some code here and there in this post to 
make things clearer.

  [1]: {{ site.baseurl }}/post/hot-patching-unconditional-jumps-x86-x86-64
  [2]: http://www.reddit.com/r/programming/comments/m3d7c/function_hooking_and_windows_dll_injection/c2xtbv8
  [3]: http://udis86.sourceforge.net/
  [4]: http://www.ragestorm.net/distorm

