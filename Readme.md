Gobinarycoverage
==============================================

![Mender logo](resources/images/mender_logo.png)

Gobinarycoverage is a tool created for instrumenting regular Go packages with a
main file, with coverage functionality, so that it can be built and run just
like a regular Go binary, but does still gather coverage information whilst
running.

The normal approach for doing this is through 'go test', and calling a main
utility function from a test function, in which case, 'go test' will capture
coverage information. However, this approach has some drawbacks. Most notably,
the 'go test' instrumented binary did not behave like out normal binaries when
the binary did fancy stuff, like opening and closing file-descriptors.

Through the approach taken by `Gobinarycoverage` the binary will function just
like normal (there is no surrounding framework playing with I/O or other
things).


## Usage

Call `gobinarycoverage <package-name>`, where `<package-name>` is the name of
the package in which the main file is located. This will then automatically add
coverage functionality to all the packages imported by main, and generate a new
'main.go' file, which is a merge of some utility functions created by
`Gobinarycoverage`, and the functions already present in the main.go file.

Most notably, a `reportCover()` function is added to the source code. This
function needs to be called before exiting the binary. This means that the
source code is not yet fully functional, it needs some human intervention.

### Example

File `main.go` before running `Gobinarycoverage` on it
```go
package main

import (
    "os"
)

func doMain() int {
   <do-stuff>
   ...
   ...
}

func main() {
    os.Exit(doMain())
}
```

After running `Gobinarycoverage` the file will now look similar to:
```go
package main

import (
    "testing"
    "fmt"
    "os"
)

func coverReport() {
   <coverage-magic>
   ...
   ...
}

func doMain() int {
   <do-stuff>
   ...
   ...
}

func main() {
    os.Exit(doMain())
}

```

Thus in order to have this program capture the coverage during an execution,
`coverReport()` needs to be called explicitly, like so:

```go
package main

import (
    "testing"
    "fmt"
    "os"
)

func coverReport() {
   <coverage-magic>
   ...
   ...
}

func doMain() int {
   <do-stuff>
   ...
   ...
}

func main() {
    ret = doMain()
    coverReport()
    os.Exit(ret)
}

```

This does require some human intervention, but it was regarded as better, than
doing magic stuff, like capturing syscalls like `os.Exit` before calling
`coverReport()`.

Some programs also have exit paths that do not go back out through `func main`.
Some minor human effort is therefore needed. For the projects this is used in
currently, this is done through a `git patch`.

Then, finally, the binary can be built, and expected to function just like the
regular binary would. Which is pretty cool.

## How it works

The tool is taking advantage of existing go tools' functionality. Notably, it
uses `go list`, in order to figure out which packages `main.go` imports. From
this information, it runs `go tool cover` on the returned packages. This will
change the source code in the given packages to add in a counter at each block,
and a `GoCover` struct to each file, which is responsible for collecting the
information.

An instrumented function will afterwards look like:

```go
func (m *MenderError) Error() string {GoCover5.Count[2] = 1;
        var err error
        if m.fatal {GoCover5.Count[4] = 1;
                err = errors.Wrapf(m.cause, "fatal error")
        } else{ GoCover5.Count[5] = 1;{
                err = errors.Wrapf(m.cause, "transient error")
        }}
        GoCover5.Count[3] = 1;return err.Error()
}
```

And the struct present in every file, collecting this information looks like:

```go
var GoCover5 = struct {
        Count     [2]uint32
        Pos       [3 * 2]uint32
        NumStmt   [2]uint16
} {
        Pos: [3 * 8]uint32{
                35, 37, 0x20025, // [0]
                39, 41, 0x20026, // [1]
        },
        NumStmt: [8]uint16{
                1, // 0
                1, // 1
        },
}
```

This struct, then has to be imported into the new `main.go` file. This means
that the tools needs to generate source code on the fly, which imports these
structs from every file it covers. Imports in the new `main.go` file will then
look something like:

```go
package main

import (
        "fmt"
        "io/ioutil"
        "testing"

        _cover0 "github.com/mendersoftware/mender/app"

        _cover1 "github.com/mendersoftware/mender/cli"
        
        ...
        ...

```

Which the `coverReport()` then takes advantage of in order to collect the
coverage information from all the packages imported.

## License

Gobinarycoverage is licensed under the Apache License, Version 2.0. See
[LICENSE](https://github.com/mendersoftware/mender/blob/master/LICENSE) for the
full license text.
