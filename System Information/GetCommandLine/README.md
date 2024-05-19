# Description

**`GetCommandLine`** is a function that retrieves the command-line string for the current process. 

# Proof of Concept

This code obtains the command-line for the current process.

```c
#include <Windows.h>
#include <iostream>

int main() {
    LPWSTR cmdLine = GetCommandLineW();
    std::wcout << L"The command line for this process is: " << cmdLine << std::endl;
    return 0;
}
```

Here is how it looks like when the code is executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/ebaf6477-ccbb-4870-a91c-d91bbc0f215b)
