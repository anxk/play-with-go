# Go 中字符串连接的几种方法的性能对比

1. 通过 `+` 连接字符串

2. 通过 `strings.Join` 连接字符串

   ```go
   package concatenate
   
   import (
   	"strings"
   )
   
   func foo() string {
   	s, e := "", "x"
   	for i := 0; i < 10; i++ {
   		s += e
   	}
   	return s
   }
   
   func bar() string {
   	e := []string{"x", "x", "x", "x", "x", "x", "x", "x", "x", "x"}
   	return strings.Join(e, "")
   }
   
   ```

   ```go
   package concatenate
   
   import (
   	"testing"
   )
   
   func BenchmarkFoo(b *testing.B) {
   	for i := 0; i < b.N; i++ {
   		foo()
   	}
   }
   
   func BenchmarkBar(b *testing.B) {
   	for i := 0; i < b.N; i++ {
   		bar()
   	}
   }
   
   ```

   

