Kernel Rop is CTF challenge available at [hexCtf2020](https://2020.ctf.link/). The challenge provides you the vulnerable kernel module you will be exploiting this module to perform privilage escallation. If you are new to kernel exploitation better start with these write-ups

1. [Low Level Adventure](https://0x434b.dev/dabbling-with-linux-kernel-exploitation-ctf-challenges-to-learn-the-ropes/) 
2. [Ikmidas](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/#the-simplest-exploit---ret2usr)


#### swapgs_restore_regs_and_return_to_usermode issue

While practicing the second part we were jumping to swapgs_restore_regs_and_return_to_usermode + 22 and there were so many push and pops i was confused how ROP chain made by lkmidas was working

```bash
.text:FFFFFFFF81200F26            mov     rdi, rsp
.text:FFFFFFFF81200F29            mov     rsp, qword ptr gs:unk_6004
.text:FFFFFFFF81200F32            push    qword ptr [rdi+30h] // SS 
.text:FFFFFFFF81200F35            push    qword ptr [rdi+28h] // RSP
.text:FFFFFFFF81200F38            push    qword ptr [rdi+20h] // EFLAGS
.text:FFFFFFFF81200F3B            push    qword ptr [rdi+18h] // CS 
.text:FFFFFFFF81200F3E            push    qword ptr [rdi+10h] // RIP
.text:FFFFFFFF81200F41            push    qword ptr [rdi]     // garbage
.text:FFFFFFFF81200F43            push    rax // garbage
.text:FFFFFFFF81200F44            jmp     short loc_FFFFFFFF81200F89
```

look closely we are saving the current `rsp` into `rdi` and then moving some value into `rsp` i don't know yet what it is then using `rdi` we are building the frame on stack again requried by `iretq` 


**How is the signal_handler trick is working ?** 

The answer to this question lies in man page of signal do `man 7 signal` and read the section called `Execution of signal handlers`

**What the hell is swapgs_restore_regs_and_return_to_usermode exactly ?**

Many authors of the write-up call this gadget function and confusingly when we try to audit it from linux kernel code we will not find this symbol because it's just label defined in the code perhaps it is for readability and code management so we can know what the following code is doing...

https://elixir.bootlin.com/linux/v5.16.10/source/arch/x86/entry/entry_64.S#L569 here is the link to read code i didn't like at&t syntax... but you are welcome to read.

**How does the value of rax remains intact when we returns to userland after reading the offset value of ksymtab_commit_creds ?**
I am not sure but probably it is because the return value of syscalls are stored in rax so in this whole process of switching from the kernel_mode to userland the value of rax remains intact since it is return value of that sycall...
my assumption can be supported by the fact that we have `push rax` in kpti trampoline code before there is `pop rax` but for some reason cannot trace it back properly it gets crashed but i am somewhat satisfied this might be the reason
