# Description

The **`listen`** function is a key function used to place a socket in a state where it is listening for incoming connections. This function is required for setting up a server socket that waits for client connections. 
When a socket is created with **`socket()`**, it is by default in a closed state. Before it can accept incoming connections, it must be bound to a local address and port using **`bind()`**, and then placed into a listening state with **`listen()`**.

```
int listen(
  SOCKET s,
  int backlog
);
```

The **`backlog`** parameter specifies the number of incoming connections that can be queued before the system starts rejecting new incoming requests.

# Proof of Concept

This sample code creates a listening TCP socket bound to port 8080, places it in a listening state to accept incoming connections.

```c
#include <iostream>
#include <Winsock2.h>
#include <Ws2tcpip.h>

#pragma comment(lib, "Ws2_32.lib")

int main() {
    WSADATA wsaData;
    int result = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (result != 0) {
        std::cout << "WSAStartup failed. Error: " << result << std::endl;
        return 1;
    }

    // Create a socket
    SOCKET listeningSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (listeningSocket == INVALID_SOCKET) {
        std::cout << "Socket creation failed. Error: " << WSAGetLastError() << std::endl;
        WSACleanup();
        return 1;
    }

    // Bind the socket
    sockaddr_in hint;
    hint.sin_family = AF_INET;
    hint.sin_port = htons(8080); // Listening on port 8080, for example
    hint.sin_addr.S_un.S_addr = INADDR_ANY; // Listen on any network interface

    if (bind(listeningSocket, (sockaddr*)&hint, sizeof(hint)) == SOCKET_ERROR) {
        std::cout << "Bind failed. Error: " << WSAGetLastError() << std::endl;
        closesocket(listeningSocket);
        WSACleanup();
        return 1;
    }

    // Put the socket in listening mode
    if (listen(listeningSocket, SOMAXCONN) == SOCKET_ERROR) {
        std::cout << "Listening failed. Error: " << WSAGetLastError() << std::endl;
        closesocket(listeningSocket);
        WSACleanup();
        return 1;
    }

    std::cout << "Listening on port 8080..." << std::endl;

    // Accept a connection (this is a blocking call)
    sockaddr_in client;
    int clientSize = sizeof(client);
    SOCKET clientSocket = accept(listeningSocket, (sockaddr*)&client, &clientSize);

    if (clientSocket == INVALID_SOCKET) {
        std::cout << "Accept failed. Error: " << WSAGetLastError() << std::endl;
        closesocket(listeningSocket);
        WSACleanup();
        return 1;
    }

    // Close the listening socket as it's no longer needed
    closesocket(listeningSocket);

    // Close the client socket when done communicating
    closesocket(clientSocket);

    // Clean up Winsock
    WSACleanup();

    return 0;
}
```

This is how the end result looks like:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/a9510f27-8648-4268-82b4-5fbd738c6cbb)

