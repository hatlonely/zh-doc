# errors

errors 包实现了函数对错误的操作。

`New` 函数创建一个仅仅包含文本消息的错误。

`Unwrap`，`Is`，`As` 函数处理哪些包装了其他错误的错误。如果一个错误的类型有 `Unwrap() error` 方法，这个错误就包含了其他错误。

如果 `e.Unwrap()` 返回一个非 `nil` 的错误 w，我们说 e 包装了 w。

`Unwrap` 取出了那个被包装的错误。如果它的参数有 `Unwrap` 方法，它会调用这个方法。否则，它会返回 `nil`。

一个创建包装错误的简单方法，调用 `fmt.Errorf` 并且使用 `%w` 动词给错误参数：

```golang
errors.Unwrap(fmt.Errorf("...%w...", ..., err, ...))
```

返回错误。

`Is` 依次取出它的第一个参数，来查找符合第二个参数的错误。它返回是否匹配。它应优先于简单的相等测试使用。

```golang
if errors.Is(err, fs.ErrExist)
```

优于

```golang
if err == fs.ErrExist
```

因为如果 err 包装了 fs.ErrExist，前者会成功。

`As` 依次展开它的第一个参数，查找可以分配给第二个指针参数的错误。如果成功，它会执行赋值，并返回 `true`，否则，它会返回 `false`。

```go
var perr *fs.PathError
if errors.As(err, &perr) {
	fmt.Println(perr.Path)
}
```

优于

```go
if perr, ok := err.(*fs.PathError); ok {
	fmt.Println(perr.Path)
}
```

因为如果 err 包装了 *fs.PathError，前面的写法会成功。

