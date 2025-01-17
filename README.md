
## 模板配置


跟着网上的教程使用[evilashz师傅的模板](https://github.com)，下载模板解压至vs的模板目录：



```


|  | %UserProfile%\Documents\Visual Studio 2022\Templates\ProjectTemplates |
| --- | --- |


```

​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021057215-1358478105.png)​


创建新项目选择刚刚新增的类型：`Beacon Object File`​。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021059852-432000355.png)​


‍


## 环境适配


生成时报错，我使用的是2022版本的，模板有点老了他这里的是vs2019。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021101559-838614641.png)​


根据底下的提示从`项目`​ \-\> `重定目标解决方案`​， 接着确定更新即可


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021102418-1018587460.png)​


但是一进来模板会报错没有引入库, 干脆就用最小测试代码：将 `Source.cpp`​重名为`Source.c`​并修改为如下：



```


|  | #include |
| --- | --- |
|  | #include |
|  | #include "beacon.h" |
|  | void go(char* buff, int len) { |
|  | BeaconPrintf(CALLBACK_OUTPUT, "Hello BOF"); |
|  | } |


```

‍


## 编译配置


在上方的生成中勾选BOF配置， 配置管理器的编译环境也一样的，就可以生成64位版本的。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021103207-731008455.png)​


但最好还是使用批生成同时生成32位和64位版本：`生成`​ \-\> `批生成`​ 在BOF那两项勾选`Win32`​和`x64`​。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021105371-91105092.png)​


创建项目时没有勾选`将解决方案和项目放在同一目录下`​，那么生成的`.obj`​文件（编译未链接的目标文件）就在`/bin/BOF`​中。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021113448-1092886392.png)​


测试如果用cs的话可以使用`inline-execute E:\TARGET\timestamp.obj`​。我这里执行成功但发现有乱码：


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021114531-589786149.png)​


‍


### 乱码问题


尝试了加上`\n`​换行来终止字符串刷新缓冲区但是不行，找到使用格式化输出宏的办法，将可变参数展开。比如这里的`INFO_FORMAT("Hello BOF");`​会被展开成`BeaconPrintf(CALLBACK_OUTPUT, "[*] Hello BOF\n");`​。



```


|  | #include |
| --- | --- |
|  | #include |
|  | #include "beacon.h" |
|  |  |
|  | #define INFO_FORMAT(fmt, ...)    BeaconPrintf(CALLBACK_OUTPUT, "[*] " fmt "\n", ##__VA_ARGS__) |
|  |  |
|  | void go(char* buff, int len) { |
|  | INFO_FORMAT("Hello BOF"); |
|  | } |


```

原先的内存中可能是这样：`"Hello BOF" <未知内存内容>`​，但使用宏之后就是这样的：`"[*] Hello BOF\n" <确定的字符串终止>`​，最后测试也没有乱码了。


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021115982-2093782952.png)​


‍


## 功能实现


实现一个修改文件时间戳的功能， BOF不能直接调用Windows API， 而是通过cs提供的函数来交互。但我这里并不是为cs编写，所以要使用Windows API函数的话，首先需要进行声明：


### Windows API声明


要修改文件时间戳， 就要用到`SetFileTime`​。它用于设置文件的创建时间、访问时间和修改时间。文档中原型如下：



```


|  | BOOL SetFileTime( |
| --- | --- |
|  | [in]           HANDLE         hFile, |
|  | [in, optional] const FILETIME *lpCreationTime, |
|  | [in, optional] const FILETIME *lpLastAccessTime, |
|  | [in, optional] const FILETIME *lpLastWriteTime |
|  | ); |


```

‍


* hFile: 文件句柄，必须有FILE\_WRITE\_ATTRIBUTES访问权限
* lpCreationTime: 文件的创建时间
* lpLastAccessTime: 文件的最后访问时间
* lpLastWriteTime: 文件的最后修改时间


那么在bof的声明中要注意这个函数是属于哪个dll， 比如这里是`kernel32.dll`​的话那要定义和调用它时就写成`KERNEL32$SetFileTime`​，完整如下：



```


|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$SetFileTime(HANDLE, const FILETIME*, const FILETIME*, const FILETIME*); |
| --- | --- |


```

cs使用这种前缀可以让BOF直接调用DLL中的原生函数， 就不需要再在导入表中声明了，这样也可以缩小BOF体积。类似的使用`CreateFileA`​来创建或打开文件时， 其原型如下：



```


|  | HANDLE CreateFileA( |
| --- | --- |
|  | [in]           LPCSTR                lpFileName, 			  // 文件名 |
|  | [in]           DWORD                 dwDesiredAccess,  	  // 访问模式 |
|  | [in]           DWORD                 dwShareMode,			  // 共享模式 |
|  | [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,  // 安全描述符 |
|  | [in]           DWORD                 dwCreationDisposition, // 创建方式 |
|  | [in]           DWORD                 dwFlagsAndAttributes,  // 文件属性 |
|  | [in, optional] HANDLE                hTemplateFile		  // 模板文件句柄 |
|  | ); |


```

BOF中声明则如下：



```


|  | DECLSPEC_IMPORT HANDLE WINAPI KERNEL32$CreateFileA(LPCSTR, DWORD, DWORD, LPSECURITY_ATTRIBUTES, DWORD, DWORD, HANDLE); |
| --- | --- |


```

以及其他要用到的api可以这样声明：



```


|  | // 其他必要的API |
| --- | --- |
|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$CloseHandle(HANDLE); // 关闭一个内核对象（如文件）的句柄 |
|  | DECLSPEC_IMPORT VOID WINAPI KERNEL32$GetSystemTime(LPSYSTEMTIME); // 获取当前系统时间（UTC时间） |
|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$SystemTimeToFileTime(LPSYSTEMTIME, LPFILETIME); // 将SYSTEMTIME结构转换为FILETIME结构。 |


```

‍


### 参数处理


BOF的入口函数就是这里的go， `inline-execute`​执行BOF时先调用这个。其中先定义并初始化一个解析器来解析传入的参数，`timestamp`​这个至少也是要一个参数路径的，先从一个来：



```


|  | void go(char* buff, int len) { |
| --- | --- |
|  | datap parser; |
|  | char* filepath; |
|  |  |
|  | // 解析Beacon传入的参数 |
|  | BeaconDataParse(&parser, buff, len); |
|  | filepath = BeaconDataExtract(&parser, NULL); |
|  |  |
|  | // 参数验证 |
|  | if (!filepath) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] please provide file path"); |
|  | return; |
|  | } |
|  | } |


```

那解析多个参数呢， 一样的：



```


|  | BeaconDataParse(&parser, buff, len); |
| --- | --- |
|  | sourceFile = BeaconDataExtract(&parser, NULL); |
|  | targetFile = BeaconDataExtract(&parser, NULL); |
|  |  |
|  | if (!sourceFile || !targetFile) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[!] Error: Two file paths required\n"); |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] Usage: inline-execute timestamp.o \"source_file\" \"target_file\"\n"); |
|  | return; |
|  | } |
|  |  |
|  | BeaconPrintf(CALLBACK_OUTPUT, "[-] Source: %s\n", sourceFile); |
|  | BeaconPrintf(CALLBACK_OUTPUT, "[-] Target: %s\n", targetFile); |


```

‍


### 时间处理


接着继续，获取系统时间然后修改成我们希望的时间，比如`2020年1月1日 00:00:00`​。然后把他转换为文件时间格式：



```


|  | SYSTEMTIME st; |
| --- | --- |
|  | FILETIME ft; |
|  | KERNEL32$GetSystemTime(&st); |
|  |  |
|  | st.wYear = 2020; |
|  | st.wMonth = 1; |
|  | st.wDay = 1; |
|  | st.wHour = 0; |
|  | st.wMinute = 0; |
|  | st.wSecond = 0; |
|  |  |
|  | KERNEL32$SystemTimeToFileTime(&st, &ft); |


```

‍


### 文件操作


准备好了要修改的时间后就尝试打开文件获取句柄：



```


|  | HANDLE hFile = KERNEL32$CreateFileA( |
| --- | --- |
|  | filepath,                         			// 文件路径 |
|  | FILE_WRITE_ATTRIBUTES,           			// 只需要写属性权限 |
|  | FILE_SHARE_READ | FILE_SHARE_WRITE, 		// 允许其他进程读写 |
|  | NULL,                            			// 默认安全属性 |
|  | OPEN_EXISTING,                   			// 只打开已存在的文件 |
|  | FILE_ATTRIBUTE_NORMAL,          		    // 使用标准属性 |
|  | NULL                            			// 不使用模板 |
|  | ); |
|  |  |
|  | if (hFile == INVALID_HANDLE_VALUE) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] can not open file: %s", filepath); |
|  | return; |
|  | } |


```

‍


### 时间戳修改


最后使用`SetFileTime`​修改三个时间属性：创建时间、访问时间、修改时间。结束后关闭句柄。



```


|  | if (!KERNEL32$SetFileTime(hFile, &ft, &ft, &ft)) { |
| --- | --- |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] failed to change timestamp"); |
|  | } else { |
|  | BeaconPrintf(CALLBACK_OUTPUT, "[+] success: %s", filepath); |
|  | } |
|  |  |
|  | KERNEL32$CloseHandle(hFile); |


```

这样就简单完成了修改一个文件时间戳的功能，完整代码如下：



```


|  | #include |
| --- | --- |
|  | #include |
|  | #include "beacon.h" |
|  |  |
|  | // 声明Windows API函数 |
|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$SetFileTime(HANDLE, const FILETIME*, const FILETIME*, const FILETIME*); |
|  | DECLSPEC_IMPORT HANDLE WINAPI KERNEL32$CreateFileA(LPCSTR, DWORD, DWORD, LPSECURITY_ATTRIBUTES, DWORD, DWORD, HANDLE); |
|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$CloseHandle(HANDLE); |
|  | DECLSPEC_IMPORT VOID WINAPI KERNEL32$GetSystemTime(LPSYSTEMTIME); |
|  | DECLSPEC_IMPORT BOOL WINAPI KERNEL32$SystemTimeToFileTime(LPSYSTEMTIME, LPFILETIME); |
|  |  |
|  | void go(char* buff, int len) { |
|  | datap parser; |
|  | char* filepath; |
|  |  |
|  | BeaconDataParse(&parser, buff, len); |
|  | filepath = BeaconDataExtract(&parser, NULL); |
|  |  |
|  | if (!filepath) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] please provide file path"); |
|  | return; |
|  | } |
|  |  |
|  | SYSTEMTIME st; |
|  | FILETIME ft; |
|  | KERNEL32$GetSystemTime(&st); |
|  |  |
|  | st.wYear = 2020; |
|  | st.wMonth = 1; |
|  | st.wDay = 1; |
|  | st.wHour = 0; |
|  | st.wMinute = 0; |
|  | st.wSecond = 0; |
|  |  |
|  | KERNEL32$SystemTimeToFileTime(&st, &ft); |
|  |  |
|  | HANDLE hFile = KERNEL32$CreateFileA( |
|  | filepath, |
|  | FILE_WRITE_ATTRIBUTES, |
|  | FILE_SHARE_READ | FILE_SHARE_WRITE, |
|  | NULL, |
|  | OPEN_EXISTING, |
|  | FILE_ATTRIBUTE_NORMAL, |
|  | NULL |
|  | ); |
|  |  |
|  | if (hFile == INVALID_HANDLE_VALUE) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] can not open file: %s", filepath); |
|  | return; |
|  | } |
|  |  |
|  | if (!KERNEL32$SetFileTime(hFile, &ft, &ft, &ft)) { |
|  | BeaconPrintf(CALLBACK_ERROR, "[-] failed to change timestamp"); |
|  | } |
|  | else { |
|  | BeaconPrintf(CALLBACK_OUTPUT, "[+] sunccess: %s", filepath); |
|  | } |
|  |  |
|  | KERNEL32$CloseHandle(hFile); |
|  | } |


```

‍


## 测试


编译还是同上使用批生成，我这里测试的可以成功修改：


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021129263-1358463062.png)​


‍


## 优化编译


为了更好的在苛刻环境下使用，我想继续压缩体积，找到的参数以及解释如下：


* \-Os: 优化大小（比\-O2生成更小的代码）
* \-fno\-asynchronous\-unwind\-tables: 禁用异常展开表
* \-fno\-ident: 删除编译器版本信息
* \-fpack\-struct\=8: 结构体8字节对齐
* \-falign\-functions\=1: 函数1字节对齐
* \-s: 删除符号表
* \-ffunction\-sections: 每个函数放入单独的段
* \-fdata\-sections: 每个数据项放入单独的段
* \-fno\-exceptions: 禁用异常处理
* \-fno\-stack\-protector: 禁用栈保护
* \-mno\-stack\-arg\-probe: 禁用栈探测


64位使用的编译命令如下:



```


|  | x86_64-w64-mingw32-gcc-8.1.0.exe -c .\Source.c -o timestamp.o -Os -fno-asynchronous-unwind-tables -fno-ident -fpack-struct=8 -falign-functions=1 -s -ffunction-sections -fdata-sections -fno-exceptions -fno-stack-protector -mno-stack-arg-probe |
| --- | --- |


```

针对于编译32位版本的命令（ 如果没有就用批生成， 重命名即可）:



```


|  | i686-w64-mingw32-gcc-8.1.0.exe -c .\Source.c -o timestamp.x86.o -Os -fno-asynchronous-unwind-tables -fno-ident -fpack-struct=8 -falign-functions=1 -s -ffunction-sections -fdata-sections -fno-exceptions -fno-stack-protector -mno-stack-arg-probe |
| --- | --- |


```


> 注：这里生成的是.o而不是.obj只是自己的需求为了统一一下，obj是Windows平台的默认目标文件扩展名，而.o是Unix/Linux平台的扩展名。它们本质和功能上是一样的，只是命名习惯不同。


‍


## 最后


这里只是简单的示例，要使用最好要有一个锚定文件，以他的时间作为目标来修改。细节不赘述，详细请跳转[Github](https://github.com)。最终版本的使用测试如下:


​![image](https://img2023.cnblogs.com/blog/3038812/202501/3038812-20250108021136553-589312315.png)​


‍


## 参考


* [Visual\-Studio\-BOF\-template](https://github.com)
* [【武器开发】\| 开发你的第一个BOF](https://github.com):[wgetcloud全球加速服务机场](https://wa7.org)
* [GCC Optimization Options](https://github.com)
* [Windows API](https://github.com)


