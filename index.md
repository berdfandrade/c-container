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

It's turned off by default in Linux at the time of this writing, but many distributions apply patches to turn it on in a limited way.

But all of these issues apply to hosts with user namespaces compiled in; it doesn't really matter whether we use user namespaces or not, especially since I'll be preventing nested user namespaces. So I'll only use a user namespace if they're available.

(The user-namespace handling in this code was originally pretty broken. Jann Horn in particular gave great feedback. Thanks!)

## contained.c

This program can used like this, to run ` /misc/img/bin/sh in /misc/img` as root:

```
    [lizzie@empress l-c-i-500-l]$ sudo ./contained -m ~/misc/busybox-img/ -u 0 -c /bin/sh
    => validating Linux version...4.7.10.201610222037-1-grsec on x86_64.
    => setting cgroups...memory...cpu...pids...blkio...done.
    => setting rlimit...done.
    => remounting everything with MS_PRIVATE...remounted.
    => making a temp directory and a bind mount there...done.
    => pivoting root...done.
    => unmounting /oldroot.oQ5jOY...done.
    => trying a user namespace...writing /proc/32627/uid_map...writing /proc/32627/gid_map...done.
    => switching to uid 0 / gid 0...done.
    => dropping capabilities...bounding...inheritable...done.
    => filtering syscalls...done.
    / # whoami
    root
    / # hostname
    05fe5c-three-of-pentacles
    / # exit
    => cleaning cgroups...done.

```

So, a skeleton for it:

- `contained.c`

```c
    /* -*- compile-command: "gcc -Wall -Werror -lcap -lseccomp contained.c -o contained" -*- */
/* This code is licensed under the GPLv3. You can find its text here:
   https://www.gnu.org/licenses/gpl-3.0.en.html */

#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <grp.h>
#include <pwd.h>
#include <sched.h>
#include <seccomp.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sys/capability.h>
#include <sys/mount.h>
#include <sys/prctl.h>
#include <sys/resource.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <sys/utsname.h>
#include <sys/wait.h>
#include <linux/capability.h>
#include <linux/limits.h>

struct child_config {
    int argc;
    uid_t uid;
    int fd;
    char *hostname;
    char **argv;
    char **mount_dir;
};

<<capabilities>>

<<mounts>>

<<syscalls>>

<<resources>>

<<child>>

<<choose-hostname>>

int main (int argc, char **argv)
{
	struct child_config config = {0};
	int err = 0;
	int option = 0;
	int sockets[2] = {0};
	pid_t child_pid = 0;
	int last_optind = 0;
	while ((option = getopt(argc, argv, "c:m:u:"))) {
		switch (option) {
		case 'c':
			config.argc = argc - last_optind - 1;
			config.argv = &argv[argc - config.argc];
			goto finish_options;
		case 'm':
			config.mount_dir = optarg;
			break;
		case 'u':
			if (sscanf(optarg, "%d", &config.uid) != 1) {
				fprintf(stderr, "badly-formatted uid: %s\n", optarg);
				goto usage;
			}
			break;
		default:
			goto usage;
		}
		last_optind = optind;
	}
finish_options:
	if (!config.argc) goto usage;
	if (!config.mount_dir) goto usage;

<<check-linux-version>>

	char hostname[256] = {0};
	if (choose_hostname(hostname, sizeof(hostname)))
		goto error;
	config.hostname = hostname;

<<namespaces>>

	goto cleanup;
usage:
	fprintf(stderr, "Usage: %s -u -1 -m . -c /bin/sh ~\n", argv[0]);
error:
	err = 1;
cleanup:
	if (sockets[0]) close(sockets[0]);
	if (sockets[1]) close(sockets[1]);
	return err;
}

```
