# Description

**`GetUserName`** is a Windows API function that retrieves the name of the user associated with the current thread. It's used to obtain information about the user account that is currently running a piece of code.

```
BOOL GetUserName(
  LPWSTR  lpBuffer, // An output parameter that, on successful execution, contains the name of the user. 
  LPDWORD lpnSize
);
```

# Proof of Concept

This code sample retrieves the current user's name using the Windows API GetUserNameW function and prints it to the console.

```c
#include <windows.h>
#include <Lmcons.h>
#include <vector>
#include <iostream>

int main() {
    std::vector<wchar_t> name(UNLEN + 1);
    DWORD size = UNLEN + 1;

    if (GetUserNameW(name.data(), &size)) {
        // Using std::wcout for wide character output.
        std::wcout << L"User Name: " << name.data() << std::endl;
    }
    else {
        std::cerr << "Failed to get user name, error code: " << GetLastError() << std::endl;
    }

    return 0;
}
```

This is the result:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/c20c94bd-c7f5-4f5c-8f4b-de15d8817ab2)



