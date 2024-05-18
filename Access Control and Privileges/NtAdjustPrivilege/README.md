# Description

**`NtAdjustTokenPrivileges`** is a function in the Windows Native API used to adjust the privileges in a specified access token. This function is part of the low-level system services that provide more direct access to the underlying operations of the Windows operating system.

# Proof of Concept

This sample code dynamically loads system functions from **`ntdll.dll`** to open a process token and adjust specified privileges using the low-level NT API functions.

```c
#include <windows.h>
#include <winternl.h>
#include <iostream>
#include <vector>
#include <ntstatus.h>

// Define the function signature for NtAdjustPrivilegesToken
typedef NTSTATUS(NTAPI* PFN_NtAdjustPrivilegesToken)(
    HANDLE TokenHandle,
    BOOLEAN DisableAllPrivileges,
    PTOKEN_PRIVILEGES NewState,
    ULONG BufferLength,
    PTOKEN_PRIVILEGES PreviousState,
    PULONG ReturnLength);

// Define the function signature for NtOpenProcessToken
typedef NTSTATUS(NTAPI* PFN_NtOpenProcessToken)(
    HANDLE ProcessHandle,
    ACCESS_MASK DesiredAccess,
    PHANDLE TokenHandle);

// Utility function to enable multiple privileges in the token
bool EnablePrivileges(HANDLE tokenHandle, const std::vector<std::wstring>& privileges) {
    // Initialize structure to zero privilege count
    TOKEN_PRIVILEGES tp;
    tp.PrivilegeCount = 0;
    std::vector<LUID> luids(privileges.size());

    // Retrieve LUID for each privilege and populate TOKEN_PRIVILEGES structure
    for (size_t i = 0; i < privileges.size(); ++i) {
        if (!LookupPrivilegeValue(nullptr, privileges[i].c_str(), &luids[i])) {
            std::wcerr << L"LookupPrivilegeValue failed for " << privileges[i] << L". Error: " << GetLastError() << std::endl;
            return false;
        }
        tp.Privileges[tp.PrivilegeCount].Luid = luids[i];
        tp.Privileges[tp.PrivilegeCount].Attributes = SE_PRIVILEGE_ENABLED;
        tp.PrivilegeCount++;
    }

    // Load ntdll.dll to access NtAdjustPrivilegesToken
    HMODULE hNtdll = GetModuleHandle(L"ntdll.dll");
    if (!hNtdll) {
        std::wcerr << L"Failed to load ntdll.dll." << std::endl;
        return false;
    }

    // Get the function pointer to NtAdjustPrivilegesToken
    PFN_NtAdjustPrivilegesToken pNtAdjustPrivilegesToken = (PFN_NtAdjustPrivilegesToken)GetProcAddress(hNtdll, "NtAdjustPrivilegesToken");
    if (!pNtAdjustPrivilegesToken) {
        std::wcerr << L"Failed to get NtAdjustPrivilegesToken function." << std::endl;
        return false;
    }

    // Call NtAdjustPrivilegesToken to adjust the privileges
    NTSTATUS status = pNtAdjustPrivilegesToken(tokenHandle, FALSE, &tp, sizeof(TOKEN_PRIVILEGES), nullptr, nullptr);
    if (status != STATUS_SUCCESS) {
        std::wcerr << L"NtAdjustPrivilegesToken failed. NTSTATUS: " << status << std::endl;
        return false;
    }

    return true;
}

int main() {
    HANDLE tokenHandle;

    // Load NtOpenProcessToken from ntdll.dll
    PFN_NtOpenProcessToken pNtOpenProcessToken = (PFN_NtOpenProcessToken)GetProcAddress(GetModuleHandle(L"ntdll.dll"), "NtOpenProcessToken");
    if (!pNtOpenProcessToken) {
        std::wcerr << L"Failed to get NtOpenProcessToken function." << std::endl;
        return 1;
    }

    // Open the current process's token with TOKEN_ADJUST_PRIVILEGES
    NTSTATUS status = pNtOpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES, &tokenHandle);
    if (status != STATUS_SUCCESS) {
        std::wcerr << L"Failed to open process token. NTSTATUS: " << status << std::endl;
        return 1;
    }

    // Define privileges to enable
    std::vector<std::wstring> privileges = { SE_DEBUG_NAME, SE_TAKE_OWNERSHIP_NAME };
    if (EnablePrivileges(tokenHandle, privileges)) {
        std::wcout << L"Successfully enabled the requested privileges." << std::endl;
    }
    else {
        std::wcout << L"Failed to enable one or more privileges." << std::endl;
    }

    // Always close handles opened
    CloseHandle(tokenHandle);
    return 0;
}
```

When we are executing this code, the following two privileges are enabled to our current process:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/c5d375fd-4105-4f2b-ad1e-1d5e4d6308b3)


