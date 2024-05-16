# Description

The **`GetIpNetTable`** function retrieves the IP-to-physical address mapping table used by the local computer. This table is commonly known as the ARP (Address Resolution Protocol) cache. The ARP cache maps IPv4 addresses to the physical MAC addresses of devices on the same subnet.

The Address Resolution Protocol (ARP) is a network protocol used to map an IP address to a physical machine address that is recognized in the local network. In most cases, this physical address refers to the Media Access Control (MAC) address on a Local Area Network (LAN).

When a device (let's call it Device A) wants to communicate with another device (Device B) on the same LAN but only knows Device B's IP address, it broadcasts an ARP request on the network. This request asks, *"Who has IP address X? Tell Device A,"* where X is Device B's IP address.

# Proof of Concept


This code determines the necessary buffer size, and then retrieves and prints the IP-to-physical (MAC) address mapping table for the local computer using the **`GetIpNetTable`** function.

```c
#include <iostream>
#include <windows.h>
#include <iphlpapi.h>
#include <vector>

#pragma comment(lib, "iphlpapi.lib")
#pragma comment(lib, "Ws2_32.lib")

int main() {
    DWORD dwSize = 0;
    DWORD dwRetVal;

    // First call to get the necessary buffer size
    dwRetVal = GetIpNetTable(nullptr, &dwSize, FALSE);

    if (dwRetVal != ERROR_INSUFFICIENT_BUFFER) {
        std::cerr << "GetIpNetTable failed on the first call. Error: " << dwRetVal << std::endl;
        return 1;
    }

    // Allocate the buffer for the IP net table
    std::vector<BYTE> buffer(dwSize);

    // Pointer to the IP net table
    PMIB_IPNETTABLE pIpNetTable = reinterpret_cast<PMIB_IPNETTABLE>(buffer.data());

    // Second call to actually get the data
    dwRetVal = GetIpNetTable(pIpNetTable, &dwSize, FALSE);

    if (dwRetVal == NO_ERROR) {
        std::cout << "Number of entries: " << pIpNetTable->dwNumEntries << std::endl;
        for (DWORD i = 0; i < pIpNetTable->dwNumEntries; i++) {
            MIB_IPNETROW row = pIpNetTable->table[i];
            std::cout << "Entry " << i + 1 << std::endl;
            std::cout << "  Interface Index: " << row.dwIndex << std::endl;

            // Convert the IP address to a string
            IN_ADDR IpAddr;
            IpAddr.S_un.S_addr = (u_long)row.dwAddr;
            std::cout << "  IP Address: " << inet_ntoa(IpAddr) << std::endl;

            // Convert the physical address (MAC) to a readable string
            std::cout << "  Physical Address: ";
            for (DWORD j = 0; j < row.dwPhysAddrLen; j++) {
                printf("%.2X ", (int)row.bPhysAddr[j]);
            }
            std::cout << std::endl;

            std::cout << "  Type: ";
            switch (row.dwType) {
            case MIB_IPNET_TYPE_OTHER:
                std::cout << "Other";
                break;
            case MIB_IPNET_TYPE_INVALID:
                std::cout << "Invalid";
                break;
            case MIB_IPNET_TYPE_DYNAMIC:
                std::cout << "Dynamic";
                break;
            case MIB_IPNET_TYPE_STATIC:
                std::cout << "Static";
                break;
            default:
                std::cout << "Unknown";
            }
            std::cout << std::endl << "---------------------------------" << std::endl;
        }
    }
    else {
        std::cerr << "GetIpNetTable failed on the second call. Error: " << dwRetVal << std::endl;
        return 1;
    }

    return 0;
}
```

This is how the result looks like. It prints the ARP table entries, which include the IP addresses and physical addresses (MAC addresses) associated with each network interface.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/7617e9d0-b6d2-447d-b4fe-3870b1c91ee5)

