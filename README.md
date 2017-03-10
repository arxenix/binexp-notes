# binexp-notes
notes to remember about binary exploitation, the stack and heap. This all assumes 32-bit architecture.


## Program Layout
![Process Memory](http://i.imgur.com/bcU5U0Y.png)

## Registers
`esp` - **Stack Pointer** - register that always points to top of the stack. Changed whenever stuff is pushed/popped from stack.

`ebp` - **Base Pointer** - register that points to the start of the current stack frame. More convenient to reference function parameters & local variables than using esp directly.

`eip` - **Instruction Pointer** - register that contains address of next function to be executed.


## Stack
`esp` - register that always points to top of the stack.

Stack grows *downward*. When we push onto stack, 4 is subtracted from esp, and when we pop from stack, 4 is added to esp.

Say we wanted to create an array of byte values on the stack, 100 items long. We want to store the pointer to the base of this array in *edi*.:
```asm
sub esp, 100  ; num of bytes in our array
mov edi, esp  ; copy address of 100 bytes area to edi
```
Then to destroy: `add esp, 100`

Data on stack accessed with positive offset. i.e. `esp + 4`. Negative offset should not be used because it is reading past the top of the stack (unreliable)

## Stack Frame
Convenient way for functions to be represented/handled in assembly.

### Example 1:
```C
void MyFunction3(int x, int y, int z)
{
  int a, int b, int c;
  ...
  return;
}
```

```asm
_MyFunction3:
  ; start stack frame
  push ebp  ; save the (old) value of ebp by pushing it into the stack
  mov ebp, esp  ; ebp now points to the top of the stack
  sub esp, 12  ; space allocated on the stack for the local variables a,b,c. sizeof(a)+sizeof(b)+sizeof(c)
  ; x,y,z can now be referenced with the base pointer:    x = [ebp + 8], y = [ebp + 12], z = [ebp + 16]
  ; a,b,c can be referenced with esp or ebp:    a = [ebp - 4] = [esp + 8], b = [ebp - 8] = [esp + 4], c = [ebp - 12] = [esp]
  
  ; end stack frame
  mov esp, ebp    ; point esp back to old value (remove space for local variables)
  pop ebp  ; restore old value of ebp, which was pushed onto stack in the start sequence
  ret 12 ; return to caller, and remove 12 bytes of parameters from stack (a,b,c)
```

Would be called with:
```asm
push 2 ; c
push 5 ; b
push 10 ; a
call _MyFunction3
```

**Important to remember**: 
the `call` intruction is equivalent to

```
push eip + 2 ; return address is current address + size of two instructions B
jmp _MyFunction3
```

It pushes `eip` onto the stack! This is so the stack frame knows what address to return to!

### Example 2:
```asm
_Question1:
  push ebp
  mov ebp, esp
  sub esp, 4
  mov eax, [ebp + 8]
  mov ecx, 2
  mul ecx
  mov [esp + 0], eax
  mov eax, [ebp + 12]
  mov edx, [esp + 0]
  add eax, edx
  mov esp, ebp
  pop ebp
  ret ;optionally ret 8 to clear pushed function parameters. otherwise, needs to be handled by caller
```

```C
int Question1(int x, int y)
 {
    int z;
    z = x * 2;
    return y + z;
 }
 ```


## Simple value buffer overflow
```c
void vuln(char *input){
    char buf[16];
    int secret = 0;
    strcpy(buf, input);

    if (secret == 0xc0deface){
        give_shell();
    }else{
        printf("The secret is %x\n", secret);
    }
}
```

`strcpy()` doesn't do length-checking or anything. If we have control over the input, we can overflow the buffer, and write arbitrary data to the stack. What does the stack look like in this program?

```
//top of stack (low mem address)
buf                               [16 bytes]
secret                            [4 bytes int (probably)]
saved EBP (saved stack frame ptr) [4 bytes]
saved EIP (return address)        [4 bytes]
input*                            [4 bytes]

...
// bottom of stack (high mem address)
```


We can overwrite secret, because strcpy() will keep reading from our input, go past the 16 bytes of the buffer, and start overwriting secret.

Setting `input` to `AAAAAAAAAAAAAAAA\xce\xfa\xde\xc0` would fill **buff** to `AAAAAAAAAAAAAAAA` and overwrite **secret** to `0xc0deface`. (keep track of endianness!)

## Basic ROP (return-oriented programming)

A simple buffer overflow let us overwrite a value on the stack, but... we can go further. What's preventing us from overwriting EIP?

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* This never gets called! */
void give_shell(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

void vuln(char *input){
    char buf[16];
    strcpy(buf, input);
}

int main(int argc, char **argv){
    if (argc > 1)
        vuln(argv[1]);
    return 0;
}
```

In this case, our `vuln` function is still vulnerable to a buffer overflow attack. At first glance, it doesn't seem like there's any way to execute the `give_shell()` function though.

Let's try and predict the stack again...

```
//top of stack (low mem address)
buf                               [16 bytes]
saved EBP (saved stack frame ptr) [4 bytes]
saved EIP (return address)        [4 bytes]
input*                            [4 bytes]

...
// bottom of stack (high mem address)
```

What happens when we overwrite buf, saved EBP, and then overwrite EIP. We can set EIP to whatever we want. This would make the program return to whatever address we want when it reaches EIP!

The address of the `give_shell()` function can be found with `print &give_shell` in gdb. For ex, let's say it is `0x80484ad`.

Ideally, we would set `input` to `'A'*20+'\xad\x84\x04\x08'`... however, it won't work. Why? In reality, there is probably stuff between buf and the saved EBP. Certain compilers can add stuff like padding in between. Our stack actually looks like this:

```
//top of stack (low mem address)
buf                               [16 bytes]
some crap                         [??? bytes]
saved EBP (saved stack frame ptr) [4 bytes]
saved EIP (return address)        [4 bytes]
input*                            [4 bytes]

...
// bottom of stack (high mem address)
```



So, how do we find the correct offset now?

**Noob way:** Trial and error


**Proper way:** GDB! We can add a breakpoint on `strcpy`. Then get the first arg to strcpy (address of buf) with `x/wx $esp`. This prints out 1 word from the top of the stack. We can then compare to ebp, which will point to our saved ebp: `p/x $ebp`.


Subtracting the 2 will get the distance from `buf` to `ebp`! In this case, it turns out to be 24. Add 4 more to overwrite ebp, then the next 4 bytes will overwrite esp. So, our exploit becomes `'A'*28+'\xad\x84\x04\x08'`.

## SHELLCODE!!!

We used ROP to overwrite EIP, and make the program flow jump to an address, which was another function.


What if we define our own function in the buffer, and overwrite EIP to make the program jump to our own function? That's exactly how shellcode works! Using this technique, we can execute whatever code we want.

//TODO example

## ret2libc (bypass NX)
If NX is on, the stack cannot be executed, so shellcode doesn't work. Instead, we can only execute data in areas such as the program .text section.

Libc is very big, if the binary has a linked libc, we can set EIP to point to a function within libc, such as `system()`! 

How do we control its arguments? Remember how stack frames work:
```
//top of stack
function local variables
ebp
eip
function args
// bottom of stack
```

Before we call a function, it is the caller's responsibility to push function arguments onto the stack, and save EIP. (it is the responsibility of the function to save EBP and push local variables)


`system()` takes in 1 argument, a `char *` for the string command to execute. We can locate the address of the string `/bin/sh` in the binary.

Now, exploit string can look like this: `buffer_overflow + system_address + "AAAA" + bin_sh_address`.

It will jump to the system() call, and the function args will be on the stack correctly! It will segfault after, because it will return to 0x41414141 address. To get shell without it exiting, run a hanging command like `(python pwn.py; cat) | ./binary`

## ROP Gadgets and ROP Chainz (bypass NX)
Same idea as before -- use bits of code from the program's `.text` section and jump to them instead. EIP Doesn't have to point to the start of a function, it can point anywhere. We can find "gadgets" which contain useful bits of assembly instruction we might want to execute. 

Must end with the `ret` instruction, which pops the top value off the stack, and jumps to it (end of a function). We can therefore chain multiple ROP Gadgets together for arbitrary assembly instruction execution!

We jump to the first ROP gadget, which runs some assembly code and returns. Then execution jumps to our next ROP gadget because it thinks it is EIP!

Ex: `buffer_overflow + gadget_1_addr + gadget_2_addr + gadget_3_addr`



Use https://github.com/JonathanSalwan/ROPgadget for finding them.

## GOT

## ret2plt

## Stack Canaries/Cookies

## Heap Overflow
