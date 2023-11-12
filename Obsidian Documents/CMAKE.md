# 概述

弱类型语言

写到文件`CMakeList.txt`中。
![](Pasted%20image%2020230610185158.png)
# 变量

使用set进行变量命名，大小写敏感。
set(varName value)
所有变量都是string。set(myVar a b c) # myVar = "a;b;c"
可以使用未定义变量，相当于空字符串。
## 环境变量

可以使用set设置当前camke的环境变量
`set(ENV{PATH} "$ENV{PATH}:/opt/myDir")`
## 内置变量(预定义变量)

```bash
PROJECT_NAME : 通过 project() 指定项目名称
PROJECT_SOURCE_DIR : 工程的根目录
PROJECT_BINARY_DIR : 执行 cmake 命令的目录
CMAKE_CURRENT_SOURCE_DIR : 当前 CMakeList.txt 文件所在的目录CMAKE_CURRENT_BINARY_DIR : 编译目录，可使用 add subdirectory 来修改EXECUTABLE_OUTPUT_PATH : 二进制可执行文件输出位置LIBRARY_OUTPUT_PATH : 库文件输出位置
BUILD_SHARED_LIBS : 默认的库编译方式 ( shared 或 static ) ，默认为 static CMAKE_C_FLAGS : 设置 C 编译选项
CMAKE_CXX_FLAGS : 设置 C++ 编译选项
CMAKE_CXX_FLAGS_DEBUG : 设置编译类型 Debug 时的编译选项CMAKE_CXX_FLAGS_RELEASE : 设置编译类型 Release 时的编译选项
CMAKE_GENERATOR : 编译器名称
CMAKE_COMMAND : CMake 可执行文件本身的全路径CMAKE_BUILD_TYPE : 工程编译生成的版本， Debug / Release
```
# 函数

#### 添加源文件

aux_source_directory 查找在某个路径下的所有源文件。搜集所有在指定路径下的源文件的文件名，将输出结果列表储存在指定的变量中。定义参与编译的源代码文件
`aux_source_directory(< dir > < variable >)`

#### 编译生成动态/静态库

`add_library(库名称 (SHARED) 源文件)`SHARED代表动态库，默认为静态库
#### find_package(xxxx)
```bash
find_package() ： 有其搜索路径。有两种模式module和config。module模式会寻找cmake/Module/findxxx.cmake。方便在项目中引入外部依赖包。如果没有该.cmake文件可以自行制作。
1. 安装
# clone该项目
git clone https://github.com/google/glog.git 
# 切换到需要的版本 
cd glog
git checkout v0.40  

# 根据官网的指南进行安装
cmake -H. -Bbuild -G "Unix Makefiles"
cmake --build build
cmake --build build --target install

2. 引入
find_package(GLOG)
add_executable(glogtest glogtest.cc)
if(GLOG_FOUND)
    # 由于glog在连接时将头文件直接链接到了库里面，所以这里不用显示调用target_include_directories
    target_link_libraries(glogtest glog::glog)
else(GLOG_FOUND)
    message(FATAL_ERROR ”GLOG library not found”)
endif(GLOG_FOUND)
```

# 参考文档

- 公众号：[源知源为](https://mp.weixin.qq.com/s/SnFZkxoKaLeqXrDZWq7x2w)
- cmake cook book ，感觉写的一般
- cmake tutorial
