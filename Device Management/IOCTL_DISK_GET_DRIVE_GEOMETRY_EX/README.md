# Description

**`IOCTL_DISK_GET_DRIVE_GEOMETRY_EX`** is an IOCTL code used in Windows to retrieve extended information about the geometry of a disk drive.

# Proof of Concept

The sample code snippet opens a handle to the first physical drive **`(PhysicalDrive0)`**, uses the **`IOCTL_DISK_GET_DRIVE_GEOMETRY_EX`** control code to fetch extended geometry information about the drive.

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

    DISK_GEOMETRY_EX dg;
    DWORD bytesReturned;

    // Send IOCTL_DISK_GET_DRIVE_GEOMETRY_EX to get the drive's geometry
    BOOL result = DeviceIoControl(
        hDevice,                          // Handle to the drive
        IOCTL_DISK_GET_DRIVE_GEOMETRY_EX, // Control code
        NULL,                             // No input buffer
        0,                                // No input buffer size
        &dg,                              // Output buffer
        sizeof(dg),                       // Size of output buffer
        &bytesReturned,                   // Number of bytes returned
        NULL);                            // No overlapped structure

    if (result) {
        std::cout << "Disk size: " << dg.DiskSize.QuadPart << " bytes" << std::endl;
        std::cout << "Media type: " << dg.Geometry.MediaType << std::endl;
        // Add more outputs as needed based on the DISK_GEOMETRY_EX structure
    }
    else {
        std::cerr << "DeviceIoControl failed with error code: " << GetLastError() << std::endl;
    }

    CloseHandle(hDevice);
    return 0;
}
```

The total size of the physical drive in bytes is being displayed as well as a value of **12** that maps to **FixedMedia**, which means that the drive is a fixed hard disk

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/389a5847-6eef-49ff-aaf1-90d950f13400)

