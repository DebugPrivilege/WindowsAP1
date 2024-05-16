# Summary

Windows Management Instrumentation can be used to create processes on both local and remote Windows systems. This is achieved through the **`Win32_Process`** WMI class using its **`Create`** method. 
The **`Win32_Process.Create`** method allows you to start a new process to execute a command or launch an application. For creating processes on the local machine, we can use the **`IWbemServices::ExecMethod`** method after setting up a WMI session to invoke the **`Create`** method of the **`Win32_Process`** class. 

```
uint32 Create(
  [in]  string                   CommandLine,
  [in]  string                   CurrentDirectory,
  [in]  Win32_ProcessStartup     ProcessStartupInformation,
  [out] uint32                   ProcessId
);
```

# Proof of Concept

This POC demonstrates how to use Windows Management Instrumentation (WMI) to execute a command by invoking the **`Create`** method of the **`Win32_Process`** WMI class. 

```c
#include <iostream>
#include <Windows.h>
#include <Wbemidl.h>
#include <comdef.h>

#pragma comment(lib, "wbemuuid.lib")

HRESULT InitializeCOMSecurity() {
    // Initialize COM security to allow all clients to connect
    return CoInitializeSecurity(
        NULL,                        // Security descriptor
        -1,                          // COM negotiates service
        NULL,                        // Authentication services
        NULL,                        // Reserved
        RPC_C_AUTHN_LEVEL_DEFAULT,   // Default authentication level for proxies
        RPC_C_IMP_LEVEL_IMPERSONATE, // Default Impersonation level for proxies
        NULL,                        // Authentication info
        EOAC_NONE,                   // Additional capabilities
        NULL                         // Reserved
    );
}

void ExecuteCmdUsingWMI(IWbemServices* pSvc) {
    IWbemClassObject* pClass = NULL;
    HRESULT hr = pSvc->GetObject(_bstr_t(L"Win32_Process"), 0, NULL, &pClass, NULL);
    if (FAILED(hr)) {
        std::cerr << "Could not get Win32_Process class. Error code = 0x"
            << std::hex << hr << std::endl;
        return;
    }

    IWbemClassObject* pInParamsDefinition = NULL;
    hr = pClass->GetMethod(L"Create", 0, &pInParamsDefinition, NULL);
    if (FAILED(hr)) {
        std::cerr << "Could not get the Create method. Error code = 0x"
            << std::hex << hr << std::endl;
        pClass->Release();
        return;
    }

    IWbemClassObject* pClassInstance = NULL;
    hr = pInParamsDefinition->SpawnInstance(0, &pClassInstance);
    if (FAILED(hr)) {
        std::cerr << "Could not spawn class instance. Error code = 0x"
            << std::hex << hr << std::endl;
        pInParamsDefinition->Release();
        pClass->Release();
        return;
    }

    // Set the command line
    VARIANT varCommand;
    VariantInit(&varCommand);
    varCommand.vt = VT_BSTR;
    varCommand.bstrVal = _bstr_t(L"cmd.exe /c echo Hello World > C:\\temp\\hello.txt").Detach();

    // Execute the method
    hr = pClassInstance->Put(L"CommandLine", 0, &varCommand, 0);
    if (SUCCEEDED(hr)) {
        IWbemClassObject* pOutParams = NULL;
        hr = pSvc->ExecMethod(_bstr_t(L"Win32_Process"), _bstr_t(L"Create"), 0,
            NULL, pClassInstance, &pOutParams, NULL);

        if (SUCCEEDED(hr)) {
            std::cout << "Successfully executed command." << std::endl;
        }
        else {
            std::cerr << "Could not execute method. Error code = 0x"
                << std::hex << hr << std::endl;
        }

        if (pOutParams)
            pOutParams->Release();
    }
    else {
        std::cerr << "Could not set the command. Error code = 0x"
            << std::hex << hr << std::endl;
    }

    VariantClear(&varCommand);
    pClassInstance->Release();
    pInParamsDefinition->Release();
    pClass->Release();
}

int main() {
    HRESULT hr = CoInitializeEx(0, COINIT_MULTITHREADED);
    if (FAILED(hr)) {
        std::cerr << "Failed to initialize COM library. Error code = 0x"
            << std::hex << hr << std::endl;
        return 1;
    }

    hr = InitializeCOMSecurity();
    if (FAILED(hr)) {
        std::cerr << "Failed to initialize security. Error code = 0x"
            << std::hex << hr << std::endl;
        CoUninitialize();
        return 1;
    }

    IWbemLocator* pLoc = NULL;
    hr = CoCreateInstance(CLSID_WbemLocator, 0, CLSCTX_INPROC_SERVER,
        IID_IWbemLocator, (LPVOID*)&pLoc);

    if (FAILED(hr)) {
        std::cerr << "Failed to create IWbemLocator object. Error code = 0x"
            << std::hex << hr << std::endl;
        CoUninitialize();
        return 1;
    }

    IWbemServices* pSvc = NULL;
    // Connect to the root\cimv2 namespace with the current user
    hr = pLoc->ConnectServer(_bstr_t(L"ROOT\\CIMV2"), NULL, NULL, 0, NULL, 0, 0, &pSvc);
    if (FAILED(hr)) {
        std::cerr << "Could not connect. Error code = 0x"
            << std::hex << hr << std::endl;
        pLoc->Release();
        CoUninitialize();
        return 1;
    }

    ExecuteCmdUsingWMI(pSvc);

    pSvc->Release();
    pLoc->Release();
    CoUninitialize();
    return 0;
}
```

In this screenshot, we are able to observe that we were able to use WMI to create a process and write a file to disk in the C:\Temp folder.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/95f67967-3819-4e45-8187-9f98ac211959)


