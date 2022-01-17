## Return Oriented Programming-ROP

Return Oriented Programming (or ROP) is the idea of chaining together small snippets of assembly with stack control to cause the program to do more complex things.

As we saw in buffer overflows, having stack control can be very powerful since it allows us to overwrite saved instruction pointers, giving us control over what the program does next. Most programs don't have a convenient give_shell function however, so we need to find a way to manually invoke system or another exec function to get us our shell.

## 32 bit

Imagine we have a program similar to the following:

```C
#include <stdio.h>
#include <stdlib.h>

char name[32];

int main() {
    printf("What's your name? ");
    read(0, name, 32);

    printf("Hi %s\n", name);

    printf("The time is currently ");
    system("/bin/date");

    char echo[100];
    printf("What do you want me to echo back? ");
    read(0, echo, 1000);
    puts(echo);

    return 0;
}
```
We obviously have a stack buffer overflow on the echo variable which can give us EIP control when main returns. But we don't have a give_shell function! So what can we do?

We can call system with an argument we control! Since arguments are passed in on the stack in 32-bit Linux programs (see calling conventions), if we have stack control, we have argument control.

When main returns, we want our stack to look like something had normally called system. Recall what is on the stack after a function has been called:

```cmd
        ...                                 // More arguments
        0xffff0008: 0x00000002              // Argument 2
        0xffff0004: 0x00000001              // Argument 1
ESP ->  0xffff0000: 0x080484d0              // Return address
```

So `main's` stack frame needs to look like this:

```cmd
        0xffff0008: 0xdeadbeef              // system argument 1
        0xffff0004: 0xdeadbeef              // return address for system
ESP ->  0xffff0000: 0x08048450              // return address for main (system's PLT entry)
```

Then when main returns, it will jump into system's PLT entry and the stack will appear just like system had been called normally for the first time.

Note: we don't care about the return address system will return to because we will have already gotten our shell by then!

## Arguments

This is a good start, but we need to pass an argument to system for anything to happen. As mentioned in the page on ASLR, the stack and dynamic libraries "move around" each time a program is run, which means we can't easily use data on the stack or a string in libc for our argument. In this case however, we have a very convenient name global which will be at a known location in the binary (in the BSS segment).

## Putting it together

Our exploit will need to do the following:

1. Enter "sh" or another command to run as name
2. Fill the stack with

-----------------------------------------------------

1. Garbage up to the saved EIP
2. The address of system's PLT entry
3. A fake return address for system to jump to when it's done
4. The address of the name global to act as the first argument to system


## 64 bit

In 64-bit binaries we have to work a bit harder to pass arguments to functions. The basic idea of overwriting the saved RIP is the same, but as discussed in calling conventions, arguments are passed in registers in 64-bit programs. In the case of running system, this means we will need to find a way to control the RDI register.

To do this, we'll use small snippets of assembly in the binary, called "gadgets." These gadgets usually pop one or more registers off of the stack, and then call ret, which allows us to chain them together by making a large fake call stack.

For example, if we needed control of both RDI and RSI, we might find two gadgets in our program that look like this (using a tool like rp++ or ROPgadget):

```CMD
0x400c01: pop rdi; ret
0x400c03: pop rsi; pop r15; ret
```

We can setup a fake call stack with these gadets to sequentially execute them, poping values we control into registers, and then end with a jump to system.

## Example

```cmd
        0xffff0028: 0x400d00            // where we want the rsi gadget's ret to jump to now that rdi and rsi are controlled
        0xffff0020: 0x1337beef          // value we want in r15 (probably garbage)
        0xffff0018: 0x1337beef          // value we want in rsi
        0xffff0010: 0x400c03            // address that the rdi gadget's ret will return to - the pop rsi gadget
        0xffff0008: 0xdeadbeef          // value to be popped into rdi
RSP ->  0xffff0000: 0x400c01            // address of rdi gadget
```

Stepping through this one instruction at a time, main returns, jumping to our pop rdi gadget:

```cmd
RIP = 0x400c01 (pop rdi)
RDI = UNKNOWN
RSI = UNKNOWN

        0xffff0028: 0x400d00            // where we want the rsi gadget's ret to jump to now that rdi and rsi are controlled
        0xffff0020: 0x1337beef          // value we want in r15 (probably garbage)
        0xffff0018: 0x1337beef          // value we want in rsi
        0xffff0010: 0x400c03            // address that the rdi gadget's ret will return to - the pop rsi gadget
RSP ->  0xffff0008: 0xdeadbeef          // value to be popped into rdi
```

pop rdi is then executed, popping the top of the stack into RDI:

```cmd
RIP = 0x400c02 (ret)
RDI = 0xdeadbeef
RSI = UNKNOWN

        0xffff0028: 0x400d00            // where we want the rsi gadget's ret to jump to now that rdi and rsi are controlled
        0xffff0020: 0x1337beef          // value we want in r15 (probably garbage)
        0xffff0018: 0x1337beef          // value we want in rsi
RSP ->  0xffff0010: 0x400c03            // address that the rdi gadget's ret will return to - the pop rsi gadget
```

The RDI gadget then rets into our RSI gadget:

```cmd
RIP = 0x400c03 (pop rsi)
RDI = 0xdeadbeef
RSI = UNKNOWN

        0xffff0028: 0x400d00            // where we want the rsi gadget's ret to jump to now that rdi and rsi are controlled
        0xffff0020: 0x1337beef          // value we want in r15 (probably garbage)
RSP ->  0xffff0018: 0x1337beef          // value we want in rsi
```

RSI and R15 are popped:

```cmd
cRIP = 0x400c05 (ret)
RDI = 0xdeadbeef
RSI = 0x1337beef

RSP ->  0xffff0028: 0x400d00            // where we want the rsi gadget's ret to jump to now that rdi and rsi are controlled
```

And finally, the RSI gadget rets, jumping to whatever function we want, but now with RDI and RSI set to values we control.

