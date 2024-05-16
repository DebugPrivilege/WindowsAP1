# Summary

**`CreateProcessWithLogonW`** is a Windows API function originating from **`advapi32.dll`** that allows a program to create a new process that runs under the security context of a specified user. **`CreateProcessWithLogonW`** relies on the Secondary Logon service. The Secondary Logon service is responsible for enabling and managing the creation of processes under alternative credentials. This service is for scenarios where tasks need to be performed with different permissions than the currently logged-in user, without requiring the user to log off and log back in with the different credentials.

```
BOOL CreateProcessWithLogonW(
  LPCWSTR               lpUsername,
  LPCWSTR               lpDomain,
  LPCWSTR               lpPassword,
  DWORD                 dwLogonFlags,
  LPCWSTR               lpApplicationName,
  LPWSTR                lpCommandLine,
  DWORD                 dwCreationFlags,
  LPVOID                lpEnvironment,
  LPCWSTR               lpCurrentDirectory,
  LPSTARTUPINFOW        lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation
);
```

Without the Secondary Logon service being active, it's not possible to initiate a process with different user credentials. This service is designed to handle requests for creating processes with a different security context than the one currently active. Here is an example where this function is not working because this service is disabled:

![image](https://github.com/DebugPrivilege/Win32-API/assets/63166600/b4f8cbf5-863b-47bb-a5b6-79be4e122f7b)

# Proof of Concept

This POC demonstrates how to create two separate processes, each running under different user credentials, by utilizing the **`CreateProcessWithLogonW`** function to start a command prompt with specified usernames and passwords.

```c
#include <Windows.h>
#include <iostream>
#include <string>

bool CreateProcessWithCredentials(const std::wstring& username, const std::wstring& domain, const std::wstring& password, const std::wstring& cmd)
{
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    if (!CreateProcessWithLogonW(username.c_str(), domain.c_str(), password.c_str(), LOGON_WITH_PROFILE, NULL, const_cast<LPWSTR>(cmd.c_str()), CREATE_NEW_CONSOLE, NULL, NULL, &si, &pi))
    {
        DWORD errorCode = GetLastError();
        LPWSTR errorMessage = NULL;
        FormatMessageW(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
            NULL, errorCode, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPWSTR)&errorMessage, 0, NULL);
        std::wcerr << L"CreateProcessWithLogonW failed for user " << username << L" with error code " << errorCode << L": " << errorMessage << std::endl;
        LocalFree(errorMessage);
        return false; // Return false on failure
    }

    // Optionally, close handles immediately after creation to avoid resource leaks.
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);

    return true; // Return true on success
}

int main()
{
    // Credentials and command to run
    std::wstring username1 = L"SVC_Account";
    std::wstring domain1 = L"."; // Use "." for local machine
    std::wstring password1 = L"Hunter2";
    std::wstring cmd = L"cmd.exe";

    std::wstring username2 = L"SVC_Breakglass";
    std::wstring domain2 = L"."; // Use "." for local machine
    std::wstring password2 = L"Passw0rd!";

    // Create processes with different credentials
    CreateProcessWithCredentials(username1, domain1, password1, cmd);
    CreateProcessWithCredentials(username2, domain2, password2, cmd);

    return 0;
}

```

In this scenario, a new command prompt is initiated under the credentials of a different user.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/497dff23-7330-40dc-8916-65d916dd2f18)


As previously mentioned, **`CreateProcessWithLogonW`** depends on the Secondary Logon service. To conduct an experiment, we could temporarily stop this service and then execute our POC to observe whether the service restarts automatically.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/df0a4780-bb69-471e-ab3a-daf7d5302a20)


When the POC has been executed, the service is automatically in a running state.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bd5463b5-8cfb-4caf-8289-1410f36556a5)

