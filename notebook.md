# 一. 主要记录学习rl_sar过程中的思考与一些基础知识点
## 1.rl_sim.cpp & rl_sim.hpp & CMakeLists.txt
> 目前只关注ROS1，因此暂时不看ROS2相关代码。

1.1 param\<T>(key, var, default)  
param是一个C++的模版函数
\<T>是模版参数，代表读取的参数类型；代码中\<std::string>说明读取的数据是string字符类型。  
key：第一个参数 "ros_namespace" 是你要查找的参数名。  
var：第二个参数 this->ros_namespace 是接收值的变量。  
default：第三个参数 "" 是默认值：如果参数服务器上没有这个参数，就把空字符串赋值给 this->ros_namespace。  
robot_name同理  

1.2 void RL::ReadYamlBase(std::string robot_path)函数  
该函数在rl_sdk.cpp中，下面看里面具体写了什么内容:  
首先获取机器人配置文件路径config_path，详细请查看1.3  
接着通过YAML-CPP库将.yaml文件中的内容读取到ModelParams params 的结构体中。其中一些读取向量的操作等需要的时候细看。

1.3 add_definitions(-DCMAKE_CURRENT_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}")  

- 其中CMAKE_CURRENT_SOURCE_DIR是CMake的内置变量，可以获取当前CMakeLists.txt的绝对路径；  
add_definitions()的作用是向所有编译单元添加编译器宏定义，相当于在编译时加上 -D 参数。  
`例如：add_definitions(-DDEBUG) 相当于在每个cpp文件中写入#define DEBUG。`

- add_definitions() 是老接口，不支持作用域控制，不能针对于某个target，只能全局添加，从CMake3.0起推荐使用add_compile_definitions()或者target_compile_definitions()。  
下面是对add_compile_definitions的测试：  
`add_compile_definitions(CMAKE_CURRENT_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}")测试通过`  
下面是target_compile_definitions的写法：  
`target_compile_definitions(my_target PRIVATE CURRENT_SRC_DIR="${CMAKE_CURRENT_SOURCE_DIR}")`  
其中my_target是你在 CMake 中指定的目标（比如 main.cpp 编译生成的可执行文件、通过 add_executable() 或 add_library() 创建的目标）  
如果第二个参数设置为PRIVATE代表只会影响my_target，不会传递到其他依赖的目标。如果你想让其它依赖于 my_target 的目标也能使用这个宏定义，可以改成 INTERFACE 或 PUBLIC。  

1.4 YAML-CPP开源库  
YAML::Node 是 YAML-CPP 库中的一个核心类，代表 YAML 文件中的任意一个“节点”。  
它是一个变种容器（variant）类型，可以表示：
- 一个 标量值（如 3.14、"robot1"）；  
- 一个 序列（如 [1, 2, 3]，对应 C++ 中的 std::vector）；
- 一个 映射（如 {name: "robot", dt: 0.002}，对应 std::map 或 std::unordered_map）；
- 或者是嵌套结构（节点中有节点）。  
也就是说，一个 YAML::Node 可以代表 YAML 文件的任何层级的结构。  
示例代码：  
> config.yaml:  
robot:  
&emsp; name: "go1"  
&emsp; dt: 0.002  
&emsp; sensors: [imu, camera, lidar]  


>#include <yaml-cpp/yaml.h>  
#include <iostream>  
#include <vector>  
int main() {  
&emsp; YAML::Node config = YAML::LoadFile("config.yaml");  
&emsp; YAML::Node robot_node = config["robot"];       // 一个 map 类型的 node  
&emsp; std::string name = robot_node["name"].as<std::string>();  
&emsp; double dt = robot_node["dt"].as<double>();   
&emsp; std::vector<std::string> sensors = robot_node["sensors"].  as<std::vector<std::string>>();  
&emsp; std::cout << "robot_node is a map with fields: name, dt, sensors" << std::endl;  
&emsp; std::cout << "name: " << name << std::endl;  
&emsp; std::cout << "dt: " << dt << std::endl;  
&emsp; std::cout << "sensors: ";  
&emsp; for (const auto& s : sensors)  
&emsp; &emsp; std::cout << s << " ";  
&emsp; std::cout << std::endl;  
&emsp; return 0;  
}  

输出结果：  
> robot_node is a map with fields: name, dt, sensors  
name: go1  
dt: 0.002  
sensors: imu camera lidar  

- YAML::LoadFile() 是 YAML-CPP 库提供的函数，用来从磁盘上加载一个 YAML 文件并返回一个 YAML::Node 类型的对象，即YAML格式的树结构。并且可以通过返回的Node访问任何加载的数据。这也可以解释了Yaml文件书写的时候是有规范的。  
当加载的根节点不存在时，会返回一个空的节点。  

- .as<T>() 是 YAML-CPP 中的一个成员函数，它用来将 YAML::Node 中的内容转换为 C++ 中的 指定类型 T。它是模板函数，可以根据你指定的类型 T 来解析 YAML 节点中的数据，并转换为相应的 C++ 类型。  

1.5 FSMManager  
todo  

1.6 CMake中的set()  
todo  


1.7 signal(SIGINT, signalHandler)
- 其中signal是一个信号处理函数，SIGINT表示终止进程，由Ctrl+C触发，该函数的功能是收到STGINT信号后，不退出，而是执行signalHandler函数。  

- 在linux系统中，有很多的标准信号，他们都可以被触发且可以进行信号处理。  

| 信号名    | 数值(常见) | 默认动作        | 说明                                 |  
|-----------|-----------|---------------|--------------------------------------|  
| SIGHUP    | 1         | 终止进程      | 终端挂起或控制进程终止               |  
| SIGINT    | 2         | 终止进程      | 终端中断 (Ctrl+C)                    |  
| SIGQUIT   | 3         | Core Dump     | 退出并生成core (Ctrl+\)             |  
| SIGILL    | 4         | Core Dump     | 非法指令                             |  
| SIGTRAP   | 5         | Core Dump     | 断点或调试异常                       |  
| SIGABRT   | 6         | Core Dump     | 调用abort()                          |  
| SIGBUS    | 7         | Core Dump     | 总线错误                             |  
| SIGFPE    | 8         | Core Dump     | 浮点异常(如除以0)                   |  
| SIGKILL   | 9         | 终止进程      | 强制杀死进程(不可捕获、不可忽略)     |  
| SIGUSR1   | 10        | 终止进程      | 用户自定义信号1                      |  
| SIGSEGV   | 11        | Core Dump     | 无效内存访问(段错误)                 |  
| SIGUSR2   | 12        | 终止进程      | 用户自定义信号2                      |  
| SIGPIPE   | 13        | 终止进程      | 向已关闭的管道写数据                 |  
| SIGALRM   | 14        | 终止进程      | 定时器alarm()到期                    |  
| SIGTERM   | 15        | 终止进程      | kill默认发送的终止信号               |  
| SIGCHLD   | 17        | 忽略          | 子进程退出时发送给父进程             |  
| SIGCONT   | 18        | 继续进程      | 恢复被暂停的进程                     |  
| SIGSTOP   | 19        | 停止进程      | 暂停进程(不可捕获、不可忽略)         |  
| SIGTSTP   | 20        | 停止进程      | 终端暂停(Ctrl+Z)                     |  
| SIGTTIN   | 21        | 停止进程      | 后台进程尝试从终端读取               |  
| SIGTTOU   | 22        | 停止进程      | 后台进程尝试向终端写入               |  
  

1.8 基于范围的for循环
for (const auto& name : names)  
{  
    std::cout << name << std::endl;  
}  
这是范围循环的语法，相当于遍历容器names中的每个元素，将每个元素赋值给name。  

1.9 execlp()  
execlp() 是一个 C 标准库函数，用于在当前进程中执行一个新的程序。  
它的函数原型为：  
```c  
int execlp(const char *file, const char *arg, ...);  
```  
- file 参数指定要执行的程序的路径名。  
- arg 参数指定要传递给新程序的命令行参数，必须以 NULL 结尾。  
- execlp() 会替换当前进程的代码段和数据段，因此调用它后，不会返回。  
- 在本代码中采用sh，其中第一个sh代表从/bin/sh中查找sh程序，第二个sh代表sh程序的路径名，-c代表后面的命令是一个字符串，而不是一个文件名。shell是命令行解释器，用于解释和执行用户输入的命令。  
- 本函数参数的结尾一定是NULL/nullptr，否则会导致未定义行为。  

1.10 对于std::map类型的数据详细见：  
https://blog.csdn.net/weixin_74396256/article/details/143272463  

1.11 对于std::function和std::bind的详细介绍：  
https://blog.csdn.net/bandaoyu/article/details/106084948  

1.12 