# Description

The Windows Sockets API defines how software should access network services, particularly TCP/IP. It is designed to be an interface for developing Windows-based applications that can communicate with other TCP/IP-based networks and systems, including the internet.

A "socket" in the context of networking is an endpoint for sending and receiving data across a network. It is a fundamental concept in network programming, which is allowing different processes or systems to communicate with each other over a network using standard protocols like TCP or UDP.

The **`connect`** function that is originating from **`ws2_32.dll`** establishes a connection to a specified socket. The **`name`** parameter is a pointer to a sockaddr structure that contains the destination address and port number for the connection.

```
int connect(
  SOCKET         s,
  const sockaddr *name,
  int            namelen
);
```

The **`send`** function is used to write outgoing data on a connected socket. The **`s`** parameter specifies the socket descriptor, and the **`buf`** parameter points to a buffer of the data to be sent.

```
int send(
  SOCKET s,
  const char *buf,
  int len,
  int flags
);
```

The **`recv`** function is used to receive data from a socket, which could be either a TCP stream socket or a UDP datagram socket, depending on how the socket is configured.

```
int recv(
  SOCKET s,
  char   *buf,
  int    len,
  int    flags
);
```

# Proof of Concept

This POC demonstrates how to establish a TCP connection to a remote server using Windows Sockets, send an HTTP GET request, and receive the server's response.

```c
#include <iostream>
#include <Winsock2.h>
#include <Ws2tcpip.h>

#pragma comment(lib, "Ws2_32.lib")

int main() {
    // Initialize Winsock
    WSADATA wsaData;
    int result = WSAStartup(MAKEWORD(2, 2), &wsaData);

    if (result != 0) {
        std::cout << "WSAStartup failed. Error: " << result << std::endl;
        return 1;
    }

    // Create a socket
    SOCKET sock = WSASocketW(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);

    if (sock == INVALID_SOCKET) {
        std::cout << "WSASocketW failed. Error: " << WSAGetLastError() << std::endl;
        WSACleanup();
        return 1;
    }

    // Set up the sockaddr_in structure for the remote server
    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(80); // Port number (e.g., 80 for HTTP)

    // Convert the IPv4 address string to binary format
    result = inet_pton(AF_INET, "93.184.216.34", &serverAddr.sin_addr);

    if (result <= 0) {
        std::cout << "inet_pton failed. Error: " << WSAGetLastError() << std::endl;
        closesocket(sock);
        WSACleanup();
        return 1;
    }

    // Connect to the remote server
    result = connect(sock, (SOCKADDR*)&serverAddr, sizeof(serverAddr));

    if (result == SOCKET_ERROR) {
        std::cout << "Connection failed. Error: " << WSAGetLastError() << std::endl;
        closesocket(sock);
        WSACleanup();
        return 1;
    }

    std::cout << "Connected to the remote server successfully." << std::endl;

    // Send an HTTP GET request
    const char* httpRequest = "GET / HTTP/1.1\r\n"
        "Host: example.com\r\n"
        "Connection: close\r\n"
        "\r\n";
    result = send(sock, httpRequest, strlen(httpRequest), 0);

    if (result == SOCKET_ERROR) {
        std::cout << "Failed to send the request. Error: " << WSAGetLastError() << std::endl;
        closesocket(sock);
        WSACleanup();
        return 1;
    }

    std::cout << "Request sent successfully." << std::endl;

    // Receive the response
    char buffer[4096];
    int bytesReceived;

    while ((bytesReceived = recv(sock, buffer, sizeof(buffer) - 1, 0)) > 0) {
        buffer[bytesReceived] = '\0';
        std::cout << buffer;
    }

    if (bytesReceived < 0) {
        std::cout << "Failed to receive the response. Error: " << WSAGetLastError() << std::endl;
    }
    else {
        std::cout << "Response received successfully." << std::endl;
    }

    // Close the socket
    closesocket(sock);

    // Clean up Winsock
    WSACleanup();

    return 0;
}
```

The output displayed indicates that the code executed successfully. It shows that a connection was made to the remote server, the HTTP GET request was sent, and the response was received and printed to the console. 

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/ffe418f1-42ab-4887-82ab-4ac8e1913038)



