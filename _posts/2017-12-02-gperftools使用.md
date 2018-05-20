---
title: gperftools使用
date: 2017-12-02 23:00:00
categories:
- Linux工具
tags:
- Linux工具
---

# gperftools

## 1 安装

```
yum install -y gperftools
```

## 2 使用

gperftools其实有4个组件，用来分析不同的性能，例如内存、执行时间等，帮助找到系统的瓶颈。Cpu Profiler是其中一个组件，用于分析执行时间。

Cpu Profiler:

```
gcc [...] -o myprogram -lprofiler
CPUPROFILE=/tmp/MyProfile ./myprogram
pprof --text ./myprogram /tmp/MyProfile > /tmp/MyProfile.txt
```

或者gdb调试过程中，在需要的地方启动、停止Cpu Profiler.

例如，demo.c

```
void consumeSomeCPUTime1(int input){ 
 int i = 0; 
 input++; 
 while(i++ < 10000){ 
   i--;  i++;  i--;  i++; 
 } 
}; 
 
void consumeSomeCPUTime2(int input){ 
 input++; 
 consumeSomeCPUTime1(input); 
 int i = 0; 
 while(i++ < 10000){ 
   i--;  i++;  i--;  i++; 
 } 
}; 
 
int stupidComputing(int a, int b){ 
 int i = 0; 
 while( i++ < 10000){  
   consumeSomeCPUTime1(i); 
 } 
 int j = 0; 
 while(j++ < 5000){ 
   consumeSomeCPUTime2(j); 
 } 
 return a+b; 
}; 
 
int smartComputing(int a, int b){ 
 return a+b; 
}; 
 
void main(){ 
 int i = 0;
 printf("reached the start point of performance bottle neck\n"); 
 sleep(5);//ProfilerStart("CPUProfile");
 while( i++ < 10){ 
   printf("Stupid computing return : %d\n",stupidComputing(i, i+1)); 
   printf("Smart computing return %d\n",smartComputing(i+1, i+2)); 
 }
 printf("should teminate profiling now.\n");  
 sleep(5);//ProfilerStop();
}
```

step1. 编译：

```
gcc demo.c -g -o demo -lprofiler
```

step2. 调试

方法1：

```
CPUPROFILE=/tmp/MyProfile ./myprogram
pprof --text ./myprogram /tmp/MyProfile > /tmp/MyProfile.txt
```

方法2：

```
gdb demo // 启动 gdb 并选择你的程序为 gdb 的启动目标 
(gdb)b main1.c:37 // 对应于耗时模块的起始点 
(gdb)b main1.c:42 // 对应于耗时模块的终止点 
(gdb)r // 运行 
(gdb)p ProfilerStart("MyProfile")
(gdb)c // 继续程序运行 
(gdb)p ProfilerStop()
```

方法1和方法2都将产生分析结果MyProfile，执行如下命令将分析结果转换成txt形式：

```
[root@master tmp]# pprof --text ./demo MyProfile > MyProfile.txt 
Using local file ./demo.
Using local file MyProfile.
[root@master tmp]# cat MyProfile.txt 
Total: 2428 samples
    1836  75.6%  75.6%     1836  75.6% consumeSomeCPUTime1
     592  24.4% 100.0%     1213  50.0% consumeSomeCPUTime2
       0   0.0% 100.0%     2428 100.0% __libc_start_main
       0   0.0% 100.0%     2428 100.0% _start
       0   0.0% 100.0%     2428 100.0% main
       0   0.0% 100.0%     2428 100.0% stupidComputing
```

