尝试了一下，Windows 下 Codeblocks 利用 Mingw 编译器编写动态链接库，结果虽然 dll 是生成了，但过程还是不太满意。因为 dll 是 Windows 下才用到的，似乎用 GNU 的编译器不太合适，最终还是改用 VS2015 (虽然它很‘臃肿’)。然后，我就尝试在 Linux 上编写`.so`(shared object, 共享库。和 dll 类似)文件。    
在 Linux 上，我尝试了 JetBrains 家的 Clion ，它的代码提示，确实比 Codeblock 好，这点值得肯定。而且在 Linux 上的构建速度比在 Windows 上快(相同硬件条件下) 。虽然 Clion 是收费软件，但是有学生优惠。我用 edu 邮箱申请一年期的免费使用权，到期之后还可以用 edu 邮箱再次验证使用。在能力范围之内，能不用破解软件，就不用破解软件。要**尊重同行的劳动**。 

回到正题，静态库、动态库在不同系统下的对应关系：  
linux： `.a` (Archive libraries) 和 `.so`(Shared object)  ;     
Windows： `.lib` 和 `.dll`(Dynamic-link library)

### 先编写一个库试试  
首先用 Clion 新建一个项目 "hello"，main.cpp 的内容为：  

```
#include <iostream>
using namespace std;

void hi()
{
    cout << "Hello,nihao" << endl;
}
int add(int a,int b)
{
    return a + b;
}
```

CMakeLists.txt 的内容：  
```
cmake_minimum_required(VERSION 3.6)
project(hello)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

set(SOURCE_FILES main.cpp)
#动态库
add_library(hello SHARED ${SOURCE_FILES})
#静态库
add_library(hell ${SOURCE_FILES})
```
编译生成`libhello.so`。 可用`nm -C libhello.so`来查看符号表：  
```
0000000000201040 B __bss_start
0000000000201040 b completed.7585
                 U __cxa_atexit@@GLIBC_2.2.5
                 w __cxa_finalize@@GLIBC_2.2.5
0000000000000840 t deregister_tm_clones
00000000000008d0 t __do_global_dtors_aux
0000000000200dd0 t __do_global_dtors_aux_fini_array_entry
0000000000201038 d __dso_handle
0000000000200de0 d _DYNAMIC
0000000000201040 D _edata
0000000000201048 B _end
00000000000009e4 T _fini
0000000000000910 t frame_dummy
0000000000200dc0 t __frame_dummy_init_array_entry
0000000000000af0 r __FRAME_END__
0000000000201000 d _GLOBAL_OFFSET_TABLE_
00000000000009cf t _GLOBAL__sub_I_main.cpp
                 w __gmon_start__
00000000000009fc r __GNU_EH_FRAME_HDR
00000000000007b8 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000200dd8 d __JCR_END__
0000000000200dd8 d __JCR_LIST__
                 w _Jv_RegisterClasses
0000000000000880 t register_tm_clones
0000000000201040 d __TMC_END__
0000000000000940 T hi()
0000000000000972 T add(int, int)
0000000000000986 t __static_initialization_and_destruction_0(int, int)
                 U std::ostream::operator<<(std::ostream& (*)(std::ostream&))@@GLIBCXX_3.4
                 U std::ios_base::Init::Init()@@GLIBCXX_3.4
                 U std::ios_base::Init::~Init()@@GLIBCXX_3.4
                 U std::cout@@GLIBCXX_3.4
                 U std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)@@GLIBCXX_3.4
00000000000009ed r std::piecewise_construct
0000000000201041 b std::__ioinit
                 U std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)@@GLIBCXX_3.4
```


### 动态库显式调用(运行时装入)  
新建一个项目 "openso"，以下是`main.cpp`的内容：  
```
#include <iostream>
#include "dlfcn.h"//显式调用的头文件

using namespace std;

int main() {

    cout << "C++ dlopen" << endl;

    // 打开库文件
    cout << "Opening libhello.so..." << endl;
    //动态库 libhello.so 的绝对路径
    void *handle = dlopen("/home/yang/Desktop/openso/lib/libhello.so", RTLD_LAZY);

    if (!handle) {
        cerr << "Cannot open library: " << dlerror() << endl;
        return 1;
    }

    // 加载符号表
    cout << "Loading symbol hi..." << endl;
    typedef void (*hi_t)();

    // 错误
    dlerror();
    hi_t hi = (hi_t) dlsym(handle, "hi");
    const char *dlsym_error = dlerror();
    if (dlsym_error) {
        cerr << "Cannot load symbol 'hi': " << dlsym_error << endl;
        dlclose(handle);
        return 1;
    }

    cout << "Calling hi()..." << endl;
    hi();

    // 加载符号表
    cout << "Loading symbol add..." << endl;
    typedef int (*add_t)(int, int);

    // 错误
    dlerror();
    add_t add = (add_t) dlsym(handle, "add");
    if (dlsym_error) {
        cerr << "Cannot load symbol 'add': " << dlsym_error << endl;

        dlclose(handle);
        return 1;
    }

    cout << "Calling the add()..."<< endl;
    cout << add(2, 3) << " is the result"<< endl;

    // 关闭库文件
    cout << "Closing library..."<< endl;
    dlclose(handle);
}
```

CMakeLists.txt 的内容： 
```
cmake_minimum_required(VERSION 3.6)
project(openso)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(SOURCE_FILES main.cpp)
add_executable(openso ${SOURCE_FILES})
target_link_libraries(openso ${CMAKE_DL_LIBS})
```
编译，也可以在项目的根目录下执行`g++ -o main main.cpp -ldl`。

### 动态库隐式调用(编译时装入)  
新建一个项目 "hideso"，以下是`main.cpp`的内容：  
```
#include <iostream>
using namespace std;

void hi();
int  add(int,int);

int main(int argc,char *argv[])
{
    hi();
    cout << endl;
    cout << add(13,12) << endl;
    return 0;
}  
```

CMakeLists.txt 的内容：  
```   
cmake_minimum_required(VERSION 3.6)
project(hideso)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

set(SOURCE_FILES main.cpp)
add_executable(hideso ${SOURCE_FILES})
target_link_libraries(hideso /home/yang/Desktop/hideso/libhello.so)
```
编译，也可以在项目的根目录下执行` g++ -o main main.cpp ./libhello.so`。

在执行隐式链接的程序之前，需要设置 LD_LIBRARY_PATH 环境变量，或者把前面生成的 libhello.so 复制到系统路径下，否则报错：  
```
error while loading shared libraries: libhello.so: cannot open shared object file: No such file or directory
```
运行可在项目根目录下执行：  
```
export LD_LIBRARY_PATH=.
./main
```
还可直接用`ldd`命令查看其所隐式调用的库。  
对于C语言编译的库，C++ 调用时需要这样做(保证C/C++ 兼容性，注意`cplusplus`前面是两个'_')：    
```
#include <iostream>
using namespace std;

#ifdef __cplusplus
extern "C"
{
#endif
void hi();
int add(int,int);
#ifdef __cplusplus
}
#endif

int main(int argc,char *argv[])
{
    hi();
    cout << endl;
    cout << add(12,13) << endl;
    return 0;
}
```
当然了，C++ 编写的库，C 是无法调用的。

### CMake 相关:       
> [CMake Wiki](https://cmake.org/Wiki/CMake)  
> [CMake 常用命令和变量](http://elloop.github.io/tools/2016-04-10/learning-cmake-2-commands)  
> [CMake使用进阶](http://linghutf.github.io/2016/06/16/cmake/)  
