# time

## Timer

## Ticker

## time.Tick()


```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Printf("start:%v\n", time.Now())

	// timer
	timer := time.NewTimer(1 * time.Second)
	// tick
	tick := time.Tick(2 * time.Second)
	// ticker
	ticker := time.NewTicker(3 * time.Second)

	// 让程序结束
	doneChan := make(chan bool)
	go func() {
		time.Sleep(9 * time.Second)
		doneChan <- true
	}()

	for {
		select {
		case t := <-timer.C:
			fmt.Printf("timer: %v\n", t)
		case t := <-tick:
			fmt.Printf("tick: %v\n", t)
		case t := <-ticker.C:
			fmt.Printf("ticker: %v\n", t)
		case <-doneChan:
			fmt.Printf("done!")
			return
		}
	}
}

// 结果

```



> **Timer 和 Ticker 的区别：** 简单来说就是，Timer在设定的时间间隔后，执行一次，根据需要可以重置间隔时间或者终止计时器；Ticker 在每间隔设定的时间后，就执行一次，也可以终止，但不能调整间隔时间

> **Ticker 和 time.Tick() 的区别：** time.Tick() 返回的其实就是一个Ticker结构对象的一个C属性；简单说，time.Tick() 生成的 Ticker 是在不需要停止的；Ticker 是可以在需要的时候，主动调用 `func (t *Ticker) Stop()` 方法来停止这个计时器的，但需要注意 stop 方法不会停止向





## 注意 for select break 的迷惑



for select 中 break 不会直接跳出循环，如下，会一直执行 line:21

```go
import (
	"log"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	done := make(chan bool)
	go func() {
		time.Sleep(3 * time.Second)
		done <- true
	}()

	for {
		select {
		case <-done:
			log.Printf("done")
			break
		case next := <-ticker.C:
			log.Printf("tick at:%v", next)
		}
	}
	log.Print("DONE")
	
	/* output:
	2022/05/25 11:05:36 tick at:2022-05-25 11:05:36.70893 +0800 CST m=+1.001147595
    2022/05/25 11:05:37 tick at:2022-05-25 11:05:37.707951 +0800 CST m=+2.000175865
    2022/05/25 11:05:38 tick at:2022-05-25 11:05:38.708089 +0800 CST m=+3.000322100
    2022/05/25 11:05:38 done
    2022/05/25 11:05:39 tick at:2022-05-25 11:05:39.707903 +0800 CST m=+4.000144014
    2022/05/25 11:05:40 tick at:2022-05-25 11:05:40.708585 +0800 CST m=+5.000833305
    2022/05/25 11:05:41 tick at:2022-05-25 11:05:41.708082 +0800 CST m=+6.000338123
    2022/05/25 11:05:42 tick at:2022-05-25 11:05:42.708788 +0800 CST m=+7.001052323
    2022/05/25 11:05:43 tick at:2022-05-25 11:05:43.708337 +0800 CST m=+8.000608917
    ....
	*/
}
```



那如何跳出循环呢？

方式1: 定义 label 使用 break label 语法

方式2: 外层定义变量，在需要跳出的地方修改变量值，select 代码段后判断

```go
package main

import (
	"log"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	done := make(chan bool)
	go func() {
		time.Sleep(3 * time.Second)
		done <- true
	}()

	canBreak := false
	for {
		select {
		case <-done:
			log.Printf("done")
			canBreak = true
			// ticker.Stop()
			// break
		case next := <-ticker.C:
			log.Printf("tick at:%v", next)
			// default:
		}

		if canBreak {
			log.Printf("loop break")
			break
		}
	}
	log.Print("DONE")
    /* output:
	2022/05/25 11:10:45 tick at:2022-05-25 11:10:45.88878 +0800 CST m=+1.000399016
    2022/05/25 11:10:46 tick at:2022-05-25 11:10:46.8888 +0800 CST m=+2.000426487
    2022/05/25 11:10:47 tick at:2022-05-25 11:10:47.889501 +0800 CST m=+3.001135773
    2022/05/25 11:10:47 done
    2022/05/25 11:10:47 loop break
    2022/05/25 11:10:47 DONE
	*/
}

```

