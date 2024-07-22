---
title: 使用Pyinjector对exe注入自定义Python代码
date: 2024/07/09 16:30
updated: 2024/07/09 16:30
tags: [逆向,Python,安全技术,渗透测试]
categories: 安全技术
description: 介绍一种能够注入exe后获取到python源代码的方法
cover: https://i1.wp.com/wx4.sinaimg.cn/large/a15b4afely1fnt9nsbnknj21hc0u0k6b.jpg
---

##  简介

当一个应用内置了Python解释器(一些打包工具[Pyinstaller、pyarmor]、游戏引擎[游戏逻辑依赖Python])时，可以通过这种方式注入自定义代码，从而获取到应用的Python相关源代码


## 尝试注入Pyinstaller打包的文件


写一个测试用的test.py然后给它打包

![](https://static.dawnnnnnn.com/2024/07/288a73a298e9dccf0faf1304ab0d5d7b.png)

![](https://static.dawnnnnnn.com/2024/07/3de59025c3c176681857f2dcd97f7d5d.png)

运行test.py

![](https://static.dawnnnnnn.com/2024/07/65cf997cd48822d6c9589cc9f4434791.png)

然后写一个code.py，和exe放到同一目录下，Pyinjector注入时会自动注入这个文件

![](https://static.dawnnnnnn.com/2024/07/ab9088a03db758ea25bf4466043e9042.png)

然后通过Process Hacker 2工具注入Pyinjector的x64 dll

![](https://static.dawnnnnnn.com/2024/07/ff90e36e3548f5cbbb30827e1222c7af.png)

![](https://static.dawnnnnnn.com/2024/07/e343356ce72b84dc3963bf6d9ee1af73.png)


注入效果

![](https://static.dawnnnnnn.com/2024/07/fc610879d88382aee0493248625a5b88.png)

现在有了dis就可以借助uncompyle6等工具还原Python源代码了，简单分析的话直接看dis结果也可以


## Pyinjector源码分析

[Pyinjector仓库](https://github.com/call-042PE/PyInjector)

其实这个项目非常简单，只需要关注两个文件`SDK.cpp`和`dllmain.cpp`

```c++
// SDK.cpp
void SDK::InitCPython()
{
    HMODULE hPython = 0x0;
    const char* pythonVersions[] = { "37", "38", "39", "310", "311", "312" }; // 定义一些支持的Python版本
    const int numVersions = sizeof(pythonVersions) / sizeof(pythonVersions[0]);

    for (int i = 0; i < numVersions; ++i) { // 遍历pythonVersions列表，使用GetModuleHandleA函数获取已经加载到进程地址空间的指定Python DLL模块的句柄
        char pythonDllName[15];
        snprintf(pythonDllName, sizeof(pythonDllName), "Python%s.dll", pythonVersions[i]);

        hPython = GetModuleHandleA(pythonDllName);

        if (hPython) //找到了就退出
            break;
    }

    // 通过GetProcAddress函数从Python3.x.dll中获取各种导出函数的函数指针
    Py_SetProgramName = (_Py_SetProgramName)(GetProcAddress(hPython, "Py_SetProgramName")); // 设置Python解释器的程序名
    PyEval_InitThreads = (_PyEval_InitThreads)(GetProcAddress(hPython, "PyEval_InitThreads")); // 初始化Python线程支持
    PyGILState_Ensure = (_PyGILState_Ensure)(GetProcAddress(hPython, "PyGILState_Ensure")); // 确保当前线程拥有全局解释器锁（GIL）
    PyGILState_Release = (_PyGILState_Release)(GetProcAddress(hPython, "PyGILState_Release")); // 释放全局解释器锁（GIL）
    PyRun_SimpleStringFlags = (_PyRun_SimpleStringFlags)(GetProcAddress(hPython, "PyRun_SimpleStringFlags")); // 执行Python代码字符串
}
```


```c++
//dllmain.cpp
DWORD WINAPI MainThread(HMODULE hModule)
{
    sdk.InitCPython(); //初始化
    Py_SetProgramName(sdk.random_string(10).c_str()); //设定一个名称
    PyEval_InitThreads(); // 初始化线程并获取全局解释器锁

    PyGILState_STATE s = PyGILState_Ensure(); // 数确保当前线程持有全局解释器锁
    PyRun_SimpleString(sdk.ReadFile("code.py").c_str()); // 读取code.py文件并运行这段代码
    //PyRun_SimpleString("import os, inspect\nwith open(\"code.py\",\"r\") as file:\n   data = file.read()\nexec(data)"); // OLD METHOD EASILY "BYPASSABLE" BY CREATING A JUNK METHOD NAMED EXEC
    PyGILState_Release(s); // 释放当前线程持有的全局解释器锁
    FreeLibraryAndExitThread(hModule, 0);
    CloseHandle(hModule);
}
```

分析完上面的关键代码，我认为有两个地方需要思考：
1. pythonVersions能否扩大支持范围？
   1. 我认为是可以扩大支持范围的，程序的原理是从Python[3.x].dll中获取导出函数的指针，那么假如Python2.7中含有PyRun_SimpleStringFlags导出函数，它理应可以获取到指针。在扩大范围的时候要先去Python对应版本源码中是否包含这个导出函数
2. Py_SetProgramName、PyEval_InitThreads、PyGILState_Ensure、PyGILState_Release是否是必要的？毕竟我们需要的功能只是执行代码
   1. 我认为这些函数都是不必要的，这些应该只是会影响程序的稳定性，假如不新建一个线程来执行代码的话，它可能会在主线程中被执行。当然，这个需要编译一下项目再实测一下，但我暂时没有编译环境


第二个问题对我们的逆向工作影响很大。对于一些特殊的打包环境，比如游戏引擎，它会将Python编译进它的引擎中，自然就找不到Python[version].dll。这时我们应该通过IDA等方式直接去找`PyRun_SimpleStringFlags`的函数地址，通过修改这个项目的相关代码就可以实现对特殊打包EXE的代码注入


## 总结

刚刚我们探讨了Pyinjector对Windows下exe的注入，那么能否对其它平台的可执行文件进行注入呢？其它平台也有一些动态链接库的注入工具，我认为原理上是可行的，但凭借这个项目肯定是不能