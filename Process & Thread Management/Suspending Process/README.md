# Summary

**`NtSuspendProcess`** is an undocumented function from the **`ntdll.dll`** library in Windows operating systems. This function allows a process to be suspended.

# Proof of Concept

This POC demonstrates how to suspend a specific process identified by the user, utilizing an undocumented function **`NtSuspendProcess`** from **`ntdll.dll`**

```c
#include <Windows.h>
#include <TlHelp32.h>
#include <iostream>
#include <string>

// Define a function pointer type for NtSuspendProcess, an undocumented function.
typedef LONG(NTAPI* NtSuspendProcess)(IN HANDLE ProcessHandle);

// Function to suspend a process given its process ID.
void SuspendProcess(DWORD processId)
{
    // Open the process with all access rights.
    HANDLE processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processId);
    if (processHandle == NULL)
    {
        std::cout << "OpenProcess failed. Error code: " << GetLastError() << std::endl;
        return;
    }

    // Dynamically locate the address of NtSuspendProcess in ntdll.dll.
    NtSuspendProcess pfnNtSuspendProcess = (NtSuspendProcess)GetProcAddress(
        GetModuleHandle(L"ntdll"), "NtSuspendProcess");
    if (pfnNtSuspendProcess == NULL)
    {
        std::cout << "GetProcAddress failed. Error code: " << GetLastError() << std::endl;
        CloseHandle(processHandle);
        return;
    }

    // Call the function to suspend the process.
    LONG status = pfnNtSuspendProcess(processHandle);
    if (status != 0)
    {
        std::cout << "NtSuspendProcess failed. Error code: " << status << std::endl;
    }
    CloseHandle(processHandle);
}

int main()
{
    DWORD processId = 0;
    std::wstring processName;

    // Prompt the user to enter the name of the process to suspend.
    std::wcout << L"Enter the name of the process: ";
    std::getline(std::wcin, processName);

    // Create a snapshot of all processes in the system.
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (hSnapshot == INVALID_HANDLE_VALUE)
    {
        std::cout << "CreateToolhelp32Snapshot failed. Error code: " << GetLastError() << std::endl;
        return 1;
    }

    // Iterate through the snapshot to find the process by name.
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(PROCESSENTRY32);

    if (Process32First(hSnapshot, &pe))
    {
        do
        {
            // If the process is found, store its process ID.
            if (!_wcsicmp(pe.szExeFile, processName.c_str()))
            {
                processId = pe.th32ProcessID;
                break;
            }
        } while (Process32Next(hSnapshot, &pe));
    }
    CloseHandle(hSnapshot);

    // If the process was not found, inform the user.
    if (!processId)
    {
        std::wcout << processName << L" not found." << std::endl;
        return 1;
    }

    // Suspend the process using its process ID.
    SuspendProcess(processId);

    std::cout << "The specified process has been suspended." << std::endl;
    return 0;
}
```

In this example, we are suspending all the threads from the **notepad.exe** process.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/6ed89e75-1c06-4deb-a7d7-c0ff0ab04db2)

We can now use tools like **Process Explorer** to examine whether the threads that resides within the specified process have been suspended:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/fe7cb21e-fd6f-4180-ae5b-964314a05ba4)

