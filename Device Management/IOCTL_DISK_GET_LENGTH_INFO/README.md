# Description

**`IOCTL_DISK_GET_LENGTH_INFO`** control code is used to retrieve the length (size in bytes) of a disk or a volume. Malware can call **`DeviceIoControl`** with the parameter **`0x7405C`** to check the size of the disk. This can be used as an anti-VM and anti-sandbox technique. See: https://twitter.com/d4rksystem/status/1536433878839214083

# Proof of Concept

 This sample code example contains how to use **`IOCTL_DISK_GET_LENGTH_INFO`** to obtain the size of a disk

```c
#include <Windows.h>
#include <iostream>

int main() {
    // Open a handle to the physical drive
    HANDLE hDevice = CreateFile(
        L"\\\\.\\PhysicalDrive0",  // Targeting the first physical drive
        GENERIC_READ,              // Read access to the drive
        FILE_SHARE_READ | FILE_SHARE_WRITE, // Share mode
        NULL,                      // No security attributes
        OPEN_EXISTING,             // Opens the existing drive
        0,                         // No special attributes
        NULL);                     // No template file

    if (hDevice == INVALID_HANDLE_VALUE) {
        std::cerr << "CreateFile failed with error code: " << GetLastError() << std::endl;
        return 1;
    }

    GET_LENGTH_INFORMATION lengthInfo;
    DWORD bytesReturned;

    // Send IOCTL_DISK_GET_LENGTH_INFO to get the drive's length
    BOOL result = DeviceIoControl(
        hDevice,                        // Handle to the drive
        IOCTL_DISK_GET_LENGTH_INFO,     // Control code
        NULL,                           // No input buffer
        0,                              // No input buffer size
        &lengthInfo,                    // Output buffer
        sizeof(lengthInfo),             // Size of output buffer
        &bytesReturned,                 // Number of bytes returned
        NULL);                          // No overlapped structure

    if (result) {
        std::cout << "Disk size: " << lengthInfo.Length.QuadPart << " bytes" << std::endl;
    }
    else {
        std::cerr << "DeviceIoControl failed with error code: " << GetLastError() << std::endl;
    }

    CloseHandle(hDevice);
    return 0;
}
```

The output shows the size of a disk in bytes:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bd311c73-d6f7-4202-867c-31bcca4e2fa4)

