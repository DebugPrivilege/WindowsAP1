# Description

**`CreateService`** is a function that creates a new service object and adds it to the specified service control manager database.

```
SC_HANDLE CreateService(
  SC_HANDLE hSCManager,
  LPCWSTR   lpServiceName,
  LPCWSTR   lpDisplayName,
  DWORD     dwDesiredAccess,
  DWORD     dwServiceType,
  DWORD     dwStartType,
  DWORD     dwErrorControl,
  LPCWSTR   lpBinaryPathName,
  LPCWSTR   lpLoadOrderGroup,
  LPDWORD   lpdwTagId,
  LPCWSTR   lpDependencies,
  LPCWSTR   lpServiceStartName,
  LPCWSTR   lpPassword
);
```

# Proof of Concept

This sample code manages a Windows service named "CmdService" that can be installed, uninstalled, and runs cmd.exe under the SYSTEM account.

```c
#include <Windows.h>
#include <iostream>

#pragma comment(lib, "Advapi32.lib")

// Service name
const wchar_t* serviceName = L"CmdService";

// Entry point for the service
void WINAPI ServiceMain(DWORD argc, LPWSTR* argv);
void WINAPI ServiceCtrlHandler(DWORD ctrlCode);
void InstallService();
void UninstallService();

// Service status variables
SERVICE_STATUS serviceStatus;
SERVICE_STATUS_HANDLE serviceStatusHandle;

int wmain(int argc, wchar_t* argv[]) {
    // Check if there are command line arguments to install or uninstall the service
    if (argc > 1) {
        if (_wcsicmp(argv[1], L"install") == 0) {
            // Install the service
            InstallService();
            return 0;
        } else if (_wcsicmp(argv[1], L"uninstall") == 0) {
            // Uninstall the service
            UninstallService();
            return 0;
        }
    }

    // Service table to link the service name with the ServiceMain function
    SERVICE_TABLE_ENTRY serviceTable[] = {
        { (LPWSTR)serviceName, (LPSERVICE_MAIN_FUNCTION)ServiceMain },
        { nullptr, nullptr }
    };

    // Start the service control dispatcher for this service
    if (!StartServiceCtrlDispatcher(serviceTable)) {
        std::cerr << "StartServiceCtrlDispatcher failed. Error: " << GetLastError() << std::endl;
    }

    return 0;
}

// Main function for the service
void WINAPI ServiceMain(DWORD argc, LPWSTR* argv) {
    // Register the service control handler
    serviceStatusHandle = RegisterServiceCtrlHandler(serviceName, ServiceCtrlHandler);
    if (!serviceStatusHandle) {
        return;
    }

    // Initialize the service status structure
    serviceStatus.dwServiceType = SERVICE_WIN32_OWN_PROCESS;
    serviceStatus.dwCurrentState = SERVICE_START_PENDING;
    serviceStatus.dwControlsAccepted = SERVICE_ACCEPT_STOP | SERVICE_ACCEPT_SHUTDOWN;
    serviceStatus.dwWin32ExitCode = 0;
    serviceStatus.dwServiceSpecificExitCode = 0;
    serviceStatus.dwCheckPoint = 0;
    serviceStatus.dwWaitHint = 0;

    // Report initial status to the SCM
    if (!SetServiceStatus(serviceStatusHandle, &serviceStatus)) {
        return;
    }

    // Report running status when initialization is complete
    serviceStatus.dwCurrentState = SERVICE_RUNNING;
    if (!SetServiceStatus(serviceStatusHandle, &serviceStatus)) {
        return;
    }

    // Start cmd.exe under SYSTEM account
    STARTUPINFO si = { sizeof(si) };
    PROCESS_INFORMATION pi;
    if (!CreateProcess(
        L"C:\\Windows\\System32\\cmd.exe", // Application name
        nullptr,                          // Command line
        nullptr,                          // Process security attributes
        nullptr,                          // Primary thread security attributes
        FALSE,                            // Handles are not inherited
        0,                                // Creation flags
        nullptr,                          // Use parent's environment block
        nullptr,                          // Use parent's starting directory 
        &si,                              // Pointer to STARTUPINFO structure
        &pi                               // Pointer to PROCESS_INFORMATION structure
    )) {
        std::cerr << "CreateProcess failed. Error: " << GetLastError() << std::endl;
    } else {
        // Close process and thread handles as they are no longer needed
        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
    }

    // The service will continue running until it is stopped
    while (serviceStatus.dwCurrentState == SERVICE_RUNNING) {
        Sleep(1000); // Sleep to simulate work being done by the service
    }

    // Report stopped status to the SCM
    serviceStatus.dwCurrentState = SERVICE_STOPPED;
    SetServiceStatus(serviceStatusHandle, &serviceStatus);
}

// Service control handler function
void WINAPI ServiceCtrlHandler(DWORD ctrlCode) {
    switch (ctrlCode) {
        case SERVICE_CONTROL_STOP:
        case SERVICE_CONTROL_SHUTDOWN:
            // Handle stop and shutdown control codes
            serviceStatus.dwCurrentState = SERVICE_STOP_PENDING;
            SetServiceStatus(serviceStatusHandle, &serviceStatus);
            serviceStatus.dwCurrentState = SERVICE_STOPPED;
            SetServiceStatus(serviceStatusHandle, &serviceStatus);
            return;
        default:
            break;
    }
}

// Function to install the service
void InstallService() {
    // Open the Service Control Manager
    SC_HANDLE schSCManager = OpenSCManager(nullptr, nullptr, SC_MANAGER_CREATE_SERVICE);
    if (!schSCManager) {
        std::cerr << "OpenSCManager failed. Error: " << GetLastError() << std::endl;
        return;
    }

    // Get the full path to the executable
    wchar_t path[MAX_PATH];
    GetModuleFileName(nullptr, path, MAX_PATH);

    // Create the service
    SC_HANDLE schService = CreateService(
        schSCManager,
        serviceName,         // Service name
        serviceName,         // Display name
        SERVICE_ALL_ACCESS,  // Desired access
        SERVICE_WIN32_OWN_PROCESS, // Service type
        SERVICE_DEMAND_START,      // Start type
        SERVICE_ERROR_NORMAL,      // Error control type
        path,                      // Path to the service executable
        nullptr,                   // No load ordering group
        nullptr,                   // No tag identifier
        nullptr,                   // No dependencies
        nullptr,                   // LocalSystem account
        nullptr                    // No password
    );

    if (!schService) {
        std::cerr << "CreateService failed. Error: " << GetLastError() << std::endl;
        CloseServiceHandle(schSCManager);
        return;
    }

    std::cout << "Service installed successfully" << std::endl;

    // Close service and SCM handles
    CloseServiceHandle(schService);
    CloseServiceHandle(schSCManager);
}

// Function to uninstall the service
void UninstallService() {
    // Open the Service Control Manager
    SC_HANDLE schSCManager = OpenSCManager(nullptr, nullptr, SC_MANAGER_CONNECT);
    if (!schSCManager) {
        std::cerr << "OpenSCManager failed. Error: " << GetLastError() << std::endl;
        return;
    }

    // Open the service
    SC_HANDLE schService = OpenService(schSCManager, serviceName, SERVICE_STOP | DELETE);
    if (!schService) {
        std::cerr << "OpenService failed. Error: " << GetLastError() << std::endl;
        CloseServiceHandle(schSCManager);
        return;
    }

    // Stop the service if it is running
    SERVICE_STATUS serviceStatus;
    ControlService(schService, SERVICE_CONTROL_STOP, &serviceStatus);

    // Delete the service
    if (!DeleteService(schService)) {
        std::cerr << "DeleteService failed. Error: " << GetLastError() << std::endl;
    } else {
        std::cout << "Service uninstalled successfully" << std::endl;
    }

    // Close service and SCM handles
    CloseServiceHandle(schService);
    CloseServiceHandle(schSCManager);
}
```
