The term "sandboxing" refers to the practice of isolating a running process or program in a restricted environment, similar to how a child plays in a sandbox within defined boundaries. The first known use of the term "sandboxing" in the context of computer security was in the 1970s, when it was used to describe the practice of isolating untrusted software in a virtualized environment to prevent it from causing harm to the underlying system.

The concept of sandboxing evolved over time, with the development of new technologies such as virtualization and containerization, which made it easier and more efficient to isolate and run processes in a sandboxed environment. With the increasing prevalence of software vulnerabilities, malware and cyber attacks, the use of sandboxing has become more widespread as a way to protect systems and networks from malicious code.

## How it started

Lets take a look at how sandboxing arose from the beginning. Back in 1950's right these computers you often um wrote programs on punch cards and when you loaded these programs into the computer they would just run in the bare metal of the computer just as close to the hardware that was the only thing the computer was executing. The problem was was,you could execute only one program at a time and the process was omnipotent and could  damage the hardware (could do anything on the machine).

### Userspace and OS separation

In 1960s hardware measures were taken to saparate "system" and "process" code. The split of of an operating system and the user space arose did two things they enabled bit more protection so a
typical process running in your machine couldn't destroy it and two they allowed multiple processes to run at the same time so we slowly started developing multi-processed systems. But the problem was the processes too can clobber each others. In early prototypes you could have one process in one part of the memory space another process another part of the memory space and they would happily use their own parts of that memory space to avoid collaborating each other. The problem is that they could still clobber each other if not carefully handled. 

Again hardware measures were taken to separate memory space of all processes. We created essentially a sort of sandboxing and we separated these processes into their own memory spaces we created virtual memory nowadays most computers you interact with will have virtual memory support where multiple processes will have their own view of memory so they don't clobber each other.

### In Process separation 

Sandboxing measures started rising let's say in the 90s. The creation of scripting languages have a implicit separation between the interpreter and the interpreted code. The interpreter has you know low level operations that it does and so forth the interpreter code just calls into the interpreter. It's a sort of isolation that is not meant to be a security isolation but is often actually utilized as such.

### Browser Hacking 

Web browsers are probably the biggest example of all potential functionality being stuffed into one program um to keep up with the needs of a rapidly expanding web. In the early 2000s, we saw the rise of youtube the rise of even earlier than that video sharing and game sharing or you know game  creation platforms um a lot of rich web content and this was powered by things such as adobe flash, activex java applets. These sort of technologies that all ran in your browser with the full permissions of the user that ended up with wild west of security. You had new vulnerabilities constantly being discovered and the exploitation of these vulnerabilities would give an attacker essentially full control of your system as if they were acting as your, it led to this huge explosion and drive-by downloads for a while, they were the biggest threat on the internet you would accidentally click a malicious link it would load an activex control that would uh have a vulnerability and then you you'd be over your machine is infected.


## The rise of sandboxing 

The idea being that any code and data (reminder: code and data are roughly the same thing)  don't trust should run with basically zero privileges. 
1.  Spawn "privileged" parent process.
2.  Spawn "sandboxed" child processes.
3. When a child needs to perform a privileged action, it asks the parent.  

![[Sandbox1.PNG]]


# Chroot

In the context of linux, chroot changes the meaning of / (root directory). 
When you are on the normal linux system and you do `/` is the root of your file system chroot changes where that root sets it to another directory that is already on your file system.

>Suppose, you have a _jail_ directory in _/temp_ (/temp/jail).
>Doing `chroot(/tmp/jail)`  will disallow the process from getting out of the jail. The process that called `chroot()` and all of its children would  see a file system view that is rooted at /temp/jail as opposed to the old slash 

### Effects of chroot

###### `chroot("/tmp/jail")` has two effects:
-   For this process, change the meaning of "_/_" to mean "_/tmp/jail_".
-   and everything under that: "_/flag_" becomes "_/tmp/jail/flag_"
-   For this process, change "_/tmp/jail/.._" to go to "_/tmp/jail_"

##### `chroot("/tmp/jail")` does NOT:
-   Close resources that reside outside of the jail.
-   _cd (chdir())_ into the jail. 
-  Change the current working directory
-   Do anything else!

### Chroot Escapes
-  Since chroor does not close prevoiusly opened resurces!
Similar to open and execve, Linux has openat and execveat:
`int open(char *pathname, int flags);`
`int openat(int dirfd, char *pathname, int flags);`

`int execve(char *pathname, char **argv, char **envp);
`int execveat(int dirfd, char *pathname, char **argv, char **envp, int flags);

-dirfd can be a file descriptor representing any open()ed directory, or the special value AT_FDCWD (note: chroot()  does not change the current working directory)!
- The kernel has no memory of previous chroots for a process!
What happens if you chroot again? This will over write the previous chroot.
- Missing other forms of isolation:
-PID  ( it doesn't provide process id isolation)
-network  ( if we launch a network service inside the chroot it uses their same resources as network services outside of the chroot)
-IPC   


# Seccomp

Seccomp (short for "secure computing mode") is a Linux kernel feature that:
-   allow certain system calls
-   disallow certain system calls
-   filter allowed and disallowed system calls based on argument variables
> seccomp rules are inherited by children!

**How it works**
Seccomp filters are implemented as Berkeley Packet Filter (BPF) programs, which are executed by the kernel when a system call is made. When a system call is made that does not match the filter, the kernel sends a SIGKILL signal to the process, causing it to terminate.
See http://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html 

### Secomp escape
Generally, to do anything useful, a sandboxed process needs to be able to communicate with the privileged process.
Normally, this means allowing the sandboxed process to use *some* system calls. This opens up some attack vectors:
-   permissive policies
-   syscall confusion  
-   kernel vulnerabilities in the syscall handlers

	##### Permissive Policies
Combination of:
1.  System calls are complex, and there are a lot of them...
2.  Developers might avoid breaking functionality by erring on the side of permissiveness.

Well-known example: depending on system configuration, allowing the ptrace() system call could let a sandboxed process to "puppet" a non-sandboxed process.
Some less well-known effects:
-   sendmsg() can transfer file descriptors between processes
-   prctl() has bizarre possible effects
-   process_vm_writev() allows direct access to other process' memory

	##### Syscall Confusion
Many 64-bit architectures are backwards compatible with their 32-bit ancestors:
On some systems (including amd64), you can switch between 32-bit mode and 64-bit mode in the same process, so the kernel must be ready for either.
Interestingly, system call numbers differ between architectures, including 32-bit and 64-bit variants of the same architecture!
Policies that allow both 32-bit and 64-bit system calls can fail to properly sandbox one or the other mode.
Example: `exit()` is syscall 60 _(mov rax, 60; syscall)_ on **amd64**, 1 _(mov eax, 1; int 0x80)_ on x86.

