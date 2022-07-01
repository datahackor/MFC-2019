# 用VS2019编译MFC源代码(Compile MFC sources with Visual Studio 2019)
MFC源代码可以编译成私有的静态库或者DLL，本例支持 <b>ARM64</b> .
# 介绍
自从Visual C++ 版本 2008以后不再提供编译IDE绑定的MFC源代码，所需要的文件，比如 makefiles <b>atlmfc.mak</b> 和 DEF 文件不再提供。

作者因为需要一个无DAO类的x86 MFC DLL. 静态链接的方式，希望加入自己DAO类（兼容MySQL）, 这就需要覆盖目标文件 <b>dao*.obj</b> , 但是MFC object 文件不支持. 

因此需要构建自己的 MFC vcxproj 文件.

本工程包含全部的必要文件来构建 MFC 源代码成 DLL.

# 背景, DEF 文件和 LTCG(链接时代码生成)
第一个拦路虎就是缺失 DEF 文件. MFC DLL 通过一个DEF(模块定义文件)来导出它的符号. 链接器使用该文件来从DLL创建LIB文件. 老的 VC++ 2005 倒是有 DEF 文件，但是新版MFC添加了很多新的特征功能.
不过自Visual Studio 2015后，LTCG已经被废弃.

通过研究后决定用工具: <b>dumpexts.exe</b>. 来自己生成一个DEF文件.

此工具在 pre-link Build Event 时运行. 此时obj文件已经生成，dumpexts通过分析obj文件生成 DEF文件.

当首次构建 DLL 时, 要确保 LTCG(链接时代码生成)关闭. 因为使用LTCG的话生成的 object 文件就不能被 <b>dumpexts.exe</b> 识别. 不过产生完成DEF文件后LTCG是可以使用的. 

# 使用本代码
克隆本工程, 使用vs2019打开解决方案, 一个私有的MFC DLL就可以使用了!

为了避免版权纠纷, MFC 源代码是编译时从目录 $(VCToolsInstallDir) 获取. 本工程并不提供，但有一个文件例外: ctlpset.cpp. 原来的文件会产生编译错误, 因此本工程包含了该文件.

```
1>------ Build started: Project: MFCDLL, Configuration: Release Win32 ------
1>ctlpset.cpp
1>C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt\corecrt_memcpy_s.h(50): error C4789: buffer 'var' of size 24 bytes will be overrun; 24 bytes will be written starting at offset 8
1>Done building project "MFCDLL.vcxproj" -- FAILED.
```
错误在文件约395行的 sizeof(var), 修改为sizeof(var.llVal)!

当前的字符集配置为 UNICODE. 如果使用 Multi Byte Character Set 那么文件 <b>afxtaskdialog.cpp</b> 将被排除在外.

在MFC的静态库工程中，原始的文件 <b>mfcdll.mak</b> 中的文件 <b>dllmodul.cpp</b> 用不同的标志编译2次. 因此该文件被复制到文件 <b>dllmodulX.cpp</b> . 工程编译一个LIB文件，提供给使用MFC DLL的 可执行程序来链接(有点糊涂？).

MFCres 工程是个德语资源 DLL . 但未充分测试.

# 其它
作者不理解的是，需要修改IDE链接设定选项 /ENTRY 来指定程序入口才能完成构建(除了Release|Win32之外). UNICODE 模式被设定为 <b>wWinMainCRTStartup</b> .

# ARM64
为了成功编译 ARM64 文件 <b>dispimpl_supporting.h</b> 被重建. 在 <b>objcall_.s</b> 中 使用 <b>armasm.exe</b> 来创建 <b>objcall_.obj</b> 来解决未知符号的问题(unknown symbols):
```
 UNSUPPORTEDPLAT_PARAMS (_ARM64_PARAMS)
 UnsupportedplatParamsReset
 UnsupportedplatParamsAddFloat
 UnsupportedplatParamsAddDouble
 ```
为了测试 Active X Control 的功能，本例添加了 Circ 工程来测试ARM64的 floats 和 doubles 值. 打开APP的 Aboutbox 并点击  'Circle Active X control', 点击圆外边. 加载active X 控件使用无注册COM. 在 ARM64 , MessageBox 来显示传递 float 和 double. x86 版本的 circ.dll 必须以管理员来注册用 <b>regsvr32 circ.dll</b> .

构建 ARM64 前请先构建 dumpexts 的 x64 (Debug and Release)版本. 必须用 x64 dumpexts.exe 来构建 ARM64 MFC.
