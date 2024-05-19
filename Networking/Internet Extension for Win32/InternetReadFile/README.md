# Description

**`InternetReadFile`** is a function that reads data from an open internet handle. This function is part of the WinINet API, which provides a set of functions for interacting with internet protocols such as HTTP, FTP, and others.

```
BOOL InternetReadFile(
  HINTERNET hFile, --> A handle to an internet file.
  LPVOID lpBuffer, --> A pointer to a buffer that receives the data read from the internet file.
  DWORD dwNumberOfBytesToRead, --> A pointer to a variable that receives the number of bytes read.
  LPDWORD lpdwNumberOfBytesRead
);
```

# Proof of Concept

This code downloads content from a specified URL to a local file using the **`InternetOpen`**, **`InternetOpenUrl`**, and **`InternetReadFile`** APIs from the **`WinINet`** library to handle the internet session, open the URL, and read the content.

```c
#include <iostream>
#include <fstream>
#include <string>
#include <Windows.h>
#include <WinInet.h>
#include <vector>
#include <memory>

#pragma comment(lib, "wininet.lib")

struct DownloadInfo {
    std::wstring url;
    std::wstring save_path;
};

// Function to convert a narrow string to a wide string
std::wstring to_wstring(const std::string& str) {
    int size_needed = MultiByteToWideChar(CP_UTF8, 0, str.c_str(), (int)str.size(), nullptr, 0);
    std::wstring wstr(size_needed, 0);
    MultiByteToWideChar(CP_UTF8, 0, str.c_str(), (int)str.size(), &wstr[0], size_needed);
    return wstr;
}

int main(int argc, char* argv[]) {
    // Check if the correct number of arguments is provided
    if (argc != 3) {
        std::cerr << "Usage: " << argv[0] << " <url> <save_path>\n";
        std::cerr << "Example: POC.exe https://github.com/0xlane/BypassUAC/raw/master/x64/Release/BypassUAC.exe C:\\Temp\\BypassUAC.exe\n";
        return 1;
    }

    // Initialize DownloadInfo with provided arguments
    DownloadInfo download_info{ to_wstring(argv[1]), to_wstring(argv[2]) };

    // Open an internet handle
    HINTERNET hInternet = InternetOpen(L"MyDownloader", INTERNET_OPEN_TYPE_DIRECT, nullptr, nullptr, 0);
    if (!hInternet) {
        std::cerr << "InternetOpen failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Ensure the internet handle is closed properly
    std::unique_ptr<void, decltype(&InternetCloseHandle)> internet_closer(hInternet, InternetCloseHandle);

    // Open the URL
    HINTERNET hUrl = InternetOpenUrl(hInternet, download_info.url.c_str(), nullptr, 0, INTERNET_FLAG_RELOAD, 0);
    if (!hUrl) {
        std::cerr << "InternetOpenUrl failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Ensure the URL handle is closed properly
    std::unique_ptr<void, decltype(&InternetCloseHandle)> url_closer(hUrl, InternetCloseHandle);

    // Open the output file
    std::ofstream output_file(download_info.save_path, std::ios::binary);
    if (!output_file) {
        std::cerr << "Failed to open output file." << std::endl;
        return 1;
    }

    // Buffer to store downloaded data
    constexpr DWORD buffer_size = 4096;
    std::vector<char> buffer(buffer_size);
    DWORD bytes_read = 0;

    // Read from the URL and write to the file
    while (InternetReadFile(hUrl, buffer.data(), buffer_size, &bytes_read) && bytes_read > 0) {
        output_file.write(buffer.data(), bytes_read);
    }

    // Close the output file
    output_file.close();

    if (output_file.fail()) {
        std::cerr << "Failed to write to output file." << std::endl;
        return 1;
    }

    std::cout << "File downloaded successfully.\n";
    return 0;
}
```

This is how it looks when we execute the code. File is downloaded to disk:

![image](https://github.com/DebugPrivilege/WindowsAP1/assets/63166600/bf4d5f7e-eabe-468e-be98-e69a6c2f15c2)

