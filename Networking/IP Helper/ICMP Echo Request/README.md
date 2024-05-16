# Description

An ICMP (Internet Control Message Protocol) echo request is a network diagnostic tool primarily used to test the reachability of a host on an IP network. It is part of the ICMP protocol, which is used for sending error messages and operational information indicating success or failure when communicating with another IP address.

It is commonly known as a "ping," named after the utility that sends these requests. When an ICMP echo request is sent to a host, that host is expected to respond with an ICMP echo reply, acknowledging the receipt of the request. This exchange allows the sender to confirm that the target host is reachable and to measure round-trip time (RTT). This is the time it takes for the echo request to go from the sender to the target host and for the reply to return.

# Proof of Concept

This code sample demonstrates the use of ICMP functions to send an ICMP echo request to a specified IP address or hostname and receive an echo reply. 
The **`IcmpCreateFile`** is a Windows API function that opens a handle. This handle allows applications to send ICMP messages such as echo requests and receive echo replies. 
**`IcmpSendEcho`** is a function that sends an ICMP Echo request to the specified destination and returns any replies. 

```
DWORD IcmpSendEcho(
  HANDLE                   IcmpHandle,
  IPAddr                   DestinationAddress,
  LPVOID                   RequestData,
  WORD                     RequestSize,
  PIP_OPTION_INFORMATION   RequestOptions,
  LPVOID                   ReplyBuffer,
  DWORD                    ReplySize,
  DWORD                    Timeout
);
```

What is important to note is that the **`ReplyBuffer`** parameter. It is a pointer to the buffer to receive the reply(ies). This buffer will receive an array of **`ICMP_ECHO_REPLY`** structures followed by the options and data for the echo reply.

```
   +0x000 Address          : Uint4B
   +0x004 Status           : Uint4B
   +0x008 RoundTripTime    : Uint4B
   +0x00c DataSize         : Uint2B
   +0x00e Reserved         : Uint2B
   +0x010 Data             : Ptr64 Void
   +0x018 Options          : ip_option_information
```

Here is the code sample:

```c
#include <iostream>
#include <vector>
#include <winsock2.h>
#include <ws2tcpip.h>
#include <iphlpapi.h>
#include <icmpapi.h>

// Link against Winsock and IP Helper libraries
#pragma comment(lib, "Ws2_32.lib")
#pragma comment(lib, "Iphlpapi.lib")

int main(int argc, char* argv[]) {
    // Check for correct command-line usage
    if (argc != 2) {
        std::cerr << "Usage: " << argv[0] << " <IP address or hostname>" << std::endl;
        return 1;
    }

    // Initialize Winsock
    WSADATA wsaData;
    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        std::cerr << "WSAStartup failed." << std::endl;
        return 1;
    }

    // Prepare hints for getaddrinfo call
    struct addrinfo hints {}, * result = nullptr;
    hints.ai_family = AF_INET; // Restrict to IPv4 addresses
    // Resolve the hostname or IP address
    if (getaddrinfo(argv[1], nullptr, &hints, &result) != 0) {
        std::cerr << "getaddrinfo failed to resolve hostname." << std::endl;
        WSACleanup();
        return 1;
    }

    // Check if any address was resolved
    if (result == nullptr) {
        std::cerr << "No addresses resolved." << std::endl;
        WSACleanup();
        return 1;
    }

    // Create ICMP handle for sending echo requests
    HANDLE icmpFile = IcmpCreateFile();
    if (icmpFile == INVALID_HANDLE_VALUE) {
        std::cerr << "IcmpCreateFile failed." << std::endl;
        freeaddrinfo(result);
        WSACleanup();
        return 1;
    }

    // Buffer for the ICMP reply
    std::vector<char> replyBuffer(sizeof(ICMP_ECHO_REPLY) + 32); // Additional space for data

    // Send ICMP echo request
    DWORD retVal = IcmpSendEcho(icmpFile, reinterpret_cast<sockaddr_in*>(result->ai_addr)->sin_addr.s_addr,
        nullptr, 0, nullptr, replyBuffer.data(), static_cast<DWORD>(replyBuffer.size()), 1000);
    if (retVal != 0) {
        // Process the ICMP echo reply
        auto* echoReply = reinterpret_cast<PICMP_ECHO_REPLY>(replyBuffer.data());
        char replyAddrStr[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &echoReply->Address, replyAddrStr, sizeof(replyAddrStr));
        std::cout << "Received reply from: " << replyAddrStr
            << " Status: " << echoReply->Status
            << " RTT: " << echoReply->RoundTripTime << "ms" << std::endl;
    }
    else {
        std::cerr << "IcmpSendEcho reported error: " << GetLastError() << std::endl;
    }

    // Clean up
    IcmpCloseHandle(icmpFile);
    freeaddrinfo(result);
    WSACleanup();

    return 0;
}
```

This is how it looks when the code is executed:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/c68acbe9-c806-4b0d-87ea-4cbd82251e02)


