# Golang Development Guide

## Leading Materials

As a professional Go programmer, these documents are very necessary to read.

- https://golang.org/doc/effective_go.html
- https://github.com/golang/go/wiki/CodeReviewComments
- https://github.com/uber-go/guide/blob/master/style.md

## Styles

In an existing project, when the new code conflicts with the old style, we should always choose consistency. If the old style is really bad, we should be refactoring it all.

First off, use [gofmt](https://golang.org/cmd/gofmt/) to format all source code files. The [gopls](https://github.com/golang/tools/blob/master/gopls/README.md) will do it for you automatically.

Most of the styles below are described in the leading materials. But in our experience, there are also some important items developers tend to ignore. So we list them here to enhance the impression.

### Error Message

The error message should be clean, contain enough context without redundant words, use `:` as the main separator.

```go
// Clean and good style
fmt.Errorf("create file %s failed: %v", fileName, err)

// Redundant and bad style
fmt.Errorf("create file %s err: %v", fileName, err)
fmt.Errorf("create file %s failed, err: %v", fileName, err)
```

### [Initialism](https://github.com/golang/go/wiki/CodeReviewComments#initialisms)

Use `HTTPHandler` instead of `HttpHandler`, etc.

### [Error Naming](https://github.com/golang/go/wiki/Errors)

Type name examples: `HTTPError`, `FileError`
Value name examples: `ErrNotFound`, `ErrExisted`

### [Imports](https://github.com/golang/go/wiki/CodeReviewComments#imports)

Use `standard lib`, `own lib`, `third-party lib`:

```go
import (
        "fmt"
        "hash/adler32"
        "os"

        "github.com/megaease/easegateway/pkg/plugin/backend"
        "github.com/megaease/easegateway/pkg/plugin/validator"

        "github.com/foo/bar"
        "rsc.io/goversion/version"
)
```

### Package Names

Please follow the principles the articles below described:

- https://rakyll.org/style-packages/
- https://blog.golang.org/package-names

NOTE: Use `plugin` instead of `plugins`.

### [Import Alias](https://github.com/uber-go/guide/blob/master/style.md#import-aliasing)

```go
import (
        "fmt"
        "os"
        "runtime/trace"

        // Alias when the conflict happened
        nettrace "golang.net/x/trace"
        // Alias when the package name does not match the last element of the import path.
        yaml "gopkg.in/yaml.v2"
)
```

### Global Names Convetions

This is a kind of new convention, we should follow it in the next development gradually.

- https://github.com/uber-go/guide/blob/master/style.md#prefix-unexported-globals-with-_

### Error Design

Put the error wrapping design in a simple way: Wrapping an error makes that error part of your API. If you don't want to commit to supporting that error as part of your API in the future, you shouldn't wrap the error.

- https://blog.golang.org/go1.13-errors
- https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

### Error Generating

Consistent generating errors: we only use `fmt.Errorf` to generate original errors:

- Good for grep-tools.
- Good for largely refactoring

### Avoid Naked Parameters

- https://github.com/uber-go/guide/blob/master/style.md#avoid-naked-parameters

```go
t.AddDate(1 /* year */, 10 /* month */ , 22 /* day */)
```

### Functional Options

It contains a lot of design concerns, these articles below describe very well:

- https://github.com/uber-go/guide/blob/master/style.md#functional-options
- https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis

### Test

Use Test Tables to do clean unit tests:

- https://github.com/uber-go/guide/blob/master/style.md#test-tables
- https://dave.cheney.net/2019/05/07/prefer-table-driven-tests

## Gotcha

The famous [50 Shades of Go](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/) contains some common gotchas.

### [Reader](https://golang.org/pkg/io/#Reader) Data Lifecycle

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

`Reader` is one of the most famous interfaces in golang. The `p` needs to copy to a new slice, if we need the content of it after `Read` returned.

### Context Assignment

```go
ctx = context.Backgroupd()

if true {
        // ctx here is a new variable, will be release after the if-branch exited.
        ctx, _ := context.WithCancel(ctx)
}

useOldCtx(ctx)
```

### Reuse Address of Variable in Range Loop

```go
type MyStruct struct {
        A int
}
values := []MyStruct{MyStruct{1}, MyStruct{2}, MyStruct{3}}
output := []*MyStruct{}
for _, v := range values {
        // First common mistake
        output = append(output, &v)
        // Second common mistake
        go func() {
          fmt.Println(v)
        }()
}
fmt.Println("output: ", output)
```

Because `v` is one variable in the same address with only updating content in the loop. So `&v` is always the same, this makes output to `[&MyStruct{3}, &MyStruct{3}, &MyStruct{3}]`

When you got the same results not diverse results as expected after some loops, most of the bugs come from this feature.

### Append Make-Slice

```go
a := make([]int, 100)
// len(a) == 101
a = append(a, 23)
```

### Nil Slice

To check if a slice is empty, always use `len(s) == 0.` Do not check for `nil`:

- https://github.com/uber-go/guide/blob/master/style.md#nil-is-a-valid-slice

### Nil Map

Nil slice is valid to `append`, nil map is invalid to manipulate. It's easy to forget to initialize it when the map lies in a struct.

```go
var s []int
// Correct
s = append(s, 1)

var m map[string]int
// panic
m["a"] = 1
```

### Int Slice Range

```go
a = []int{10, 11, 12}
// It's easier than you think to forget 2 values of range clause.
for i := range a {
        // No compilation error.
        // want: 10, 11, 12
        // got:  0, 1, 2
        fmt.Println(i)
}
```

### String Range

```go
data := "A\xfe\x02\xff\x04"
for _,v := range data {
        fmt.Printf("%#x ",v)
}
//prints: 0x41 0xfffd 0x2 0xfffd 0x4 (not ok)

fmt.Println()
for _,v := range []byte(data) {
        fmt.Printf("%#x ",v)
}
//prints: 0x41 0xfe 0x2 0xff 0x4 (good)

```

### Range to Change Elements

```go
type MyStruct struct {
        A int
}

// By default, use map[string]*MyStruct
// 1: Avoid large data copy.
// 2: Changeable elements.
m := map[string]MyStruct{
        "one":   MyStruct{1},
        "two":   MyStruct{2},
        "three": MyStruct{3},
}

for _, v := range m {
        // Never changed.
        v.A++
}
```

### Nil Interface

Interface contains two fiedls: type index and data pointer. It is nil only if both type index and data pointer are nil. The official FAQ [nil error](https://golang.org/doc/faq#nil_error) give a great example:

```go
func returnsError() error {
        var p *MyError = nil
        if bad() {
                p = ErrBad
        }
        return p // Will always return a non-nil error.
}
```

### Break Statement

A "break" statement terminates execution of the innermost "for", "switch", or "select" statement within the same function.

```go
done := make(chan struct{})
data := make(chan int)

go func() {
        for {
                select {
                case <-done:
                        // Never exit the goroutine.
                        break
                case d := <-data:
                        fmt.Println(d)
                }
        }
}()
```

### Method Set

It's a complex theory for this behavior, you can check out:

- [wiki page](https://github.com/golang/go/wiki/MethodSets)
- [stackoverflow answer](https://stackoverflow.com/a/48874650/1705845)

```go
type (
        I interface {
                Foo()
                Bar()
        }

        T struct{}
)

func (t T) Foo()  {}
func (t *T) Bar() {}

func main() {
        t := T{}
        t.Bar()
        t.Foo()

        // Compilation failed: T does not implement I
        // It will succeed if `t := &T{}`
        var i I = t

        m := map[int]T{1: T{}}
        // Compilation failed: cannot take the address of map value
        m[1].Bar()
}
```

## Advanced Study

- Channel Behaviors in Every State: https://go101.org/article/channel.html
- Spec: https://golang.org/ref/spec
- FAQ: https://golang.org/doc/faq
- Wiki: https://github.com/golang/go/wiki