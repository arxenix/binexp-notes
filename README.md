# binexp-notes
notes to remember about binary exploitation, the stack and heap


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
push eip + 2 ; return address is current address + size of two instructions
jmp _MyFunction3
```

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
