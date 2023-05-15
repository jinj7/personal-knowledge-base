
# 1. regexp

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    pattern := regexp.MustCompile("welcome ([A-z]*) New ([A-z]*) city")
    welcomeMessage := "Hello guys, welcome to New York city"

    subStr := pattern.FindStringSubmatch(welcomeMessage)
    fmt.Println(subStr)

    for _, s := range subStr {
        fmt.Println("Match:", s)
    }
}
```

Output:
```sh
[welcome to New York city to York]
Match: welcome to New York city
Match: to
Match: York
```

## 1.1 Naming subexpressions

Subexpressions can also be named to ease processing of the resulting outputs.

Here is how to name a subexpression: within parentheses, add a question mark, followed by an uppercase P, followed by the name within angle brackets.

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    pattern := regexp.MustCompile("welcome (?P<val1>[A-z]*) New (?P<val2>[A-z]*) city")
    welcomeMessage := "Hello guys, welcome to New York city"

    subStr := pattern.FindStringSubmatch(welcomeMessage)
    fmt.Println(subStr)

    idx := pattern.SubexpIndex("val1")
    if idx != -1 {
        fmt.Println("val1:", subStr[idx])
    }

    idx = pattern.SubexpIndex("val2")
    if idx != -1 {
        fmt.Println("val2:", subStr[idx])
    }
}
```

Output:
```sh
[welcome to New York city to York]
val1: to
val2: York
```

## 1.2 Raw Strings

It’s convenient to use \`raw strings\` when writing regular expressions, since both ordinary string literals and regular expressions use backslashes for special characters.

A raw string, delimited by backticks, is interpreted literally and backslashes have no special meaning.

```go
grep, _ := regexp.Compile(`\w{1,}`)
grep, _ := regexp.Compile("\\w{1,}")
grep, _ := regexp.Compile("\w{1,}")     /// illegal
```

&nbsp;


# 2. pprof

Package pprof serves via its HTTP server runtime profiling data in the format expected by the pprof visualization tool.

To use pprof, link this package into your program:
```go
import _ "net/http/pprof"
```

Add "net/http" and "log" to your imports and the following code to your main function:
```go
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

View all stacks of goroutines:
```sh
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```
