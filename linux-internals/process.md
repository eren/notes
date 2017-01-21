# Starting a Program From Shell
Shell (in this case bash) has a `reader_loop` which reads the characters from the user and executes accordingly. It has a number of
checks before reader loop such as checking /dev/tty, checking if in debug mode, reading .bashrc etc.


## Bash Reader Loop and Execve
Mainly, the most important part while running a program is this workflow:

```
execute_command
--> execute_command_internal
----> execute_simple_command
------> execute_disk_command
--------> shell_execve
```

`shell_execve` calls `execve` system call which has the following signature: 

```
int execve(const char *filename, char *const argv [], char *const envp[]);
```

The calls made by `execve` are:

```
execve
-->  do_execve
---->  do_execveat_common
```

The real work is done by do_execveat_common which is called with following argument:

```
do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
```

* The first argument is the file descriptor that represent directory with our application, in our case the `AT_FDCWD` means that the given pathname is interpreted relative to the current working directory of the calling process.
* The fifth argument is flags.

## Preperation and Checks in do_execveat_common

- Check filename pointer
- Check running process limit. If limit exceeds, return with error. If not, clear `PF_NPROC_EXCEEDED` bit.
- Unshare files of the current task to eliminate potential leak of the execve'd binary's file descriptor.
- Begin to create `bprm` structure which holds the arguments that are used when loading binaries such as:

   `vma` field which represents single memory area over a contiguous interval in a given address space where the application will be loaded 

   `mm` field which is a memory descriptor of the binary, pointer to the top of memory and many other different fields respectively.

- Create `bprm` credentials using `prepare_bprm_creds` function.

   `cred` structure is initialized and it contains the security context of a task for example `real uid` of the task, `real guid` of the task, `uid` and `guid` for the virtual file system operations etc.

- Check if we can safely run the binary using `check_unsafe_exec` and then set current process to `in_execve` state.
- Call `do_open_execat`

## Inside do_open_execat
- Check the flags that we passed to the `do_execveat_common` function (remember that we have `0` in the `flags`)
- Search and open executable file on the disk (as a side node, this part involves VFS, path finding, inodes, etc).
- Check the binary is not from `noexec` mountpoint. Avoid executing a binary from filesystems that do not contain executable binaries like proc or sysfs
- Initialize file structure and return pointer on this structure.
- Run `sched_exec()`

```c
file = do_open_execat(fd, filename, flags);
retval = PTR_ERR(file);
if (IS_ERR(file))
    goto out_unmark;

sched_exec();
```

The `sched_exec` function is used to determine the least loaded processor that can execute the new program and to migrate the current process to it.


# Links
[How does the Linux kernel run a program](https://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-4.html) by 0xax
[Anatomy of a Progmam in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/) by Gustavo Duarte
[Anatomy of Linux process management](http://www.osnews.com/story/9691/Anatomy_of_the_Linux_boot_process) by IBM Developer Works
