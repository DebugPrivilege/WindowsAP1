# Description

**`TerminateProcess`** is a Windows API function that is used to unconditionally terminate a specified process and all of its threads. This function provides a way for applications to force a process to end, without waiting for it to clean up resources.

```
BOOL TerminateProcess(
  HANDLE hProcess,
  UINT   uExitCode
);
```

In order to terminate a process, the handle must have the **`PROCESS_TERMINATE`** access right. We can obtain this handle through functions like **`OpenProcess`** for example.

# Proof of Concept

This POC enumerates all running processes and terminates process names that matches any in the predefined list of target process names. It uses the **`CreateToolhelp32Snapshot`**, **`Process32FirstW`**, and **`Process32NextW`** functions to iterate through the processes, and **`OpenProcess`** combined with **`TerminateProcess`** to stop the specified processes.

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>

// Array of process names to stop
static const wchar_t* processes_to_stop[] = { L"notepad.exe", L"sqlmangr.exe", L"supervise.exe", L"Defwatch.exe", L"winword.exe", L"QBW32.exe", L"QBDBMgr.exe", L"qbupdate.exe", L"axlbridge.exe", L"httpd.exe", L"fdlauncher.exe", L"MsDtSrvr.exe", L"java.exe", L"360se.exe", L"360doctor.exe", L"wdswfsafe.exe", L"fdhost.exe", L"dbeng50.exe", L"GDscan.exe", L"vxmon.exe", L"vsnapvss.exe", L"CagService.exe", L"DellSystemDetect.exe", L"ProcessHacker.exe", L"procexp64.exe", L"Procexp.exe", L"WireShark.exe", L"dumpcap.exe", L"Autoruns.exe", L"Autoruns64.exe", L"Autoruns64a.exe", L"Autorunsc.exe", L"Autorunsc64.exe", L"Sysmon.exe", L"Sysmon64.exe", L"procexp64a.exe", L"procmon.exe", L"Procmon64.exe", L"procmon64a.exe", L"ADExplorer.exe", L"ADExplorer64.exe", L"ADExplorer64a.exe", L"tcpview.exe", L"tcpview64.exe", L"tcpview64a.exe", L"VeeamDeploymentSvc.exe", L"windbg.exe", L"cdb.exe", L"cdb64.exe", L"ida64.exe", L"OllyDbg.exe", L"bintext.exe", L"TTTracerElevated.exe", L"TTDInject.exe", L"pestudio.exe", L"capa-v1.0.0-win.exe", L"netmon.exe", L"Fiddler.exe" };

// Function to stop the specified processes
void StopProcesses() {
    HANDLE hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    PROCESSENTRY32W pEntry = {};
    pEntry.dwSize = sizeof(pEntry);
    BOOL hRes = Process32FirstW(hSnapShot, &pEntry);
    while (hRes)
    {
        for (const auto& process : processes_to_stop) {
            if (lstrcmpW(process, pEntry.szExeFile) == 0) {
                HANDLE hProcess = OpenProcess(PROCESS_TERMINATE, FALSE, pEntry.th32ProcessID);
                if (hProcess != nullptr)
                {
                    TerminateProcess(hProcess, 0);
                    CloseHandle(hProcess);
                }
                break;
            }
        }
        hRes = Process32NextW(hSnapShot, &pEntry);
    }
    CloseHandle(hSnapShot);
}

int main()
{
    StopProcesses();
    std::cout << "Processes terminated successfully." << std::endl;
    return 0;
}
```

Every process that is running and contains the name of the predefined list, will get terminated.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bfd0e370-a5ef-42ad-8dac-59e36dc8275c)

