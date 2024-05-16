# Summary

**`WinHttpConnect`** is a function from the WinHTTP (Windows HTTP Services) API, a more modern and server-oriented alternative to WinINet. WinHTTP provides developers with a HTTP client application programming interface to send requests through the HTTP protocol primarily for server-based or service-oriented applications.

```
HINTERNET WinHttpConnect(
  HINTERNET     hSession,
  LPCWSTR       pswzServerName,
  INTERNET_PORT nServerPort,
  DWORD         dwReserved
);
```

# Proof of Concept

This POC utilizes the WinINet API to establish an internet connection to "example.com", sends an HTTP GET request to the root path ("/") and retrieves the response.

```c
#include <windows.h>
#include <winhttp.h>
#include <iostream>

#pragma comment(lib, "winhttp.lib")

int main() {
    // Specify the server and resource to connect to
    LPCWSTR serverName = L"example.com";
    LPCWSTR resource = L"/";

    // Initialize a WinHTTP session
    HINTERNET hSession = WinHttpOpen(L"A WinHTTP Example Program/1.0",
        WINHTTP_ACCESS_TYPE_DEFAULT_PROXY,
        WINHTTP_NO_PROXY_NAME,
        WINHTTP_NO_PROXY_BYPASS, 0);
    if (!hSession) {
        std::cerr << "WinHttpOpen failed: " << GetLastError() << std::endl;
        return 1;
    }

    // Specify an HTTP server
    HINTERNET hConnect = WinHttpConnect(hSession, serverName,
        INTERNET_DEFAULT_HTTP_PORT, 0);
    if (!hConnect) {
        std::cerr << "WinHttpConnect failed: " << GetLastError() << std::endl;
        WinHttpCloseHandle(hSession);
        return 1;
    }

    // Create an HTTP request handle
    HINTERNET hRequest = WinHttpOpenRequest(hConnect, L"GET", resource,
        NULL, WINHTTP_NO_REFERER,
        WINHTTP_DEFAULT_ACCEPT_TYPES,
        0);
    if (!hRequest) {
        std::cerr << "WinHttpOpenRequest failed: " << GetLastError() << std::endl;
        WinHttpCloseHandle(hConnect);
        WinHttpCloseHandle(hSession);
        return 1;
    }

    // Send a request
    BOOL result = WinHttpSendRequest(hRequest,
        WINHTTP_NO_ADDITIONAL_HEADERS, 0,
        WINHTTP_NO_REQUEST_DATA, 0,
        0, 0);
    if (!result) {
        std::cerr << "WinHttpSendRequest failed: " << GetLastError() << std::endl;
        WinHttpCloseHandle(hRequest);
        WinHttpCloseHandle(hConnect);
        WinHttpCloseHandle(hSession);
        return 1;
    }

    // Receive a response
    result = WinHttpReceiveResponse(hRequest, NULL);
    if (!result) {
        std::cerr << "WinHttpReceiveResponse failed: " << GetLastError() << std::endl;
        WinHttpCloseHandle(hRequest);
        WinHttpCloseHandle(hConnect);
        WinHttpCloseHandle(hSession);
        return 1;
    }

    // Read the data
    DWORD bytesRead = 0;
    char buffer[4096];
    do {
        result = WinHttpReadData(hRequest, buffer, sizeof(buffer), &bytesRead);
        if (result) {
            std::cout.write(buffer, bytesRead);
        }
    } while (result && bytesRead > 0);

    // Clean up
    WinHttpCloseHandle(hRequest);
    WinHttpCloseHandle(hConnect);
    WinHttpCloseHandle(hSession);

    return 0;
}
```

This is how the POC looks like when we execute the code:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/f553157e-497c-4eed-86da-85ecfbf02d04)

