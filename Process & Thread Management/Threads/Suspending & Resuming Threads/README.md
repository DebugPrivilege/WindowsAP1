# Summary

**`SuspendThread`** and **`ResumeThread`** are Windows API functions used to suspend and resume the execution of a thread within a process. These functions are primarily used for debugging purposes or to control thread execution in a specific manner that cannot be achieved through normal synchronization methods.

# Proof of Concept

This POC demonstrates how to enumerate processes and their threads, suspend or resume specific threads based on their ID, and display the updated state of these threads.

```c
#include <iostream>
#include <iomanip>
#include <windows.h>
#include <TlHelp32.h>
#include <vector>
#include <string>

// Enum to represent the state of a thread.
enum class ThreadState {
    Running,
    Suspended,
    Unknown
};

// Structure to hold information about a thread.
typedef struct {
    DWORD threadID;
    ThreadState state;
} ThreadInfo;

// Structure to hold information about a process including its threads.
typedef struct {
    DWORD processPID;
    std::wstring processName;
    std::vector<ThreadInfo> threadInfos;
} ProcessInfo;

// Function to suspend a thread by its ID.
bool suspendThread(DWORD threadID)
{
    // Open the thread with suspend/resume permissions.
    HANDLE threadHandle = OpenThread(THREAD_SUSPEND_RESUME, FALSE, threadID);
    if (threadHandle == INVALID_HANDLE_VALUE) {
        return false;
    }
    // Suspend the thread and close its handle.
    DWORD threadState = SuspendThread(threadHandle);
    CloseHandle(threadHandle);
    return (threadState != (DWORD)-1);
}

// Function to resume a thread by its ID.
bool resumeThread(DWORD threadID) {
    // Open the thread with suspend/resume permissions.
    HANDLE threadHandle = OpenThread(THREAD_SUSPEND_RESUME, FALSE, threadID);
    if (threadHandle == INVALID_HANDLE_VALUE) {
        return false;
    }
    // Resume the thread and close its handle.
    DWORD threadState = ResumeThread(threadHandle);
    CloseHandle(threadHandle);
    return (threadState != (DWORD)-1);
}

// Function to get information about a process and its threads by the process name.
ProcessInfo getProcessInfo(const std::wstring& wprocessName)
{
    // Take a snapshot of all processes in the system.
    HANDLE snap_handler = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snap_handler == INVALID_HANDLE_VALUE) {
        return {};
    }
    PROCESSENTRY32 processEntry = { sizeof(PROCESSENTRY32) };
    ProcessInfo info = {};
    // Iterate through the snapshot to find the specified process.
    while (Process32Next(snap_handler, &processEntry)) {
        if (_wcsicmp(processEntry.szExeFile, wprocessName.c_str()) == 0) {
            info.processPID = processEntry.th32ProcessID;
            info.processName = processEntry.szExeFile;
            // For the found process, take another snapshot for threads.
            HANDLE snap_handler_threads = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);
            if (snap_handler_threads == INVALID_HANDLE_VALUE) {
                CloseHandle(snap_handler);
                return {};
            }
            THREADENTRY32 threadEntry = { sizeof(THREADENTRY32) };
            // Iterate through the threads snapshot to find threads belonging to the process.
            while (Thread32Next(snap_handler_threads, &threadEntry)) {
                if (threadEntry.th32OwnerProcessID == info.processPID) {
                    ThreadInfo threadInfo = { threadEntry.th32ThreadID, ThreadState::Running };
                    info.threadInfos.push_back(threadInfo);
                }
            }
            CloseHandle(snap_handler_threads);
            break;
        }
    }
    CloseHandle(snap_handler);
    return info;
}

// Function to print detailed information about a process's threads in a tabular format.
void print_table(const ProcessInfo& info) {
    // Print headers
    std::wcout << std::left
        << std::setw(15) << L"PID"
        << std::setw(25) << L"Process Name"
        << std::setw(15) << L"Thread ID"
        << std::setw(45) << L"Thread State" << std::endl;
    std::wcout << std::wstring(100, L'=') << std::endl;
    // Print each thread's information
    for (const auto& threadInfo : info.threadInfos) {
        std::wstring threadStateStr;
        // Convert the ThreadState enum to a readable string
        switch (threadInfo.state) {
        case ThreadState::Running:
            threadStateStr = L"Running";
            break;
        case ThreadState::Suspended:
            threadStateStr = L"Suspended";
            break;
        case ThreadState::Unknown:
        default:
            threadStateStr = L"Unknown";
            break;
        }
        // Print thread information
        std::wcout << std::left
            << std::setw(15) << info.processPID
            << std::setw(25) << info.processName
            << std::setw(15) << threadInfo.threadID
            << std::setw(45) << threadStateStr << std::endl;
    }
}

// Function to update the state of a thread in the ProcessInfo structure.
void updateThreadState(ProcessInfo& info, DWORD threadID, ThreadState newState) {
    // Find the thread in the vector and update its state.
    auto it = std::find_if(info.threadInfos.begin(), info.threadInfos.end(),
        [threadID](const ThreadInfo& threadInfo) {
            return threadInfo.threadID == threadID;
        });
    if (it != info.threadInfos.end()) {
        it->state = newState;
    }
}

int main(int argc, char* argv[]) {
    // Check for correct number of arguments and display usage if incorrect.
    if (argc < 3) {
        std::cerr << "Usage: " << argv[0] << " <process_name> [--suspend-thread <thread_id> | --resume-thread <thread_id> | --get-thread]" << std::endl;
        return 1;
    }
    // Convert the process name argument from char* to std::wstring.
    std::string processName(argv[1]);
    std::wstring wprocessName(processName.begin(), processName.end());
    // Get the process information including its threads.
    ProcessInfo info = getProcessInfo(wprocessName);
    // Parse the command-line arguments to perform actions like suspending/resuming threads.
    if (argc >= 3) {
        if (std::string(argv[2]) == "--suspend-thread") {
            if (argc < 4) {
                std::cerr << "Usage: " << argv[0] << " <process_name> --suspend-thread <thread_id>" << std::endl;
                return 1;
            }
            // Suspend the specified thread and update its state in the info structure.
            DWORD threadID = std::stoul(argv[3]);
            if (suspendThread(threadID)) {
                updateThreadState(info, threadID, ThreadState::Suspended);
            }
            else {
                std::cerr << "Failed to suspend thread " << threadID << "." << std::endl;
            }
        }
        else if (std::string(argv[2]) == "--resume-thread") {
            if (argc < 4) {
                std::cerr << "Usage: " << argv[0] << " <process_name> --resume-thread <thread_id>" << std::endl;
                return 1;
            }
            // Resume the specified thread and update its state in the info structure.
            DWORD threadID = std::stoul(argv[3]);
            if (resumeThread(threadID)) {
                updateThreadState(info, threadID, ThreadState::Running);
            }
            else {
                std::cerr << "Failed to resume thread " << threadID << "." << std::endl;
            }
        }
        else if (std::string(argv[2]) == "--get-thread") {
            // Print the table of threads for the process and exit.
            print_table(info);
            return 0;
        }
    }
    // If no specific action was requested, print the updated table of threads.
    print_table(info);
    return 0;
}
```

In this example, we are enumerating all the threads within **notepad.exe**

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/a9bc0940-7622-4805-8a09-c853017c8e4a)


Here we are making a call to **`SuspendThread`** to suspend a specific thread.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/93dc3dbd-4911-48cb-99c2-73077ff952bc)


To finalize it, we could resume it again by making a call to **`ResumeThread`**.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/2f37ea12-a4bd-40d5-8df0-f9d9ca4220a7)



