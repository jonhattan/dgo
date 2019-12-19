# dgo (Dynamic Go)

[![](https://goreportcard.com/badge/github.com/lyraproj/dgo)](https://goreportcard.com/report/github.com/lyraproj/dgo)
[![](https://img.shields.io/badge/godoc-reference-blue.svg)](https://godoc.org/github.com/lyraproj/dgo)
[![](https://github.com/lyraproj/dgo/workflows/Dgo%20Build/badge.svg)](https://github.com/lyraproj/dgo/actions)
[![](https://coveralls.io/repos/github/lyraproj/dgo/badge.svg)](https://coveralls.io/github/lyraproj/dgo)

Dgo's main objectives are: Type Constraints, Immutability, Collections, Serialization, Encapsulation, Extendability,
and Performance. It is designed to make working with dynamic values an effortless and type safe task.

## Install
Dgo is a go module and if the Go version is < 1.13, go modules must be enabled. This is done by setting the environment
variable GO111MODULE=on before an attempt is made to install:
```sh
export GO111MODULE=on
```
### Using dgo as a library
To use dgo, first install the latest version of the library:
```sh
go get github.com/lyraproj/dgo
```
Next, include needed packages in your application. Pick what you need from the following packages:
```go
import (
  "github.com/lyraproj/dgo/dgo"
  "github.com/lyraproj/dgo/typ"
  "github.com/lyraproj/dgo/tf"
  "github.com/lyraproj/dgo/vf"
)
```

## Type Constraints

### What's wrong with Go's type system?
Go is a typed language but the types are not very descriptive. It is for instance not possible to declare a type
that corresponds only to a specific range of integers, a string that must confirm to a specific pattern, or a slice
that can contain only integers or floats. The only generic type provided by Go is the empty interface `interface{}`
which corresponds to every possible value in the system. A slice containing ints and floats must be declared as
`[]interface{}` but doing so creates a list that can also contain any other type of value. What's needed is a type
such as `[](int|float)`. Other examples can be `map[string](string|int)` (string keyed map of string
and int values), or `[]0..15` (slice of integers ranging from 0 - 15).

In many situations it is desirable to declare more restrictive type constraints.
[TypeScript](https://www.typescriptlang.org/docs/handbook/basic-types.html) has good model for such constraints.
Others can be found in [Python Type Hints](https://www.python.org/dev/peps/pep-0484/) and
[Puppet Types](/puppetlabs/puppet-specifications/blob/master/language/types_values_variables.md).

### Language syntax
Dgo defines a [type language of its own](docs/types.md) which is designed to be close to Go itself. A parser
and a stringifier are provided for this syntax. New parsers and stringifiers can be added to support other syntaxes. 

### Type Assignability
As with go reflect, types can be compared for assignability. A type is assignable from another type if the other
type is equally or more restricitive, e.g. the type `int` is assignable from the range `0..10` (and all other
integer ranges). The type `string` is assignable from the pattern `/abc/`, the type `"a"|"b"|"c"` or
any other type that restricts a string. A length constrained `string[10,20]` is assignable from `string[12,17]`, etc.

### Type Instance check
A type can be used to validate if a value is an instance of that type. The integer `3` is an instance of the
range type `1..4`, the string `"abc"` is an instance of the pattern type `/b/`, etc.

### The type of a value
All values have a type that is backed by the value itself. The type will consider its value, and only that value,
to be an instance of itself. E.g. string "hello" is represented by the type `"hello"`. That type in turn is assignable
to `string`, `string[5]`, `string[0,10]`, `"hello"|"goodbye"`, but it is not assignable to `string[0,4]` or
`"hi"|"bye"`. In other words, the value type is assignable to another type if the value that it represents is an
instance of that other type.

#### Collection types
The default type for an Array or a Map can be overridden. This is particularly useful when working with
mutable collections. If an explicit type is given to the collection, it will instead ensure that any modifications
made to it will be in conformance with that type. The default type is backed by the collection and changes
dynamically when the collection is modified.

## Immutability

Non primitives in Go (array, slice, map, struct) are mutable and it's the programmers responsibility to ensure that
access to such values are synchronized when they are accessed from multiple go routines.

Dgo guarantees that all values can be 100% immutable by exposing all values through interfaces and hiding the
implementation, thus enabling concurrency safe coding without the need to synchronized use of shared resources
using mutexes.

An`Array` or a `Map` can be created as a mutable collection but can be made immutable by calling the method `Freeze()`
or `Copy(true)` (argument `true` requests a frozen copy). Both calls are recursive and ensures that the collection
and all its contained values are frozen. `Freeze` performs an in-place recursive freeze of all values while `Copy`
will copy unfrozen objects before freezing them to ensure that the original and all its contained values
are not frozen as a consequence of the call.

A frozen object can never be unfrozen. The only way to resume mutability is to do `Copy(false)` which returns a
mutable copy.

## Serialization
Support for JSON is built in to Dgo. YAML support is provided by the [dgoyaml](https://github.com/lyraproj/dgoyaml)
module which also provides a CLI validate parameter types. Support for [gob](https://golang.org/pkg/encoding/gob/) is in
the pipeline.

Transformations between dgo and [cty](https://github.com/zclconf/go-cty) is provided by the
[dgocty](https://github.com/lyraproj/dgocty) module 

Transformations between dgo and [pcore](https://github.com/lyraproj/pcore) is provided by the
[dgocty](https://github.com/lyraproj/dgopcore) module 

## Encapsulation

It's often desirable to encapsulate common behavior of values in a way that relieves the programmer from trivial
concerns. For instance:

- In Go, you cannot define a generic behavior for equality comparison. It's either `==` or `reflect.DeepEqual()`
 and both have limitations. `==` cannot compare slices or structs containing slices. The DeepEqual method compares
 _all_ fields of a struct, exported and unexported. Dgo solves this by letting all values implement the `Equals()`
 method. This method is then used throughout the Dgo framework.
- Go has no concept of a hash code. Keys in hashes may only be values that are considered comparable on Go. Dgo
solves this by letting all values implement the `HashCode()` method. The hash code can be used for several purposes,
including map keys and computing unique sets of values.
- Since Go has generic value (besides the `interface{}`, it provides no way to specify generic natural ordering. Dgo
 provides a `Comparable` interface.

## Extendability

The functionality of the Dgo basic types is exposed through interfaces to enable the type/value system to be expanded.

## Performance

Dgo is designed with performance in mind. The Map implementation is, although it's generic, actually faster
than a `map[string]interface{}`. It is a lot faster than a `map[interface{}]interface{}` required if you want to
use dynamic keyes with Go. Nevertheless, the Dgo Map can use any value as a key, even arrays, maps, and types.

The dgo.Value implementation for primitives like Bool, Integer, and Float are just redefined Go types, and as such,
they consume the same amount of memory and reuses the same logic i.e.: 
```go
type integer int64 // implements dgo.Integer
type float float64 // implements dgo.Float
type boolean bool  // implements dgo.Boolean
```

The dgo.String is different because it caches the hash code once it has been computed. A string is stored as:
```go
type hstring struct {
  s string
  h int
}
```

## How to get involved
We value your opinion very much. Please don't hesitate to reach out. Opinions, ideas, and contributions are more
than welcome. Ping us on the [Puppet Cloudnative Slack](https://app.slack.com/client/T0AQJ2CTU/CHPSJ9L4F), create
an [issue](../../issues), or file a [PR](../../pulls). 

## Pipeline
- dgo annotations for go struct members, e.g.
    ```go
    type Input struct {
      Variables map[string]interface{} `dgo:"map[string]dgo"`
      Parameters map[string]interface{} `dgo:"map[string]{name: string[1], type: dgo, required?: bool}"`
    }
    ```
- Go [gob](https://golang.org/pkg/encoding/gob/) support to enable full binary exchange of values and types.
- Distributed type aliases (aliases using URI reference)
- Type extension, i.e. how a type such as a map with explicit associations can be extended by another type.
