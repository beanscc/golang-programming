### spin 自旋锁



什么是自旋锁？



自旋锁可以使线程在未获得锁的时候不被挂起阻塞，而是执行一个空循环，在若干个空循环后，若可以获得锁，则继续，否则可能被挂起



使用自旋锁，线程的执行连续性较强，被挂起的几率较小。因此对于锁竞争不是很激烈，锁占用时间短的并发线程有一定的积极意义；但对于锁竞争激烈，单个线程锁占用时间较长的并发程序不适用，自旋锁在自旋后仍然无法得到锁，浪费 cpu，且最终将被挂起，浪费系统资源





缺点？

- 过多占用cpu问题：若已获得锁的线程长时间占用锁，将导致未获得锁的线程长时间空循环占用cpu，浪费 cpu 资源（可设置超时时间）
- 死锁问题：已获得锁的线程在递归调用中，无法二次获得锁，需要等待自己已获得的自旋锁的释放，造成死锁问题（可通过实现可重入性，解决）

优点？

​	自旋锁不会发生线程状态切换，会一直处于用户态，即线程一直处于 activie



公平性：是否先到先得锁？即前一线程释放锁后，是否第一个等待该锁的线程可以获得该锁

​	不公平会导致 “线程饥饿”

可重入行性：



















