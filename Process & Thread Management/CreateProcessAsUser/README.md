# Summary

**`CreateProcessAsUser`** is a Windows API function originating from **`advapi32.dll`** that creates a new process and its primary thread. The new process runs in the security context of a specified user account, which is different from the calling process's security context. This function is particularly useful in scenarios where an application needs to run a task under the credentials of a user other than the one currently logged on or the one under which the application is running.

Here are the parameters of this function:

```
BOOL CreateProcessAsUser(
  HANDLE                hToken,
  LPCWSTR               lpApplicationName,
  LPWSTR                lpCommandLine,
  LPSECURITY_ATTRIBUTES lpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL                  bInheritHandles,
  DWORD                 dwCreationFlags,
  LPVOID                lpEnvironment,
  LPCWSTR               lpCurrentDirectory,
  LPSTARTUPINFOW        lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation
);
```


The **`LogonUser`** function originating from **`advapi32.dll`** performs a user logon operation. This function verifies the user's credentials and returns a handle to a token that represents the logged-on user. With the token obtained from **`LogonUser`**, we can then call **`CreateProcessAsUser`** to create a new process that runs under the security context of the specified user.

```
BOOL LogonUser(
  LPCWSTR lpszUsername,     // User's name
  LPCWSTR lpszDomain,       // Domain or server name, NULL for local
  LPCWSTR lpszPassword,     // User's password
  DWORD   dwLogonType,      // Type of logon operation
  DWORD   dwLogonProvider,  // Logon provider
  PHANDLE phToken           // Handle to the token that represents the logged-on user
);
```

This call stack shows how high-level API functions such as **`CreateProcessAsUserW`** eventually boil downs to a native system call **`(NtCreateUserProcess)`** to perform the actual process creation. 

```
0:000> knL
 # Child-SP          RetAddr               Call Site
00 000000d3`1ad3dbe8 00007ffc`8fa319fb     ntdll!NtCreateUserProcess
01 000000d3`1ad3dbf0 00007ffc`8fa72fb3     KERNELBASE!CreateProcessInternalW+0x23eb
02 000000d3`1ad3f590 00007ffc`90a5b250     KERNELBASE!CreateProcessAsUserW+0x63
03 000000d3`1ad3f600 00007ff7`2de51233     ADVAPI32!CreateProcessAsUserWStub+0x60
04 000000d3`1ad3f670 00007ff7`2de51500     POC!main+0x143
05 (Inline Function) --------`--------     POC!invoke_main+0x22
06 000000d3`1ad3f7a0 00007ffc`9053257d     POC!__scrt_common_main_seh+0x10c
07 000000d3`1ad3f7e0 00007ffc`9268aa58     KERNEL32!BaseThreadInitThunk+0x1d
08 000000d3`1ad3f810 00000000`00000000     ntdll!RtlUserThreadStart+0x28
```


# Proof of Concept

This POC demonstrates how to authenticate a local user using the **`LogonUser`** function and create a new process with **`CreateProcessAsUser`** running under that user's security context, without displaying a window.

```c
#include <Windows.h>
#include <UserEnv.h>
#include <iostream>

#pragma comment(lib, "Userenv.lib") // Link against Userenv library for environment block functions.

// Function to display detailed error messages
void ShowErrorMessage(DWORD errorCode) {
    LPWSTR errorMessage = nullptr;
    FormatMessage(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
        nullptr, errorCode, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPWSTR)&errorMessage, 0, nullptr);
    wprintf(L"%s\nError Code: 0x%X\n", errorMessage, errorCode);
    LocalFree(errorMessage);
}

int main() {
    HANDLE hToken = NULL;
    // Log on the local user "SVC_Account" with the specified password
    if (!LogonUser(L"SVC_Account", L".", L"Passw0rd!", LOGON32_LOGON_SERVICE, LOGON32_PROVIDER_DEFAULT, &hToken)) {
        ShowErrorMessage(GetLastError());
        return 1;
    }

    LPVOID lpEnvironment = nullptr;
    // Generate an environment block for the new process
    if (!CreateEnvironmentBlock(&lpEnvironment, hToken, FALSE)) {
        ShowErrorMessage(GetLastError());
        CloseHandle(hToken); // Clean up the token handle
        return 1;
    }

    STARTUPINFO si = {};
    PROCESS_INFORMATION pi = {};
    si.cb = sizeof(STARTUPINFO);

    WCHAR commandLine[] = L"whoami"; // The command to be executed

    // Create a new process as the user represented by hToken
    if (!CreateProcessAsUser(hToken, nullptr, commandLine, nullptr, nullptr, FALSE,
        CREATE_UNICODE_ENVIRONMENT | CREATE_NO_WINDOW, lpEnvironment, nullptr, &si, &pi)) {
        ShowErrorMessage(GetLastError());
        DestroyEnvironmentBlock(lpEnvironment); // Clean up the environment block
        CloseHandle(hToken); // Clean up the token handle
        return 1;
    }

    // Wait for the created process to finish
    WaitForSingleObject(pi.hProcess, INFINITE);

    DWORD exitCode;
    GetExitCodeProcess(pi.hProcess, &exitCode); // Get the exit code of the process
    wprintf(L"Process exited with code: %lu\n", exitCode);

    // Cleanup
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
    DestroyEnvironmentBlock(lpEnvironment);
    CloseHandle(hToken);

    return 0;
}
```

The calling process needs the **`"Replace a process level token"`** privileg when using the **`CreateProcessAsUser`** function. This privilege is required because **`CreateProcessAsUser`** assigns the primary token of the new created process, which allows the calling process to specify the security context (the user account) under which the new process will run.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bcf68f5e-a436-4536-8dfb-39c9aae59d36)

In my lab setup with Sysmon installed, we observe a log entry indicating the execution of **whoami.exe** under an alternate account:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/a3049045-3147-4ee5-95ac-9ccb06471be5)


This is how the pseudo code looks like when we compile the code and then load it into IDA:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/b623e71c-a5d7-4160-93c0-f31b5ca823b0)

