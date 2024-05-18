# Description

**`GetComputerName`** is a function of **`kernel32.dll`** that retrieves the NetBIOS name of the local computer on which the application is running. This name is established at system startup, when the system reads it from the registry.

```
BOOL GetComputerName(
  LPWSTR  lpBuffer, // Contains the name of the computer.
  LPDWORD lpnSize //  Receives the actual size of the computer name
);
```

# Proof of Concept

This sample code retrieves the name of the local computer using the Windows API GetComputerNameW function and prints it to the console.

```c
#include <windows.h>
#include <iostream>
#include <vector>

int main() {
    // MAX_COMPUTERNAME_LENGTH is defined in WinDef.h and represents the maximum length for a computer name in Windows.
    // Adding 1 for the null terminator.
    std::vector<wchar_t> name(MAX_COMPUTERNAME_LENGTH + 1);
    DWORD size = MAX_COMPUTERNAME_LENGTH + 1;

    if (GetComputerNameW(name.data(), &size)) {
        // Using std::wcout for wide character output.
        std::wcout << L"Computer Name: " << name.data() << std::endl;
    }
    else {
        std::cerr << "Failed to get computer name, error code: " << GetLastError() << std::endl;
    }

    return 0;
}
```

Here is the result:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/0c433bbe-6904-4ec9-89f1-39a8c4442e20)

