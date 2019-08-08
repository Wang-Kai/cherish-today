# Go 代码测试

### Main

在执行测试代码块时，常常需要执行一些前置条件或后置收尾工作，这个时候就需要用到 **TestMain**，如果测试文件中包含 **TestMain** 函数，则测试时会先调用 **TestMain** 而不是直接调用测试用例。最终，**TestMain** 需要调用 `os.Exit` 来结束测试流程。

```go
func TestMain(m *testing.M)
```

示例场景：

```go
func TestMain(m *testing.M) {
	// 执行 解析命令行参数、连接DB 等前置工作
	res := m.Run()
	
	// 执行收尾工作
	// ...
	
	//结束测试
	os.Exit(res)
}
```

### Examples

Example 函数可以比对示例程序的标准输出和预想的输出。**Example Function** 必须以注释行为结尾，并且第一行注释为 `Output:`。测试执行对比时会忽略前后的空格。

```golang
func ExampleHello() {
	fmt.Println("hello!")
	// Output: hello
}
```

```
=== RUN   ExampleHello
--- FAIL: ExampleHello (0.00s)
got:
hello!
want:
hello
```

如果输出是无序的，可以使用 `Unordered output:`，其会忽略行之间的比对顺序

```golang
func ExamplePerm() {
    for _, value := range Perm(4) {
        fmt.Println(value)
    }
    // Unordered output: 4
    // 2
    // 1
    // 3
    // 0
}
```

上面是对方法的测试，如果是对 `Type`、`method M on type T` 做测试的话，要做如下声明：

```golang
func Example() { ... }
func ExampleF() { ... }
func ExampleT() { ... }
func ExampleT_M() { ... }
```

假使对同一类型有多个测试例子，需要添加不同的后缀名，后缀名以小写开头：

```golang
func Example_suffix() { ... }
func ExampleF_suffix() { ... }
func ExampleT_suffix() { ... }
func ExampleT_M_suffix() { ... }
```


