# Summary

**`NtSetInformationThread`** is an function exported by **`ntdll.dll`**. It's part of the Native API, which provides low-level system functionalities. This function allows modifying various settings and properties of a thread within the calling process.

One common use of **`NtSetInformationThread`** is to hide a thread from debuggers, using the **`ThreadHideFromDebugger`** (0x11) information class. This can be an attempt to make debugging more difficult for reverse engineers.

```
NTSTATUS NtSetInformationThread(
    HANDLE ThreadHandle,
    THREADINFOCLASS ThreadInformationClass,
    PVOID ThreadInformation,
    ULONG ThreadInformationLength
);
```

Here is an example of LockBit 3.0 using this technique, which has been discussed here: https://news.sophos.com/en-us/2022/11/30/lockbit-3-0-black-attacks-and-leaks-reveal-wormable-capabilities-and-tooling/

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/f501e57b-1111-4bff-a0f4-0d18a5883b54)



# Proof of Concept

This POC demonstrates how to hide threads from a debugger using the **`NtSetInformationThread`** function from **`ntdll.dll`** with the **`ThreadHideFromDebugger`** flag. It creates two threads: one for creating and deleting files in a loop, and another for enumerating processes. Both threads, as well as the main thread, are hidden from any attached debugger to make debugging harder.

```c
#include <Windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>

#define NUM_FILES 100
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 10

typedef enum _THREADINFOCLASS {
    ThreadHideFromDebugger = 17,
} THREADINFOCLASS;

typedef NTSTATUS(NTAPI* pNtSetInformationThread)(HANDLE, THREADINFOCLASS, PVOID, ULONG);

// Function to hide a thread from debugger
void HideThreadFromDebugger(HANDLE hThread) {
    HMODULE hNtdll = GetModuleHandleW(L"ntdll.dll");
    if (hNtdll) {
        auto NtSetInformationThread = (pNtSetInformationThread)GetProcAddress(hNtdll, "NtSetInformationThread");
        if (NtSetInformationThread) {
            NtSetInformationThread(hThread, ThreadHideFromDebugger, NULL, 0);
        }
    }
}

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(fileName.c_str(), GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
            if (hFile != INVALID_HANDLE_VALUE) {
                DWORD written;
                WriteFile(hFile, "Data", 4, &written, NULL);
                CloseHandle(hFile);
                DeleteFile(fileName.c_str());
            }
        }
    }
    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot != INVALID_HANDLE_VALUE) {
            PROCESSENTRY32 pe32;
            pe32.dwSize = sizeof(PROCESSENTRY32);
            if (Process32First(hSnapshot, &pe32)) {
                do {
                    std::wcout << L"Process ID: " << pe32.th32ProcessID << L", Process Name: " << pe32.szExeFile << std::endl;
                } while (Process32Next(hSnapshot, &pe32));
            }
            CloseHandle(hSnapshot);
        }
    }
    return 0;
}

int main() {
    // Hide the main thread from the debugger
    HideThreadFromDebugger(GetCurrentThread());

    HANDLE hThreads[2];
    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, NULL);
    // Hide the first worker thread from the debugger
    HideThreadFromDebugger(hThreads[0]);

    hThreads[1] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, NULL);
    // Hide the second worker thread from the debugger
    HideThreadFromDebugger(hThreads[1]);

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    for (auto& hThread : hThreads) {
        CloseHandle(hThread);
    }

    std::cout << "Operations completed." << std::endl;

    return 0;
}
```

In this example, we use the same code but without the anti-debugging technique. Upon loading the compiled program into WinDbg and setting breakpoints on specific functions, everything works as expected.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/478f8a84-2a6e-4d7e-a034-369d264abba3)


When we are trying to do this with the code that contains the anti-debugging technique, it doesn't work. It keeps saying that the *Debuggee is running..*

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/2651c63d-25f8-4e48-a921-5f14b51dbadc)

