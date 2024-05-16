# Summary

**`NtQuerySystemInformation`** is a function exported by the **`ntdll.dll`** library in Windows operating systems. It's part of the Native API, which provides low-level interfaces to various system information and capabilities. It is widely used by developers and system utilities to query various kinds of system information that are not directly available through the Win32 API.

```
__kernel_entry NTSTATUS NtQuerySystemInformation(
  [in]            SYSTEM_INFORMATION_CLASS SystemInformationClass,
  [in, out]       PVOID                    SystemInformation,
  [in]            ULONG                    SystemInformationLength,
  [out, optional] PULONG                   ReturnLength
);
```

**`NtQuerySystemInformation`** is a versatile function that can be used to retrieve a wide range of system information. By specifying **`SystemProcessInformation`** as the **`SystemInformationClass`**, developers can retrieve information about all processes and threads running on the system.

# Proof of Concept

This POC demonstrates how to enumerate all processes running on a Windows system by using the **`NtQuerySystemInformation`** function from the **`ntdll.dll`**

```c
// Credits goes to: https://www.mdsec.co.uk/2022/08/fourteen-ways-to-read-the-pid-for-the-local-security-authority-subsystem-service-lsass/

#include <Windows.h>
#include <winternl.h>
#include <ntstatus.h>
#include <iostream>
#include <string.h>
#include <vector>
#include <memory>
#include <iomanip>

// NtQuerySystemInformation function prototype
typedef NTSTATUS(WINAPI* PNtQuerySystemInformation)(
    SYSTEM_INFORMATION_CLASS SystemInformationClass,
    PVOID SystemInformation,
    ULONG SystemInformationLength,
    PULONG ReturnLength
    );

void EnumerateProcesses() {
    NTSTATUS status;
    DWORD length = 1024;
    std::unique_ptr<BYTE[]> processList;

    // Get the address of NtQuerySystemInformation
    PNtQuerySystemInformation NtQuerySystemInformation = (PNtQuerySystemInformation)GetProcAddress(
        GetModuleHandle(L"ntdll.dll"),
        "NtQuerySystemInformation"
    );

    if (!NtQuerySystemInformation) {
        std::cout << "GetProcAddress(NtQuerySystemInformation) failed: " << GetLastError() << std::endl;
        return;
    }

    // Allocate memory for the process list
    processList = std::make_unique<BYTE[]>(length);

    // Retrieve the list of processes
    status = NtQuerySystemInformation(
        SystemProcessInformation,
        processList.get(),
        length,
        &length
    );

    if (status == STATUS_INFO_LENGTH_MISMATCH) {
        // The buffer was too small, so allocate a new one with the correct size
        processList = std::make_unique<BYTE[]>(length);

        // Retrieve the list of processes again
        status = NtQuerySystemInformation(
            SystemProcessInformation,
            processList.get(),
            length,
            &length
        );
    }

    if (status != STATUS_SUCCESS) {
        std::cout << "NtQuerySystemInformation failed: " << status << std::endl;
        return;
    }

    // Print header
    std::wcout << L"+-------+---------------------+" << std::endl;
    std::wcout << L"| PID   | Process Name        |" << std::endl;
    std::wcout << L"+-------+---------------------+" << std::endl;

    DWORD offset = 0;
    PSYSTEM_PROCESS_INFORMATION processEntry = NULL;

    // Iterate through the list of processes
    do {
        processEntry = (PSYSTEM_PROCESS_INFORMATION)(processList.get() + offset);

        // Convert the process name to a wide string
        std::wstring processName(processEntry->ImageName.Buffer, processEntry->ImageName.Length / sizeof(wchar_t));
        DWORD processId = HandleToUlong(processEntry->UniqueProcessId);

        // Print the process name and process ID
        std::wcout << L"| " << std::setw(5) << std::left << processId << L"| ";
        std::wcout << std::setw(19) << std::left << processName << L"|" << std::endl;

        offset += processEntry->NextEntryOffset;
    } while (processEntry->NextEntryOffset);

    // Print footer
    std::wcout << L"+-------+---------------------+" << std::endl;
}

int main() {
    EnumerateProcesses();
    return 0;
}
```

In this screenshot, we were able to enumerate all the running processes via **`NtQuerySystemInformation`**:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/9851d6b2-334a-498a-8ecc-331f017f192c)

