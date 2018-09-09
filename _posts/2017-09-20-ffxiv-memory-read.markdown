---
layout: post
title: "FinalFantasyXIV 内存数据的读取（1）"
date:   2017-09-20 8:00:00 +0800
categories: windows c++ memory
---

前几天突发奇想，想通过模拟按键的方式来操作ffxiv，以实现类似于电影拍摄中的视角移动，为广大游戏视频制作者提供帮助。  

但是ffxiv作为一款使用diretX开发的游戏，其游戏窗口屏蔽了在外的按键模拟信息，简单的`keybd_event()`和`mouse_event()`并无法实现对游戏内的模拟操作，在网上查询一系列资料后，发现只要能够获取窗口的句柄，便能够轻松地使用`PostMessage()`方式模拟按键信息，实现代码如下：

{% highlight c %}
int main(int argc, char const *argv[])
{
	HWND hwnd = FindWindow(0, L"最终幻想XIV");
	if (hwnd == 0)cout << "Error:Can't find window!" << endl;
	else {
		cout << "Log:Find hwnd is " << hwnd << endl;
		DWORD proc_id;
		GetWindowThreadProcessId(hwnd, &proc_id);
		cout << "Log:Get PID is " << proc_id << endl;
		PostMessage(hwnd, WM_KEYDOWN, VkKeyScan('a'), 0);
		Sleep(500);
		PostMessage(hwnd, WM_KEYUP, VkKeyScan('a'), 0);
	}
	system("pause");
}
{% endhighlight %}

但在不经意中了解到，获得窗口句柄，然后使用`GetWindowThreadProcessID()`即可获得窗口对应的进程ID，再用`OpenProcess()`甚至可以获得进程权限，进行对其内存的读写。如下：

{% highlight c %}
int main(int argc, char const *argv[])
{
	HWND hwnd = FindWindow(0, L"最终幻想XIV");
	if (hwnd == 0)cout << "Error:Can't find window!" << endl;
	else {
		cout << "Log:Find hwnd is " << hwnd << endl;
		DWORD proc_id;
		GetWindowThreadProcessId(hwnd, &proc_id);
		cout << "Log:Get PID is " << proc_id << endl;
		HANDLE hProc = OpenProcess(PROCESS_ALL_ACCESS, FALSE, procId);
		DWORD adr = xxxx;
		DWORD pointed;
		ReadProcessMemory(hProc, (LPCVOID)adr, &pointed, 4,NULL)
	}
	system("pause");
}
{% endhighlight %}

>adr即为需要提前找好的内存地址，`ReadProcessMemory()`会读取数据存放在pointed中。（adr的寻找、偏移的确定都会在第二篇中进行讲解）

这样读取虽然简单粗暴，可是当程序运行时，动态数据会加载到内存中，要想定位变量位置，需要采用`基址+偏移`的方式来构成指针，在复杂的情况下，可能会有多次偏移，经过多层指针才能得到目标数据所在的位置，而这次的ffxiv就是如此。

通过其他工具直接获取数据地址，再使用自己编写的程序直接读取，这样未免太蠢，这个地址在每次ffxiv运行时都会变化，需要找到方法确定动态的基址和静态的偏移，两者结合便可找到数据的位置，无需借助其他工具。

而确定基址，便是确定`ffxiv_dx11.exe`的入口地址，在游戏进程下，会有多个模块，而模块入口地址的确定有多种方法，在这里我提供一种方法，在现情境下，我们只需找到`ffxiv_dx11.exe`的入口地址。

{% highlight c %}
#include <Tlhelp32.h>

DWORD dwGetModuleBaseAddress(DWORD dwProcessIdentifier, TCHAR *lpszModuleName)
{
	HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, dwProcessIdentifier);
	DWORD dwModuleBaseAddress = 0;
	if (hSnapshot == INVALID_HANDLE_VALUE) {
		cout << "ERROR: CreateToolhelp Failed" << endl;
		cout << GetLastError() << endl;
	}
	else
	{
		MODULEENTRY32 ModuleEntry32 = { 0 };
		ModuleEntry32.dwSize = sizeof(MODULEENTRY32);
		if (Module32First(hSnapshot, &ModuleEntry32))
		{
			do
			{
				if (_tcscmp(ModuleEntry32.szModule, lpszModuleName) == 0)
				{
					dwModuleBaseAddress = (DWORD)ModuleEntry32.modBaseAddr;
					cout << "OK" << endl;
					break;
				}
			} while (Module32Next(hSnapshot, &ModuleEntry32));
		}
		else {
			cout << "ERROR: Module32First Failed" << endl;
		}
		CloseHandle(hSnapshot);
	}
	return dwModuleBaseAddress;
}
{% endhighlight %}

使用`CreateToolhelp32Snapshot()`来获取ffxiv游戏进程的一份快照，该快照中含有进程下模块的信息。使用VS调试，可以看到模块的信息。

多次打开`ffxiv_dx11.exe`，获取`ffxiv_dx11.exe`的入口地址，观察规律，会发现有惊喜。