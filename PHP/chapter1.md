## PHP基本架构

### 一、php简介

#### php的相关组成

- SAPI

SAPI是PHP的接入层，接收用户的请求，然后调用php内核提供的一些接口完成php脚本的执行，所以严格意义上SAPI并不算php内核的一部分

php中常用的SAPI有cli、php-fpm，cli是命令行下执行php脚本的实现：bin/php script.php 而php-fpm比较复杂，实现了网络处理模块，用于与web服务器交互

- Zend引擎

php代码从编译到执行都是由Zend完成的，Zend整体由两个部分组成：

1. 编译器：负责将php代码编译为抽象语法树，然后进一步编译为可执行的opcodes，这个过程相当于GCC的工作，编译器是一个语言实现的基础
2. 执行器：负责执行编译器输出的opcodes，也就是执行php脚本中编写的代码逻辑

- 扩展

#### FPM

FPM是PHP FastCGI运行模式的一个进程管理器，FPM的核心功能是进程管理。

FastCGI是web服务器(如Nginx、Apache)和处理程序的一种通信协议，与http类似的一种应用层 **通信协议**。

在网络场景下，php并没有像golang那样实现 **http网络库**，而是实现了FastCGI协议，与web服务器配合实现了http的处理，**web服务器**处理http请求，然后将解析的结果再通过FastCGI协议转发给处理程序，程序处理完将结果返回给web服务器，web服务器再返回给用户。

网络处理一般的处理模型：**多进程、多线程**。**多进程模型**通常是主进程只负责管理子进程，而基本的网络事件由各个子进程处理，
