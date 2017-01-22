# Starting a Program From Shell
Shell (in this case bash) has a `reader_loop` which reads the characters from the user and executes accordingly. It has a number of
checks before reader loop such as checking /dev/tty, checking if in debug mode, reading .bashrc etc.


## Bash Reader Loop and Execve
Mainly, the most important part while running a program is this workflow:

```
execute_command
--> execute_command_internal
----> execute_simple_command
------> execute_disk_command - calls make_child() and in turn calls fork()
--------> shell_execve - this is called in the child.
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
    - `vma` field which represents single memory area over a contiguous interval in a given address space where the application will be loaded 

    - `mm` field which is a memory descriptor of the binary, pointer to the top of memory and many other different fields respectively.

- Create `bprm` credentials using `prepare_bprm_creds` function.

   `cred` structure is initialized and it contains the security context of a task for example `real uid` of the task, `real guid` of the task, `uid` and `guid` for the virtual file system operations etc.

- Check if we can safely run the binary using `check_unsafe_exec` and then set current process to `in_execve` state.
- Call `do_open_execat`

## Inside do_open_execat
- Check the flags that we passed to the `do_execveat_common` function (remember that we have `0` in the `flags`)
- Search and open executable file on the disk (as a side node, this part involves VFS, path finding, inodes, etc).
- Check the binary is not from `noexec` mountpoint. Avoid executing a binary from filesystems that do not contain executable binaries like proc or sysfs
- Initialize file structure and return pointer on this structure.

## Back to do_execveat_common
```c
# ...
# ...
file = do_open_execat(fd, filename, flags);
retval = PTR_ERR(file);
if (IS_ERR(file))
    goto out_unmark;

sched_exec();
# ...
# ...
```
The `sched_exec` function is used to determine the least loaded processor that can execute the new program and to migrate the current process to it.

- Check the file descriptor of given binary
    - Does it start with / ?
    - Does it contain `AT_FDCWD` (which we passed earlier) describing given executable binary is interpreted relative to the current working directory of the calling process.

- Set `bprm->interp` with the same filename. This is updated later with the real program interpreter. This depends on the binary format of the program.

- Call `bprm_mm_init` which initializes `mm_struct` structure representing address space of a process (memory management is a lengthy topic in linux so let's leave it there).

- Calculate the count of command line arguments, count of environment variables and set it to `bprm->argc` and `bprm->envc` respectively checking `MAX_ARGS_STRINGS`.

- Call `prepare_binprm` which fills `uid` from inode and reads `128` bytes from the executable file to check the type of the executable program. The rest of the program is read later.

- Copy the filename of the executable binary file, command line arguments and environment variables to the `linux_bprm` with the call of the `copy_strings_kerne`l function.

- Set the pointer to the top of new program's stack that we set in the `bprm_mm_init` function:

    ```
    bprm->exec = bprm->p;
    ```
- Call `exec_bprm`

## Executing The Program (exec_bprm)
- Store the `pid` and the `pid` seen from the namespace of the current task.
- Call `search_binary_handler`. There are different binary handles in the linux kernel:
   * `binfmt_script` - support for interpreted scripts that are starts from the [#!](https://en.wikipedia.org/wiki/Shebang_%28Unix%29) line;
   * `binfmt_misc` - support different binary formats, according to runtime configuration of the Linux kernel;
   * `binfmt_elf` - support [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) format;
   * `binfmt_aout` - support [a.out](https://en.wikipedia.org/wiki/A.out) format;
   * `binfmt_flat` - support for [flat](https://en.wikipedia.org/wiki/Binary_file#Structure) format;
   * `binfmt_elf_fdpic` - Support for [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) [FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF) binaries;
   * `binfmt_em86` - support for Intel [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) binaries running on [Alpha](https://en.wikipedia.org/wiki/DEC_Alpha) machines.

- `search_binary_handler` tries each of the handlers with `load_binary` call and passes `bprm` structure. If given handler supports the program, it starts to prepare the executable binary for execution. In our case, we will look at ELF.

- `binfmt_elf` checks the ELF magic number. Remember this information is available already in `bprm` structure as `128` bytes were read beforehand.

## ELF Loading and Starting Thread
- `load_elf_binary` checks the architecture and type of the executable file
- Load program header table
- Read program interpreter (`.interp` section) and libraries linked with the executable file from the disk and load them into memory
- Setup the stack for the program and map into a correct location
- Map `.bss` and `brk` section and do many other things.
- At the end, call `start_thread` and pass 3 arguments.
   * Set of [registers](https://en.wikipedia.org/wiki/Processor_register) for the new task;
   * Address of the entry point of the new task;
   * Address of the top of the stack for the new task.

- `start_thread` calls `start_thread_common` which fills:
    - `fs` segment register with zero
    - `es` and ds with the value of the data segment register
    - sets new values to the `instruction pointer`, `cs` segments etc
    - Call `force_iret` macro that force a system call return via iret instruction

With the creation of a new thread to be run in userspace, now we can return from `exec_binprm` and we're in `do_execveat_common` again. After the `exec_binprm` finishes its execution, memory for structures that were allocated before were released and returns.

After we returned from the execve system call handler, execution of our program will be started. We can do it, because **all context related information already configured for this purpose. As we saw the `execve` system call does not return control to a process, but code, data and other segments of the caller process are just overwritten of the program segments.** Remember, this is a forked process from bash. Basically, this is the whole fork()/execv() overview.

The exit from our application will be implemented through the exit system call.


# Links
[How does the Linux kernel run a program](https://0xax.gitbooks.io/linux-insides/content/SysCall/syscall-4.html) by 0xax

[Anatomy of a Progmam in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/) by Gustavo Duarte

[Anatomy of Linux process management](http://www.osnews.com/story/9691/Anatomy_of_the_Linux_boot_process) by IBM Developer Works
