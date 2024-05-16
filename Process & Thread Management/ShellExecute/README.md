# Summary

**`ShellExecute`** is a function provided by the Windows API and originating from **`shellapi.h`** that is used to open a document or launch an executable. The most common use of **`ShellExecute`** is to open files using their associated application or to run executable files.

```
HINSTANCE ShellExecute(
  HWND hwnd,         // Parent window handle
  LPCSTR lpOperation, // Operation to perform ("open", "print", etc.)
  LPCSTR lpFile,     // File on which to perform the operation
  LPCSTR lpParameters, // Parameters to pass to the executable
  LPCSTR lpDirectory, // Default directory
  INT nShowCmd       // How to display the application window
);
```

The **`lpOperation`** parameter of the **`ShellExecute`** function specifies the action to be performed on the file specified by the **`lpFile`** parameter.

| lpOperation | Description                                                                                                         |
|---------------------|---------------------------------------------------------------------------------------------------------------------|
| `open`            | Opens the file or application. For documents, opens with the associated app; for executables, runs the executable. |
| `runas`           | Launches an application with administrative privileges, prompting for confirmation or an admin password.           |
| `edit`            | Opens the file for editing with its associated application, if supported.                                          |
| `explore`         | Opens Explorer with the specified folder displayed.                                                                 |
| `find`            | Opens the search results window for the specified directory.                                                        |
| `print`           | Prints the document using its associated application that supports printing.                                        |
| `properties`      | Displays the properties dialog box for the file or folder.                                                          |
| `NULL`            | Performs the default operation for the file type, typically the same as `"open"`.                                  |

The **`nShowCmd`** parameter of **`ShellExecute`** determines how the application window will be shown (e.g., minimized, maximized, hidden). It is used to control the visibility and state of the window of the launched application.

| Value               | Description                                                   | Hexadecimal |
|---------------------|---------------------------------------------------------------|-------------|
| `SW_HIDE`           | Hides the window and activates another window.                | 0x0         |
| `SW_SHOWNORMAL`     | Activates and displays a window.                              | 0x1         |
| `SW_NORMAL`         | Same as `SW_SHOWNORMAL`.                                      | 0x1         |
| `SW_SHOWMINIMIZED`  | Activates the window and displays it as a minimized window.   | 0x2         |
| `SW_SHOWMAXIMIZED`  | Activates the window and displays it as a maximized window.   | 0x3         |
| `SW_MAXIMIZE`       | Maximizes the specified window.                               | 0x3         |
| `SW_SHOWNOACTIVATE` | Displays a window in its most recent size and position.       | 0x4         |
| `SW_SHOW`           | Activates the window and displays it in its current size.     | 0x5         |
| `SW_MINIMIZE`       | Minimizes the specified window and activates the next window. | 0x6         |
| `SW_SHOWMINNOACTIVE`| Displays the window as a minimized window without activating. | 0x7         |
| `SW_SHOWNA`         | Displays the window in its current state.                     | 0x8         |
| `SW_RESTORE`        | Activates and displays the window.                            | 0x9         |
| `SW_SHOWDEFAULT`    | Sets the show state based on the `STARTUPINFO` structure.     | 0x10        |
| `SW_FORCEMINIMIZE`  | Minimizes a window, even if the thread is not responding.     | 0x11        |

# Proof of Concept

This code attempts to launch PsExec64.exe with specific arguments using **`ShellExecute`** with administrative privileges.

```c
#include <windows.h>
#include <iostream>
#include <shellapi.h> // Include for ShellExecute

int main() {
    // The full command to be run, including the path to PsExec64.exe and its arguments
    LPCWSTR applicationPath = L"C:\\Users\\User\\Desktop\\PsExec64.exe";
    LPCWSTR parameters = L"-s -i cmd.exe";

    // Use ShellExecute to launch PsExec64.exe with parameters
    HINSTANCE result = ShellExecute(NULL, L"runas", applicationPath, parameters, NULL, SW_HIDE);

    // Check if ShellExecute succeeded
    if ((int)result <= 32) {
        // ShellExecute returns a value greater than 32 if successful
        std::cerr << "ShellExecute failed with error code: " << (int)result << std::endl;
        return -1;
    }

    return 0;
}
```

Here we will receive an UAC prompt that is requesting to run PsExec with elevated privileges:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/0c1d6d91-ff3b-4ced-adf9-539766559c11)


This is the command prompt that is running under the context of **`NT AUTHORITY\SYSTEM`**.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/746e6386-c03b-4707-b2fb-b92725a25d51)


# Proof of Concept - 2

**`open`** is the default action and executes the specified file or application using its associated program. When used with executable files, it launches the application. If **`ShellExecute`** is called with the **`open`** verb for an executable, it will run with the same privilege level as the calling process.

```c
#include <windows.h>
#include <iostream>
#include <shellapi.h> // Include for ShellExecute

int main() {
    // Directly specify cmd.exe for execution
    LPCWSTR applicationPath = L"cmd.exe";
    // Use the /C switch with cmd to run the command and then terminate
    LPCWSTR parameters = L"/C vssadmin.exe delete shadows /all /quiet";

    // Use ShellExecute to run the command with elevated privileges
    HINSTANCE result = ShellExecute(NULL, L"open", applicationPath, parameters, NULL, SW_HIDE);

    // Check if ShellExecute succeeded
    if ((int)result <= 32) {
        // ShellExecute returns a value greater than 32 if successful
        std::cerr << "ShellExecute failed with error code: " << (int)result << std::endl;
        return -1;
    }

    return 0;
}
```

After compiling this code into an executable and loading it into IDA without the PDB files, the pseudocode appears as follows:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/6fb9dac7-7073-4475-a036-2e4148d76dfe)

