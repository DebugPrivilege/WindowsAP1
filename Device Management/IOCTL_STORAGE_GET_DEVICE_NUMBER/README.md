# Description

**`IOCTL_STORAGE_GET_DEVICE_NUMBER`** is an IOCTL code used with the **`DeviceIoControl`** function in the Windows API. It allows applications to retrieve information about a storage device, such as a hard disk or a volume. This information includes the type of the device, the number assigned to the device by the system, and, if applicable, the partition number.

The following fields are included:

- **`DeviceType`** Specifies the type of device, such as **`FILE_DEVICE_DISK`** for disk drives, **`FILE_DEVICE_CD_ROM`** for CD/DVD drives, etc.
- **`DeviceNumber`** Specifies the number of the device.
- **`PartitionNumber`** Specifies the partition number of the device.

# Proof of Concept

This sample code opens a handle to the **C:** drive, retrieves its device type, device number, and partition number using the **`IOCTL_STORAGE_GET_DEVICE_NUMBER`** IOCTL code. 

```c
#include <Windows.h>
#include <iostream>
#include <iomanip>

int main() {
    // Open a handle to the device
    HANDLE deviceHandle = CreateFile(
        L"\\\\.\\C:",              // Drive to open
        GENERIC_READ,              // Access to the drive
        FILE_SHARE_READ | FILE_SHARE_WRITE, // Share mode
        NULL,                      // Security attributes
        OPEN_EXISTING,             // Disposition
        0,                         // File attributes
        NULL);                     // Template file

    if (deviceHandle == INVALID_HANDLE_VALUE) {
        std::cerr << "Error: Unable to access device. Code: " << GetLastError() << std::endl;
        return 1;
    }

    STORAGE_DEVICE_NUMBER deviceNumber;
    DWORD bytesReturned = 0;

    // Send the IOCTL_STORAGE_GET_DEVICE_NUMBER control code to the device
    BOOL result = DeviceIoControl(
        deviceHandle,
        IOCTL_STORAGE_GET_DEVICE_NUMBER,
        NULL,                      // No input buffer
        0,                         // Input buffer size
        &deviceNumber,             // Output buffer
        sizeof(deviceNumber),      // Output buffer size
        &bytesReturned,            // Bytes returned
        NULL);                     // Overlapped

    if (result) {
        std::cout << "Device Type: " << deviceNumber.DeviceType << std::endl;
        std::cout << "Device Number: " << deviceNumber.DeviceNumber << std::endl;
        std::cout << "Partition Number: " << deviceNumber.PartitionNumber << std::endl;
    }
    else {
        std::cerr << "Error: DeviceIoControl failed. Code: " << GetLastError() << std::endl;
    }

    // Close the handle
    CloseHandle(deviceHandle);
    return 0;
}
```

A value of **`7`** at the device type typically corresponds to **`FILE_DEVICE_DISK`**, indicating that the device is a disk drive. A value of **3** at the partition number means that the volume is located on the third partition of the disk. 

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/1e2b018c-23f8-47fc-a978-28db237af66a)

