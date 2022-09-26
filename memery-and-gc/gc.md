## GC

- go1.5 gc: https://blog.golang.org/go15gc
- 参考：https://www.cnblogs.com/hezhixiong/p/9577199.html
- 论文《On-the-fly garbage collection: an exercise in cooperation》：https://www.cnblogs.com/rsapaper/p/14191410.html
- go 垃圾回收系列：https://zhuanlan.zhihu.com/p/104623357



## stop the world



## 标记-清除

标记-清楚算法是一种自动管理内存，基于追踪的垃圾收集算法。内存单元不会在变成垃圾时被立刻回收，而是保持不可达状态，直到到达某个阈值或者固定时间长度。这个时候系统会挂起用户程序，也就是 stw (stop the world)，转而执行垃圾回收程序。



## 三色标记法



可并发执行





算法介绍





## 写屏障

写屏障：修改原先的写逻辑，在对象新增的同时给它标记颜色，标记为"灰色"，因此打开了写屏障可以保证三色标记法可以并发的安全正确的运行



## Go GC 



先看以下几个问题：

- 何时触发 gc
- 在哪里触发？
- 触发条件是什么？
- gc 如何运行的？
- 有哪些参与者？







