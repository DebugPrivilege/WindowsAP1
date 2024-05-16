# Description

**`CreateThread`** is a Windows API function that creates a thread to execute within the virtual address space of the calling process. Threads are one of the primary means for a program to perform tasks concurrently.

Here are the parameters:

```
HANDLE CreateThread(
  LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  SIZE_T                  dwStackSize,
  LPTHREAD_START_ROUTINE  lpStartAddress,
  LPVOID                  lpParameter,
  DWORD                   dwCreationFlags,
  LPDWORD                 lpThreadId
);
```

# Proof of Concept

This POC concurrently executes two tasks in separate threads by using the **`CreateThread`** function. The first thread creates and then deletes **1000** files in the **C:\Temp** directory, repeating this process **3** times. The second thread takes a snapshot of all processes running on the system and lists their process IDs and names, repeating this enumeration 30 times.

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>
#include <vector>

#define NUM_FILES 1000
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 30

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(
                fileName.c_str(),
                GENERIC_WRITE,
                0,
                NULL,
                CREATE_ALWAYS,
                FILE_ATTRIBUTE_NORMAL,
                NULL
            );

            if (hFile == INVALID_HANDLE_VALUE) {
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
            }
        }
    }

    return 0;
}


DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    for (int i = 0; i < 2; i++) {
        CloseHandle(hThreads[i]);
    }

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

In this screenshot, we are able to observe that there are two threads performing tasks concurrently. 

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/117a1bd3-1512-4f40-a7b1-5cd67042f489)


