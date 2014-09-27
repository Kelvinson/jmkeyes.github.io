---
layout: post
title: Delimited Continuations and the Call Stack
date: 2011-05-12 14:26:56.000000000 -07:00
status: published
type: post
---

When a program invokes a subroutine it needs to know where to go when 
it's finished.

For example, when a function call happens on the x86, the caller pushes 
a number of arguments onto the stack and then calls the subroutine, 
which sets up the stack as it needs. When the caller executes the 
'call' instruction, the processor pushes a *return address* onto the 
stack and then transfers control to the subroutine. When the subroutine 
exits, it transfers control back to that return address and the program 
invokes the next set of instructions. If you analyze the stack in a 
debugger you can easily see each of these *stack frames* (or 
*activation records* if you're a pedantic academic type). All of these 
records, composed one inside another, form the entire call stack.

Each of those records represents not only an *execution history* of 
where the program has been before but also the *execution future* or 
where the program intends to finish. At any point in a computation the 
snapshot state both of the call stack and the processor's registers 
represent the entire history of the program from start to finish. See 
where I'm going with this?

A *continuation* represents the state of a program at a given point in 
its execution. That state is a small package then saved somewhere else 
and invoked like a subroutine later on. When the program invokes that 
continuation the continuation's contents replace the current execution 
context effectively rewriting the program's history. When the program 
invokes that continuation, it copies the entire saved state of the 
stack and the registers captured, which replace the entire program 
history reverting it back to the state it originally had when the 
continuation was first captured. Therefore the original function can 
never return to its caller because it's original state of execution no 
longer exists. This is obvious in the context of continuation-passing 
style, because in that style no subroutine can ever return to its 
caller because it passes its own state as an argument to the explicitly 
provided continuation argument. In assembly, a subroutine call then 
boils down to a simple control-transfer function like *jump*. It's a 
vast simplification of the typical execution scheme that requires the 
programmer to explicitly direct execution flow at a higher level 
instead of relying on the implicit setup of a function call.

A *delimited* continuation is a special case of a *full* continuation. 
Instead of replacing the entire call stack, it only replaces a specific 
number of stack frames and only rewrites the execution history of the 
program up to a specific point. This is what makes 
call-with-current-continuation so powerful. It gives the programmer 
access to freely rewrite the execution history of a program to do 
whatever he or she wants at any given point. This is also why 
exceptions are a special case of delimited continuations. When thrown, 
an exception reverts the stack by a number of frames until a calling 
subroutine in the program handles the exception. It replaces the 
execution context so that the program handles any errors immediately 
instead of generating a return code that requires the subroutine to 
finish.

How does this relate to the call stack? It's simple enough to answer 
that the nested stack frames are an implicit form of post-hoc 
continuation passing. The compiler translates the program code into a 
stripped down transfer of state across subroutine calls and the return 
address (and by extension the contents of the stack at that point) 
represents the current continuation of that stack frame to the rest of 
the program.

