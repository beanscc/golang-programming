# Concurrency is not parallelism



> 原文：https://go.dev/blog/waza-talk



```go
If there’s one thing most people know about Go, is that it is designed for concurrency. No introduction to Go is complete without a demonstration of its goroutines and channels.

But when people hear the word concurrency they often think of parallelism, a related but quite distinct concept. In programming, concurrency is the composition of independently executing processes, while parallelism is the simultaneous execution of (possibly related) computations. Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.

To clear up this conflation, Rob Pike gave a talk at Heroku’s Waza conference entitled Concurrency is not parallelism, and a video recording of the talk was released a few months ago.


The slides are available at go.dev/talks (use the left and right arrow keys to navigate).

To learn about Go’s concurrency primitives, watch Go concurrency patterns (slides).
```



https://go.dev/talks/2012/waza.slide#1





