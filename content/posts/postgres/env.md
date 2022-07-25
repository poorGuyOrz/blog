---
title: "Postgres源码编译及调试"
date: 2022-07-20T17:54:05+08:00
draft: false
---

## 源码编译

1. 源码下载 
直接下载最新源码，github上的源码每一个提交都保证是可编译运行的
```
git clone git@github.com:postgres/postgres.git
```

2. 依赖安装  
按照官网上的说明安装依赖，这里提前安装过但是没有记载，所以不贴了，也可以直接编译，在中间遇到确实的依赖再进行安装

3. 编译

为了能使用gdb调试，需要使用debug模式调试，我自己之前编译的时候发现即使指定`-enable-debug`在编译的时候发现也使用了`-O2`，所以这里建议直接修改`configure`中的`-O`和`-O2`为`-g`,
```shell
./configure --enable-depend --enable-cassert --enable-debug

make

sudo make install
```

4. 启动
```
./initdb -D /home/wen/postgres/data 
./pg_ctl -D /home/wen/postgres/data  start
./pg_ctl -D /home/wen/postgres/data  stop
./psql -p 5432
```
直接安装之后的文件在`/usr/local/pgsql/bin`中

5. 使用VSCode调试

可以使用VSCode进行调试，只需要配置`launch.json`即可。文件如下，其中program不重要，具体想调试那个进程，是使用processId选定的，调试的时候会有一个窗口让你选择调试进程。
```json
{
  // 使用 IntelliSense 了解相关属性。 
  // 悬停以查看现有属性的描述。
  // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "name": "(gdb) 附加",
      "type": "cppdbg",
      "request": "attach",
      "program": "/usr/local/pgsql/bin/postgres",
      "processId": "${command:pickProcess}",
      "MIMode": "gdb",
      "setupCommands": [
          {
              "description": "为 gdb 启用整齐打印",
              "text": "-enable-pretty-printing",
              "ignoreFailures": true
          },
          {
              "description":  "将反汇编风格设置为 Intel",
              "text": "-gdb-set disassembly-flavor intel",
              "ignoreFailures": true
          }
      ]
    }
  ]
}
```

这里有几个进程需要进行区分，由于pg是典型的CS架构，
* 你可以调试psql进程，也就是你命令行执行的进程，
* 也可以调试守护进程，他接收到新的链接之后会fork一个新进程，
* 也可以调试他fork出来的进程，这才是你命令行对应的执行进程，语句的解析执行都是在这个进程中
* 也可以调试其他的进程，，例如vocumm进程，，walwrite进程等



## gdb 调试
1. 使用脚本可以在gdb中打印出具体的执行计划的节点树

> clang-format 禁止头文件重排序 SortIncludes: false