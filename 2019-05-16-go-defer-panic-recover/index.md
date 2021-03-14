# go笔记 defer,panic,recover


官方blog传送门：https://blog.golang.org/defer-panic-and-recover

## defer

> A defer statement pushes a function call onto a list. The list of saved calls is executed after the surrounding function returns. Defer is commonly used to simplify functions that perform various clean-up actions.

defer类似cpp在对象在离开作用于后析构，defer可以多次，这样形成一个defer栈，后defer的语句在函数返回时将先被调用。

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

## panic

> Panic is a built-in function that stops the ordinary flow of control and begins panicking. When the function F calls panic, execution of F stops, any deferred functions in F are executed normally, and then F returns to its caller. To the caller, F then behaves like a call to panic. The process continues up the stack until all functions in the current goroutine have returned, at which point the program crashes. Panics can be initiated by invoking panic directly. They can also be caused by runtime errors, such as out-of-bounds array accesses.

执行panic后，函数不在往下走，会调用defer

```go
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() { fmt.Println("111") }()
	defer func() { fmt.Println("222") }()
	defer func() { fmt.Println("333") }()
	panic("panic")
}

/*
输出:
333
222
111
panic: panic

goroutine 1 [running]:
main.defer_call()
        C:/Users/lch/go/src/github.com/lchannng/gopl/defer/main.go:21 +0x98
main.main()
        C:/Users/lch/go/src/github.com/lchannng/gopl/defer/main.go:14 +0x27
*/
```

## recover

> Recover is a built-in function that regains control of a panicking goroutine. Recover is only useful inside deferred functions. During normal execution, a call to recover will return nil and have no other effect. If the current goroutine is panicking, a call to recover will capture the value given to panic and resume normal execution.

按上面所说，panic的函数并不会立刻返回，而是先defer，再返回。这时候可在defer中将panic捕获到，并阻止panic传递，save the world。

一个类似try catch的例子
```go
func Try(f func(), handler func(interface{}) {
    defer func() {
        if err := recover(); err != nil {
            handler(err)
        }
    }        
    f()
})

func main() {
    Try(func() {
        panic("oh!panic.")
    }, func(e interface{}) {
        fmt.Println(e)
    })
}
```

