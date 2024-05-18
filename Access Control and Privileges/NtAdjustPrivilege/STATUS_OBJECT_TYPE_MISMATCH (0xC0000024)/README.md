# Description

If we don't call **`NtOpenProcessToken`** in our code where we need to adjust privileges, the code will not work because we lack a valid handle to the access token of the process.  Every process in Windows has an associated access token that stores security and user information, including the privileges. 

To modify the privileges of a process, we first must obtain a handle to this token using something like **`NtOpenProcessToken`**. Without invoking one of these functions, we do not have a handle to the process's token, which will lead to an error.

# Proof of Concept

This sample code dynamically loads the **`NtAdjustPrivilegesToken`** function to attempt enabling specified privileges for an access token, although it initializes with an invalid token handle, causing the privilege adjustments to fail.

```c
#include <windows.h>
#include <winternl.h>
#include <iostream>
#include <vector>
#include <sstream>
#include <iomanip>
#include <ntstatus.h>

// Define the function signature for NtAdjustPrivilegesToken
typedef NTSTATUS(NTAPI* PFN_NtAdjustPrivilegesToken)(
    HANDLE TokenHandle,
    BOOLEAN DisableAllPrivileges,
    PTOKEN_PRIVILEGES NewState,
    ULONG BufferLength,
    PTOKEN_PRIVILEGES PreviousState,
    PULONG ReturnLength);

std::wstring ErrorCodeToNTSTATUS(DWORD errorCode) {
    std::wstringstream ss;
    ss << L"0x"
        << std::uppercase
        << std::setw(8)
        << std::setfill(L'0')
        << std::hex
        << errorCode;
    return ss.str();
}

bool EnablePrivileges(HANDLE tokenHandle, const std::vector<std::wstring>& privileges) {
    TOKEN_PRIVILEGES tp;
    tp.PrivilegeCount = 0;
    std::vector<LUID> luids(privileges.size());

    for (size_t i = 0; i < privileges.size(); ++i) {
        if (!LookupPrivilegeValue(nullptr, privileges[i].c_str(), &luids[i])) {
            std::wcerr << L"LookupPrivilegeValue failed for " << privileges[i] << L". Error: "
                << ErrorCodeToNTSTATUS(GetLastError()) << std::endl;
            return false;
        }
        tp.Privileges[tp.PrivilegeCount].Luid = luids[i];
        tp.Privileges[tp.PrivilegeCount].Attributes = SE_PRIVILEGE_ENABLED;
        tp.PrivilegeCount++;
    }

    HMODULE hNtdll = GetModuleHandle(L"ntdll.dll");
    PFN_NtAdjustPrivilegesToken pNtAdjustPrivilegesToken = (PFN_NtAdjustPrivilegesToken)GetProcAddress(hNtdll, "NtAdjustPrivilegesToken");
    if (!pNtAdjustPrivilegesToken) {
        std::wcerr << L"Failed to get NtAdjustPrivilegesToken function." << std::endl;
        return false;
    }

    NTSTATUS status = pNtAdjustPrivilegesToken(tokenHandle, FALSE, &tp, sizeof(TOKEN_PRIVILEGES), nullptr, nullptr);
    if (status != STATUS_SUCCESS) {
        std::wcerr << L"NtAdjustPrivilegesToken failed. NTSTATUS: " << ErrorCodeToNTSTATUS(status) << std::endl;
        return false;
    }

    return true;
}

int main() {
    HANDLE tokenHandle = INVALID_HANDLE_VALUE;

    std::vector<std::wstring> privileges = { SE_DEBUG_NAME, SE_TAKE_OWNERSHIP_NAME };
    if (EnablePrivileges(tokenHandle, privileges)) {
        std::wcout << L"Successfully enabled the requested privileges." << std::endl;
    }
    else {
        std::wcout << L"Failed to enable one or more privileges." << std::endl;
    }

    return 0;
}
```

This is how it looks like when we try to execute this code:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bd19d44d-1a09-480f-addc-08518d441a46)

