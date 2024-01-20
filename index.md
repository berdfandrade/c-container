# Linux containers in 500 lines of code

link : https://blog.lizzie.io/linux-containers-in-500-loc.html

 I've used Linux containers directly and indirectly for years, but I wanted to become more familiar with them. So I wrote some code. This used to be 500 lines of code, I swear, but I've revised it some since publishing; I've ended up with about 70 lines more.

I wanted specifically to find a minimal set of restrictions to run untrusted code. This isn't how you should approach containers on anything with any exposure: you should restrict everything you can. But I think it's important to know which permissions are categorically unsafe! I've tried to back up things I'm saying with links to code or people I trust, but I'd love to know if I missed anything. 

### Container setup

There are several complementary and overlapping mechanisms that make up modern Linux containers. Roughly, 

- `namespaces` are used to group kernel objects into different sets that can be accessed bu specif process trees. For example, pid namespaces limit the view of the process list to the process within the namespace. There are a couple of different kind of namespaces. I'll go into this more later. 

- `capabilities` are used here to set some coarse limits on what uid 0 can do. 

- `cgroups` is a mechanisms to limit usage of resources like memory, disk io, and cpu-time.

- `setrlimit` is another mechanism for limiting resource usage. It's older than cgroups, but can do some things cgoups can't. 

These are all Linux kernel mechanisms. Seccomp, capabilities, and `setrlimit` are all done with system calls. `cgroups` is accessed through a filesystem. 

There's a lot here, and the scope of each mechanism is pretty unclear. They overlap a lot and it's tricky to find the best way to limit things User namespaces are somewhat new, and promise to unify a lot of this behaviour. But unfortunately compiling the kernel with user namespaces neabled complicates things. Compiling with user namespaces changes the semantics of capabilities system-wide, which could cause more problems or at least confusion1. There have been a large number of privilege-escalation bugs exposed by user namespaces. "Understanding and Hardening Linux Containers" explains 

    - Despite the large upsides the user namespace provides in terms of security, due to the sensitive nature of the user namespace, somewhat conflicting security models and large amount of new code, several serious vulnerabilities have been discovered and new vulnerabilities have unfortunately continued to be discovered. These deal with both the implementation of user namespaces itself or allow the illegitimate or unintended use of the user namespace to perform a privilege escalation. Often these issues present themselves on systems where containers are not being used, and where the kernel version is recent enough to support user namespaces. 
