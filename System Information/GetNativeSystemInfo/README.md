# Description

**`GetNativeSystemInfo`** is a function that retrieves information about the current system's architecture and configuration. It can include the processor architecture, page size, and number of processors.

# Proof of Concept

This sample code retrieves and displays the system's hardware information, including the number of processors, page size, processor architecture type, the range of application-addressable memory, and the active processor mask.

```c
#include <windows.h>
#include <iostream>

int main() {
    SYSTEM_INFO si;
    GetNativeSystemInfo(&si);

    std::cout << "Hardware information: " << std::endl;
    std::cout << "Number of processors: " << si.dwNumberOfProcessors << std::endl;
    std::cout << "Page size: " << si.dwPageSize << " Bytes" << std::endl;

    std::cout << "Processor architecture: ";
    switch (si.wProcessorArchitecture) {
    case PROCESSOR_ARCHITECTURE_AMD64:
        std::cout << "x64 (AMD or Intel)" << std::endl;
        break;
    case PROCESSOR_ARCHITECTURE_ARM:
        std::cout << "ARM" << std::endl;
        break;
    case PROCESSOR_ARCHITECTURE_IA64:
        std::cout << "Intel Itanium-based" << std::endl;
        break;
    case PROCESSOR_ARCHITECTURE_INTEL:
        std::cout << "x86" << std::endl;
        break;
    }

    std::cout << "Minimum application address: " << si.lpMinimumApplicationAddress << std::endl;
    std::cout << "Maximum application address: " << si.lpMaximumApplicationAddress << std::endl;
    std::cout << "Active processor mask: " << si.dwActiveProcessorMask << std::endl;

    return 0;
}
```

This is the result:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/a560d88a-c831-477a-98ff-e92001ba3e20)
