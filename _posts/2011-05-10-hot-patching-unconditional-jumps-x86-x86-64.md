---
layout: post
title: 'Hot Patching: Unconditional Jumps on x86 and x86-64'
date: 2011-05-10 20:51:18.000000000 -07:00
status: published
type: post
---

*Hot patching* is the process of rewriting a program while it is 
actively running. It's a technique that self-modifying code often uses 
to redirect control flow and it's a great way to debug or reverse 
engineer software. It sounds like a complex task but in practice it's 
actually fairly simple to pull off.

The core of every program is the executable code and the data that 
accompanies it during it's execution. The executable code is a series 
of instructions that the processor interprets in sequence, similar to 
an actor reading a script or a teacher reading off directions to a 
class, which direct the processor to do a specific task. Redirecting 
execution flow means rewriting the instructions at a specific place to 
do something other than what it was originally coded to do. In the 
interest of brevity, I'll only be focusing on two processors here -- 
x86 and it's newer cousin x86-64 -- and the Linux/BSD C platform.

So let's start by building something to take apart! Here's a simple 
program that invokes a single function `proc1` and then terminates. 
I've also included the relevant assembler code for reference.

{% highlight c %}
#include <stdio.h>

static void proc2(int arg1) {
    printf("In procedure 2! arg:%d\n", arg1);
}

static void proc1(int arg1) {
    printf("In procedure 1! arg:%d\n", arg1);
}

int main(void) {
    proc1(42);
    return 0;
}
{% endhighlight %}

The plan here is to redirect every call to `proc1` to `proc2`. We could 
go though the program and rewrite all the calls that use the address of 
`proc1` to the other function, but that would only cover the direct 
cases and miss the indirect ones. Here's a better idea: rewrite the 
beginning of `proc1` to immediately transfer to `proc2` using the 
processor's "jump" instruction which transfers execution flow to 
another place in the program.

The format of the jump instruction is simply the opcode for the jump 
instruction and the target address. The trick with the target address 
is that it's not an absolute address: it's the offset between the 
address specified in the processor's instruction pointer when it 
executed the jump instruction and the desired target of the jump. The 
delta is an unsigned 32 bit integer so we're restricted to jumps with a 
displacement equal to or less than 2GB in either direction. That's more 
than enough on the x86.

Although we can't directly get at the instruction pointer on the x86 we 
can guess it's value by adding the source's address and the number 
five: a jump instruction is a one byte opcode and four bytes for the 
offset. Our delta is then computed using the equation 
`(destination-address - (source-address + sizeof(jump-instruction)))`.

{% highlight c %}
void write_redirection_medium(unsigned char *source, unsigned char *destination) {
    unsigned long delta = (dest - (source + 5));

    /* 0xe9 is the unconditional jump opcode. */
    source[0] = 0xe9; 

    /* Write each of the octets of the delta. */
    source[1] = (delta <<  0) & 0xff;
    source[2] = (delta <<  8) & 0xff;
    source[3] = (delta << 16) & 0xff;
    source[4] = (delta << 24) & 0xff;

    return;
}
{% endhighlight %}

Each process has memory protection on the executable code instructions 
to prevent people from rewriting them like this. In order to edit the 
process memory we have to disable the write protection. SVr4 and 
POSIX.1-2001 both name a function called `mprotect` that a process can 
use to change the attributes of its memory pages. We can use this 
function to disable the memory protections. Marking the pages writeable 
removes the write protection. (This is why patch sets like GRSecurity 
are a Good Thing™)

{% highlight c %}
#include <sys/mman.h>
#include <sys/user.h>

void alter_page_attributes(const unsigned char *address, unsigned long length, int mask) {
    /* mprotect() needs an address that is page-aligned. */
    unsigned void *addr = (void *) ((unsigned long) address & PAGE_MASK);

    if (mprotect(addr, len, mask) == -1) {
       perror("mprotect");
    }

     return;
}
{% endhighlight %}

Remove the write protections, write the changes, and restore the write 
protection. Now when the program calls `proc1(42)`, the processor will 
read the jump instruction and go directly to `proc2` instead. If you 
want to call the original function, just back up the instructions 
before overwriting them and append another jump instruction at the end 
of that block back to the function that you overwrote. When you want to 
call the original function again, just pass the address of the backed 
up block as a function pointer.

In some environments -- like x86-64 -- the operating system stores code 
in different areas of the address space that are often farther than 2GB 
apart. The unconditional jump in the previous example is inadequate 
because it can only use a 32 bit immediate value. We can still write a 
redirecting jump; we just need to do it a new way.

This time we need to use an indirect jump. Indirect jumps work by 
jumping to the absolute address in a register or a memory location. 
This is great, sure, but there is still a problem. How can we change 
the contents of a register without destroying the registers in use, 
changing the stack, or writing over too many instructions? It's a tough 
question and the answer depends on the context at that point in 
execution. Looking at the [AMD64 ABI Draft][amd64-abi-draft] we can see
that the `%r11` register is a temporary register that is not preserved
across a function call. Perfect. This time we write `movabs $addr, %r11`
over the beginning of a function and use an indirect jump `jmp *%r11`
to transfer execution.

{% highlight c %}
void write_redirection_large(unsigned char *source, unsigned char *destination) {
    /* There is no sanity in casting pointers to integers. There is only Zuul. */
    unsigned long long address = (unsigned long long) destination;

    /* 0x49 is the 'movabs' opcode. */
    source[0] = 0x49;
    /* 0xbb is the %r11 register. */
    source[1] = 0xbb; 

    /* Write each of the octets of the destination address. */
    source[2] =  (address <<  0) & 0xff;
    source[3] =  (address <<  8) & 0xff;
    source[4] =  (address << 16) & 0xff;
    source[5] =  (address << 24) & 0xff;
    source[6] =  (address << 32) & 0xff;
    source[7] =  (address << 40) & 0xff;
    source[8] =  (address << 48) & 0xff;
    source[9] =  (address << 56) & 0xff;

    /* 0x41 and 0xff are the encoded unconditional jump instruction opcode. */
    source[10] = 0x41;
    source[11] = 0xff;
    /* 0xe3 is the %r11 register. */
    source[12] = 0xe3;

    return;
}
{% endhighlight %}

It's easy to redirect execution in a process by using this method to 
jump to a loaded shared library. It's also generally a good idea to 
lock the process that you're operating on; it could accidentally read 
half-written instructions if you don't. If you want to call the 
original function, do the same thing as you would for the x86.

[amd64-abi-draft]: http://www.x86-64.org/documentation/abi.pdf

