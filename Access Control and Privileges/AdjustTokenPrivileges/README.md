# Description

**`AdjustTokenPrivileges`** is a function that is used to enable or disable privileges in an access token. For instance, a process might need elevated privileges to perform system-level operations, such as accessing protected system files or modifying system settings.

```
BOOL AdjustTokenPrivileges(
  HANDLE            TokenHandle,
  BOOL              DisableAllPrivileges,
  PTOKEN_PRIVILEGES NewState,
  DWORD             BufferLength,
  PTOKEN_PRIVILEGES PreviousState,
  PDWORD            ReturnLength
);
```

# Proof of Concept

This sample code utilizes the **`AdjustTokenPrivileges`** function to enable the SeDebugPrivilege, SeTakeOwnershipPrivilege, and SeLoadDriverPrivilege for the current process.

```c
#include <windows.h>
#include <iostream>

// Function to enable a privilege in the access token of the current process.
bool SetPrivilege(HANDLE hToken, LPCWSTR lpszPrivilege, BOOL bEnablePrivilege) {
    TOKEN_PRIVILEGES tp;
    LUID luid;

    if (!LookupPrivilegeValue(
        NULL,           // lookup privilege on local system
        lpszPrivilege,  // privilege to lookup 
        &luid)) {       // receives LUID of privilege
        std::wcout << L"LookupPrivilegeValue error: " << GetLastError() << L'\n';
        return false;
    }

    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    tp.Privileges[0].Attributes = (bEnablePrivilege) ? SE_PRIVILEGE_ENABLED : 0;

    // Enable the privilege or disable all privileges.
    if (!AdjustTokenPrivileges(
        hToken,
        FALSE,
        &tp,
        sizeof(TOKEN_PRIVILEGES),
        (PTOKEN_PRIVILEGES)NULL,
        (PDWORD)NULL)) {
        std::wcout << L"AdjustTokenPrivileges error: " << GetLastError() << L'\n';
        return false;
    }

    if (GetLastError() == ERROR_NOT_ALL_ASSIGNED) {
        std::wcout << L"The token does not have the specified privilege: " << lpszPrivilege << L'\n';
        return false;
    }

    return true;
}

int main() {
    HANDLE hToken;
    // Open a handle to the access token for the calling process.
    if (!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &hToken)) {
        std::wcout << L"OpenProcessToken error: " << GetLastError() << L'\n';
        return 1;
    }

    // Enable the SeDebugPrivilege.
    if (!SetPrivilege(hToken, SE_DEBUG_NAME, TRUE)) {
        std::wcout << L"Failed to enable SeDebugPrivilege.\n";
    }
    else {
        std::wcout << L"SeDebugPrivilege enabled successfully.\n";
    }

    // Enable the SeTakeOwnershipPrivilege.
    if (!SetPrivilege(hToken, SE_TAKE_OWNERSHIP_NAME, TRUE)) {
        std::wcout << L"Failed to enable SeTakeOwnershipPrivilege.\n";
    }
    else {
        std::wcout << L"SeTakeOwnershipPrivilege enabled successfully.\n";
    }

    // Enable the SeLoadDriverPrivilege.
    if (!SetPrivilege(hToken, SE_LOAD_DRIVER_NAME, TRUE)) {
        std::wcout << L"Failed to enable SeLoadDriverPrivilege.\n";
    }
    else {
        std::wcout << L"SeLoadDriverPrivilege enabled successfully.\n";
    }

    // Cleanup
    CloseHandle(hToken);
    //return 0;
    getchar();
}
```

Here we are executing the code:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/25f03d9a-ad09-424a-9afa-38b16f37e313)


Here we can see all the privileges that have been enabled:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/cf945331-fbb8-433e-af53-77654c9093b0)


