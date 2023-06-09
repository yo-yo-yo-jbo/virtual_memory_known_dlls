# Virtual memory and KnownDlls
Virtual Memory is instrumental to many native security concepts, as well as an important concept for modern operating systems.  
In this blogpost, I'd like to describe virtual memory and how it helps with performance while still enforcing cross-process security.

## Introduction to virtual memory
[Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory) is a concept every programmer must know.  
The idea supports the concept of program address spaces - each program "believes" it has the entire memory to itself.  
This is also why two programs that run in parallel might have different values for the same address.  
Here's a tiny program to demonstate this concept (I've done it for `POSIX` systems like Linux or macOS, but the concept is still true in Windows):

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

With this, we can see many benefits of virtual memory:
- Essentially supports implementation of memory seperation between processes, which is essential for security.
- Supports running multiple processes in parallel.
- Supports enforcing page security (whether pages can be read, written to, or executed from, or in short: RWX).

## Virtual Memory and KnownDlls
Having virtual memory is great not only for security reasons, but also for performance reasons.  
Let's assume we have two processes that want to share a big chunk of data; instead of copying that data to both address spaces, we can just *map* that memory. This means we have one physical copy of the data, referenced twice:

```

                              +----------------+
                              |                |
                              | Shared data    |  (Physical memory)
                              |                |
                              +----------------+
                                     |
                                     |
               ----------------------------------------------------
               |                                                  |
               | (Virtual memory address 0xFF991337)              |  (Virtual memory address 0x00AA1234)
               |                                                  |
        +----------------+                                 +----------------+
        |                |                                 |                |
        |  Process 1     |                                 |  Process 2     |
        |                |                                 |                |
        +----------------+                                 +----------------+
```

Now both processes share the same data; note that the data is mapped to *different virtual memory addresses*, so it's essential that data does not contain any pointers.  
This is used extensively by modern operating systems for performance, for example, let's examine Windows.  
On Windows, there are a set of DLLs known as `KnownDlls`. Those are configurable in the registry:

```shell
C:\>reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs"

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs
    *kernel32    REG_SZ    kernel32.dll
    _wow64cpu    REG_SZ    wow64cpu.dll
    _wowarmhw    REG_SZ    wowarmhw.dll
    _xtajit    REG_SZ    xtajit.dll
    advapi32    REG_SZ    advapi32.dll
    clbcatq    REG_SZ    clbcatq.dll
    combase    REG_SZ    combase.dll
    COMDLG32    REG_SZ    COMDLG32.dll
    coml2    REG_SZ    coml2.dll
    DifxApi    REG_SZ    difxapi.dll
    gdi32    REG_SZ    gdi32.dll
    gdiplus    REG_SZ    gdiplus.dll
    IMAGEHLP    REG_SZ    IMAGEHLP.dll
    IMM32    REG_SZ    IMM32.dll
    MSCTF    REG_SZ    MSCTF.dll
    MSVCRT    REG_SZ    MSVCRT.dll
    NORMALIZ    REG_SZ    NORMALIZ.dll
    NSI    REG_SZ    NSI.dll
    ole32    REG_SZ    ole32.dll
    OLEAUT32    REG_SZ    OLEAUT32.dll
    PSAPI    REG_SZ    PSAPI.DLL
    rpcrt4    REG_SZ    rpcrt4.dll
    sechost    REG_SZ    sechost.dll
    Setupapi    REG_SZ    Setupapi.dll
    SHCORE    REG_SZ    SHCORE.dll
    SHELL32    REG_SZ    SHELL32.dll
    SHLWAPI    REG_SZ    SHLWAPI.dll
    user32    REG_SZ    user32.dll
    WLDAP32    REG_SZ    WLDAP32.dll
    wow64    REG_SZ    wow64.dll
    wow64base    REG_SZ    wow64base.dll
    wow64con    REG_SZ    wow64con.dll
    wow64win    REG_SZ    wow64win.dll
    WS2_32    REG_SZ    WS2_32.dll
    xtajit64    REG_SZ    xtajit64.dll
```

Upon boot, this registry key will be read by the session manager (in `smss.exe`). Each DLL will be mapped into a `Section`, which is the Windows object type that represents memory-mapping. You can even view these sections with [Winobj](https://learn.microsoft.com/en-us/sysinternals/downloads/winobj) on a live system.  
When a new program starts and tries to load a DLL, the Windows Loader will first look if the requested DLL is a `KnownDll`. If it is - instead of reading the DLL contents from disk, it will simply load it from the mapped section. Since those DLLs are frequently used, it's assumed that most pages of that DLL are going to be `paged-in`, which means the operating system saved a lot of disk operations that are known to be slow, as well as saving memory (since the DLL is going to be mapped to multiple processes and only have one copy in physical memory).  

## Protecting with COW
So, `kernel32.dll` pages are mapped to all running processes essentially, with one physical copy. This raises an interesting question: what happens if one process overrides bytes in one of those shared memory pages? Let's find out!

```c
#include <stdio.h>
#include <Windows.h>

static
BOOL
GetPageProtections(
	PVOID pvAddr,
	PMEMORY_BASIC_INFORMATION ptMemInfo
)
{
	BOOL bResult = FALSE;

	// Query memory
	if (0 == VirtualQuery(pvAddr, ptMemInfo, sizeof(*ptMemInfo)))
	{
		printf("VirtualQuery() failed (LastError=%lu).\n", GetLastError());
		goto lblCleanup;
	}

	// Success
	bResult = TRUE;

lblCleanup:

	// Return result
	return bResult;
}

static
BOOL
PrintPageProtections(
	PVOID pvAddr
)
{
	BOOL bResult = FALSE;
	MEMORY_BASIC_INFORMATION tMemInfo = { 0 };

	// Print page protections
	if (!GetPageProtections(pvAddr, &tMemInfo))
	{
		printf("GetPageProtections() failed.\n");
		goto lblCleanup;
	}
	printf("prot(0x%p) = %.8x\n", tMemInfo.BaseAddress, tMemInfo.Protect);

	// Success
	bResult = TRUE;

lblCleanup:

	// Return result
	return bResult;
}

static
BOOL
MakePageRWX(
	PVOID pvAddr
)
{
	BOOL bResult = FALSE;
	MEMORY_BASIC_INFORMATION tMemInfo = { 0 };
	DWORD dwOldProt = 0;

	// Query memory
	if (!(GetPageProtections(pvAddr, &tMemInfo)))
	{
		printf("GetPageProtections() failed.\n");
		goto lblCleanup;
	}

	// Make page RWX
	if (!VirtualProtect(tMemInfo.BaseAddress, 1, PAGE_EXECUTE_READWRITE, &dwOldProt))
	{
		printf("VirtualProtect() failed (LastError=%lu).\n", GetLastError());
		goto lblCleanup;
	}

	// Success
	bResult = TRUE;

lblCleanup:

	// Return result
	return bResult;
}

int
main()
{
	BOOL bResult = FALSE;
	HMODULE hKernel32 = NULL;
	FARPROC pfnExitProcess = NULL;

	// Get kernel32 handle
	hKernel32 = GetModuleHandleW(L"kernel32.dll");
	if (NULL == hKernel32)
	{
		printf("GetModuleHandleW() failed (LastError=%lu).\n", GetLastError());
		goto lblCleanup;
	}

	// Get one function address as an example
	pfnExitProcess = GetProcAddress(hKernel32, "ExitProcess");
	if (NULL == pfnExitProcess)
	{
		printf("GetProcAddress() failed (LastError=%lu).\n", GetLastError());
		goto lblCleanup;
	}

	// Print page protections
	if (!PrintPageProtections(pfnExitProcess))
	{
		printf("PrintPageProtections() failed.\n");
		goto lblCleanup;
	}

	// Make page RWX
	if (!MakePageRWX(pfnExitProcess))
	{
		printf("MakePageRWX() failed.\n");
		goto lblCleanup;
	}

	// Print page protections
	if (!PrintPageProtections(pfnExitProcess))
	{
		printf("PrintPageProtections() failed.\n");
		goto lblCleanup;
	}

	// Patch one byte
	*((PBYTE)pfnExitProcess) ^= 1;

	// Print page protections
	if (!PrintPageProtections(pfnExitProcess))
	{
		printf("PrintPageProtections() failed.\n");
		goto lblCleanup;
	}

	// Patch one byte again
	*((PBYTE)pfnExitProcess) ^= 1;

	// Success
	bResult = TRUE;

lblCleanup:

	// Indicate result
	return bResult ? 0 : -1;
}
```

The code is quite simple:
1. The `GetPageProtections` function simply fetches page protections (in fact, the entire `MEMORY_BASIC_INFORMATION` structure) using the [VirtualQuery](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualquery) API.
2. The `PrintPageProtections` function simply calls `GetPageProtections` and prints the page base address and its page protections.
3. The `MakePageRWX` function makes a page readable, writable and executable (`RWX`).
4. The `main` function fetches the `kernel32!ExitProcess` function (which obviously resides in the `kernel32.dll` shared memory executable section), prints its page protections, makes it writable, prints its page protections, patches one byte (`XOR`ing with 1), prints its page protections again and then restores the byte and quits.

Let's run it:

```
prot(0x76D56000) = 00000020
prot(0x76D56000) = 00000080
prot(0x76D56000) = 00000040
```

Something magical has happened!
- When we started, the page protections were `0x20`.  According to the [Memory Protection Constants MSDN page](https://learn.microsoft.com/en-us/windows/win32/Memory/memory-protection-constants), this is `PAGE_EXECUTE_READ`, as expected.
- After changing the memory protections to `PAGE_EXECUTE_READWRITE` via the `MakePageRWX` function - the page protections changed to `0x80`! This is quite unexepcted since `PAGE_EXECUTE_READWRITE` is `0x40`. Well, `0x80` is `PAGE_EXECUTE_WRITECOPY`!
- After patching one byte, the page protections changed to `0x40` - the expected value of `PAGE_EXECUTE_READWRITE`.

Why did we get `PAGE_EXECUTE_WRITECOPY` and how did the page protections change after patching one byte?  
This mechanism is called `Copy-on-Write` (or `COW` for short). Let's see what MSDN says about `PAGE_EXECUTE_WRITECOPY`:

> Enables execute, read-only, or copy-on-write access to a mapped view of a file mapping object. An attempt to write to a committed copy-on-write page results in a private copy of the page being made for the process. The private page is marked as PAGE_EXECUTE_READWRITE, and the change is written to the new page.

This description is the essence of `Copy-On-Write` - sharable mapped pages (essentially `Sections` in Windows) have the `COW` bit turned on. When such a page is being written (which happens when we patch the byte), a private copy of it is created. This is an *important* security feature - without `COW`, patching `kernel32` code would affect all processes in the system!

Let's illustrate how the memory layout looks like before the byte patching (but after making it `RWX`):
```

                              +----------------------+
                              |                      |
                              | kernel32!ExitProcess | (shared memory)
                              |                      |
                              +----------------------+
                                     |
                                     |
               ----------------------------------------------------
               |                                                  |
               | (Virtual memory address 0x76D56000)              |  (Virtual memory address 0x76D56000)
               |                                                  |
        +----------------+                                 +----------------+
        |                |                                 |                |
        |  Legit process |                                 |  Patcher       |
        |                |                                 |                |
        +----------------+                                 +----------------+
```

And after:

```

        +----------------------+                             +----------------------+
        |                      |                             |                      |
        | kernel32!ExitProcess | (shared memory)             | kernel32!ExitProcess | (private copy)
        |                      |                             |                      |
        +----------------------+                             +----------------------+
               |                                                  |
               |                                                  |
               |                                                  |
               |                                                  |
               | (Virtual memory address 0x76D56000)              |  (Virtual memory address 0x76D56000)
               |                                                  |
        +----------------+                                 +----------------+
        |                |                                 |                |
        |  Legit process |                                 |  Patcher       |
        |                |                                 |                |
        +----------------+                                 +----------------+
```

With the help of virtual memory, the kernel can simply create a private copy without changing the virtual memory address in the patcher process.

## Summary
In this blogpost, we've described virtual memory, Copy-on-Write and how it's related to KnownDlls.  
I think these are important concepts for all modern operating systems (not just Windows). In fact, COW was even a source of several interesting vulnerabilities, for example [Dirty COW](https://en.wikipedia.org/wiki/Dirty_COW) that was a race-condition in the COW mechanism (along side a caching bit called "dirty bit", which justifies the vulnerability name).

Stay tuned!

Jonathan Bar Or
