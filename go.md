
# 1. Install

```sh
wget https://go.dev/dl/go1.19.2.linux-amd64.tar.gz

rm -rf /usr/local/go && tar -C /usr/local -xzf go1.19.2.linux-amd64.tar.gz

export PATH=$PATH:/usr/local/go/bin

go version
```

&nbsp;


# 2. Basic Commands

```sh
go mod init xxx

go mod tidy

go mod vendor
```

&nbsp;


# 3. Struct

## 3.1 embedded field

A field declared with a type but no explicit field name is called an embedded field. An embedded field must be specified as a type name T or as a pointer to a non-interface type name \*T, and T itself may not be a pointer type. The unqualified type name acts as the field name.

```c
// A struct with four embedded fields of types T1, *T2, P.T3 and *P.T4
struct {
    T1        // field name is T1
    *T2       // field name is T2
    P.T3      // field name is T3
    *P.T4     // field name is T4
    x, y int  // field names are x and y
}
```

## 3.2 tags

Go offers struct tags which are discoverable via reflection. These enjoy a wide range of use in the standard library in the JSON/XML and other encoding packages.

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "os"
    "time"
)

type User struct {
    Name          string    `json:"name"`
    Password      string    `json:"-"`
    PreferredFish []string  `json:"preferredFish,omitempty"`
    CreatedAt     time.Time `json:"createdAt"`
}

func main() {
    u := &User{
        Name:      "Sammy the Shark",
        Password:  "fisharegreat",
        CreatedAt: time.Now(),
    }

    out, err := json.MarshalIndent(u, "", "  ")
    if err != nil {
        log.Println(err)
        os.Exit(1)
    }

    fmt.Println(string(out))
}
```

| extra attributes | meanings |
| :---  | :---  |
| ,omitempty | removing empty JSON fields |
| - | ignoring private fields |

References:

1. [How To Use Struct Tags in Go](https://www.digitalocean.com/community/tutorials/how-to-use-struct-tags-in-go)
2. [Well known struct tags](https://github.com/golang/go/wiki/Well-known-struct-tags)

&nbsp;


# 4. Import Declarations

| Import declaration | Local name of Sin |
| :---  | :---  |
| import   "lib/math" | math.Sin |
| import m "lib/math" | m.Sin |
| import . "lib/math" | Sin |

To import a package solely for its side-effects (initialization), use the blank identifier as explicit package name:

```go
import _ "lib/math"
```

