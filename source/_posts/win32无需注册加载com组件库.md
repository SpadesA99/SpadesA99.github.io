---
title: win32无需注册加载com组件库
layout: post
tags:
  - com
  - alt
categories:
  - c++
date: 2022-06-24 12:09:10
---



### win32无需注册加载com组件库

##### 前言

​	常规情况下win32 com组件都是需要注册到系统才能调用，实际上还有一种方法可以不进行注册直接加载 com组件。

在vs安装目录下面的 DIA SDK 有一个函数 ```NoRegCoCreate``` 可以通过dll文件直接加载com组件库。

函数原型

```c++
HRESULT STDMETHODCALLTYPE NoRegCoCreate(  const __wchar_t *dllName,
                        REFCLSID   rclsid,
                        REFIID     riid,
                        void     **ppv);
```

头文件

```c++
#include "dia2.h"
#include "diacreate.h"
#pragma comment(lib, "diaguids.lib")
```

示例代码

```c++
const IID IID_IWinHttpRequest = { 0x06f29373,0x5c5a,0x4b54,{0xb0, 0x25, 0x6e, 0xf1, 0xbf, 0x8a, 0xbf, 0x0e} };

void TestWinHttp() {
	//初始化com库
	auto hr = CoInitialize(NULL);
	if (FAILED(hr))
	{
		printf("[-] CoInitialize failed. code %X", hr);
		return;
	}
	defer(CoUninitialize(););

	CLSID clsid;
	hr = CLSIDFromProgID(L"WinHttp.WinHttpRequest.5.1", &clsid);
	if (FAILED(hr))
	{
		printf("[-] CLSIDFromProgID failed. code %X", hr);
		return;
	}

	IWinHttpRequest* pIWinHttpRequest = NULL;
	hr = NoRegCoCreate(L"C:\\Windows\\SysWOW64\\winhttpcom.dll", clsid, IID_IWinHttpRequest, (void**)&pIWinHttpRequest);
	if (FAILED(hr))
	{
		printf("[-] Create cominterface failed. code %X", hr);
		return;
	}
	defer(if (pIWinHttpRequest) { pIWinHttpRequest->Release(); });

	VARIANT varFalse;
	VariantInit(&varFalse);
	V_VT(&varFalse) = VT_BOOL;
	V_BOOL(&varFalse) = VARIANT_FALSE;
	hr = pIWinHttpRequest->Open(L"GET", "https://www.baidu.com/", varFalse);
	if (FAILED(hr))
	{
		printf("[-] pIWinHttpRequest->Open failed. code %X", hr);
		return;
	}

	VARIANT         varEmpty;
	VariantInit(&varEmpty);
	V_VT(&varEmpty) = VT_ERROR;
	hr = pIWinHttpRequest->Send(varEmpty);
	BSTR bstrResponse = pIWinHttpRequest->GetResponseText();
	if (bstrResponse)
	{
		std::wstring body = bstrResponse;
		std::wcout << body.c_str() << std::endl;
		SysFreeString(bstrResponse);
	}
}
```

