# Description

**`IOCTL_DISK_IS_WRITABLE`** is an IOCTL that is used to check whether a disk drive is writable. When this IOCTL is issued to a disk device, it determines if the disk is currently protected against write operations. If the disk is writable, the operation completes successfully. If not, the operation fails.

# Proof of Concept

This sample code uses the **`IOCTL_DISK_IS_WRITABLE`** IOCTL to check if the first physical drive on the system **`(PhysicalDrive0)`** is writable. **`PhysicalDrive0`** in Windows is a special file that represents the first physical hard disk drive attached to the system at a low level.

```c
#include <Windows.h>
#include <iostream>

bool CheckDiskIsWritable(HANDLE hDevice) {
    DWORD bytesReturned;
    // IOCTL_DISK_IS_WRITABLE does not require input or output buffer
    BOOL result = DeviceIoControl(
        hDevice,
        IOCTL_DISK_IS_WRITABLE,
        NULL,
        0,
        NULL,
        0,
        &bytesReturned,
        NULL
    );

    return result == TRUE;
}

int main() {
    // Open a handle to the physical drive
    HANDLE hDevice = CreateFile(
        L"\\\\.\\PhysicalDrive0",   // Use the correct drive number
        GENERIC_READ,               // Read access is sufficient
        FILE_SHARE_READ | FILE_SHARE_WRITE, // Share mode
        NULL,                       // No security attributes
        OPEN_EXISTING,              // Open existing drive
        0,                          // No attributes or flags
        NULL                        // No template file
    );

    if (hDevice == INVALID_HANDLE_VALUE) {
        std::cerr << "CreateFile failed with error code: " << GetLastError() << std::endl;
        return 1;
    }

    // Check if the disk is writable
    if (CheckDiskIsWritable(hDevice)) {
        std::cout << "The disk is writable." << std::endl;
    }
    else {
        DWORD errorCode = GetLastError();
        if (errorCode == ERROR_WRITE_PROTECT) {
            std::cout << "The disk is not writable (write-protected)." << std::endl;
        }
        else {
            std::cout << "Failed to determine if the disk is writable. Error code: " << errorCode << std::endl;
        }
    }

    // Always remember to close the device handle
    CloseHandle(hDevice);

    return 0;
}
```

Within this example, the disk is writeable:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/36de25d0-feef-440e-9a80-29417ef535ef)


