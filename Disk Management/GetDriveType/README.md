# Description

**`GetDriveType`** is a function of **`kernel32.dll`** that determines the type of a specified drive. This can include whether the drive is a hard disk, a removable drive, a network drive, a CD-ROM, a RAM disk, or some other type. 

```
UINT GetDriveType(
  LPCWSTR lpRootPathName
);
```

**`GetDriveType`** returns one of the following values, which represent the type of drive:

- **`DRIVE_UNKNOWN:`** The drive type cannot be determined.
- **`DRIVE_NO_ROOT_DIR:`** The root path is invalid; for example, there is no volume mounted at the path.
- **`DRIVE_REMOVABLE:`** The drive is a type that has removable media, such as a floppy drive or flash drive.
- **`DRIVE_FIXED:`** The drive is a fixed disk, such as a hard disk drive.
- **`DRIVE_REMOTE:`** The drive is a network drive.
- **`DRIVE_CDROM:`** The drive is a CD-ROM drive.
- **`DRIVE_RAMDISK:`** The drive is a RAM disk.

# Proof of Concept

This sample code iterates through all possible drive letters from 'A' to 'Z', uses the **`GetDriveType`** function to determine and print the type of each drive present on the system to the console.

```c
#include <windows.h>
#include <iostream>

int main() {
    wchar_t drive[] = L"A:\\";
    UINT driveType;

    for (wchar_t letter = L'A'; letter <= L'Z'; ++letter) {
        drive[0] = letter; // Set the drive letter to check

        driveType = GetDriveType(drive);
        std::wcout << letter << L":\\ - ";

        switch (driveType) {
        case DRIVE_UNKNOWN:
            std::wcout << L"Unknown";
            break;
        case DRIVE_NO_ROOT_DIR:
            std::wcout << L"Drive does not exist";
            break;
        case DRIVE_REMOVABLE:
            std::wcout << L"Removable Drive";
            break;
        case DRIVE_FIXED:
            std::wcout << L"Fixed Drive";
            break;
        case DRIVE_REMOTE:
            std::wcout << L"Network Drive";
            break;
        case DRIVE_CDROM:
            std::wcout << L"CD-ROM Drive";
            break;
        case DRIVE_RAMDISK:
            std::wcout << L"RAM Disk";
            break;
        default:
            std::wcout << L"Unknown Type";
            break;
        }
        std::wcout << std::endl;
    }

    return 0;
}
```

This is how it looks:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/98cfccd5-ef3f-415a-8fa7-b95b78dad4c6)


