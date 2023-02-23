<hr>
Assembly language is a symbolic representation of the machine code that a computer can understand and execute.

In assembly language, instructions are represented using mnemonic codes, which are easier for humans to read and understand than the numeric machine code. For example, the machine code for the instruction to add two numbers might be "01010101," while the corresponding assembly language instruction might be "ADD."

<hr>

## Registers

Registers are temporary stores that may hold data, memory address or instructions.
Several general purpose registers:
- x86: eax, ecx, edx, ebx, esp, ebp, esi, edi
- amd64: rax, rcx, rdx, rbx, rsp, rbp, rsi, rdi, r8, r9, r10, r11, r12, r13, r14, r15
    #### Register Partial Access
Registers can be accessed partially, meaning that you can access the least significant bytes or most significant bytes of a word in a register. This can be useful for setting specific bits, or for reading and writing data from certain areas of the register.

> Accessing eax will sign-extend out the rest of rax. Other partial access preserve untouched parts of the register.  ![[Capture1.PNG]]

#### All partial accesses on amd64

![[Capture2.PNG]]

## Assembly Instruction
A general assembly instruction format looks like : `OPCODE OPERAND OPERAND`

#### Data manipulation instructions
Instructions can move and manipulate data in registers and memory

    MOV  : Move data from one location to another
    PUSH : Push a value onto the stack
    POP  : Pop a value from the stack
    LEA  : Load the address of a memory location

#### Control Flow instruction
Control flow is determined by conditional and unconditional jumps.
Unconditional jump: call, jmp, ret
Conditional jumps: je, jne, jge, jle

### System calls
System calls (on amd64) are triggered by:
-  Set rax to the system call number
-  Store arguments in rdi, rsi, rdx, rcx, r8, r9
-  Call the `syscall` instruction
The following assembly code that uses the `read` system call to read from the standard input (file descriptor 0) into a buffer in memory.

```

mov rdi, 0          # Set file descriptor (0 for stdin)
mov rsi, buffer     # Set buffer address
mov rdx, 100        # Set number of bytes to read
mov eax, 0          # Set system call number (0 for read)
syscall             # Make the system call

```

### Function calling conventions
In x86_64 architecture, arguments are passed on stack in reverse order and remaining arguments are passed via register rsi, rdi, rdx, rcx, r8 and r9 and the return value of function is stored in rax register.
**Registers are shared between functions, so calling conventions should agree on what registers are protected.**
- Caller-saved (Callee owned) registers are those that are expected to be modified by the called function, and it is the responsibility of the calling function (if the caller wants to preserve them) to push them before making the function call and later pop them after the function returns. The caller-saved registers in the x86-64 architecture are: `rax`,`rcx`, `rdx`, `rdi`,  `rsi`,`r8`to `r11`

- Callee-saved (Caller owned) registers, on the other hand, are expected to be preserved by the called function and restored to their original values before returning control to the calling function if the callee wants to use them. The callee-saved registers in the x86-64 architecture are: `rbx`,`rbp`,`r12` to `r15`

>It is important to note that these conventions are not enforced by the hardware, but are followed as a matter of convention to ensure compatibility between functions. A function can modify any register it wishes, but it is generally considered good practice to follow the caller-saved/callee-saved conventions to avoid unexpected behavior.