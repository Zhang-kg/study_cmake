学习链接：https://www.eglinux.com/cmake
## CMake 学习 Step1
**练习 1**

指定最低版本号：
cmake_minimum_required(VERSION 3.10)
创建 project
project(Turtorial)
添加可执行文件：
add_executable(Tutorial tutorial.cxx)

编译执行：
cmake -S . -B build（构建目录并运行 CMake）
cmake --build build（构建步骤）

**练习 2**

设置特殊变量：
set(CMAKE_CXX_STANDARD 11)
设置 CPP 标准：
set(CMAKE_CXX_STANDARD_REQUIRED True)

**练习 3**——添加版本号和配置头文件
可以在 CMakelists 中定义变量。例如打印版本
可以使用已配置的头文件。创建一个输入文件，其中包含一个或多个要替换的变量。变量有特殊语法，如 @var@。使用 configure_file 将输入文件复制给指定的输出文件，将变量替换为 CMakelists.txt 文件中 VAR 的值

设置 version
project(Tutorial VERSION 1.0)

输入文件中定义版本号，生成输出文件
configure_file(TutorialConfig.h.in TutorialConfig.h)

生成的 include 文件在 build 中，所以 cmakelist 中指定 include 文件去哪里找：
target_include_directories(Tutorial PUBLIC "${PROJECT_BINARY_DIR}")

输入文件中定义版本号变量
```cpp
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

文件中 include 对应的头文件后直接使用变量即可
```cpp
std::cout << argv[0] << " Version " << Tutorial_VERSION_MAJOR << "." << Tutorial_VERSION_MINOR << std::endl;
```

## Step 2 添加一个库

**练习 1** 添加一个库

使用 add_library 指定应由哪些源文件构成库。

可以使用一个或者多个子目录组织项目。
专门为库创建子目录，可以添加一个新的 CMakeLists.txt 一个或者多个源文件。
顶级 CmakeList 中使用 add_subdirectory

创建库后，通过 target_include_directories 和 target_link_libraries 连接到我们的可执行目标。

子目录中添加 library，并设置库的源文件
add_library(MathFunctions mysqrt.cxx)

使用新库，顶级 CMakeList 中设置：
```
add_subdirectory(MathFunctions)
```

链接新库到可执行文件
target_link_libraries(Tutorial PUBLIC MathFunctions)

指定库头文件：
```cmake
target_include_directories(Tutorial
    PUBLIC
        "${PROJECT_BINARY_DIR}"
        "${PROJECT_SOURCE_DIR}/MathFunctions"
)
```

**练习 2** 将库设置为可选

使用 option() 命令实现

设置一个变量，可以修改：
```cmake
option(USE_MYMATH "Use tutorial provided math implementation" ON)
```

使构建和链接 Math 库成为一个条件
```cmake
if(USE_MYMATH) 
	add_subdirectory(MathFunctions) 
	list(APPEND EXTRA_LIBS MathFunctions) 
	list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions") 
endif()
```

将 target_link_libraries 替换为 EXTRA_LIBS，**注意使用变量符号**
```cmake
target_link_libraries(Tutorial
    PUBLIC
        ${EXTRA_LIBS}
)
```

将 target_include_directories 替换，同样使用变量符号：
```cmake
target_include_directories(Tutorial
    PUBLIC
        "${PROJECT_BINARY_DIR}"
        ${EXTRA_INCLUDES}
)
```

为了使 Cmake 变量在 源代码中可以知晓，在 TutorialConfig.h.in 中使用如下方式：
```cpp
#cmakedefine USE_MYMATH
```

进而源代码中这样启动和关闭 Math 库：
```cpp
#ifdef USE_MYMATH
    #include "MathFunctions.h"
#endif
#ifdef USE_MYMATH
    const double outputValue = mysqrt(inputValue);
#else
    const double outputValue = sqrt(inputValue);
#endif
```

编译：
```shell
cmake -S . -B build -DUSE_MYMATH=OFF
cmake --build build
```

思考题：为什么在 USE_MYMATH 选项之后配置 TutorialConfig.h.in 很重要？如果我们将两者倒置会发生什么？

答案：我们配置之后是因为 TutorialConfig.h.in 使用了 USE_MYMATH 的值。如果我们在调用 option() 之前配置文件，我们将不会使用 USE_MYMATH 的预期值。


## Step 3 给库添加使用要求

**第一步** Step 2 中我们可以选择性链接 MathFunction 了，但是同时也要在主文件的 CMakeList 中添加很多的信息，导致主文件夹中的 CMakeLists 越来越满。从 CMake 2.x 到 CMake 3.x 开始是普通 CMake 到 Modern CMake 的转变，也是 directory-oriented CMake 到 target-oriented CMake 的转变。
target 类比一个对象，properties 就是对象的属性（成员函数、成员变量），public private 是属性的访问控制权限。

在 MathFunctions 库中的 CMakeList 中自己 include 自己：
```cmake
target_include_directories(MathFunctions
    INTERFACE
        ${CMAKE_CURRENT_SOURCE_DIR}
)
```
CMAKE_CURRENT_SOURCE_DIR 是当前 DIR

主文件夹 CMakeList 不需要设置 EXTRA_INCLUDES
```cmake
if(USE_MYMATH)
    add_subdirectory(MathFunctions)
    list(APPEND EXTRA_LIBS MathFunctions)
endif()
target_include_directories(Tutorial
    PUBLIC
        "${PROJECT_BINARY_DIR}"
)
```

这种方式下，只需要 target_link_libraries 就可以使用库。在传统方式下，手动指定库的依赖关系会变得非常复杂。

**练习 2**
使用 INTERFACE 库来指定 C++ 标准
CMake 中的 interface 库是一个空的库，没有任何源代码，同时定义了一组接口（API）以及链接其他库的依赖关系。

生成器表达式在 CMake 生成构建系统的时候根据不同配置动态生成特定的内容。
1. 条件链接：同一个编译目标，debug 和 release 版本链接不同的库文件
2. 条件定义，针对不同编译器定义不同的宏

实现这个目标的方式就是使用生成器表达式，这是在编译时自动生成的，根据条件确定是否生成。

生成器表达式的格式如 `$<>` 中间的是条件，生成器表达式可以嵌套。
生成器表达式支持的常用操作：
- 布尔表达式：逻辑运算符、字符串比较、变量查询
- 字符串值生成器表达式：条件表达式、


## Step 4 添加生成表达式

**练习 1** 使用接口库定义 C++ 标准

删除主目录 CMakeLists 中的 C++ 标准：
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

主目录 CMakeLists 中设置一个 interface 库，interface 库中没有任何源代码，仅仅设置一个标准
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

主目录 CMakeLists 中和 MathFunctions 库将 interface 库添加到 Tutorial 和 MathFunctions 中
target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)
target_link_libraries(MathFunctions tutorial_compiler_flags)

**练习 2** 使用生成式表达式添加编译警告标志
有条件地添加编译警告。
由于需要使用生成式表达式，所以设置最小的 CMake 版本为 3.15
cmake_minimum_required(VERSION 3.15)

确定系统使用哪个编译器来构建（警告标志根据编译器的不同而不同）
通过 COMPILE_LANG_AND_ID 生成式表达式完成，将结果设置在 gcc_like_cxx 和 msvc_cxx 中：
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
>这里可能是说如果使用的编译器是 xxx 其中之一，则是 gcc_like_cxx；如果是 xxx 其中之一，则是 msvc_cxx 编译器

给项目增加编译器警告标志，使用上面定义的 gcc_like_cxx 和 msvc_cxx 判断添加什么标志：
```cmake
target_compile_options(tutorial_compiler_flags
    INTERFACE
        "$<${gcc_like_cxx}:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>"
        "$<${msvc_cxx}:-W3>"
)
```
> 所有的编译选项现在都交给了 inference 库 tutorial_compiler_flags，所以这里通过也将编译选项交给库

只希望在构建期间使用这些编译选项（debug 版本），消费者不能看到编译警告（release 版本）
```cmake
target_compile_options(tutorial_compiler_flags
    INTERFACE
    "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
    "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)
```

<!-- # CMake 工程实践指南


本仓库是我(公众号：Eglinux)为了配合出 CMake 视频教程而建立的仓库，旨在记录一些 CMake 的基础知识以及视频教程中用到的例子。

CMake 学习交流群（如果二维码失效，请加我微信：eglinuxer，备注：CMake学习）：

<img src="./doc/picture/wechat.JPG" width="50%" height="50%">

## 0. 声明
本人知识有限，其中难免有不足之处。如果你发现什么地方有问题，欢迎指正，欢迎提 pull request。

本教程使用当前最新的 CMake 版本（VERSION 3.26.3）进行讲解，如果视频更新的过程中 CMake 更新了，那我也会同步使用最新的版本进行讲解。

课程对应的文档已全面转向我的个人网站，大家可以访问 https://www.eglinux.com/cmake/ 阅读文字版本教程。

---
以下目录内容会暂停一段时间，后续更新到 https://www.eglinux.com/cmake/。

## 1. 课程计划

### **第一部分：如何构建简单的可执行文件和库文件，这部分内容足以让你快速入门 CMake**
- [第 000 讲：工欲善其事必先利其器：CMake 最佳安装方法](./doc/000_how_to_install_cmake.md)
- [第 001 讲：使用 GitHub+ vscode + CMake 快速搭建一个 CMake 管理的项目仓库](./doc/001_github+vscode+cmake_to_build_a_repo.md)
- [第 002 讲：让 CMake 管理的项目真正工作起来：vscode + CMake 调试 C/C++ 项目](./doc/002_vscode+cmake_to_debug.md)
- [第 003 讲：CMake Targets 入门：CMake 如何构建简单的 Target](./doc/003_cmake_target_basic.md)

<font color="#dddd00">第一部分视频已全部更新，大家可以前往 [B站](https://www.bilibili.com/video/BV1vL41117xz/?spm_id_from=333.788&vd_source=70001201af6c9b750ff79c4703a168a6) 进行学习。</font> 

### **第二部分：全面介绍 CMake 的基础知识，为在大型项目中使 CMake 发挥最大的价值打下坚实的基础**
<font color="#dddd00">从第二部分开始，如果和平台无关的用法，我只会在一个平台演示，如果和平台相关的用法则会到用法支持的平台进行演示。</font> 

- [第 004 讲：CMake 变量之普通变量](./doc/004_cmake_var_basic.md)
- [第 005 讲：CMake 变量之环境变量](./doc/005_cmake_var_env.md)
- [第 006 讲：CMake 变量之缓存变量](./doc/006_cmake_var_cache.md)
- [第 007 讲：CMake 变量之作用域](./doc/007_cmake_var_scope.md)
- [第 008 讲：CMake 变量总结](./doc/008_cmake_var_summarize.md)
- [第 009 讲：CMake 字符串](./doc/009_cmake_string.md)
- [第 010 讲：CMake 列表](./doc/010_cmake_list.md)
- [第 011 讲：CMake 数学计算操作](./doc/011_cmake_math.md)
- [第 012 讲：CMake 流程控制之 if() 命令](./doc/012_cmake_if.md)
- [第 013 讲：CMake 流程控制之 for 循环](./doc/013_cmake_foreach.md)
- [第 014 讲：CMake 流程控制之 while 循环](./doc/014_cmake_while.md)
- [第 015 讲：CMake 流程控制之跳出循环和继续下一次循环](./doc/015_cmake_break_continue.md)
- [第 016 讲：如何使用子目录](./doc/016_cmake_add_subdirectory.md)
- [第 017 讲：子目录相关的作用域详解](./doc/017_scope_for_subdirectory.md)
- [第 018 讲：子目录中定义 project](./doc/018_project_for_subdirectory.md)
- [第 019 讲：CMake 命令之 include](./doc/019_cmake_include.md)
- [第 020 讲：项目相关的变量详解](./doc/020_project_relative_variables.md)
- [第 021 讲：CMake 提前结束处理命令：return](./doc/021_cmake_return.md)
- [第 022 讲：CMake 函数和宏基础](./doc/022_the_basics_of_functions_and_macros.md)
- [第 023 讲：CMake 函数和宏的参数处理基础](./doc/023_argument_handling_essentials.md)
- [第 024 讲：CMake 函数和宏之关键字参数](./doc/024_keyword_arguments.md)
- [第 025 讲：函数和宏返回值详解](./doc/025_functions_andmacros_returning_values.md)
- [第 026 讲：CMake 命令覆盖详解](./doc/026_overriding_commands.md)
- [第 027 讲：函数相关的特殊变量](./doc/027_special_variables_for_functions.md)
- [第 028 讲：复用 CMake 代码的其他方法](./doc/028_othrer_ways_of_invoking_cmake_code.md)
- [第 029 讲：CMake 处理参数时的一些问题说明](./doc/029_problems_with_argument_handling.md)
- [第 030 讲：CMake 属性通用命令](./doc/030_cmake_general_property_cmd.md)
- [第 031 讲：CMake 全局属性](./doc/031_cmake_global_property.md)
- [第 032 讲：目录属性](./doc/032_cmake_directory_property.md)
- [第 033 讲：Target 属性](./doc/033_cmake_target_property.md)
- [第 034 讲：源文件属性](./doc/034_cmake_source_property.md)
- [第 035 讲：CMake 其他属性](./doc/035_cmake_other_property.md)
- 
- 努力更新中...

### **第三部分：深入 CMake，探讨 CMake 精髓**
- 待更新
### **第四部分：CMake 工程实践，你要的这里都有**
- 待更新
### **第五部分：CMake 管理的开源项目带读，TA 有，我也要有**
- 待更新
### **第六部分：CMake 项目模板**
- 待更新
## 2. 如何学习

后续课程更新提醒，答疑等都会在知识星球上进行。为什么选择知识星球，因为知识星球是一个很好的可以将问答沉淀记录下来的地方。这样同样的问题，如果其他人遇到就不用再次提问了。

<img src="./doc/picture/zhishixingqiu.jpeg" width="100%" height="100%">

答疑：优先解答付费用户的疑问，当然免费用户的疑问我也会全部解答的，只是同一时间，如果有付费用户也在问问题，我将优先解答付费用户的问题。

## 3. 其他

其他未尽事宜，待后续补充。 -->