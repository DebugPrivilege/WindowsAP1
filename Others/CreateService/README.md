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

