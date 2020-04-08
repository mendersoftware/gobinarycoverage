Gobinarycoverage
==============================================

![Mender logo](mender_logo.png)

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
```
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

```
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

## License

Gobinarycoverage is licensed under the Apache License, Version 2.0. See
[LICENSE](https://github.com/mendersoftware/mender/blob/master/LICENSE) for the
full license text.
