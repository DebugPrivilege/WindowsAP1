# Description

An adapter is a hardware or software component that connects a computer or other device to a network. It enables the device to communicate with other devices on the network by facilitating data transmission over network protocols.

A few examples, but not limited to:

- **Network Interface Card (NIC):** A hardware component, typically in the form of a circuit board or chip, installed in a computer or connected externally. It provides a physical interface for connecting to a wired network, such as Ethernet.
- **Wireless Adapter:** Allows devices to connect to a wireless network.
- **Virtual Network Adapter:**  A software-emulated version of a network interface card. It can be used to interface with virtual networks, especially within virtualized environments or when using VPNs.

# Proof of Concept

The **`GetAdaptersInfo`** function is part of the IP Helper API, which provides information about the network adapters on the local computer. This code initializes a buffer to store adapter information and calls **`GetAdaptersInfo`** to fill the buffer. It then iterates through the linked list of **`IP_ADAPTER_INFO`** structures filled by **`GetAdaptersInfo`**, printing details about each adapter.

```
typedef struct _IP_ADAPTER_INFO {
  struct _IP_ADAPTER_INFO *Next;
  DWORD                   ComboIndex;
  char                    AdapterName[MAX_ADAPTER_NAME_LENGTH + 4];
  char                    Description[MAX_ADAPTER_DESCRIPTION_LENGTH + 4];
  UINT                    AddressLength;
  BYTE                    Address[MAX_ADAPTER_ADDRESS_LENGTH];
  DWORD                   Index;
  UINT                    Type;
  UINT                    DhcpEnabled;
  PIP_ADDR_STRING         CurrentIpAddress;
  IP_ADDR_STRING          IpAddressList;
  IP_ADDR_STRING          GatewayList;
  IP_ADDR_STRING          DhcpServer;
  BOOL                    HaveWins;
  IP_ADDR_STRING          PrimaryWinsServer;
  IP_ADDR_STRING          SecondaryWinsServer;
  time_t                  LeaseObtained;
  time_t                  LeaseExpires;
} IP_ADAPTER_INFO, *PIP_ADAPTER_INFO;
```

Here is the code sample:

```c
#include <iostream>
#include <windows.h>
#include <iphlpapi.h>
#include <vector>

#pragma comment(lib, "iphlpapi.lib")

int main() {
    DWORD dwSize = sizeof(IP_ADAPTER_INFO);
    DWORD dwRetVal = 0;

    // Initially allocate memory for a single adapter.
    std::vector<BYTE> buffer(dwSize);

    // The GetAdaptersInfo function might need more space. 
    // Call it once to get the actual size needed.
    dwRetVal = GetAdaptersInfo(reinterpret_cast<IP_ADAPTER_INFO*>(buffer.data()), &dwSize);

    if (dwRetVal == ERROR_BUFFER_OVERFLOW) {
        // Resize the buffer to the size returned by GetAdaptersInfo
        buffer.resize(dwSize);
        // Try again with the resized buffer
        dwRetVal = GetAdaptersInfo(reinterpret_cast<IP_ADAPTER_INFO*>(buffer.data()), &dwSize);
    }

    if (dwRetVal == ERROR_SUCCESS) {
        // Iterate through the adapter list.
        IP_ADAPTER_INFO* pAdapterInfo = reinterpret_cast<IP_ADAPTER_INFO*>(buffer.data());
        while (pAdapterInfo) {
            std::cout << "Adapter Name: " << pAdapterInfo->AdapterName << std::endl;
            std::cout << "Adapter Description: " << pAdapterInfo->Description << std::endl;

            // IP Address List
            IP_ADDR_STRING* pIpAddressList = &pAdapterInfo->IpAddressList;
            while (pIpAddressList) {
                std::cout << "IP Address: " << pIpAddressList->IpAddress.String << std::endl;
                std::cout << "IP Mask: " << pIpAddressList->IpMask.String << std::endl;
                pIpAddressList = pIpAddressList->Next;
            }
            std::cout << "---------------------------------" << std::endl;

            pAdapterInfo = pAdapterInfo->Next;
        }
    }
    else {
        std::cerr << "GetAdaptersInfo failed with error: " << dwRetVal << std::endl;
    }

    return 0;
}
```

This is the result:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/10962f92-16a5-492d-82b6-b280a2fdf8a2)

