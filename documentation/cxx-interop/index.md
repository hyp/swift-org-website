---
layout: page
title: Mix Swift and C++
official_url: https://swift.org/documentation/cxx-interop/
redirect_from: 
- /documentation/cxx-interop.html
---
<!-- {% comment %}
The width of <pre> elements on this page is carefully regulated, so we
can afford to drop the scrollbar boxes.
{% endcomment %} -->
<style>
article pre {
    overflow: visible;
}
</style>

<!-- {% comment %}
Define some variables that help us build expanding detail sections
without too much boilerplate.  We use checkboxes instead of
<details>...</details> because it allows us to:

  * Write CSS ensuring that details aren't hidden when printing.
  * Add a button that expands or collapses all sections at once.
{% endcomment %} -->
{% capture expand %}{::nomarkdown}
<input type="checkbox" class="detail">
{:/nomarkdown}{% endcapture %}
{% assign detail = '<div class="more" markdown="1">' %}
{% assign enddetail = '</div>' %}

<div class="info screenonly" markdown="1">
To facilitate use as a quick reference, the details of many guidelines
can be expanded individually. Details are never hidden when this page
is printed.
<input type="button" id="toggle" value="Expand all details now" onClick="show_or_hide_all()" />
</div>

## Table of Contents
{:.no_toc}

* TOC
{:toc}

## Introduction

This document is the reference guide describing how to mix Swift and C++. It
describes how C++ APIs get imported into Swift, and provides examples showing
how various C++ APIs can be used in Swift. It also describes how Swift APIs
get exposed to C++, and provides examples showing how the exposed Swift APIs
can be used from C++.

Bidirectional interoperability with C++ is supported in Swift 5.9 and above. 

* * *

<div class="info" markdown="1">
C++ interoperability is an actively evolving feature of Swift. The [status page](status) provides an
overview of the currently supported interoperability features.
</div>

## Overview

This section provides the basic high-level overview of how
Swift interoperates with C++.

### Enabling C++ Interoperability

Swift code interoperates with C and Objective-C APIs by default.
You must enable interoperability with C++ if you want to use
C++ APIs from Swift, or expose Swift APIs to C++.

The following guides describe how C++ interoperability can be enabled when
working with a specific build system or IDE:

<div class="links" markdown="1">
[Read how to mix Swift and C++ in an Xcode project](TODO)

[Read how to use C++ APIs from Swift in a Swift package](TODO)

[Read how to use CMake to mix Swift and C++](TODO)
</div>

Other build systems can enable C++ interoperability by passing in the required
flag to the Swift compiler:

<div class="links" markdown="1">
[Read how to enable C++ interoperability when invoking Swift compiler directly](TODO)
</div>

### Importing C++ into Swift

Header files are commonly used to describe the public interface of a
C++ library. They contain type and template definitions, and also
declarations for functions and methods, whose bodies are often placed into
implementation files that are then compiled by the C++ compiler.

The Swift compiler embeds the [Clang](https://clang.llvm.org/) compiler.
This allows Swift to import C++ header files using
[Clang modules](https://clang.llvm.org/docs/Modules.html). Clang modules
provide a more robust and efficient semantic model of C++ headers as
compared to the preprocessor-based model of directly including the contents of 
header files using the `#include` directive.

> C++20 introduced C++ modules as an alternative to header files.
> Swift cannot import C++ modules yet.

### Creating a Clang Module

In order for Swift to import a Clang module, it needs to find a 
`module.modulemap` file that describes how a collection of C++ headers maps
to a Clang module. 

<!-- {% comment %}
TODO: talk about SwiftPM generating module map for umbrealla header.
TODO: talk about Xcode generating module map?
{% endcomment %} -->

Some IDEs and build systems can generate a module map file for a
C++ build target. In other cases you might be required to create a module
map manually.

The recommended way to create a module map is to list all the header
files from a specific C++ target that you want to make available to Swift.
For example, let's say we want to create a module map for a C++
library `forestLib`. This library has two header files: `forest.h` and `tree.h`.
In this case we can follow the recommended approach and create a module map
that has two `header` directives:

```shell
module forestLib {
    header "forest.h"
    header "tree.h"

    export *
}
```

The `export *` directive is another
recommended addition to the module map.
It ensures that the types from Clang modules imported 
into the `forestLib` module are visible to Swift as well.

The module map file should be placed right next to the header files it
references. 
For example, in the `forestLib` library, the module map would
go into the `include` directory:

~~~ shell
forestLib
├── include
│   ├── forest.h
│   ├── tree.h
│   └── module.modulemap [NEW]
├── forest.cpp
└── tree.cpp
~~~

Now that `forestLib` has a module map, Swift can import it when
C++ interoperability is enabled. In order for Swift to find the `forestLib`
module, the build system must pass the import path flag (`-I`) that 
points to `forestLib/include` when it's invoking the Swift compiler. 

For more information on the syntax and the semantics of module map files, please
see Clang's
[module map language documentation](https://clang.llvm.org/docs/Modules.html#module-map-language).

### Working with Imported C++ APIs

The Swift compiler represents the imported C++ types and functions
using Swift declarations once a Clang module is imported. This allows Swift code
to use C++ types and functions as if they were Swift types and functions.

For example, the following C++ class from the `forestLib` library: 

```c++
class Tree {
public:
  Tree(TreeKind kind);
private:
  TreeKind kind;
};
```

Is represented as a `struct` inside of the Swift compiler:

```swift
struct Tree {
  init(_ kind: TreeKind)
}
```

It can then be used directly in Swift, just like any other
Swift `struct`:

```swift
import forestLib

let tree = Tree(.Oak)
```

Even though Swift has its own internal representation of the C++ type,
it doesn't use any kind of indirection to represent a
value of such type. That means that
when you're creating a
`Tree` from Swift, Swift invokes the C++ constructor directly and stores
the produced value directly into the `tree` variable.

### Exposing Swift APIs to C++

In addition to importing and using C++ APIs, the Swift compiler is also
capable of exposing Swift APIs from a Swift module to C++. This makes it
possible to gradually integrate Swift into an existing C++ codebase,
as the newly added Swift APIs can still be accessed from C++.

Swift APIs can be accessed by including a header
file that Swift generates. The generated header uses C++ types and functions
to represent Swift types and functions. When C++ interoperability is enabled,
Swift generates C++ bindings for all the supported public types and functions
in a Swift module. For example, the following Swift function:

```swift
// Swift module 'forestRenderer'
import forestLib

public func renderTreeToAscii(_ tree: Tree) -> String {
  ...
}
```

Will be present in the header generated by the Swift compiler for the
`forestRenderer` module. It can then be called directly from C++ once the C++
file includes the generated header:

```c++
#include "forestRenderer-Swift.h"
#include <string>
#include <iostream>

void printTreeArt(const Tree &tree) {
  std::cout << (std::string)forestRenderer::renderTreeToAscii(tree);
}
```

The [C++ interoperability status page](status) describes which Swift
language constructs and standard library types can be exposed to C++.

## Using C++ APIs from Swift

A wide array of C++ types and functions can be used from Swift. This section
dives into the details of how the supported types and functions can be
used from Swift in a safe and ergonomic manner.

### Invoking C++ Functions

C++ functions from imported modules can be invoked using the
familiar function call syntax from Swift. For example, this C++ function:

```c++
void printWelcomeMessage(const std::string &name);
```

Can be invoked directly from Swift as if it was a regular Swift function:

```swift
printWelcomeMessage("Thomas");
```

### C++ Structures and Classes Are Value Types By Default

Swift maps C++ structures and classes to Swift `struct` types by default.
Swift considers them to be
[value types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures#Structures-and-Enumerations-Are-Value-Types). This means that they're always copied
when they're passed around in your Swift code.

The special members of a C++ structure or class type are used by Swift when it
needs to perform a copy of a value or dispose of a value when it goes out of
scope. If the C++ type has a copy
constructor, Swift will use it when a value of such type is copied in
Swift. And if the C++ type has a destructor, Swift will call the destructor when
a Swift value of such type is destroyed.

As of Swift 5.9, C++ structures and classes with a deleted copy constructor
are not available in Swift. Non-copyable C++ structures or classes that also
have a move constructor will be available in a future version of Swift.
They will map to
[non-copyable](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md) Swift `structs`. In the meantime, such C++ types can be annotated
with [Swift-provided annotations for reference types](TODO) to make them
available in Swift. 

### Constructing C++ Types From Swift

Public constructors inside C++ structures and classes
that aren't copy or move constructors 
become initializers in Swift. 

For example, these constructors of the C++ `Color` class:

```c++
class Color {
public:
  Color();
  Color(float red, float blue, float green);
  Color(float value);

  ...
  float red, blue, green;
};
```

Become initializers. They can be called from Swift to create a value of type
`Color`: 

```swift
let theEmptiness = Color()
let oceanBlue = Color(0.0, 0.0, 1.0)
let seattleGray = Color(0.7)
```

### Accessing Data Members Of a C++ Type

The public data members of C++ structures and classes become properties
in Swift. For example, the data members of the `Color` class shown above
can be accessed just like any other Swift property:

```swift
let color: Color = getRandomColor()
print("Today I'm feeling \(color.red) red but also \(color.blue) blue")
```

### Calling C++ Member Functions

Member functions inside C++ structures and classes become methods in
Swift.
Constant member functions become `nonmutating` Swift methods, whereas
member function without a `const` qualifier become `mutating` Swift methods.
For example, this member function in the C++ `Color` class:

```c++
void Color::invert() { ... }
```

Is considered to be a `mutating` method in Swift:

```swift
var red = Color(1.0, 0.0, 0.0)
red.invert() // red becomes yellow.
```

And as such it can't be called on constant `Color` values. However,
this constant member function:

```c++
Color Color::inverted() const { ... }
```

Is not a `mutating` method in Swift, and thus it can be called on a constant
`Color` value:

```swift
let darkGray = Color(0.2, 0.2, 0.2)
let veryLightGray = darkGray.inverted()
```

Static C++ member functions become `static` Swift methods.

### Using C++ Enumerations

Scoped C++ enumerations become Swift enumerations with raw values.
All of their cases get mapped to Swift cases as well. For example,
the following C++ enumeration:

```c++
enum class TreeKind {
  Oak,
  Redwood,
  Willow
};
```

Is represented in Swift as the following enumeration:

```swift
enum TreeKind : Int32 {
  case Oak = 0
  case Redwood = 1
  case Willow = 2
}
```

As such, it can be used just like any other `enum` in Swift:

```swift
func isConiferous(treeKind: TreeKind) -> Bool {
  switch treeKind {
    case .Redwood: return true
    default: return false
  }
}
```

Unscoped C++ enumerations become Swift structures. Their cases become
variables outside of the Swift structure itself. For example, the following
unscoped enum:

```c++
enum MushroomKind {
  Oyster,
  Portobello,
  Button
}
```

Is represented in Swift as the following structure:

```swift
struct MushroomKind : Equatable, RawRepresentable {
    public init(_ rawValue: UInt32)
    public init(rawValue: UInt32)
    public var rawValue: UInt32
}
var Oyster: MushroomKind { get }
var Portobello: MushroomKind { get }
var Button: MushroomKind { get }
```

### `#import <swift/bridging>`

The following header will be in the toolchain search paths already, so you just need to import it. Once this header is imported, you'll have access to several annotations that are helpful for preparing your C++ headers (APIs) for Swift to import them. Some of the most helpful annotations:

TODO 

### Operators 

C++ operators can usually be mapped to similar Swift operators. Sometimes, to promote Swift’s idioms, operators are imported semantically rather than directly:
* `operator++` is mapped to a `successor` method in Swift
* `operator--` is mapped to a `predecessor` method in Swift 
* `operator*` is mapped to a `pointee` property
* `operator[]` is mapped to a subscript in Swift
* `operator()` is mapped to Swift's `callAsFunction`.

Note: currently Swift cannot import templated operators.

### Ranges -> Collections

Swift will attempt to import C++ ranges as Swift Collections. This means that types with a `begin` and `end` method will be automatically conformed to Swift's `Collection` protocol. When using a C++ type that's been imported as a Swift `Collection`, you'll have acceess to all of the methods that Swift `Collection`s provide by default: `map`, `filter`, and so on. You'll also have access to some extra features, such as intializers that convert the C++ type to `Array` and `Set`. For example, if `vec` is a C++ `std::vector`, you will be able to autoamtically convert it to an array like this: `Array(vec)`. You can also iterate over `vec` in a for-loop: `for element in vec { print(element) }`. Or map it to a `String`: `vec.map(String.init)`. 

**Why isn't my range beging bridged?**

Sometimes your ranges aren't bridged automatically and it can be hard to figure out why. If your range isn't automatically conformed to the Swift `Collection` protocol, it's likely that we are having some trouble importing the range's iterator type. To figure out what's wrong, try manually conforming your range to `CxxCollection`. It's likely that you will get an error that your iterator does not conform to `CxxIterator`, so again, try conforming the type manually. For example:

```c++
// C++ header file, defines a std::vector specialization:
using Vector = std::vector<int>;
```

```swift
// Manual conformance produces: does not conform to protocol 'CxxSequence'
extension Vector: CxxSequence {}

// Next, try adding the `Element` protocol requirement manually.
extension Vector: CxxSequence {
  public typealias Element = Vector.const_iterator.Pointee
}

// Next, try manually adding the iterator conformance.
extension Vector.const_iterator: UnsafeCxxInputIterator {}

// Finally, try manually adding the `Equatable` conformance.
extension Vector.const_iterator: Equatable {
    public static func==(lhs: Vector.const_iterator, rhs: Vector.const_iterator) -> Bool {
        <# for you to implement... #>
    }
}
```

It's fairly common for `Equatable` conformance to be the issue, so it may be a good idea to check that your iterator is `Equatable` as a first debugging step. It's fairly common for the compiler to be unable to automatically add an `Equatable` protocol conformance becuase we _cannot import templated operators_ (see above section). So, if your `operator==` is templated in C++, we won't be able to use it in Swift for the automatic `Equatable` conformance. You can work around this by providing a non-templated operator in C++ or just implementing it again in Swift (see example above).

**Safety**

It's unsafe to use C++ iterators, such as those returned from `begin` or `end` methods in Swift. With these C++ iterators it's easy to create lifetime issues if the iterator is used after the C++ type is destroyed (which lead to use-after-free), or even invalid memory accesses (if the iterators mis-match or read past the end of the range). So, you should use these Swift `Collection` APIs instead.

**Beyond Collections**

Swift also will automatically import C++ types that look like dictionaries as `CxxDictionary`s. `CxxDictionary` refines `CxxSequence` to provide a few other niceties, namely, a subscript operator that can look up key-value pairs, so 

 ```c++
// C++ header file, defines a std::map specialization:
using Map = std::map<std::string, std::string>;
```

```swift
func findAlex(inMap map: Map) -> std.string {
    map["Alex"]
}
```

The subscript is implemented using the map's `find` method. 

When iterating over a `CxxDictionary`, each element will be a `CxxPair`. _C++ types with a `first` and `second` member will automatically be conformed to `CxxPair`._

### Other special cases

Swift also provides a few niceties when using common C++ APIs. For example, you can construct a C++ string from a Swift string and vice-versa. You can also construct a C++ string from a string literal:
```swift
let greeting: std.string = "Hello, World!" 
```

Another example is `Optional`. To convert a C++ optional (such as `std.optional`, but any user defined type will also work) to a Swift optional, simply call `Optional(fromCxx:)`, passing the C++ optional.

Swift will also automatically synthesize `Equality` conformance, but only if a _non-templated_ `operator==` is present in C++. (We are working on `Hashable` as well.)

### Class templates

Currently, unspecialized class templates (i.e, `std.vector`) are not supported in Swift. Swift and C++ have very different models for "generic programming" and we have not yet finished developing the story around class template bridging. See some of the designs outlined in the vision document and this forum post:

* [Vision Document]( https://github.com/zoecarver/swift/blob/docs/interop-roadmap/docs/CppInteroperability/ForwardVision.md)

* [This forum post](https://forums.swift.org/t/bridging-c-templates-with-interop/55003)

### Move only types

C++ types with a move constructor and no copy constructor can be represented using Swift's new non-copyable types. These types will be imported automatically as as Swift non-copyable types when you pass `-Xfrontend -enable-experimental-move-only` to the compiler. This is expiremental, so there are lots of bugs. Use with caussion.

If a type is not copyable or moveable, it can be mapped to a Swift class or reference type (as a foreign reference type). See the secion on foreign reference types to determine if this is the correct mapping for your type.

### Reference types

Whether a C++ class type is appropriate to import as a reference type is a complex question, and there are two primary criteria that go into answering it.

The first criterion is whether object identity is part of the "value" of the type. Is comparing the address of two objects just asking whether they're stored at the same location, or it is deciding whether they represent the "same object" in a more significant sense? 

The second criterion whether objects of the C++ class are always passed around by reference.  Are objects predominantly passed around using a pointer or reference type, such as a raw pointer (`*`), raw reference (`&` or `&&`), or smart pointer (like `std::unique_ptr` or `std::shared_ptr`)?  When passed by raw pointer or reference, is there an expectation that that memory is stable and will continue to stay valid, or are receivers expected to copy the object if they need to keep the value alive independently?  If objects are generally allocated and remain at a stable address, even if that address is not semantically part of the "value" of an object, the class may be idiomatically a reference type. This will sometimes be a judgment call for the programmer.

(In the future we hope to support polymorphic C++ classes as well, but the current implementation does not suppor these types.)

The first and most important criteria is often not possible for a compiler to answer automatically by just looking at the code. So, if you want the Swift compiler to import a C++ type as a refernece type, you must communciate this via one of the three reference attribute from teh Swift bridging header: 

  - **Immortal** reference types are not designed to be managed individually by the program. Objects of these types are allocated and then intentionally "leaked" without tracking their uses. Sometimes these objects are not truly immortal: for example, they may be arena-allocated, with an expectation that they will only be referenced from other objects within the arena. Nonetheless, they aren't expected to be individually managed.

    The only reasonable thing Swift can do with immortal reference types is import them as unmanaged classes.  This is perfectly fine when objects are truly immortal.  If the object is arena-allocated, this is unsafe, but it's essentially an unavoidable level of unsafety given the choices of the C++ API.
    
    To specify that a C++ type is an immortal reference type, use the TODO attribute. 

  - **Shared** reference types are reference-counted with custom retain and release operations. In C++, this is nearly always done with a smart pointer like `std::shared_ptr` rather than expecting programmers to manually use retain and release. This is generally compatible with being imported as a managed type. Shared pointer types are either "intrusive" or "non-intrusive", which unfortunately ends up being relevant to semantics. `std::shared_ptr` is a non-intrusive shared pointer, which supports pointers of any type without needing any cooperation.  Intrusive shared pointers require cooperation but support some additional operations. Swift should endeavor to support both.
  
    To specify that a C++ type is a shared reference type, use the TODO attribute. This attribute expects two arguments: a retain and release function. These functions must be global functions that take exactly one argument and return void. The argument must be a pointer to the C++ type. Swift will call these custom retain and release functions where it would otherwise retain and release Swift classes. 
    
  - **Unique** reference types are owned by a single context at once, which must ultimately either destroy it or pass ownership of it to a different context. There are two common idioms for unique ownership in C++. The first is that the object is passed around using a raw pointer (or sometimes a reference) and eventually destroyed using the `delete` operator. The second is that this is automated using a move-only smart pointer such as `std::unique_ptr`. This kind of use of `std::unique_ptr` is often paired with "borrowed" uses that traffic in raw pointers temporarily extracted from the smart pointer; in particular, method calls on the class via `operator->` implicitly receive a raw pointer as `this`.
    
    Unique reference types have expiremental support tied to move-only types. See move only types section for more information.
  
TODO provide some examples (or maybe just link to an example project).


## Using C++ Standard Library from Swift


## Using Swift APIs from C++

This section describes how APIs in a Swift module
get exposed to C++. It then goes into details of how to use Swift APIs from C++.
