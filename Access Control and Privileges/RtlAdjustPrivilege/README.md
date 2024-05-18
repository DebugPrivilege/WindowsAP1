# Description

**`RtlAdjustPrivilege`** is part of the Windows Native API, which interfaces directly with lower-level system functionalities. Unlike the higher-level **`AdjustTokenPrivileges`** function provided by the Windows API, which requires a token handle and can manipulate multiple privileges at once, **`RtlAdjustPrivilege`** directly enables or disables a single specified privilege.

```
NTSTATUS RtlAdjustPrivilege(
    ULONG Privilege,      // The privilege to adjust
    BOOLEAN Enable,       // TRUE to enable, FALSE to disable
    BOOLEAN CurrentThread,// TRUE to affect the calling thread, FALSE for the process
    PBOOLEAN OldValue     // Optional pointer to store the previous state of the privilege
);
```

# Proof of Concept

This sample code utilizes the **`RtlAdjustPrivilege`** function to enable the SeDebugPrivilege, SeTakeOwnershipPrivilege, and SeLoadDriverPrivilege for the current process.

```c
#include <Windows.h>
#include <iostream>
#include <ntstatus.h>

// Define constants for the privileges as used by the Windows system
#define SE_SHUTDOWN_PRIVILEGE 19
#define SE_DEBUG_PRIVILEGE 20
#define SE_SYSTEMTIME_PRIVILEGE 12

// Typedef for the RtlAdjustPrivilege function pointer
typedef NTSTATUS(NTAPI* pfnRtlAdjustPrivilege)(
    ULONG Privilege,
    BOOLEAN Enable,
    BOOLEAN CurrentThread,
    PBOOLEAN WasEnabled
    );

// Global instance of the function pointer and the loaded library handle
pfnRtlAdjustPrivilege g_RtlAdjustPrivilege = nullptr;
HMODULE g_hNtdll = nullptr;

// Function to load and get the function pointer for RtlAdjustPrivilege
bool InitializePrivilegeAdjustment() {
    if (!g_hNtdll) {
        g_hNtdll = LoadLibraryW(L"ntdll.dll");
        if (g_hNtdll == nullptr) {
            std::cerr << "Failed to load ntdll.dll" << std::endl;
            return false;
        }

        g_RtlAdjustPrivilege = (pfnRtlAdjustPrivilege)GetProcAddress(g_hNtdll, "RtlAdjustPrivilege");
        if (!g_RtlAdjustPrivilege) {
            std::cerr << "Failed to get address of RtlAdjustPrivilege" << std::endl;
            FreeLibrary(g_hNtdll);
            g_hNtdll = nullptr;
            return false;
        }
    }
    return true;
}

// Function to enable a specific system privilege
BOOL EnablePrivilege(ULONG privilege) {
    BOOLEAN wasEnabled;
    if (!g_RtlAdjustPrivilege) {
        if (!InitializePrivilegeAdjustment()) {
            return FALSE;
        }
    }

    NTSTATUS status = g_RtlAdjustPrivilege(privilege, TRUE, FALSE, &wasEnabled);
    if (status != STATUS_SUCCESS) {
        std::cerr << "Failed to adjust privilege: " << privilege << ", Status: " << status << std::endl;
        return FALSE;
    }

    return TRUE;
}

int main() {
    // Initialize the privilege adjustment function
    if (!InitializePrivilegeAdjustment()) {
        return 1;  // Exit if the initialization fails
    }

    // Enable necessary privileges and check each operation
    if (EnablePrivilege(SE_SHUTDOWN_PRIVILEGE)) {
        std::cout << "Shutdown privilege enabled successfully." << std::endl;
    }
    else {
        std::cerr << "Failed to enable Shutdown privilege." << std::endl;
    }

    if (EnablePrivilege(SE_DEBUG_PRIVILEGE)) {
        std::cout << "Debug privilege enabled successfully." << std::endl;
    }
    else {
        std::cerr << "Failed to enable Debug privilege." << std::endl;
    }

    if (EnablePrivilege(SE_SYSTEMTIME_PRIVILEGE)) {
        std::cout << "System time privilege enabled successfully." << std::endl;
    }
    else {
        std::cerr << "Failed to enable System time privilege." << std::endl;
    }

    // Free the library if it's no longer needed
    if (g_hNtdll) {
        FreeLibrary(g_hNtdll);
        g_hNtdll = nullptr;
        g_RtlAdjustPrivilege = nullptr;
    }

    //return 0;
    getchar();
}
```

This is how it looks like when the code is executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/b2dc4dd9-6252-4265-9601-a529fe15bed1)


Here we can see that the privileges are enabled to the current process:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/031d15a9-661f-450c-a731-3a3450dc779e)


