---
layout: post
title:  "常量区修改"
author: "daweibayu"
tags: C++
excerpt_separator: <!--more-->
---

<!--more-->

```c++
#include <Windows.h>
#include <stdio.h>
int main()
{
	const char* a = "123456";
	const char* b = "123456";
	char* c = "654321";

	DWORD oldprot; 
	HANDLE hProcess = GetCurrentProcess(); 
	VirtualProtectEx(hProcess, (LPVOID)b, 7, PAGE_EXECUTE_READWRITE, &oldprot);
	WriteProcessMemory(hProcess, (LPVOID)b, (LPVOID)c, 7, NULL);

	printf("%s \n", a);
}
```

以上程序输出  654321



“123456”是储存在常量区的，也就是说是在编译的时候就确定的，a、b、c只是一个指向常量区的指针。

由于编译器的优化，此时 a = b

VirtualProtectEx : Changes the protection on a region of committed pages in the virtual address space of a specified process.

WriteProcessMemory:Writes data to an area of memory in a specified process. The entire area to be written to must be accessible or the operation fails.