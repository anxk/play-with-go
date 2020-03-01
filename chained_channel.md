# 通道串联的最大长度

下面代码来源于 The Go Programming Language 中的一个习题，习题大概意思是用函数串联通道，大概串联多少后计算机会崩溃？

```go
package chainedchan

// Processor represents a processor of one stage, it receives values
// from stage's in-channel, do some jobs with the values and sends the
// results to out-channel
type Processor func(in <-chan interface{}, out chan<- interface{})

// Stage represents a stage in the pipeline, note that its channels
// have zero buffer length
type Stage struct {
	in        chan interface{}
	out       chan interface{}
	processor Processor
}

// active wakeup the stage by calling its processor in a goroutine
func (s *Stage) active() {
	go s.processor(s.in, s.out)
}


func (s *Stage) In() chan interface{} {
	return s.in
}

func (s *Stage) Out() chan interface{} {
	return s.out
}

// Pipe represents a pipeline consist of several stages
type Pipe struct {
	Active bool // whether the pipeline is alive
	Length int  // amount of stages
	Stages []*Stage
}

// NewWithCap create an empty pipeline with a initial capacity
func NewWithCap(capacity int) *Pipe {
	return &Pipe{
		Stages: make([]*Stage, capacity),
	}
}

// New create an empty pipeline
func New() *Pipe {
	return &Pipe{
		Stages: make([]*Stage, 0),
	}
}

// Len returns the amount of stages in the pipeline
func (p *Pipe) Len() int {
	return p.Length
}

// First returns the first stage in the pipeline, if
// it is a new pipeline, returns nil
func (p *Pipe) First() *Stage {
	l := p.Len()
	if l == 0 {
		return nil
	}
	return p.Stages[0]
}

// Last returns the last stage in the pipeline, if
// it is a new pipeline, returns nil
func (p *Pipe) Last() *Stage {
	l := p.Len()
	if l == 0 {
		return nil
	}
	return p.Stages[p.Len()-1]
}

// Append append a new stage to the pipeline
func (p *Pipe) Append(proc Processor) {
	var (
		i chan interface{}
		o chan interface{} = make(chan interface{})
	)
	if p.Last() != nil {
		i = p.Last().out
	} else {
		i = make(chan interface{})
	}
	stage := &Stage{
		in:        i,
		out:       o,
		processor: proc,
	}
	p.Stages = append(p.Stages, stage)
	p.Length++
}

// Run active the pipeline
func (p *Pipe) Run() {
	for _, s := range p.Stages {
		go s.active()
	}
}

```

```go
package chainedchan

import (
	"fmt"
	"testing"
)

// stupidProc is a Processor which does nothing but receives items
// from the in-channel and sends them to the out-channel
func stupidProc(in <-chan interface{}, out chan<- interface{}) {
	out <- <-in
}

//increaseProc is a Processor which increases the values receives from
//the in-channel and sends them to the out-channel
func increaseProc(in <-chan interface{}, out chan<- interface{}) {
	v := <-in
	v = v.(int) + 1
	out <- v
}

// TestNew tests a fresh pipeline should be create by New()
func TestNew(t *testing.T) {
	pipe := New()
	if pipe.Len() != 0 {
		t.Error("New() should creates an empty pipeline which has no stages")
	}
}

func TestLen(t *testing.T) {
	pipe := New()
	pipe.Append(increaseProc)
	if pipe.Len() != 1 {
		t.Errorf("pipeline want length 1, got %d", pipe.Len())
	}
}

func BenchmarkAppend(b *testing.B) {
	pipe := New()
	for i := 0; i < b.N; i++ {
		pipe.Append(stupidProc)
	}
}

func ExampleIncreasePipeline() {
	pipeMaxLen := 100
	pipe := New()
	for pipeMaxLen > 0 {
		pipe.Append(increaseProc)
		pipeMaxLen--
	}
	pipe.Run()
	go func() {
		pipe.First().In() <- 0
	}()
	fmt.Println(<-pipe.Last().Out())
	// Output: 100
}

func ExampleStupidPipeline() {
	pipeMaxLen := 100
	pipe := New()
	for pipeMaxLen > 0 {
		pipe.Append(stupidProc)
		pipeMaxLen--
	}
	pipe.Run()
	go func() {
		pipe.First().In() <- 0
	}()
	fmt.Println(<-pipe.Last().Out())
	// Output: 0
}

```

