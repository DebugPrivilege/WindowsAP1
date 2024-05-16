# Description

Windows Management Instrumentation (WMI) can be used to gather information about processes running on a Windows system. WMI provides a unified interface for Windows system management and operations, which includes querying for active processes. The **`Win32_Process`** WMI class represents a process on a Windows operating system, and you can use WMI queries (using WMI Query Language, or WQL) to retrieve information about these processes.

The **`IWbemServices::ExecQuery`** method, which is responsible for executing a WQL (WMI Query Language) query against the WMI data store.

```
HRESULT ExecQuery(
  [in]  const BSTR             bstrQueryLanguage,
  [in]  const BSTR             bstrQuery,
  [in]  LONG                   lFlags,
  [in, optional] IWbemContext* pCtx,
  [out] IEnumWbemClassObject** ppEnum
);
```

**`bstrQuery`** contains the query string.  For process enumeration, the query might be something like **`"SELECT * FROM Win32_Process"`** to select all instances of the **`Win32_Process`** class.

# Proof of Concept

The function responsible for enumerating the processes in the code sample is **`pSvc->ExecQuery`**. This function executes a WQL (WMI Query Language) query to select all instances of the **`Win32_Process`** class and enumerate all the running processes.

```c
#include <iostream>
#include <Windows.h>
#include <Wbemidl.h>
#include <comdef.h>

#pragma comment(lib, "wbemuuid.lib") // Link against the WMI library

// Initializes COM and sets up a connection to the WMI service
HRESULT InitializeWMI(IWbemLocator** locator, IWbemServices** service) {
    // Initialize COM for multi-threaded use
    HRESULT hr = CoInitializeEx(NULL, COINIT_MULTITHREADED);
    if (FAILED(hr)) {
        std::cerr << "CoInitializeEx failed: " << std::hex << hr << std::endl;
        return hr;
    }

    // Set security levels for the COM library
    hr = CoInitializeSecurity(NULL, -1, NULL, NULL, RPC_C_AUTHN_LEVEL_DEFAULT, RPC_C_IMP_LEVEL_IMPERSONATE, NULL, EOAC_NONE, NULL);
    if (FAILED(hr)) {
        std::cerr << "CoInitializeSecurity failed: " << std::hex << hr << std::endl;
        CoUninitialize();
        return hr;
    }

    // Create an instance of the WbemLocator interface
    hr = CoCreateInstance(CLSID_WbemLocator, 0, CLSCTX_INPROC_SERVER, IID_IWbemLocator, reinterpret_cast<void**>(locator));
    if (FAILED(hr)) {
        std::cerr << "CoCreateInstance for IWbemLocator failed: " << std::hex << hr << std::endl;
        CoUninitialize();
        return hr;
    }

    // Connect to the WMI service on the local computer
    hr = (*locator)->ConnectServer(_bstr_t(L"ROOT\\CIMV2"), NULL, NULL, 0, 0, 0, 0, service);
    if (FAILED(hr)) {
        std::cerr << "ConnectServer failed: " << std::hex << hr << std::endl;
        (*locator)->Release();
        CoUninitialize();
        return hr;
    }

    return S_OK; // Return success
}

// Enumerates all processes using WMI and prints their names and PIDs
void EnumerateAllProcesses(IWbemServices* service) {
    IEnumWbemClassObject* enumerator = nullptr;
    // Execute a WMI query to get all Win32_Process instances
    HRESULT hr = service->ExecQuery(
        bstr_t("WQL"),
        bstr_t("SELECT Name, ProcessId FROM Win32_Process"),
        WBEM_FLAG_FORWARD_ONLY | WBEM_FLAG_RETURN_IMMEDIATELY,
        NULL,
        &enumerator);

    if (FAILED(hr)) {
        std::cerr << "ExecQuery failed: " << std::hex << hr << std::endl;
        return;
    }

    IWbemClassObject* clsObj = nullptr;
    ULONG retCnt = 0;

    // Iterate through the query results
    while (enumerator) {
        hr = enumerator->Next(WBEM_INFINITE, 1, &clsObj, &retCnt);
        if (retCnt == 0) {
            break;
        }

        VARIANT varName, varPid; // Variables to hold process name and ID
        hr = clsObj->Get(L"Name", 0, &varName, 0, 0);
        hr = clsObj->Get(L"ProcessId", 0, &varPid, 0, 0);

        if (SUCCEEDED(hr)) {
            std::wcout << L"Process Name: " << varName.bstrVal << L", PID: " << varPid.uintVal << std::endl;
        }

        VariantClear(&varName); // Clear the VARIANTs
        VariantClear(&varPid);

        clsObj->Release(); // Release the current object
    }

    enumerator->Release(); // Release the enumerator
}

// Main function to set up WMI and enumerate all processes
int main() {
    IWbemLocator* locator = nullptr;
    IWbemServices* service = nullptr;
    HRESULT hr = InitializeWMI(&locator, &service); // Initialize WMI

    if (SUCCEEDED(hr)) {
        EnumerateAllProcesses(service); // Enumerate all processes
    }

    if (service) service->Release(); // Clean up
    if (locator) locator->Release();
    CoUninitialize(); // Uninitialize COM

    return 0;
}
```

In this screenshot, we can see that we're able to enumerate all the running processes on a system via WMI.

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/0e1048a7-c91b-4621-893e-443feead05ae)


# References

- https://www.mdsec.co.uk/2022/08/fourteen-ways-to-read-the-pid-for-the-local-security-authority-subsystem-service-lsass/
