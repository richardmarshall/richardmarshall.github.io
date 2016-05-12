---
layout: post
title:  "Capital Punishment"
date:   2011-11-21 05:44:33 -0700
categories: kernel libc
---
There is no stay on these executions

After talking about fork() the next logical step is to continue with exec(). The fork()/exec() pairing, as mentioned in my previous post in this series, is a method by which new processes are started in unix like systems. exec is the stage at which a new program is loaded from disk to replace the currently running process. In this post I will cover the user land side of the exec family of functions which provide userspace the interface to start processes other than init. Most of the meat of how exec() works will be covered when I post about the kernel side of things.

Diving into the details

The library function that ultimately makes the magic happen is execve. It is the version of exec that accepts an argument vector (v) and an environment vector (e) to be used by the newly loaded program. This function is a very simple wrapper in front of the sys_execve system call. That is if you ignore the case of having bounds checking enabled in glibc. The execve function is as straightforward as they come, it simply passes it’s arguments on as is and makes the system call. In the unlikely case that bounds checking is enabled, it first iterates overs both argv and envp to make sure the strings actually terminate within legal bounding limits. The path to the executable is checked as well. The details are unimportant as it is disabled in almost all cases. When the bounds checking support is not compiled in it is a single line function.

{% highlight c %}
int __execve(const char *file, char *const argv[], char *const envp[])
{
return INLINE_SYSCALL(execve, 3, file, argv, envp);
}
{% endhighlight %}

Granted the INLINE_SYSCALL macro does expand to substantially more than 1 line, but that’s a topic for another day. As you can see execve only takes 3 arguments; a path to the executable (file), a null terminated array of strings to use as program arguments (argv), and a null terminated array of strings to use as the environment (envp). To see this in action lets take a look at a very contrived example of what is marginally equivalent to a small portion of what happens when you type “ls -l .” at a shell prompt.

{% highlight c %}
char *envp[] = {"FOO=bar", (char *)0};
char *argv[] = {"/bin/ls", "-l", ".", (char *)0};
execve("/bin/ls", argv, envp);
{% endhighlight %}

The unusual thing about execve is when it succeeds it doesn’t return. This makes sense since once execve completes without error the calling process has been replaced and there isn’t a valid place to return to. In the error case the function returns -1 and errno is set to give you an idea of what went wrong. Check out the man page execve(2) for a full list of those error numbers and what they mean. So instead of returning when all is well execution instead continues at the entry point of the new program that was loaded into memory.

It is also worth noting that the handling of scripts using the #! mechanism is done entirely in the kernel. Though with one caveat, in the case the calling program makes use of a execve wrapper function glibc does some magic in certain situations to preform this it self. If a file is passed in that is executable but the kernel fails to find a handler to run the file the library functions will try again with the file as an argument to /bin/sh. This provides a way to have simple shell scripts without a #! and can also provide lots of confusion.

I could leave it at that, as for the most part that is really all that happens in user space. However there are a plethora of convenience wrapper functions in libc to make calling execve “less” of a headache. These include, execl, execlp, execle, execv, execvp, and execvpe. They all are build on top of execve and have varying call signatures for various use cases.

Letters make the difference

The various letters after exec all have different meanings and relate to the type and number of arguments accepted and how they are handled.

execl*() refers to the fact that the arguments passed to the program being executed are included as individual arguments to the execl*() function. This proves to be the simplest way of doing argument passing if the number of arguments is fixed. Since the arguments are passed directly to the execl*() function you are limited to a fixed number of arguments at compile time.

{% highlight c %}
int execlp(const char *file, const char *arg, …);
{% endhighlight %}

Arguments are read from the stack until a NULL character is found.

execv*() means that program arguments are passed via an array of strings in the same manner as execve. The array must be terminated with a NULL character as the last element. This allows you to have more flexibility for argument passing in comparison to execl*, at the cost of having to build the array your self.

exec*e() means that environment is passed via an array of strings also in the same manner as execve. The array must be terminated with a NULL character as the last element just as the argument list in execv*().

exec*p() functions do path expansion using the PATH environment variable (or current directory is PATH is not set). Both execvp and execlp are wrappers around execvpe that does the actual path lookup before calling execve. While these functions do the same thing that a shell like bash does to allow you to type ‘ls‘ and correctly invoke ‘/bin/ls‘, bash doesn’t actually use any of the exec*p() functions and does the path look up it self.

In the exec functions that do not specify an environment vector the new program inherits the environment of the calling process.

Call signatures

For completeness the following is a list of example calls for each of the execve helper functions:

execl:

{% highlight c %}
execl("/bin/ls", "/bin/ls", "-l", ".", (char *)0);
{% endhighlight %}

execlp:

{% highlight c %}
execlp("ls", "ls", "-l", ".", (char *)0);
{% endhighlight %}

execle:

{% highlight c %}
char *envp[] = {"FOO=bar", (char *)0};
execle("/bin/ls", "/bin/ls", "-l", ".", (char *)0, envp);
{% endhighlight %}

execv:

{% highlight c %}
char *argv[] = {"/bin/ls", "-l", ".", (char *)0};
execv("/bin/ls", argv);
{% endhighlight %}

execvp:

{% highlight c %}
char *argv[] = {"ls", "-l", ".", (char *)0};
execvp("ls", argv);
{% endhighlight %}

execvpe:

{% highlight c %}
char *envp[] = {"FOO=bar", (char *)0};
char *argv[] = {"ls", "-l", ".", (char *)0};
execvpe("ls", argv, envp);
{% endhighlight %}

That concludes this brief description of execve and friends. Stay tuned for further posts that dive into the kernel implementation of this functionality.
