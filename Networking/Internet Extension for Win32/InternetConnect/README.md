# Summary

**`InternetConnect`** is a function from the WinINet API in Windows that establishes a connection to a specific Internet service. The connection it establishes can then be used for operations like sending requests to a server, uploading data, or downloading files.

```
HINTERNET InternetConnect(
  HINTERNET hInternet,
  LPCTSTR   lpszServerName,
  INTERNET_PORT nServerPort,
  LPCTSTR   lpszUsername,
  LPCTSTR   lpszPassword,
  DWORD     dwService,
  DWORD     dwFlags,
  DWORD_PTR dwContext
);
```

# Proof of Concept

This POC demonstrates making a simple HTTP GET request to "example.com" using the WinINet API to establish a connection, send the request, and read the response.

```c
#include <windows.h>
#include <wininet.h>
#include <iostream>

#pragma comment(lib, "wininet.lib")

int main() {
    // Initialize WinINet
    HINTERNET hInternet = InternetOpen(L"WinInet", INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, 0);
    if (hInternet == NULL) {
        std::cerr << "InternetOpen failed: " << GetLastError() << std::endl;
        return 1;
    }

    // Connect to the HTTP server
    HINTERNET hConnect = InternetConnect(hInternet, L"example.com", INTERNET_DEFAULT_HTTP_PORT, NULL, NULL, INTERNET_SERVICE_HTTP, 0, 0);
    if (hConnect == NULL) {
        std::cerr << "InternetConnect failed: " << GetLastError() << std::endl;
        InternetCloseHandle(hInternet);
        return 1;
    }

    // Open an HTTP request
    HINTERNET hRequest = HttpOpenRequest(hConnect, L"GET", L"/", NULL, NULL, NULL, 0, 0);
    if (hRequest == NULL) {
        std::cerr << "HttpOpenRequest failed: " << GetLastError() << std::endl;
        InternetCloseHandle(hConnect);
        InternetCloseHandle(hInternet);
        return 1;
    }

    // Send the request
    BOOL result = HttpSendRequest(hRequest, NULL, 0, NULL, 0);
    if (!result) {
        std::cerr << "HttpSendRequest failed: " << GetLastError() << std::endl;
        InternetCloseHandle(hRequest);
        InternetCloseHandle(hConnect);
        InternetCloseHandle(hInternet);
        return 1;
    }

    // Read the response
    char buffer[4096];
    DWORD bytesRead;
    while (InternetReadFile(hRequest, buffer, sizeof(buffer), &bytesRead) && bytesRead > 0) {
        std::cout.write(buffer, bytesRead);
    }

    // Close handles
    InternetCloseHandle(hRequest);
    InternetCloseHandle(hConnect);
    InternetCloseHandle(hInternet);

    return 0;
}
```

This is the result that we're getting:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/fe94efe9-88b3-4d39-a0ac-f8bef0e42f2d)

