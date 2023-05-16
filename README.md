# Virtual memory and KnownDlls




## Introduction to memory
[Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory) is a concept every programmer must know.  
The idea supports the concept of program address spaces - each program "believes" it has the entire memory to itself.  
This is also why two programs that run in parallel might have different values for the same address (POSIX):

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>

int main()
{
        uint32_t* addr = NULL;

        // Allocate one integer
        addr = malloc(sizeof(*addr));
        if (NULL == addr)
        {
                printf("Allocation error\n");
                goto cleanup;
        }

        // Fork and indicate
        *addr = fork();
        printf("[%p] = 0x%.8x\n", addr, *addr);
        fflush(stdout);

cleanup:

        // Free resources
        if (NULL != addr)
        {
                free(addr);
                addr = NULL;
        }

        // Return
        return 0;
}
```

When running, you might get an output looking like this:
```shell
[0x560fe8d872a0] = 0x00000060
[0x560fe8d872a0] = 0x00000000
```

This shows the same address (`0x560fe8d872a0` in my run, but you might get a different address) having two different values in two different processes (we've created a new process with the `fork` system call).
