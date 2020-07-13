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

> **Ticker 和 time.Tick() 的区别：** time.Tick() 返回的其实就是一个Ticker结构对象的一个C属性；简单说，time.Tick() 生成的 Ticker 是在不需要停止的；Ticker 是可以在需要的时候，主动调用 `func (t *Ticker) Stop()` 方法来停止这个计时器的
