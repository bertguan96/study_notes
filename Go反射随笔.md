# Go随笔

## 反射

和Java一样Go的反射，也是用于简化代码量，同时go中提供reflect包，用于扩展reflect.type的不足。reflect实现了运行时反射的能力。

其中typeof和valueof是其实现的两个核心函数：

- [`reflect.TypeOf`](https://draveness.me/golang/tree/reflect.TypeOf) 能获取类型信息；
- [`reflect.ValueOf`](https://draveness.me/golang/tree/reflect.ValueOf) 能获取数据的运行时表示；

demo如下所示：

```go
	t := reflect.TypeOf(reflect_study.ReflectDemo{})
	println(t.String())
```

其作用优点和Java中的class.forName类似，可以拿到对象所包括的所有类型等，而valueOf则是可以获取到当前对象的值。

总的来说Class clazz = class.forName() = reflect.Typeof + reflect.ValueOf（我的个人理解）

**反射的三大法则**

反射是一把双刃剑，反射机制作为元编程来说，适量的反射机制可以减少重复代码，

1. 从 `interface{}` 变量可以反射出反射对象；
2. 从反射对象可以获取 `interface{}` 变量；
3. 要修改反射对象，其值必须可设置；

由于Go中反射的使用和Java十分类似，这里就不叙述其具体使用方法了。

具体接口和使用方法可以前往如下链接获取查询：[Go反射](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/#432-%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%80%BC)

## 接口

接口最核心的作用在于解除耦合，这一点在任何语言中都是一样的。其本质是一个中层，对已有代码封装（隐藏了实现层），同时不影响新代码的使用。例如：数据库就是如此，用户不需要关心数据库是如何实现的，只需要使用好SQL语句得到结果即可。

**Go中的接口**

go中的接口和Java中所有不同，Java中是需要显式地去实现接口，而Go中只需要隐式实现即可。

Java实现接口代码如下：

```java
package demo;

interface BaseInterface {

    int testInterface();
}
// java需要显式的去实现接口，而在Go语言中不需要类似的方式
public class InterfaceTest implements BaseInterface {
    @Override
    public int testInterface() {
        return 0;
    }
}
```

Go接口实现:

```java
import "fmt"

type TestInterface interface {
   Error() string
}

type TestCallback struct {
   Code    int `json:"code"`
   Message int `json:"message"`
}

func (e *TestCallback) Error() string {
   return fmt.Sprintf("%d, code=%d", e.Message, e.Code)
}
```

细心的人可能会发现上述代码根本就没有 implements接口的地方，这是为什么呢？Go 语言中**接口的实现都是隐式的**，我们只需要实现 将对应方法实现就实现了 `error` 接口。Go 语言实现接口的方式与 Java 完全不同：

- 在 Java 中：实现接口需要显式地声明接口并实现所有方法；
- 在 Go 中：实现接口的所有方法就隐式地实现了接口；

|                      | 结构体实现接口 | 结构体指针实现接口 |
| :------------------: | :------------: | ------------------ |
|   结构体初始化变量   |      通过      | 不通过             |
| 结构体指针初始化变量 |      通过      | 通过               |

## 并发（一）

**Context**

context.Context的作用是在不同 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期。每一个context.Context都会从最顶层的 Goroutine 一层一层传递到最下层。context.Context可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层。

作用：传递信号，及时抛出异常。

使用context同步信号案例：

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, 500*time.Millisecond)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}
```

**Cancel**

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)  // 新建一个新的上下文，用于取消进程
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // 父上下文不会触发取消信号
	}
	select {
	case <-done:
		child.cancel(false, parent.Err()) // 父上下文已经被取消，此时进行子上下文取消
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock() // 启动一个互斥锁
		if p.err != nil {
			child.cancel(false, p.err)
		} else {
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

总结：用于Goroutine中同步数据。

