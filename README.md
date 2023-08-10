
# ftrace API

- https://www.kernel.org/doc/html/next/trace/ftrace.html
- https://elixir.bootlin.com/linux/v6.4/source/include/linux/ftrace.h
- https://elixir.bootlin.com/linux/v6.4/source/kernel/trace/ftrace.c
- https://elixir.bootlin.com/linux/v6.4/source/kernel/livepatch/patch.c

# Kprobe-based Event Tracing

- https://docs.kernel.org/trace/kprobetrace.html

# Arm 64 Calling Convention

- https://docs.google.com/document/d/1LJPxLj9xIdlZghBYg-9NCe_LYMWEp5QXFjGCzbUfTR8/edit?usp=sharing

# instruction_pointer (struct pt_regs.pc)

```c
static inline unsigned long instruction_pointer(struct pt_regs *regs)
{
	return regs->pc;
}
```

- https://elixir.bootlin.com/linux/v6.4/source/arch/arm64/include/asm/ptrace.h#L367
- https://elixir.bootlin.com/linux/v6.4/source/arch/arm64/include/asm/ptrace.h#L184

# vfs_write

`ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)`

- https://elixir.bootlin.com/linux/v6.4/source/fs/read_write.c#L564

# Run time Crash Utility Attach

```
# crash/crash /home/paran/vmlinux-have-debug-symbol

> mod -s ftracehook  /home/paran/ftracehook/build/ftracehook.ko
```

# eventfd_write

`static ssize_t eventfd_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)`

- https://elixir.bootlin.com/linux/v6.4/source/fs/eventfd.c#L341
- https://elixir.bootlin.com/linux/v6.4/source/fs/eventfd.c#L272

eventfd (Inter Process Communication)

- https://man7.org/linux/man-pages/man2/eventfd.2.html

```
The following program creates an eventfd file descriptor and then
forks to create a child process.  While the parent briefly
sleeps, the child writes each of the integers supplied in the
program's command-line arguments to the eventfd file descriptor.
When the parent has finished sleeping, it reads from the eventfd
file descriptor.

The following shell session shows a sample run of the program:

   $ ./a.out 1 2 4 7 14
   Child writing 1 to efd
   Child writing 2 to efd
   Child writing 4 to efd
   Child writing 7 to efd
   Child writing 14 to efd
   Child completed write loop
   Parent about to read
   Parent read 28 (0x1c) from efd
```

```c
#include <sys/eventfd.h>
#include <unistd.h>
#include <inttypes.h>           /* Definition of PRIu64 & PRIx64 */
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>             /* Definition of uint64_t */

#define handle_error(msg) \
   do { perror(msg); exit(EXIT_FAILURE); } while (0)

int
main(int argc, char *argv[])
{
   int efd;
   uint64_t u;
   ssize_t s;

   if (argc < 2) {
       fprintf(stderr, "Usage: %s <num>...\n", argv[0]);
       exit(EXIT_FAILURE);
   }

   efd = eventfd(0, 0);
   if (efd == -1)
       handle_error("eventfd");

   switch (fork()) {
   case 0:
       for (int j = 1; j < argc; j++) {
           printf("Child writing %s to efd\n", argv[j]);
           u = strtoull(argv[j], NULL, 0);
                   /* strtoull() allows various bases */
           s = write(efd, &u, sizeof(uint64_t));
           if (s != sizeof(uint64_t))
               handle_error("write");
       }
       printf("Child completed write loop\n");

       exit(EXIT_SUCCESS);

   default:
       sleep(2);

       printf("Parent about to read\n");
       s = read(efd, &u, sizeof(uint64_t));
       if (s != sizeof(uint64_t))
           handle_error("read");
       printf("Parent read %"PRIu64" (%#"PRIx64") from efd\n", u, u);
       exit(EXIT_SUCCESS);

   case -1:
       handle_error("fork");
   }
}
```