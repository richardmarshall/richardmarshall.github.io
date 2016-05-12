---
layout: post
title:  "A fork() in the road"
date:   2011-01-18 05:44:33 -0700
categories: linux libc
---

Go fork yourself

Several months ago a good friend of mine suggested that I write a post about process creation. Initially I planned on writing a single post on fork, clone, exec, and friends however after thinking about the scope of the topic I’ve decided to break the subject up into several posts. This is the first and will cover the libc magic of the fork function.

For clarity i will be referring to library functions by name (ex fork) and system calls as sys_name (ie sys_fork). Several of the examples will be making use of the strace and ltrace utilities so if you’re not familiar with them, now would be a good time to read their man pages.

Back to basics

While I am assuming there is a certain level of existing knowledge of how unix style process creation works, a quick overview seems appropriate. This topic has been covered at length by many a author much more eloquent than I, and if you are new to these ideas I would suggest a more in depth review of the material else where.

In the before time new processes were created by duplicating the calling process in it’s enterty. That meant that all of the process was copied, the kernel data structures, page tables, and the memory allocated to the process. I’m sure you can imagine how slow this would get as processes eat more and more memory that would need to get copied each time a new process was created. This describes a very naive simplistic version of what happened at a fork that hasn’t actually been the way it implemented for a very long time. The modern version of the same process does mostly the same things, copies the kernel data structures and page tables associated with the process. However the actual memory allocated to the process is not copied. Instead the page table entries for both the parent and the child are marked as copy-on-write (COW). This allows the child to share the allocated memory with the parent and new pages are only allocated to the child when a write occurs. COW provides a mechanizm by which all the memory in both the parent and child is shared until one of the who writes to a page. When a write occurs a duplicate is created for the writing process (parent or child). Allowing a single set of pages to service both processes with minimal copying overhead. This process continues until all the pages have been duplicated, one of the processes exit, or exec is called. The details of how COW is implemented is outside the scope of this post but it is an important concept that I will more than likely write another post about in the future. The important thing to take from this is COW prevents needless data duplication and reduces the overhead of creating a new process to a much more manageable level. Since one of the more common situations that requires creating a new process is executing a new program it makes sense to try and duplicate as little as possible from the parent.

Once a new process has been created by fork what is it to do? Well if you type “ls” at your command prompt bash forks and then execs ls. This means that the ls image is loaded from disk and replaces the child copy of bash. The fork/exec pattern is very common indeed; but from a functional level it is no different than any other process creation that doesn’t result in an exec. The only difference is that the logic in the process doing the fork is to have the child immediate call exec. There is nothing requiring the child to do that, and in many cases it won’t. The kernel and libc don’t care what the child does and while I am going to continue talking about exec as far as the process creation portion side of things is concerned the work is complete.

So exec. While in linux there is no actual “exec” library function, conceptually exec loads a new process image from disk to replace the currently running one. Most of the environment from the calling process is preserved after an exec, however there are a few things that get cleaned up. Such as signal handlers are reset to defaults, memory mappings are unmapped, shared memory segments are detached, etc. Once exec completes execution resumes at the entry point to the loaded process.

You may be wondering if all processes come from a parent process fork‘ing where did the first one come from? Lets just say that in the beginning there was nothing and the kernel said “let there be pid 1″ and so it was. Simply put the kernel creates the first process as part of the initial boot up and from that point onwards all new processes are created with fork.

That concludes a rather brief background on the fork/exec concept the details of many of the pieces described will be covered in this and subsequent posts in this series.

Behind the libc curtain

The description of the libc side of fork will use glibc as the reference implementation. That noted there is quite a lot of linux specific stuff to follow in this section since fork is so tightly wound with the linux threading code.

The first bit of libc magic around fork is that the library wrapper function fork does not actually call sys_fork but instead uses sys_clone. So I could have named this post “a clone() in the road” but that doesn’t have the same ring to it. With the integration of the nptl (native posix thread library) into glibc (happened in v2.3.2 in case you care.) the usage of the sys_fork call on most Linux systems went the way of the dodo.

A very contrived example will show the fork -> sys_clone relationship.

{% highlight bash %}
[/tmp]: ltrace -e fork sh -c ‘ls’
fork() = 2785
<ls output omitted for brevity>
— SIGCHLD (Child exited) —
+++ exited (status 0) +++
{% endhighlight %}

Since ltrace gives us information about library calls this shows the library function fork() being called by ‘sh’ to spawn ‘ls’.

{% highlight bash %}
[/tmp]: strace -e fork sh -c ‘ls’
<ls output omitted for brevity>
{% endhighlight %}

Here with strace we are looking at system calls and can see that sys_fork is not being called.

{% highlight bash %}
[/tmp]: strace -e clone sh -c ‘ls’
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7fdad87d29d0) = 2789
<ls output omitted for brevity>
— SIGCHLD (Child exited) @ 0 (0) —
{% endhighlight %}

As promised a sys_clone is being done by the library call to fork(). sys_clone accepts a wide range of arguments that allow the child process to share various amounts of data with the parent. That fact is why clone is so important to threading, but that’s a topic for later. The arguments passed to sys_clone by fork result in the same behavior as when using sys_fork so this was a mostly transparent library change.

The fork wrapper is a bit more involved then one might initially imagine. Without looking into things it might appear that since fork takes no arguments all it would need to do is setup a hard coded set of arguments for sys_clone and trigger the syscall to let the kernel do it’s business. However that is not the case. Due to complications with threading there was a need for the ability to register handlers to be called before and after a call to fork. pthread_atfork provides this mechanism which is commonly used by multi-threaded libraries to protect internal state during a fork in a single threaded process making use of said library. The details of why this functionality is important is outside the scope of this article but needless to say it is important. If you are interested in why the man page for pthread_atfork is a good start. What is important right now is the fact that these handlers can be registered and they need to be dealt with during the fork process.

The handlers that are registered are stored in a single linked list which is walked by the fork code. The structure contains function pointers that perform the tasks that the code that registered the fork handler needs done. The structure is defined as follows:

{% highlight c %}
struct fork_handler
{
struct fork_handler *next;
void (*prepare_handler) (void);
void (*parent_handler) (void);
void (*child_handler) (void);
void *dso_handle;
unsigned int refcntr;
int need_signal;
};
{% endhighlight %}

The fields that are most relevant to this topic are the *_handler fields. These are the call-back function pointers mentioned earlier. Their names are pretty self evident on when they are called. But for due diligence the prepare_handler is called in the parent in the preparation for a call to sys_clone. parent_handler is called after the fork in the parent process, and child_handler is called from the child also after the fork. refcntr is used to prevent the list from being removed after a call to fork has already started.

Once all of the prepare handlers have been dealt with the actual call to sys_clone happens. This is done via a macro ARCH_FORK which on linux ends up calling sys_clone. Once the “fork” has happened two different code paths are followed depending on if execution is in the parent or the child.

In the child the first order of business is to reset some libc locks so the child gets a fresh lock states. The call-backs registered for the child are then run. In the parent the call-backs for, the parent, are run. Mostly the same in both cases just subtle differences in regards to lock states.

The final step is to return the pid value returned by sys_clone. In the child value will be 0 and the parent will be the new pid of the child. This simply makes detecting and providing different behavior depending on which process the code continues in easy. Often in the child the first thing done is to exec a new program, but that is the topic for the next post in this series.

That concludes the first of the process creation series. While I do intend to start working on the next part after completing this one other posts may wiggle their way in between each new post in this series. See you next time.
