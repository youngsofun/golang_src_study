
# escape

定义： [wiki](https://en.wikipedia.org/wiki/Escape_analysis)

下面完全是我从实用角度理解：

1. 对象能分配在栈上当然最好，但对象或其部分的地址如果被引用，这个引用的生命周期有可能超过这个栈，只好分配在堆上。

2. 对每个函数判断参数和local变量是否escape。比如 funcA 本地有个v，调用 funcB(v)，而funcB会让参数v escape，那么对funcA，v是要分配在堆上的。


编译时加 -m，就可以看到变量的escape情况（还能看到inline）。

对于各种情况，golang的test是个很好的参考。



[例子](http://golang.org/test/escape_array.go#L69) :

```
func hugeLeaks1(x **string, y **string) { // ERROR "leaking param content: x" "hugeLeaks1 y does not escape" "mark escaped content: x"
        a := [10]*string{*y}
        _ = a
        // 4 x 4,000,000 exceeds MaxStackVarSize, therefore it must be heap allocated if pointers are 4 bytes or larger.
        b := [4000000]*string{*x} // ERROR "moved to heap: b"
        _ = b
    }
    
```

ERROR 是-m会输出
比如 （同一行会输出多行，并且可能不连续的）

```
  6 ab.go:12: can inline slice0
  5 ab.go:15: moved to heap: i
  4 ab.go:16: &i escapes to heap
```

74行说明 调用这个函数会照成 x escape

test中可见还有很多不合理的地方被标记为 [BAD](http://golang.org/test/escape_slice.go)



