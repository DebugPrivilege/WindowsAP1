# Description

**`ZwTerminateProcess`** is a function in the Windows Native API, part of the **`ntdll.dll`** library, which allows for the termination of a specified process. This function is the low-level counterpart to the Win32 API function **`TerminateProcess`**, but it operates at a closer level to the kernel.

```
NTSYSAPI NTSTATUS ZwTerminateProcess(
  [in, optional] HANDLE   ProcessHandle,
  [in]           NTSTATUS ExitStatus
);
```

# Proof of Concept

This POC demonstrates how to enumerate running processes to find a process by name, and then terminate it using the Native API function **`ZwTerminateProcess`**. Dynamic linking is employed here through the use of the **`GetProcAddress`** and **`GetModuleHandle`** functions to obtain a pointer to the **`ZwTerminateProcess`** function from the **`ntdll.dll`** library at runtime.

```c
#include <iostream>
#include <windows.h>
#include <TlHelp32.h>
#include <vector>
#include <string>

typedef NTSTATUS(WINAPI* ZwTerminateProcessFn)(HANDLE ProcessHandle, NTSTATUS ExitStatus);

struct ProcessInfo {
    DWORD processPID = 0;
    std::wstring processName;
};

std::vector<ProcessInfo> getAllProcessInfo()
{
    std::vector<ProcessInfo> processInfos;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    if (snapshot == INVALID_HANDLE_VALUE) {
        std::cerr << "Error creating snapshot: " << GetLastError() << std::endl;
        return processInfos;
    }

    PROCESSENTRY32 processEntry = { sizeof(PROCESSENTRY32) };
    if (Process32First(snapshot, &processEntry)) {
        while (Process32Next(snapshot, &processEntry)) {
            ProcessInfo info;
            info.processPID = processEntry.th32ProcessID;
            info.processName = processEntry.szExeFile;
            processInfos.push_back(info);
        }
    }
    else {
        std::cerr << "Error getting process information: " << GetLastError() << std::endl;
    }

    CloseHandle(snapshot);
    return processInfos;
}

int main(int argc, char** argv)
{
    if (argc != 2) {
        std::cerr << "Usage: " << argv[0] << " <process name>" << std::endl;
        return 1;
    }

    std::wstring targetProcessName = std::wstring(argv[1], argv[1] + strlen(argv[1]));
    std::vector<ProcessInfo> processInfos = getAllProcessInfo();
    bool found = false;
    DWORD processPID = 0;
    for (const auto& info : processInfos)
    {
        if (info.processName == targetProcessName) {
            found = true;
            processPID = info.processPID;
            break;
        }
    }

    if (!found) {
        std::cerr << "Error: process " << argv[1] << " not found" << std::endl;
        return 1;
    }

    HANDLE processHandle = OpenProcess(PROCESS_TERMINATE, FALSE, processPID);
    if (processHandle == NULL) {
        std::cerr << "Error opening process: " << GetLastError() << std::endl;
        return 1;
    }

    ZwTerminateProcessFn ZwTerminateProcess = (ZwTerminateProcessFn)GetProcAddress(GetModuleHandle(L"ntdll.dll"), "ZwTerminateProcess");
    if (!ZwTerminateProcess) {
        std::cerr << "Error getting ZwTerminateProcess: " << GetLastError() << std::endl;
        return 1;
    }

    NTSTATUS status = ZwTerminateProcess(processHandle, 0);
    if (status != 0) {
        std::cerr << "Error terminating process with ZwTerminateProcess: " << status << std::endl;
        return 1;
    }

    std::wcout << "Process " << targetProcessName << " with PID " << processPID << " terminated successfully." << std::endl;

    CloseHandle(processHandle);
    return 0;
}
```

Here we can see that we used the **`ZwTerminateProcess`** function to terminate notepad.exe

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/056f1ba1-9921-428d-84eb-ff442c389b86)


