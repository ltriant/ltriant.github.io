---
layout: post
title:  "Debugging EV and Perl’s Signal Handlers"
date:   2019-03-23 00:00:00 +1100
description: "Diving into EV's internals to debug a signal handling problem"
---
A little while ago, I ran into a weird problem with a small Perl system that I had been developing. What followed was one of the more interesting debugging experiences of my career so far, involving digging into event loops, and how Perl and Linux handle signals.

The program is made up of a non-blocking parent process, which, among other functions, is mainly responsible for supervising several child processes, spawning replacement children should any exit unexpectedly.

The parent process uses [AnyEvent](https://metacpan.org/pod/AnyEvent) for the non-blocking functionality, and the code specifically forces the use of the [EV](https://metacpan.org/pod/EV) backend event loop (which is a wrapper to the [libev](http://software.schmorp.de/pkg/libev.html) library), instead of letting AnyEvent auto-detect an event loop at runtime. This is done so that - across many codebases - the same event loop is always used.

Also relevant to the story is that at startup, the parent process announces itself to a system which, for the purpose of this post, is like a service discovery system. To do this, it uses [Sys::Hostname](https://metacpan.org/pod/Sys::Hostname) to get the current host’s hostname.

A simple example of this functionality might look something like this:

```perl
#!/usr/bin/env perl

use v5.10;
use warnings;
use strict;

use EV;
use AnyEvent;

use Sys::Hostname qw(hostname);

main();

sub main {
	announce_myself();

	# Any state about the workers is stored in this hash, keyed by pid
	my %workers;

	foreach my $worker (1 .. 5) {
		make_child(\%workers);
	}

	say "[$$] new parent here, looking after children for the rest of my life";

	AnyEvent->condvar->recv;
}

sub announce_myself {
	my $hostname = hostname();

	say "[$$] announcing myself to service discovery";

	# Imagine there's more code here to announce this host with
	# service discovery
}

sub make_child {
	my ($worker_state) = @_;

	my $pid = fork;

	# fork() failed, bail out
	if (not defined $pid) {
		die "fork(): $!";
	}

	if ($pid == 0) {
		# Child proc runs here
		child_main();
		exit(0);
	}

	# Parent proc runs here
	my $child_handler = AnyEvent->child(
		pid => $pid,
		cb  => sub {
			say "[$$] child $pid died, respawning a new one...";
			make_child($worker_state);
			delete $worker_state->{$pid};
		},
	);

	$worker_state->{$pid} = {
		child_w => $child_handler,
	};
}

sub child_main {
	say "  [$$] child ready, randomly doing stuff";

	while (1) {
		sleep(rand(10));

		# Imagine some real work is being done here, instead of
		# sleeping for random periods of time
	}
}
```

When run, the output looks something like this:

```
$ perl basic-supervisor.pl
[92815] announcing myself to service discovery
  [92816] child ready, randomly doing stuff
  [92817] child ready, randomly doing stuff
  [92818] child ready, randomly doing stuff
  [92819] child ready, randomly doing stuff
  [92820] child ready, randomly doing stuff
[92815] new parent here, looking after children for the rest of my life
```

And now, in another terminal, if a process were to be killed:

```
$ kill 92816
```

The supervisor will fork a new child to replace it:

```
[92815] child 92816 died, respawning a new one...
  [92822] child ready, randomly doing stuff
```

Great! It does exactly as expected! One of the child processes was killed, the parent process was notified via the event loop, and it replaced the dead process with a new one.

This code was developed and due to be deployed onto a host, running Perl 5.8.9 on CentOS 5.

## The Problem

Part-way through development, this particular system was replatformed into a new provider, upgraded to CentOS 6, and also upgraded to Perl 5.20.1.

So I redeployed everything onto a new development host based on the new production servers’ architecture... and the process management functionality completely stopped working; any time a child process exited unexpectedly, the parent never noticed and thus, a new child process was never forked to replace it.

The first thing I did - through a very tedious process - was ensure all of the CPAN dependencies (and there were many that the code above doesn’t use) were the same versions to the working version of the code; it wouldn’t have been the first time that a dependency version difference was the cause of a weird problem like this.

But that didn’t fix anything.

I then wondered if the problem was within AnyEvent itself, so I replaced the call to `AnyEvent->child` with the equivalent `EV::child` call.

That didn’t fix anything either, but I was now convinced that AnyEvent wasn’t the source of the problem, so I became suspicious of EV.

To confirm my suspicion, I changed the AnyEvent backend from EV to the pure-Perl backend, by setting the `AE_MODEL` environment variable to Perl. And everything worked again!

<pre>
$ AE_MODEL=Perl perl basic-supervisor.pl
[27239] announcing myself to service discovery
  [27242] child ready, randomly doing stuff
  [27243] child ready, randomly doing stuff
  [27244] child ready, randomly doing stuff
  [27245] child ready, randomly doing stuff
  [27246] child ready, randomly doing stuff
[27239] new parent here, looking after children for the rest of my life
  <strong>[27239] child 27246 died, respawning a new one...
  [27247] child ready, randomly doing stuff</strong>
</pre>

I also then discovered that using the [Event](https://metacpan.org/pod/Event) and [POE](https://metacpan.org/pod/POE) event loops also worked.

So, the problem was definitely related to EV.

## SIGCHLD

Before going any further, here’s a super-brief refresher on how a parent process is notified of a status change in one of its child processes.

When a new process is forked, the parent process must do one of two things:

1. Wait on that process via the `wait4` syscall, invoked from the `waitpid` function; or

1. Declare that it will ignore child signals, which will result in the child being inherited by PID 1 (init, upstart, systemd, etc)

In order to take an action when a child dies, the parent must wait.

When the child has a status change (e.g. it exits, or is killed, or is stopped, or is continued), the kernel sends an interrupt to the parent process, and the `wait` or `waitpid` functions will return, bringing with it the information necessary to know if the child exited or if it was killed, what the exit code was, what signal killed it, etc.

A basic handler looks like this in C:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void)
{
	pid_t pid = fork();

	if (pid < 0) {
		perror("fork");
		exit(1);
	}

	if (pid == 0) {
		// Child process, just do nothing
		while (1) sleep(1);
		exit(0);
	}

	// Parent process, wait on the child proc forever
	printf("waiting on child pid: %d\n", pid);

	int status;
	waitpid(pid, &status, 0);

	printf("status: %d\n", status);

	if (WIFSIGNALED(status)) {
		printf(" SIGCHLD\n");

		if (WTERMSIG(status) == SIGTERM) {
			printf(" child rx'd SIGTERM\n");
		}
	}

	return 0;
}
```

Compile and run in one terminal:

```
$ gcc -o sigchld sigchld.c
$ ./sigchld
waiting on child pid: 95242
```

And then send a SIGTERM to the child from another terminal:

```
$ kill -TERM 95242
```

And then the first terminal will catch the child dying and know how the child died:

```
status: 15
 SIGCHLD
 child rx'd SIGTERM
```

So, at a low level, the only way a parent can know if a child process exited and do something about it (e.g. invoke a callback), is via a call to the `wait4` syscall, which is invoked via the `waitpid` function.

Going back to the Perl supervisor code, I wanted to verify that the parent received the SIGCHLD interrupt, and that the `wait4` syscall was called.

<pre>
$ strace -e trace=process perl basic-supervisor.pl > /dev/null

execve("/opt/rh/rh-perl520/root/usr/bin/perl", ["perl", "basic-supervisor.pl"], [/* 33 vars */]) = 0
arch_prctl(ARCH_SET_FS, 0x7ff0a781c700) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27874
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27874, si_status=0, si_utime=0, si_stime=0} ---
wait4(27874, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 27874
<strong>clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27876</strong>
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27877
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27878
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27879
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27880
<strong>--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=27876, si_status=SIGTERM, si_utime=0, si_stime=0} ---</strong>
</pre>

The first highlighted line shows a child with process ID 27876 being created via the clone syscall.

The second line shows that the SIGCHLD interrupt was indeed sent to the parent process when the child was killed with a SIGTERM.

But there’s no `wait4` syscall.

It looks like the SIGCHLD handler that EV registered isn’t being triggered at all.

## EV’s Child Signal Handler

Without diving too deep into what essentially is just following function calls, EV’s child signal handler is setup in the following steps:

1. When the EV module is `use`’d, the `default_loop` XSUB (in EV.xs) is called

1. `default_loop` calls the C function `ev_default_loop` (in libev/ev.c)

1. `ev_default_loop` calls `ev_signal_start` (in libev/ev.c)

1. `ev_signal_start` creates a signal handler via the [POSIX sigaction API](https://en.wikipedia.org/wiki/Sigaction)

The key takeaway is that EV’s child handler is created almost immediately after startup. and it is created from within the libev C library.

Looking again at the first few lines of the strace output from above:

<pre>
$ strace -e trace=process perl basic-supervisor.pl > /dev/null
execve("/opt/rh/rh-perl520/root/usr/bin/perl", ["perl", "basic-supervisor.pl"], [/* 33 vars */]) = 0
arch_prctl(ARCH_SET_FS, 0x7ff0a781c700) = 0

<strong>clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff0a781c9d0) = 27874</strong>

<strong>--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=27874, si_status=0, si_utime=0, si_stime=0} ---</strong>

<strong>wait4(27874, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 27874</strong>
</pre>

We can see that some kind of child process is created, a SIGCHLD is sent to the parent, *and* there’s a successful `wait4` syscall.

But what is it? It’s not my code…

## Sys::Hostname

Finally, we get to this key, core Perl module.

The purpose of the hostname function, which is exported by [Sys::Hostname](https://metacpan.org/pod/Sys::Hostname), is to - in case the function name wasn’t clear - get the hostname of the current machine. [The code](https://metacpan.org/release/perl/source/ext/Sys-Hostname/Hostname.pm) attempts to get this information by using a few different methods in order to be portable across many platforms.

One of those methods includes calling out to the hostname command. The code looks like this:

```perl
eval {
    local $SIG{__DIE__};
    local $SIG{CHLD};
    $host = `(hostname) 2>/dev/null`; # BSDish
}
```

Perl’s SIGCHLD handler is overwritten. But it has been declared local, so the *original* handler should be put back into `$SIG{CHLD}` after the block of code has finished. This code does the right thing.

However, because the original handler in this case is EV’s handler, which was declared in the libev world, the Perl interpreter is unable to replace the handler.

The hostname function is called later at runtime, long after EV has setup its SIGCHLD handler.

Finally, I understand the problem:

1. libev (via EV) sets up a SIGCHLD handler at Perl’s compile-time

1. `Sys::Hostname::hostname()` overwrites the SIGCHLD handler temporarily, but Perl is unable to reinstate libev’s SIGCHLD handler afterwards

1. This results in there being no SIGCHLD handler at all

The problem can be further broken down into a much more minimal example:

```perl
#!/usr/bin/env perl

use v5.10;
use warnings;
use strict;

use EV;
use AnyEvent;

# XXX comment this out, and it works
{ local $SIG{CHLD}; }

my $done = AnyEvent->condvar;

my $pid = fork;

die "fork(): $!" if not defined $pid;

if ($pid == 0) {
	exit 5;
}

my $w = AnyEvent->child(
	pid => $pid,
	cb  => sub {
		my ($pid, $status) = @_;
		say "pid $pid exited with status $status";
		$done->send;
	},
);

$done->recv;
```

The code forks a child, the child exits immediately, and the parent waits for the child to exit, but it never catches the child exiting, so it runs forever.

If line 11 is commented out, then everything works just fine.

Although it’s not documented in a place that’s super-easy to locate, the [perldoc for perlxs](https://perldoc.perl.org/perlxs.html#CAVEATS) hints at this being a problem:
> XS code has full access to system calls including C library functions. It thus has the capability of interfering with things that the Perl core or other modules have set up, **such as signal handlers** or file handles. It could mess with the memory, or any number of harmful things. Don’t.

> In general, the perl interpreter views itself as the center of the universe as far as the Perl program goes. XS code is viewed as a help-mate, to accomplish things that perl doesn’t do, or doesn’t do fast enough, but always subservient to perl. The closer XS code adheres to this model, the less likely conflicts will occur.

## Possible Solutions

To fix (or work around) this problem, there are several options:

1. **Modify libev and EV.** I’m not entirely sure how, given that the signal handling is setup in libev, which is the C library that the Perl EV module wraps. This would require a lot of work for the developer(s) of libev, especially given that libev also has wrappers in Python, Ruby, etc, so the change is quite intrusive, even to other communities.

1. **Load EV at runtime after getting the hostname.** This can either be done by removing the use EV statement and setting the `AE_MODEL` environment variable to EV prior to running the script. This will cause EV to be loaded when the event loop is first invoked (i.e. the call to `AnyEvent->condvar->recv` in the code above). But if the hostname function is *ever* called after EV is loaded, the problem will be silently introduced at runtime. This could be disastrous in a production environment.

1. **Stop using Sys::Hostname.** The [POSIX](https://metacpan.org/pod/POSIX) library exports the uname function, which can fill the same need and won’t clobber Perl’s SIGCHLD handler. However, although this solves the problem with Sys::Hostname, it *doesn’t* solve the problem for any other modules that exhibit this behavior, whether they’re publicly available via CPAN, or internal only.

1. **Stop using EV.** The pure-Perl, the Event and the POE event loops all work fine, however I rely on EV’s speed in other systems, and the more consistent I can keep the usage of event loops, the less cognitive load required, so I don’t like this option as much, but it’s a compromise that I *could* make.

1. **Stop using EV‘s child handlers.** For the platform I’m targeting, this can be done with the [POSIX::SigAction](https://metacpan.org/pod/POSIX#POSIX::SigAction) class, as an abstraction to the [POSIX sigaction API](https://en.wikipedia.org/wiki/Sigaction).

In the end, option 5 is the best compromise for my situation; it adds a little extra complexity by requiring hand-rolled child handlers rather than relying on AnyEvent and EV’s abstraction, but it allows the continued use of EV, and it future-proofs the code from any other Perl modules that might mess with SIGCHLD.

The modified supervisor code ends up looking like this:

```perl
#!/usr/bin/env perl

use v5.10;
use warnings;
use strict;

use EV;
use AnyEvent;
use POSIX qw();

use Sys::Hostname qw(hostname);

main();

sub main {
	announce_myself();

	my $sigset = POSIX::SigSet->new;
	my $signal = POSIX::SigAction->new(\&sigchld_handler, $sigset, POSIX::SA_NODEFER);
	POSIX::sigaction(POSIX::SIGCHLD, $signal);

	foreach my $worker (1 .. 5) {
		make_child();
	}

	say "[$$] new parent here, looking after children for the rest of my life";

	AnyEvent->condvar->recv;
}

sub announce_myself {
	my $hostname = hostname();

	say "[$$] announcing myself to service discovery";

	# Imagine there's more code here to announce this host with
	# service discovery
}

sub sigchld_handler {
	while (1) {
		my $kid = waitpid(-1, POSIX::WNOHANG);

		if ($kid <= 0) {
			last;
		}

		say "[$$] child $kid died, respawning a new one...";
		make_child();
	}
}

sub make_child {
	my $pid = fork;

	# fork() failed, bail out
	if (not defined $pid) {
		die "fork(): $!";
	}

	if ($pid == 0) {
		# Child proc runs here
		child_main();
		exit(0);
	}
}

sub child_main {
	say "  [$$] child ready, randomly doing stuff";

	while (1) {
		sleep(rand(10));

		# Imagine some real work is being done here, instead of
		# sleeping for random periods of time
	}
}
```

## How did this all happen anyway?

How did an upgrade from Perl 5.8.9 on CentOS 5 to Perl 5.20.1 on CentOS 6 cause this problem anyway?

Looking closer at the Sys::Hostname code, one of the first methods for determining the hostname is to call a `ghname` function, which Sys::Hostname defines in its XS code.

```perl
# method 1' - try to ask the system
$host = ghname() if defined &ghname;
```

I thought it was funny that it only calls the `ghname` function *if* it’s defined. I presume this might be for machines where a compiler may not be available to build the XS module.

On the servers that were exhibiting the pathological behaviour, the compiled XS code (the .so file) was unable to be loaded, therefor this function was never called. Unfortunately, I was never able to figure out *why* the XS code couldn’t be loaded as I no longer have access to that system to further investigate.

On the older servers that worked perfectly fine, the compiled XS code *was* loaded successfully, therefor the `ghname` function was called instead of falling back to the hostname command.

In the end, it was a little disappointing to not find out the rest of the story, but I was satisfied enough with what I’d discovered, as well as what I’d learned about some of the libraries that I heavily rely on for high-performance servers, and about signal handling in general.
