# Description

Windows Task Scheduler API is a set of COM interfaces that allow developers to interact with the Windows Task Scheduler service. To use the API, we typically start by initializing the COM library and then creating an instance of the **`ITaskService`** interface. From there, we can connect to the Task Scheduler service.

# Proof of Concept

This sample code initializes the COM library and connect to the Windows Task Scheduler service. From there, it creates a scheduled task that launches **`cmd.exe`** as **`NT AUTHORITY\SYSTEM`**.

```c
#include <iostream>
#include <windows.h>
#include <taskschd.h>
#include <comutil.h>

#pragma comment(lib, "taskschd.lib")
#pragma comment(lib, "comsuppw.lib")

// Utility template function to safely release COM objects.
template<typename T>
void SafeRelease(T** ppT) {
    if (*ppT) {
        (*ppT)->Release(); 
        *ppT = nullptr;    
    }
}

int main() {
    // Initialize COM library for use by concurrent multithreaded applications.
    HRESULT hr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
    if (FAILED(hr)) {
        std::cerr << "Failed to initialize COM library.\n";
        return 1;
    }

    // Create an instance of the Task Service.
    ITaskService* pService = nullptr;
    hr = CoCreateInstance(CLSID_TaskScheduler, nullptr, CLSCTX_INPROC_SERVER, IID_ITaskService, reinterpret_cast<void**>(&pService));
    if (FAILED(hr)) {
        std::cerr << "Failed to create TaskService instance.\n";
        CoUninitialize();
        return 1;
    }

    // Connect to the Task Scheduler service.
    hr = pService->Connect(_variant_t(), _variant_t(), _variant_t(), _variant_t());
    if (FAILED(hr)) {
        std::cerr << "Failed to connect to Task Scheduler service.\n";
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Retrieve the root task folder.
    ITaskFolder* pRootFolder = nullptr;
    hr = pService->GetFolder(_bstr_t(L"\\"), &pRootFolder);
    if (FAILED(hr)) {
        std::cerr << "Failed to get root folder.\n";
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Create a task definition object to create a new task.
    ITaskDefinition* pTask = nullptr;
    hr = pService->NewTask(0, &pTask);
    if (FAILED(hr)) {
        std::cerr << "Failed to create new task.\n";
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Set the registration information for the task.
    IRegistrationInfo* pInfo = nullptr;
    hr = pTask->get_RegistrationInfo(&pInfo);
    if (SUCCEEDED(hr)) {
        pInfo->put_Author(_bstr_t(L"COM Scheduled Task Example"));
        SafeRelease(&pInfo);
    }

    // Create a trigger collection to hold triggers for the task.
    ITriggerCollection* pTriggerCollection = nullptr;
    hr = pTask->get_Triggers(&pTriggerCollection);
    if (FAILED(hr)) {
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Create a time trigger.
    ITrigger* pTrigger = nullptr;
    hr = pTriggerCollection->Create(TASK_TRIGGER_TIME, &pTrigger);
    if (FAILED(hr)) {
        SafeRelease(&pTriggerCollection);
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Query for the ITimeTrigger interface to set properties specific to time triggers.
    ITimeTrigger* pTimeTrigger = nullptr;
    hr = pTrigger->QueryInterface(IID_ITimeTrigger, reinterpret_cast<void**>(&pTimeTrigger));
    if (FAILED(hr)) {
        SafeRelease(&pTrigger);
        SafeRelease(&pTriggerCollection);
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Calculate and set the start time to 5 minutes from now.
    SYSTEMTIME st;
    GetLocalTime(&st);
    FILETIME ft;
    SystemTimeToFileTime(&st, &ft);
    ULARGE_INTEGER uli;
    uli.LowPart = ft.dwLowDateTime;
    uli.HighPart = ft.dwHighDateTime;
    uli.QuadPart += 5 * 60 * 10000000LL; // Convert 5 minutes to 100-nanosecond intervals.
    FileTimeToSystemTime((FILETIME*)&uli, &st);

    wchar_t startTime[32];
    swprintf(startTime, 32, L"%04d-%02d-%02dT%02d:%02d:%02d", st.wYear, st.wMonth, st.wDay, st.wHour, st.wMinute, st.wSecond);
    pTimeTrigger->put_StartBoundary(_bstr_t(startTime)); // Set the start boundary for the trigger.
    SafeRelease(&pTimeTrigger);
    SafeRelease(&pTrigger);
    SafeRelease(&pTriggerCollection);

    // Create a collection to store actions for the task.
    IActionCollection* pActionCollection = nullptr;
    hr = pTask->get_Actions(&pActionCollection);
    if (FAILED(hr)) {
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Create an action for the task. In this case, execute a command line operation.
    IAction* pAction = nullptr;
    hr = pActionCollection->Create(TASK_ACTION_EXEC, &pAction);
    if (FAILED(hr)) {
        SafeRelease(&pActionCollection);
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    // Query for the IExecAction interface to set properties specific to execution actions.
    IExecAction* pExecAction = nullptr;
    hr = pAction->QueryInterface(IID_IExecAction, reinterpret_cast<void**>(&pExecAction));
    if (FAILED(hr)) {
        SafeRelease(&pAction);
        SafeRelease(&pActionCollection);
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    pExecAction->put_Path(_bstr_t(L"C:\\Windows\\System32\\cmd.exe")); // Set the path of the executable.
    SafeRelease(&pExecAction);
    SafeRelease(&pAction);
    SafeRelease(&pActionCollection);

    // Register the task under the root folder with specific settings for NT AUTHORITY\SYSTEM.
    IRegisteredTask* pRegisteredTask = nullptr;
    hr = pRootFolder->RegisterTaskDefinition(_bstr_t(L"NewScheduledTask"),
        pTask,
        TASK_CREATE_OR_UPDATE,
        _variant_t(L"S-1-5-18"), // SID for NT AUTHORITY\SYSTEM
        _variant_t(),
        TASK_LOGON_SERVICE_ACCOUNT,
        _variant_t(L""),
        &pRegisteredTask);
    if (FAILED(hr)) {
        std::cerr << "Failed to register task. Error: " << hr << '\n';
        SafeRelease(&pTask);
        SafeRelease(&pRootFolder);
        SafeRelease(&pService);
        CoUninitialize();
        return 1;
    }

    std::cout << "Task successfully registered to run as SYSTEM after 5 minutes.\n";
    SafeRelease(&pRegisteredTask);
    SafeRelease(&pTask);
    SafeRelease(&pRootFolder);
    SafeRelease(&pService);
    CoUninitialize();

    return 0;
}
```

Here we are executing our code:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/e728033c-b0e6-4dde-a1b2-4eb0299ff258)


The result shows that the task scheduler has been successfully created:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/0aeb4803-ac49-4f1d-b4f6-126c37eaaffb)


