# Description

**`WTSEnumerateProcesses`** is a function provided by the Windows Terminal Services API **`(Wtsapi32.dll)`**. It's used to enumerate processes running on a terminal server or a local computer. This function can be particularly useful in Remote Desktop Services sessions to retrieve information about processes running in all sessions on the server.

```
BOOL WTSEnumerateProcesses(
  IN HANDLE hServer,
  IN DWORD Reserved,
  IN DWORD Version,
  OUT PWTS_PROCESS_INFO* ppProcessInfo,
  OUT DWORD* pCount
);
```

# Proof of Concept

This POC utilizes the **`WTSEnumerateProcesses`** function from the Windows Terminal Services API to list all processes running on the current server or workstation.

```c
#include <Windows.h>
#include <tchar.h>
#include <WtsApi32.h>
#include <iostream>
#include <iomanip>

#pragma comment(lib, "WtsApi32.lib")

void EnumerateAllProcesses() {
    PWTS_PROCESS_INFO info = nullptr;
    DWORD cnt = 0;
    BOOL Result;

    Result = WTSEnumerateProcesses(WTS_CURRENT_SERVER_HANDLE, 0, 1, &info, &cnt);

    if (!Result) {
        std::cout << "WTSEnumerateProcesses() failed : " << GetLastError() << std::endl;
        return;
    }

    std::wcout << std::left << std::setw(35) << L"Process name"
        << std::setw(12) << L"PID"
        << std::setw(15) << L"Session ID"
        << L"Username" << std::endl;
    std::wcout << std::setw(67) << std::setfill(L'-') << L"" << std::setfill(L' ') << std::endl;

    for (DWORD i = 0; i < cnt; i++) {
        LPWSTR username = NULL;
        DWORD usernameSize = 0;
        WTSQuerySessionInformation(WTS_CURRENT_SERVER_HANDLE, info[i].SessionId, WTSUserName, &username, &usernameSize);

        std::wcout << std::left << std::setw(35) << info[i].pProcessName
            << std::setw(12) << info[i].ProcessId
            << std::setw(15) << info[i].SessionId
            << username << std::endl;
        WTSFreeMemory(username);
    }
    WTSFreeMemory(info);
}

int main() {
    EnumerateAllProcesses();
    return 0;
}
```

In this screenshot, we can see all the running processes with the associated PID, Session ID, and the Username.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/7e7f44f7-8611-4ab8-b479-436bc24d657c)

