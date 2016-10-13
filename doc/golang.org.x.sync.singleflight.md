# 概要

doc: <https://godoc.org/golang.org/x/sync/singleflight>

`singleflight` 保证注册的函数同一时间只会被执行一次。

# 示例

```go
import (
	"fmt"
	"golang.org/x/sync/singleflight"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var g singleflight.Group

	// fn只会被调用一次
	var count int32
	var fn = func() (interface{}, error) {
		time.Sleep(10 * time.Millisecond)
		return atomic.AddInt32(&count, 1), nil
	}

	group := &sync.WaitGroup{}

	// Do 方法是同步的，可以立即拿到结果
	for i := 0; i < 5; i++ {
		i := i
		group.Add(1)
		go func() {
			defer group.Done()
			v, err, shared := g.Do("key", fn)
			fmt.Printf("[%d] value=%v, err=%v, shared=%t\n", i, v, err, shared)
		}()
	}

	// DoChan 是异步方法，通过 chan Result 获取结果
	for i := 0; i < 5; i++ {
		i := i
		group.Add(1)
		go func() {
			defer group.Done()
			c := g.DoChan("key", fn)
			result := <-c
			fmt.Printf("[%d] value=%v, err=%v, shared=%t\n", 5+i, result.Val, result.Err, result.Shared)
		}()
	}

	group.Wait()

	// 现在，没有其他goroutine需要执行fn了，所有下面两次调用会重新执行fn

	v, err, shared := g.Do("key", fn)
	fmt.Printf("[%d] value=%v, err=%v, shared=%t\n", 10, v, err, shared)

	c := g.DoChan("key", fn)
	result := <-c
	fmt.Printf("[%d] value=%v, err=%v, shared=%t\n", 11, result.Val, result.Err, result.Shared)
}
```

输出示例：

```bash
[3] value=1, err=<nil>, shared=true
[1] value=1, err=<nil>, shared=true
[2] value=1, err=<nil>, shared=true
[9] value=1, err=<nil>, shared=true
[5] value=1, err=<nil>, shared=true
[7] value=1, err=<nil>, shared=true
[6] value=1, err=<nil>, shared=true
[8] value=1, err=<nil>, shared=true
[4] value=1, err=<nil>, shared=true
[0] value=1, err=<nil>, shared=true
[10] value=2, err=<nil>, shared=false
[11] value=3, err=<nil>, shared=false
```



在多个 `goroutine` 同时调用 `fn` 时（`singleflight` 其实是通过 `key` 来判断是否重复执行的）, `fn` 只被执行一次。

当 `fn` 被执行完毕，`key` 就从 `singleflight.Group` 删除了，后续再调用 `Do()` 或 `DoChan()` 的话，`fn` 会再次被调用。