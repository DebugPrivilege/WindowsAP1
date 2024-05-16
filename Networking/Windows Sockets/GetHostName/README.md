# Description

The **`gethostname`** function retrieves the standard host name for the local computer. It is a very simple function that only contains two parameters:

- **`name:`** A pointer to a buffer that will receive the host name.
- **`namelen:`** The length of the buffer.

# Proof of Concept

This code sample retrieves and displays the hostname of the local computer using the Winsock **`gethostname`** function.

```c
#include <winsock2.h>
#include <iostream>

#pragma comment(lib, "Ws2_32.lib")

int main() {
    // Initialize Winsock
    WSADATA wsaData;
    // WSAStartup is required to initiate the use of the Winsock DLL
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        // If WSAStartup fails, it returns a non-zero value and we print an error message
        std::cerr << "WSAStartup failed.\n";
        return 1;
    }

    char hostname[256]; // Buffer to store the hostname
    // gethostname fills the buffer with the local host's name
    if (gethostname(hostname, sizeof(hostname)) == 0) {
        std::cout << "Hostname: " << hostname << std::endl;
    }
    else {
        std::cerr << "Failed to get hostname.\n";
    }

    // Cleanup Winsock resources
    WSACleanup();
    return 0;
}
```

This is how it looks like when the code is executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/d11d67dc-7121-4060-b14c-774c073f6b81)


