# Go 中通道的速度

下面代码来自 The Go Programming Language 中的一道习题，题目大概意思是启动两个 goroutine，让它们通过两个无缓冲的通道进行 ping-pong 通信，那么每秒能够通信多少次？

通过下面的测试，通信速度大概是 250 万次 / 秒，当然这个数值受运行环境影响并且和测试的代码也有关系，但是至少可以说明 Go 中通过通道通信的速度在百万量级每秒。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	conversation(time.Second * 5)
}

func conversation(duration time.Duration) {
	var (
		ch1         = make(chan bool)
		ch2         = make(chan bool)
		msg         = true
		count int64 = 0
	)

	// start two goroutines A and B, in this situation count was safe
	go role(ch1, ch2, msg, &count)
	go role(ch2, ch1, msg, &count)

	// initial signal to start the communication between A and B
	ch1 <- msg

	select {
	case <-time.NewTimer(duration).C:
		fmt.Printf("%d times communication per second", count/int64(duration.Seconds()))
		return
	}
}

// role reads a bool value from in-channel, increases count and then sends msg to out-channel
func role(in <-chan bool, out chan<- bool, msg bool, count *int64) {
	for {
		<-in
		*count++
		out <- msg
	}
}

```

