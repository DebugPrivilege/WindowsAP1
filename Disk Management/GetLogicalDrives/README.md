# Description

The **`GetLogicalDrives`** of **`kernel32.dll`** is a function that retrieves a bitmask representing the currently available disk drives. Each bit represents one drive, with the least significant bit (bit 0) representing drive A, bit 1 representing drive B, and so on.

# Proof of Concept:

This code assumes that the system has a maximum of 32 logical drives, which is a reasonable assumption for Windows systems.

```c
#include <iostream>
#include <windows.h>
#include <bitset>
#include <string>

int main() {
    DWORD driveBitMask = GetLogicalDrives();
    if (driveBitMask == 0) {
        std::cerr << "GetLogicalDrives failed with error code: " << GetLastError() << std::endl;
        return 1;
    }

    std::bitset<32> driveBits(driveBitMask);
    std::cout << "Available drives: ";

    // Iterate through each bit in the bitmask
    for (int i = 0; i < 32; ++i) {
        if (driveBits.test(i)) {
            // Convert the bit index to the corresponding drive letter
            char driveLetter = 'A' + i;
            std::cout << driveLetter << ":\\ ";
        }
    }

    std::cout << std::endl;
    return 0;
}
```

This is how the result looks like:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/7b045dbd-14b4-4803-b3f8-3d8b5abe02e8)


