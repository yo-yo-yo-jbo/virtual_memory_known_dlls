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

How is this achieved? The idea is that memory addresses (like `0x560fe8d872a0`) do not represent RAM; in fact, your RAM might not be big enough to save all memory for all programs running simultanously! Instead, memory is split into chunks called *pages* (traditionally of size 4K, but there are already modern operating systems using a different number). Each page can either reside in RAM (physical memory) or on the hard drive. Then:
1. When addressing an address, its page entry is checked (an address is essentially a page + offset). There are translation schemes in the kernel between virtual page addresses and physical memory.
2. If that translation indicates the page resides in physical memory, it's simply used.
3. Otherwise, that page resides in hard-drive, which means it has to be fetched. "Discovering" a page referenced by a program does not reside in RAM is called a `Page Fault`, and fetching it from hard drive into physical memory is called `Paging-in`.
4. If the RAM is full, it means another page needs to be removed from RAM into the hard drive to make room for the newly fetched page. This is called `Paging-out`.
5. There are sophisciated algorithms to determine which pages are supposed to be paged-out first; most of them rely on paging out the `Least Recently Used (LRU)` pages; you can read more [here](https://en.wikipedia.org/wiki/Cache_replacement_policies).

