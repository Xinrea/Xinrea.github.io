---
title: "GetSaveFileName 引发的工作目录变更"
date: 2024-08-12
draft: false
---

近日在处理 C++ Win32 程序异常时，采用 [Minidump](https://learn.microsoft.com/en-us/windows/win32/debug/minidump-files) 来保存程序崩溃时的栈记录，生成的 dmp 文件保存在配置数据目录下。
如果程序启动时检查发现存在 dmp 文件，则弹出提示框让用户选择路径来保存该文件，主要逻辑如下所示：

```cpp
LONG WINAPI unhandled_handler(EXCEPTION_POINTERS* e) {
    const wstring dumpfile = LAppDefine::documentPath + L"/minidump.dmp";
    HANDLE hFile = CreateFile(dumpfile.c_str(), GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile && (hFile != INVALID_HANDLE_VALUE)) {
        MINIDUMP_EXCEPTION_INFORMATION mdei;
        mdei.ThreadId = GetCurrentThreadId();
        mdei.ExceptionPointers = e;
        mdei.ClientPointers = FALSE;

        MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(),
                          hFile, MiniDumpNormal, &mdei, NULL, NULL);
        CloseHandle(hFile);
    }
    return EXCEPTION_CONTINUE_SEARCH;
}

int main() {
  SetUnhandledExceptionFilter(unhandled_handler);
  // ...
  if (std::filesystem::exists(std::filesystem::path(filepath))) {
    OPENFILENAME ofn;                         // Common dialog box structure
    wchar_t szFile[260] = L"minidump.dmp\0";  // Buffer for file name

    ZeroMemory(&ofn, sizeof(ofn));
    ofn.lStructSize = sizeof(ofn);
    ofn.hwndOwner = nullptr;
    ofn.lpstrFile = szFile;
    ofn.nMaxFile = sizeof(szFile) / sizeof(wchar_t);
    ofn.lpstrFilter = L"Dump Files\0*.dmp\0";
    ofn.nFilterIndex = 1;
    ofn.lpstrFileTitle = NULL;
    ofn.nMaxFileTitle = 0;
    ofn.lpstrInitialDir = NULL;
    ofn.Flags = OFN_PATHMUSTEXIST | OFN_OVERWRITEPROMPT;

    if (GetSaveFileName(&ofn) == TRUE) {
       CopyFile(filepath.c_str(), ofn.lpstrFile, TRUE);
       DeleteFile(filepath.c_str());
    }
  }
  // ...
  struct stat statBuf {};
  if (stat(relative_path, &statBuf) == 0) {
    //...
  }
}
```

实现后程序出现了奇怪的问题，当有崩溃报告存在时，后续在通过 stat 获取其他文件时必然失败，因此还导致了其他地方发生崩溃，继而程序永远无法正常启动。

后来经过逐步调试，发现只要 GetSaveFileName() 调用成功（用户选定路径并确认）就会导致后续的 stat 出现问题。考虑到 stat 使用的路径为相对路径，猜测 GetSaveFileName() 会改变程序当前的工作目录，且经查询文档确认了这一点。

而 ofn.Flags 也提供了解决这一问题的方法：OFN_NOCHANGEDIR。
