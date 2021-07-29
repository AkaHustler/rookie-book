## PHP基本架构

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

- ##### 概述

FPM是PHP FastCGI运行模式的一个进程管理器，FPM的核心功能是进程管理。

FastCGI是web服务器(如Nginx、Apache)和处理程序的一种通信协议，与http类似的一种应用层 **通信协议**。

在网络场景下，php并没有像golang那样实现 **http网络库**，而是实现了FastCGI协议，与web服务器配合实现了http的处理，**web服务器**处理http请求，然后将解析的结果再通过FastCGI协议转发给处理程序，程序处理完将结果返回给web服务器，web服务器再返回给用户。

网络处理一般的处理模型：**多进程、多线程**。**多进程模型**通常是主进程只负责管理子进程，而基本的网络事件由各个子进程处理，nginx，fpm就是这种模式；另一种 **多线程模型**与多进程类似，只不过是线程粒度，通常会由 **主线程监听、接收请求**，然后交由子线程处理，memcached就是这种模式，有时候也是采用多进程那种模式：主线程志管理子线程，而不管网络事件，各个子线程监听、接收、处理请求，memcached使用udp协议时采用的就是这种模式。

- ##### 基本实现

**fpm**的实现就是创建一个master进程，在master进程中 **创建并监听socket**，然后fork出多个子进程，这些子进程各自accept请求，子进程的处理很简单，在启动后 **阻塞** 在accept上，有请求后开始读取请求数据，读取完成后开始处理然后再返回，期间是不会处理其他请求的，即 **fpm的子进程同时只能响应一个请求**，只有把这个请求处理完成才会accept下一个请求。而nginx的子进程通过epoll处理套接字，如果一个请求数据还未发送完成则会处理下一个请求，即 **一个进程同时连接多个请求，是非阻塞的模型，只处理活跃的套接字**。

fpm的master进程和worker进程不会直接进行通信，**master通过共享内存获取worker进程的信息，比如worker进程当前状态、已处理请求数等**，当master要杀掉一个worker进程时则通过发送信号的方式通知worker进程。

fpm可以监听多个端口，每个端口对应一个**worker pool**，而每个pool下对应多个worker进程，类似nginx的server概念。

具体实现worker pool通过 **fpm_worker_pool_s** 这个结构表示，多个worker pool组成一个单链表

``` c++
struct fpm_worker_pool_s {
struct fpm_worker_pool_s *next; //指向下一个worker pool
struct fpm_worker_pool_config_s *config; //conf配置:pm、max_children、
start_servers...
int listening_socket; //监听的套接字
...
//以下这个值用于master定时检查、记录worker数
struct fpm_child_s *children; //当前pool的worker链表
int running_children; //当前pool的worker运行总数
int idle_spawn_rate;
int warn_max_children;
struct fpm_scoreboard_s *scoreboard; //记录worker的运行信息，比如空闲、忙碌worker数
...
}
```

- ##### FPM的初始化

1. fpm的初始化

fpm的启动流程，

在fork后worker进程返回了监听的套接字继续main()后面的处理，而master将永远阻塞在fpm_event_loop()

2. 请求处理

**fpm_run()**执行后将fork出worker进程，worker进程返回 **main()** 中继续向下执行，后面的流程就是worker进程不断accept请求，然后执行php脚本并返回。整体流程如下：

(1) 等待请求：worker进程阻塞在 **fcgi_accept_request()** 等待请求

(2) 解析请求：fastcgi请求到达后被worker接收，然后开始接收并解析数据，直到request请求完全到达

(3) 请求初始化：执行 **php_request_startup()**，此阶段会调用每个扩展的 **PHP_RINIT_FUNCTION()**

(4) 编译、执行：由 **php_execute_script()** 完成php脚本的编译、执行

(5) 关闭请求：请求完成后执行 **php_execute_shutdown()**，此阶段会调用每个扩展的 **PHP_RSHUTDOWN_FUNCTION()**，然后进入步骤(1)等待下一个请求

worker进程一次请求的处理被划分5个阶段，worker处理到各个阶段时会将当前阶段更新到 **fpm_scoreboard_proc_s->request_stage**，master进程正是通过这个标识判断worker进程是否空闲。

3. 进程管理

master管理worker进程的三种不同方式

**static**：启动时master按照 **pm.max_children** 配置fork出相应的worker进程，即 **worker进程数时固定不变的**。

**dynamic**：动态进程管理，启动时master按照 **pm.start_servers** 初始化一定数量worker，运行中master发现空闲worker数低于 **pm.min_spare_servers** 配置数(表示请求数比较多，worker处理不过来)则会fork worker进程，但总的worker数不能超过 **pm.max_children**，如果master发现空闲worker数多于 **pm.max_spare_servers**(表示闲着的worker过多)则会杀掉一些worker

**ondemand**：此方式很少用，在启动时不分配worker进程，等到有请求了后再通知master进程fork worker进程，总的worker进程不超过 **pm.max_children**，处理完后worker进程不会立即退出，当空闲时间超过 **pm.process_idle_timeout**后再退出。

- PHP执行的几个阶段

1. 模块初始化阶段
2. 请求初始化阶段
3. 执行php脚本阶段
4. 请求结束阶段
5. 模块关闭阶段
