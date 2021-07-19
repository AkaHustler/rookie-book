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

