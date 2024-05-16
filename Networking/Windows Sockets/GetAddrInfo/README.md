# Summary

**`GetAddrinfo`** is a socket function in the Windows Sockets API that resolves domain names into their corresponding IP addresses and prepares the structures needed to open a socket to that address.

```
int WSAAPI getaddrinfo(
  [in, optional]  const char            *pNodeName,
  [in, optional]  const char            *pServiceName,
  [in, optional]  const struct addrinfo *pHints,
  [out]           struct addrinfo       **ppResult
);
```

# Proof of Concept

This POC a TCP connection to "example.com" on the HTTP port (80), sends an HTTP GET request, receives the HTTP response, and then closes the connection. It uses Winsock APIs to **resolve** the address, create a socket, connect to the server, and send and receive data.

```c
#include <iostream>
#include <Winsock2.h>
#include <Ws2tcpip.h>

#pragma comment(lib, "Ws2_32.lib")

int main() {
    WSADATA wsaData;
    int result;

    // Initialize Winsock
    result = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (result != 0) {
        std::cerr << "WSAStartup failed: " << result << std::endl;
        return 1;
    }

    // Resolve the server address and port
    struct addrinfo* resultList = nullptr, * ptr = nullptr, hints;
    ZeroMemory(&hints, sizeof(hints));
    hints.ai_family = AF_UNSPEC; // Allow IPv4 or IPv6
    hints.ai_socktype = SOCK_STREAM; // TCP stream sockets
    hints.ai_protocol = IPPROTO_TCP; // TCP protocol

    // Resolve the address and port
    if (getaddrinfo("example.com", "http", &hints, &resultList) != 0) {
        WSACleanup();
        std::cerr << "getaddrinfo failed" << std::endl;
        return 1;
    }

    // Print resolved addresses
    std::cout << "Resolved addresses for example.com:" << std::endl;
    for (ptr = resultList; ptr != nullptr; ptr = ptr->ai_next) {
        char addressString[INET6_ADDRSTRLEN];
        void* address;

        if (ptr->ai_family == AF_INET) {
            address = &((struct sockaddr_in*)ptr->ai_addr)->sin_addr;
        }
        else {
            address = &((struct sockaddr_in6*)ptr->ai_addr)->sin6_addr;
        }

        inet_ntop(ptr->ai_family, address, addressString, sizeof(addressString));
        std::cout << "  " << addressString << std::endl;
    }

    // Attempt to connect to the first address returned by the call to getaddrinfo
    SOCKET connectSocket = INVALID_SOCKET;
    ptr = resultList; // use the first IP address to which we could connect
    // Create a SOCKET for connecting to the server
    connectSocket = socket(ptr->ai_family, ptr->ai_socktype, ptr->ai_protocol);
    if (connectSocket == INVALID_SOCKET) {
        std::cerr << "Error at socket(): " << WSAGetLastError() << std::endl;
        freeaddrinfo(resultList);
        WSACleanup();
        return 1;
    }

    // Connect to the server
    result = connect(connectSocket, ptr->ai_addr, (int)ptr->ai_addrlen);
    if (result == SOCKET_ERROR) {
        std::cerr << "Connection failed: " << WSAGetLastError() << std::endl;
        closesocket(connectSocket);
        connectSocket = INVALID_SOCKET;
    }
    freeaddrinfo(resultList); // No longer needed

    if (connectSocket == INVALID_SOCKET) {
        std::cerr << "Unable to connect to server!" << std::endl;
        WSACleanup();
        return 1;
    }
    std::cout << "Successfully connected to example.com." << std::endl;

    // Send an HTTP GET request
    const char* sendbuf = "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n";
    result = send(connectSocket, sendbuf, (int)strlen(sendbuf), 0);
    if (result == SOCKET_ERROR) {
        std::cerr << "send failed: " << WSAGetLastError() << std::endl;
        closesocket(connectSocket);
        WSACleanup();
        return 1;
    }

    std::cout << "Request sent successfully. Bytes sent: " << result << std::endl;

    // Receive the response
    char recvbuf[512];
    int recvbuflen = 512;
    do {
        result = recv(connectSocket, recvbuf, recvbuflen, 0);
        if (result > 0) {
            recvbuf[result] = '\0'; // Null-terminate the buffer
            std::cout << recvbuf;
        }
        else if (result == 0)
            std::cout << "Connection closed" << std::endl;
        else
            std::cerr << "recv failed: " << WSAGetLastError() << std::endl;
    } while (result > 0);

    // Close the socket
    closesocket(connectSocket);
    WSACleanup();
    return 0;
}
```

In this screenshot, we can see that we were able to connect to example.com, send 56 bytes to it, and were able to resolve the corresponding IP address that is associated with the URL.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/889d3575-975a-48e1-9920-bdf8730fe616)

