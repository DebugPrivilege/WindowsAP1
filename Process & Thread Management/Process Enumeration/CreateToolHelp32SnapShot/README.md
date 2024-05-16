# Summary

**`CreateToolhelp32Snapshot`** is a Windows API function that captures a snapshot of certain types of system information, depending on the parameters passed to it. This function is part of the Tool Help library and is commonly used for enumerating processes, threads, modules, and heaps within the system at a specific point in time. This function  is closely integrated with other APIs within the same library, such as **`Process32First`** and **`Process32Next`** to work properly. 

The parameters of this function:

```
HANDLE CreateToolhelp32Snapshot(
  DWORD dwFlags,
  DWORD th32ProcessID
);
```

The **`dwFlags`** parameter supports the following values:

- **`TH32CS_SNAPHEAPLIST`** Includes all heaps of the process specified in th32ProcessID in the snapshot.
- **`TH32CS_SNAPMODULE`** Includes all modules of the process specified in th32ProcessID in the snapshot. 
- **`TH32CS_SNAPPROCESS`** Includes all processes in the system in the snapshot.
- **`TH32CS_SNAPTHREAD`** Includes all threads in the system in the snapshot.

# Proof of Concept

This POC captures a snapshot of all currently running processes on the system using the **`CreateToolhelp32Snapshot`** function, then iterates through this snapshot to collect each process's ID and name.

```c
#include <iostream>
#include <windows.h>
#include <TlHelp32.h>
#include <vector>
#include <iomanip>

struct ProcessInfo {
    DWORD processPID;
    std::wstring processName;
};

std::vector<ProcessInfo> GetAllProcessInfo() {
    std::vector<ProcessInfo> processInfos;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    if (snapshot != INVALID_HANDLE_VALUE) {
        PROCESSENTRY32 processEntry = { sizeof(processEntry) };
        if (Process32First(snapshot, &processEntry)) {
            do {
                processInfos.push_back({ processEntry.th32ProcessID, processEntry.szExeFile });
            } while (Process32Next(snapshot, &processEntry));
        }
        else {
            std::cerr << "Failed to retrieve process information: " << GetLastError() << std::endl;
        }

        CloseHandle(snapshot);
    }
    else {
        std::cerr << "Failed to create snapshot: " << GetLastError() << std::endl;
    }

    return processInfos;
}

int main() {
    auto processInfos = GetAllProcessInfo();

    std::wcout << std::left << std::setw(10) << L"PID" << L"Process Name" << std::endl;
    std::wcout << std::wstring(60, L'-') << std::endl; 

    for (const auto& info : processInfos) {
        std::wcout << std::setw(10) << info.processPID << info.processName << std::endl;
    }

    return 0;
}
```

In this screenshot, we can see that we have used this API to enumerate a list of running processes:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/1f43167b-55e0-4722-8e5f-01765719e1d2)
