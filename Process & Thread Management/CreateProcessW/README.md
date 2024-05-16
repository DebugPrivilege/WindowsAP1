# Summary

**`CreateProcessW`** is a function in the Windows API that is originating from **`kernelbase.dll`** and used to create a new process and its primary thread. The process can run any executable file and it provides control over how the process is created through different parameters that allow developers to specify how the new process should be launched and configured. 

Here are the parameters of this function:

```
BOOL CreateProcessW(
  [in, optional]      LPCWSTR               lpApplicationName,
  [in, out, optional] LPWSTR                lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCWSTR               lpCurrentDirectory,
  [in]                LPSTARTUPINFOW        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

The **`dwCreationFlags`** parameter of the **`CreateProcessW`** function provides control over the creation of a new process. It consists of a DWORD value that can include one or more flags. These flags can be combined using the bitwise OR operator to specify the priority class and control various aspects of the process's behavior. 

This table summarizes the various flags that can be used with the **`dwCreationFlags`** parameter in **`CreateProcessW`** to control the behavior of the process creation:

| Flag                              | Description                                                                                   | Hexadecimal Value |
|-----------------------------------|-----------------------------------------------------------------------------------------------|-------------------|
| `CREATE_BREAKAWAY_FROM_JOB`       | Allows the child process to break away from the job that its parent process is a part of.    | `0x01000000`      |
| `CREATE_DEFAULT_ERROR_MODE`       | The new process does not inherit the error mode of the calling process. Instead, the new process gets the default error mode. | `0x04000000`      |
| `CREATE_NEW_CONSOLE`              | The new process has a new console, instead of inheriting its parent's console.               | `0x00000010`      |
| `CREATE_NEW_PROCESS_GROUP`        | The new process is the root process of a new process group.                                   | `0x00000200`      |
| `CREATE_NO_WINDOW`                | The process is a console application that is being run without a console window.             | `0x08000000`      |
| `CREATE_PROTECTED_PROCESS`        | The process is to be created with a protected process. The process is created with a level of privilege that is different from the caller process. | `0x00040000`      |
| `CREATE_PRESERVE_CODE_AUTHZ_LEVEL`| Preserves the code integrity authorization level.                                             | `0x02000000`      |
| `CREATE_SEPARATE_WOW_VDM`         | The new process is to run in its own Virtual DOS Machine (VDM).                               | `0x00000800`      |
| `CREATE_SHARED_WOW_VDM`           | The new process is to share a Virtual DOS Machine (VDM) with other processes that are also created with this flag. | `0x00001000`      |
| `CREATE_SUSPENDED`                | The primary thread of the new process is created in a suspended state, and does not run until the `ResumeThread` function is called. | `0x00000004`      |
| `CREATE_UNICODE_ENVIRONMENT`      | Indicates the environment block pointed to by lpEnvironment uses Unicode characters.         | `0x00000400`      |
| `DEBUG_ONLY_THIS_PROCESS`         | The calling thread gets debugging rights over the new process.                                | `0x00000002`      |
| `DEBUG_PROCESS`                   | The calling thread gets debugging rights over the new process, as well as any process it spawns. | `0x00000001`    |
| `DETACHED_PROCESS`                | For console applications, the new process does not inherit its parent's console window.      | `0x00000008`      |
| `EXTENDED_STARTUPINFO_PRESENT`    | The process is created with an extended startup information; the `lpStartupInfo` parameter specifies a `STARTUPINFOEX` structure. | `0x00080000` |
| `INHERIT_PARENT_AFFINITY`         | The process inherits the affinity of its parent process.                                      | `0x00010000`      |


# Proof of Concept - Launch Application

Here is some sample code that launches Process Explorer application located at **`C:\Users\User\Desktop\procexp64.exe`** **without** any command line arguments, using the **`CreateProcessW`** function.

```c
#include <windows.h>
#include <iostream>

int main() {
    // The full path to the executable to be run
    LPCWSTR applicationName = L"C:\\Users\\User\\Desktop\\procexp64.exe";

    // Since lpCommandLine can be empty, we pass NULL
    LPWSTR commandLine = NULL; // No command line arguments are needed

    // Process security attributes; NULL means the process handle cannot be inherited
    LPSECURITY_ATTRIBUTES processAttributes = NULL;

    // Thread security attributes; NULL means the thread handle cannot be inherited
    LPSECURITY_ATTRIBUTES threadAttributes = NULL;

    // Handle inheritance option; FALSE means handles are not inherited
    BOOL inheritHandles = FALSE;

    // Creation flags; 0 means no special flags are set
    DWORD creationFlags = 0;

    // Environment block; NULL means the new process uses the parent's environment block
    LPVOID environment = NULL;

    // Current directory; NULL means the new process uses the parent's current directory
    LPCWSTR currentDirectory = NULL;

    // STARTUPINFO specifies startup parameters for the new process
    STARTUPINFOW startupInfo;
    ZeroMemory(&startupInfo, sizeof(startupInfo));
    startupInfo.cb = sizeof(startupInfo);

    // PROCESS_INFORMATION receives identification information about the new process
    PROCESS_INFORMATION processInformation;
    ZeroMemory(&processInformation, sizeof(processInformation));

    // Create the process
    BOOL result = CreateProcessW(
        applicationName,          // Application name (full path to the executable)
        commandLine,              // Command line (NULL, since no arguments are provided)
        processAttributes,        // Process attributes
        threadAttributes,         // Thread attributes
        inheritHandles,           // Inheritance option
        creationFlags,            // Creation flags
        environment,              // Environment block
        currentDirectory,         // Current directory
        &startupInfo,             // Pointer to STARTUPINFO structure
        &processInformation       // Pointer to PROCESS_INFORMATION structure
    );

    // Check if CreateProcessW succeeded
    if (!result) {
        // If CreateProcessW failed, print the error code
        std::cerr << "CreateProcessW failed with error code: " << GetLastError() << std::endl;
        return -1;
    }

    // Wait for the created process to finish
    WaitForSingleObject(processInformation.hProcess, INFINITE);

    // Close handles to the process and its primary thread
    CloseHandle(processInformation.hProcess);
    CloseHandle(processInformation.hThread);

    return 0;
}
```

This is how the result will look like:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/ac4b33ea-da05-4a75-aee4-1e663de9b5ce)

# Proof of Concept - Specifying Command-Line Arguments

Here is some sample code that launches PsExec with specific command-line arguments to elevate to **`NT\AUTHORITY SYSTEM`**.

```c
#include <windows.h>
#include <iostream>

int main() {
    // The full path to the executable to be run
    LPCWSTR applicationName = L"C:\\Users\\User\\Desktop\\PsExec64.exe";

    // Command line to be executed; includes arguments for PsExec64.exe
    // NOTE: Make sure there's a space
    WCHAR commandLine[] = L" -s -i cmd.exe";

    // Process security attributes; NULL means the process handle cannot be inherited
    LPSECURITY_ATTRIBUTES processAttributes = NULL;

    // Thread security attributes; NULL means the thread handle cannot be inherited
    LPSECURITY_ATTRIBUTES threadAttributes = NULL;

    // Handle inheritance option; FALSE means handles are not inherited
    BOOL inheritHandles = FALSE;

    // Creation flags; 0 means no special flags are set
    DWORD creationFlags = 0;

    // Environment block; NULL means the new process uses the parent's environment block
    LPVOID environment = NULL;

    // Current directory; NULL means the new process uses the parent's current directory
    LPCWSTR currentDirectory = NULL;

    // STARTUPINFO specifies startup parameters for the new process
    STARTUPINFOW startupInfo;
    ZeroMemory(&startupInfo, sizeof(startupInfo));
    startupInfo.cb = sizeof(startupInfo);

    // PROCESS_INFORMATION receives identification information about the new process
    PROCESS_INFORMATION processInformation;
    ZeroMemory(&processInformation, sizeof(processInformation));

    // Create the process
    BOOL result = CreateProcessW(
        applicationName,          // Application name (full path to the executable)
        commandLine,              // Command line (arguments for PsExec64.exe)
        processAttributes,        // Process attributes
        threadAttributes,         // Thread attributes
        inheritHandles,           // Inheritance option
        creationFlags,            // Creation flags
        environment,              // Environment block
        currentDirectory,         // Current directory
        &startupInfo,             // Pointer to STARTUPINFO structure
        &processInformation       // Pointer to PROCESS_INFORMATION structure
    );

    // Check if CreateProcessW succeeded
    if (!result) {
        // If CreateProcessW failed, print the error code
        std::cerr << "CreateProcessW failed with error code: " << GetLastError() << std::endl;
        return -1;
    }

    // Wait for the created process to finish
    WaitForSingleObject(processInformation.hProcess, INFINITE);

    // Close handles to the process and its primary thread
    CloseHandle(processInformation.hProcess);
    CloseHandle(processInformation.hThread);

    return 0;
}
```

Here we can see a new command prompt being launched as **`NT AUTHORITY\SYSTEM`** when calling PsExec with specific command-line arguments:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/826a11bf-c155-4f35-96d6-4ddd04854396)


# Proof of Concept - Creation Flag

This code is using the **`CreateProcessW`** function with the **`CREATE_SUSPENDED`** flag. The **`CREATE_SUSPENDED`** flag causes the process to start in a suspended state, allowing the code to perform any necessary operations (not shown here) before resuming the process with **`ResumeThread`**.

```c
#include <windows.h>
#include <iostream>

int main() {
    LPCWSTR applicationName = L"C:\\Users\\User\\Desktop\\PsExec64.exe";
    WCHAR commandLine[] = L" -s -i cmd.exe"; // Arguments for PsExec64.exe

    STARTUPINFOW startupInfo;
    ZeroMemory(&startupInfo, sizeof(startupInfo));
    startupInfo.cb = sizeof(startupInfo);

    PROCESS_INFORMATION processInformation;
    ZeroMemory(&processInformation, sizeof(processInformation));

    BOOL result = CreateProcessW(
        applicationName,          // Full path to the executable
        commandLine,              // Command line (arguments for the executable)
        NULL,                     // Process attributes
        NULL,                     // Thread attributes
        FALSE,                    // Handle inheritance option
        CREATE_SUSPENDED,         // Creation flags - create process in a suspended state
        NULL,                     // Environment block
        NULL,                     // Current directory
        &startupInfo,             // Pointer to STARTUPINFO structure
        &processInformation       // Pointer to PROCESS_INFORMATION structure
    );

    if (!result) {
        std::cerr << "CreateProcessW failed with error code: " << GetLastError() << std::endl;
        return -1;
    }

    // Resume the suspended process
    ResumeThread(processInformation.hThread);

    // Wait for the created process to finish
    WaitForSingleObject(processInformation.hProcess, INFINITE);

    // Close handles to the process and its primary thread
    CloseHandle(processInformation.hProcess);
    CloseHandle(processInformation.hThread);

    return 0;
}
```

# Pseudo code

This pseudo code shows a new created process for **`PsExec64.exe`** with specific arguments, in a suspended state, using **`CreateProcessW`**, and sets its working directory to **`C:\Temp`**. After successful creation, it resumes the process, waits for it to complete and then cleans up resources.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/f8e221a1-7d55-43c8-aa6a-950714b32e4e)

Here is another example of pseudo code that shows a series of commands being executed such as deleting shadow copies, clearing the Security event log, retrieving the system's UUID, and disabling the recovery mode for Windows. This is very common with Ransomware samples.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/fa6758f2-6718-49da-b76a-b3beaae8de87)


