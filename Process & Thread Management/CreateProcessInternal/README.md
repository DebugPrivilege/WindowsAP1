# Summary

**`CreateProcessInternal`** is an undocumented Windows API function that is not officially exposed by Microsoft for public use. In Windows Internals, it is mentioned that both **`CreateProcess`** and **`CreateProcessAsUser`** ultimately invoke **`CreateProcessInternal`**. This API initiates the process creation from user space. Finally, it invokes **`NtCreateUserProcess`** to handle operations within the kernel space.

This call stack indicates that a process creation request in a user application (POC) initiates with **`CreateProcessW`** and progresses through **`CreateProcessInternalW`** in **`KERNELBASE`**, eventually reaching **`NtCreateUserProcess`** in **`ntdll`** to perform the actual process creation in kernel space.

```
0:000> knL
 # Child-SP          RetAddr               Call Site
00 00000045`caafd478 00007ff9`e9e719fb     ntdll!NtCreateUserProcess
01 00000045`caafd480 00007ff9`e9eb00c6     KERNELBASE!CreateProcessInternalW+0x23eb
02 00000045`caafee20 00007ff9`ec5961f4     KERNELBASE!CreateProcessW+0x66
03 00000045`caafee90 00007ff7`bec5111c     KERNEL32!CreateProcessWStub+0x54
04 (Inline Function) --------`--------     POC!ExecuteCommand+0xab
05 00000045`caafeef0 00007ff7`bec51650     POC!main+0x11c
06 (Inline Function) --------`--------     POC!invoke_main+0x22
07 00000045`caaff810 00007ff9`ec59257d     POC!__scrt_common_main_seh+0x10c
08 00000045`caaff850 00007ff9`eca2aa58     KERNEL32!BaseThreadInitThunk+0x1d
09 00000045`caaff880 00000000`00000000     ntdll!RtlUserThreadStart+0x28
```

# Proof of Concept

This code dynamically loads the **`KernelBase.dll`** to access and use the undocumented **`CreateProcessInternalW`** function to launch **`cmd.exe`** with a new console window.

```c
#include <windows.h>
#include <tchar.h>
#include <iostream>

// Define a function pointer type for CreateProcessInternalW for easier usage
typedef BOOL(WINAPI* CreateProcessInternalFunction)(
    HANDLE hToken,
    LPCTSTR lpApplicationName,
    LPTSTR lpCommandLine, 
    LPSECURITY_ATTRIBUTES lpProcessAttributes,
    LPSECURITY_ATTRIBUTES lpThreadAttributes,
    BOOL bInheritHandles,
    DWORD dwCreationFlags,
    LPVOID lpEnvironment,
    LPCTSTR lpCurrentDirectory,
    LPSTARTUPINFO lpStartupInfo,
    LPPROCESS_INFORMATION lpProcessInformation,
    PHANDLE hNewToken); // Not used in this context

int main() {
    STARTUPINFO startupInfo;
    PROCESS_INFORMATION processInfo;
    ZeroMemory(&startupInfo, sizeof(startupInfo));
    startupInfo.cb = sizeof(startupInfo);
    ZeroMemory(&processInfo, sizeof(processInfo));

    // Load KernelBase.dll to access CreateProcessInternalW
    HMODULE kernelBaseModule = LoadLibrary(_T("KernelBase.dll"));
    if (kernelBaseModule == NULL) {
        std::cout << "LoadLibrary failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Get the address of CreateProcessInternalW from KernelBase.dll
    CreateProcessInternalFunction createProcessInternal = 
        (CreateProcessInternalFunction)GetProcAddress(kernelBaseModule, "CreateProcessInternalW");
    if (createProcessInternal == NULL) {
        std::cout << "GetProcAddress failed. Error: " << GetLastError() << std::endl;
        FreeLibrary(kernelBaseModule); // Clean up loaded library on failure
        return 1;
    }

    // Define the application to launch and creation flags
    LPCTSTR applicationName = _T("C:\\Windows\\System32\\cmd.exe");
    DWORD creationFlags = CREATE_NEW_CONSOLE;

    // Execute the command without a specific command line (cmd.exe alone)
    if (!createProcessInternal(NULL, // No token modification
                               applicationName, // Full path to cmd.exe
                               NULL, // No additional command line arguments
                               NULL, // Default process security attributes
                               NULL, // Default thread security attributes
                               FALSE, // Handle inheritance option
                               creationFlags, // Creation flags
                               NULL, // Use parent's environment block
                               NULL, // Use parent's current directory
                               &startupInfo, // Pointer to STARTUPINFO structure
                               &processInfo, // Pointer to PROCESS_INFORMATION structure
                               NULL)) { // No new token is generated
        std::cout << "CreateProcessInternal failed. Error: " << GetLastError() << std::endl;
        FreeLibrary(kernelBaseModule); // Clean up loaded library on failure
        return 1;
    }

    // Wait for the launched process to finish
    WaitForSingleObject(processInfo.hProcess, INFINITE);

    // Clean up process and thread handles
    CloseHandle(processInfo.hProcess);
    CloseHandle(processInfo.hThread);

    // Free the dynamically loaded library
    FreeLibrary(kernelBaseModule);

    return 0;
}
```

In this instance, the **`CreateProcessInternal`** function successfully initiated a new **`cmd.exe`** console:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/6d8524bf-1a6c-4971-a535-41eab675ed72)


