# Java进程CPU使用率高排查

> 原文: https://www.linuxhot.com/java-cpu-used-high.html

这里记录下, 以防丢失.

下面是排查步骤, 非常实用:

* jps获取Java进程的PID
* jstack pid > java.txt 导出CPU占用高进程的线程栈
* top -H -p PID 查看对应进程的哪个线程占用CPU过高
* echo "obase=16; PID" | bc 将线程的PID转换为16进制
* 在第二步导出的java.txt中查找转换成为16进制的线程PID找到对应的线程栈
* 分析负载高的线程栈都是什么业务操作, 优化程序并处理问题
