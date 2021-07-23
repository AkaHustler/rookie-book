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

### 变量

#### 变量内部实现

- 变量内部实现

变量有两个组成部分：变量名、变量值，php中将其对应为：**zval，zend_value**，php中的 **变量的内存** 是通过 **引用计数** 进行管理的，而且php7中 **引用计数**是在zend_value而不是在zval上，变量之间的传递、赋值通常是针对zend_value。

1. 变量的基础结构

``` c
//zend_types.h
typedef struct _zval_struct zval;
typedef union _zend_value {
zend_long lval; //int整形
double dval; //浮点型
zend_refcounted *counted;
zend_string *str; //string字符串
zend_array *arr; //array数组
zend_object *obj; //object对象
zend_resource *res; //resource资源类型
zend_reference *ref; //引用类型，通过&$var_name定义的
zend_ast_ref *ast; //下面几个都是内核使用的value
zval *zv;
void *ptr;
zend_class_entry *ce;
zend_function *func;
struct {
uint32_t w1;
uint32_t w2;
} ww;
} zend_value;
```

``` c
struct _zval_struct {
zend_value value; //变量实际的value
  union {
  	struct {
  	ZEND_ENDIAN_LOHI_4( //这个是为了兼容大小字节序，小字节序就是下面的顺序，大字节序则下面4个顺序翻转
  zend_uchar type, //变量类型
  zend_uchar type_flags, //类型掩码，不同的类型会有不同的几种属性，内存管理会用到
  zend_uchar const_flags,
  zend_uchar reserved) //call info，zend执行流程会用到
  } v;
  uint32_t type_info; //上面4个值的组合值，可以直接根据type_info取到4个对应位置的值
  } u1;
  union {
  uint32_t var_flags;
  uint32_t next; //哈希表中解决哈希冲突时用到
  uint32_t cache_slot; /* literal cache slot */
  uint32_t lineno; /* line number (for ast nodes)
  */
  uint32_t num_args; /* arguments number for EX(Thi
  s) */
  uint32_t fe_pos; /* foreach position */
  uint32_t fe_iter_idx; /* foreach iterator index */
  } u2; //一些辅助值
};
```

**zval** 结构比较简单，内嵌一个 union类型的zend_value，保存具体变量类型的值或指针，还有两个union：u1，u2

u1：变量的类型就通过 **u1.v.type** 区分，另一个值 **type_flags** 为类型掩码，在变量的内存管理、gc中会用到

u2：辅助值

从 **zend_value**可以看出，除了 **long、double**类型直接存储值外，其他类型存储的都是指针，指向各自的结构。

2. 类型

标量类型：最简单的类型是 **true、false、long、double、null**，其中true、false、null没有value，直接根据type区分，而long、double的值直接存在value中；zend_long、double，也就是标量类型不需要额外的value指针

字符串：PHP中字符串通过 **zend_string**表示：

``` c
struct _zend_string {
	zend_refcounted_h gc;
	zend_ulong h; /* hash value */
	size_t len;
	char val[1];
};
```

1. gc：变量引用信息，比如当前value的引用数，所有用到 **引用计数** 的变量类型都会有这个结构
2. h：哈希值，数组中计算索引时会用到
3. len：字符串长度，通过该值保证二进制安全
4. val：字符串内容，变长struct，分配时按len长度申请内存

字符串还可分为具体的几类，通过flaf保存：zval.value->gc.u.flags

数组：数组的底层实现就是普通的有序HashTable

对象/资源：对象比较常见。资源指的是tcp连接、文件句柄等，这种类型可以随便定义struct，通过ptr指向

引用：引用实际是指向另外一个PHP变量，对它的修改会直接改动实际指向的zval，通过 **&** 操作符产生一个引用变量，**&** 首先会创建一个 **zend_reference** 结构，内嵌了一个zval，这个zval的value指向 原来zval的value（如果是 **布尔、整形、浮点**则直接复制原来的值），然后将原来zval的类型修改为 **IS_REFERENC**，原zval的value指向新创建的 **zend_reference** 结构。

``` c
struct _zend_reference {
	zend_refcounted_h gc;
	zval val;
};
```

PHP中的引用只可能有一层，不会出现一个引用指向另外一个引用的情况，就是没有C语言中**指针的指针** 的概念

3. 内存管理

如果使用硬拷贝方式，赋值、函数传参硬拷贝一个副本，最终各变量的值是互相独立的，不回出现多个变量共用一个value的情况，但是弊端是效率低，如果是只读操作，硬拷贝的话就会有一份多余数据，这个问题解放方案是：**引用计数+写时复制(copy on write)**，php变量的管理正式基于这两点实现的。

(1)**引用计数**：是指在value中增加一个字段 **refcount** 记录指向当前value的数量，变量赋值、函数传参时，不硬拷贝一份数据，而是 **refcount++**，变量销毁时将 **refcount--**，等到refcount减为0时，表示已经没有变量引用这个value，将它销毁即可。

但从**zend_value**的结构可以看出，并不是所有数据类型都会用到引用计数，**long、double** 直接都是硬拷贝，只有value是 **指针**那几种类型才可能会用到引用计数。

并不是所有的php比变量都会用到引用计数，标量：true/false/double/long/null是硬拷贝自然不需要，除此之外还有两个特殊类型也不会用到：interned string(内部字符串，即上文提到的IS_STR_INTERNED、immutable array)，通过**zval.u1.type_flag** 类型掩码来区分是否支持饮用计数

- **interned string**：内部字符串
- **immutable array**：只有在用opcache的时候才会用到这种类型

(2)写时复制

上面的 **引用计数**，多个变量可能指向同一个value，然后通过 **refcount** 统计引用计数，这时候如果一个变量试图更改value的内容，则会 **重新拷贝** 一份value数据修改，同时断开旧的指向，只有在有必要的时候(写)才会发生硬拷贝。

不是所有类型都可以copy，比如对象，资源，事实上只有 **string、array两种支持**，于引用计数相同，也是通过 **zval.u1.type_flag** 标识value是否可复制。

(3)变量回收

(4)垃圾回收

#### 数组

#### 静态变量

#### 全局变量

#### 常量
