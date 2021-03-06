
# What is cgen?
At its simplest, cgen is a tool that lets you write C code mixed with metaprogramming in JavaScript.

At its best, cgen is a tool that lets you build higher and higher level abstractions for code generation to help you write C code with less and less effort.

Core features:
* C with metaprogramming in JavaScript
* Automated #includes and makefile generation from a single dependency list for .h/.c file pairs
* Ability to integrate with third-party C libraries
* Extreme extensibility

[Jump to examples](#tutorial)

[Jump to the cooler example](#getting-more-opinionated-with-our-class-design)

# Quick Start

To get started with cgen, first install the JavaScript version of [ribosome](http://sustrik.github.io/ribosome/download.html). Next:

```bash
$ mkdir cgen_hello_world
$ cd cgen_hello_world
$ curl https://raw.githubusercontent.com/saltzm/cgen/master/cgen.js.dna -o cgen.js.dna
```

Create a file called `HelloWorld.cdna` in the `cgen_hello_world` directory you
just created and copy and paste the following:
```c
defineModule({
  name: "HelloWorld",
  executable: true, // This should be executable, yay. See entry point below.
  project_deps: [], 
  external_deps: ["assert", "stdio", "stdlib"],
  external_libs: []
})

defineEntryPoint({
  name: "Main",
  module: "HelloWorld"
})

defineType({
  name: "Nothing",
  ctype: "void"
})

defineFunction({
  name: "Main",
  module: "HelloWorld",
  visibility: "private",
  inp: {},
  out: t.Nothing, 
  def: () => {
    var greetings = [ "Hello", "Bonjour", "Hola" ]
    greetings.forEach((greeting) => {
.     printf("%s, world!\n", "@{greeting}");
    })
  }
})
```

And create a file called `package.js.dna` in the same directory with the following line:
```
./!include('HelloWorld.cdna')
```

To generate the C code and Makefile, run the following command (assuming an alias 'ribosome' to run 'node ribosome.js' from wherever it's located):
```bash
$ # Assuming you've downloaded ribosome.js to this directory
$ node ribosome.js cgen.js.dna
```

And then to run the code:
```bash
$ make
$ ./build/HelloWorld
Hello, world!
Bonjour, world!
Hola, world!
```

Inside the source directory you'll see a file HelloWorld.h (that's empty except a header guard) and HelloWorld.c: 

```c
#include "HelloWorld.h"

#include "assert.h"
#include "stdio.h"
#include "stdlib.h"


static
void
Main() {
    printf("%s, world!\n", "Hello");
    printf("%s, world!\n", "Bonjour");
    printf("%s, world!\n", "Hola");
}

int main() {
    Main();
}
```

If you're unconvinced that this is useful or concerned that this is too verbose, check out 
[an example of using cgen to write classes in C](#getting-more-opinionated-with-our-class-design) for a more powerful example.

# Use Case

cgen is designed for people who appreciate the simplicity of C as a language, but who want an easy way to implement higher level abstractions that would be difficult to implement with macros alone. Using JavaScript as a metaprogramming language for C leads to a flexibility that's hard to match. Examples of abstractions you could build (some of which are in examples):
* [Classes](#building-a-class-abstraction)
* [Generic containers](#generic-containers)
* [Adding a reference count to all classes](#getting-more-opinionated-with-our-class-design)
* Enforce adherence of classes to an interface
* Generate serialization code for structs automatically
* Define a state machine using JavaScript, automatically defining the necessary enums and structs for states, events, and messages

Beyond constructs that generate C code directly, it's also easy to add arbitrary JavaScript code to do other things, like text your boss every time you run cgen. (Okay, there's probably a more practical example, like generating documentation or running a protobuf compiler, but that's less fun.) 

This paradigm is extremely powerful, and I'm excited to see where it leads.

# Personal Motivation
C++ template metaprogramming is Turing complete... But would you ever really want to write a whole program in it?

I've been writing almost exclusively in C++ for work since 2015. Even after having read several books about C++ best practices, API design, design patterns, and so on, I still bump into parts of the language from time to time that leave me scratching my head and rummaging through StackOverflow. (In the latest case, I was trying and failing to pass a move-only type into a lambda capture for a lambda being passed as a function parameter. Turns out it doesn't work with the standard library - you need something like [unique_function](https://naios.github.io/function2/).) Needless to say, this can be frustrating at times.

The seed of inspiration for this project was planted in my brain about a year and a half ago when I was reading through the [ZeroMQ docs](http://zguide.zeromq.org/page:all) for fun (they're a good read, actually, written by Pieter Hintjens, one of the contributors), and came across this tidbit:
>... my preferred language for systems programming is C (upgraded to C99, with a constructor/destructor API model and generic containers). There are two reasons I like this modernized C language. First, **I'm too weak-minded to learn a big language like C++. Life just seems filled with more interesting things to understand.** Second, I find that this specific level of manual control lets me produce better results, faster.

This quote, in addition to the section on [code generation](http://zguide.zeromq.org/page:all#Code-Generation), inspired me and led me down a rabbit hole to the iMatix projects [gsl (the code generation tool described in the ZeroMQ docs)](https://github.com/imatix/gsl), [zproject](https://github.com/zeromq/zproject) and [zproto](https://github.com/zeromq/zproto). 

I was inspired by the concept of ["model-oriented programming" as described in the gsl readme](https://github.com/imatix/gsl#model-oriented-programming). I loved the idea that the programming language itself is the wrong layer of abstraction for solving certain problems, and that just because you *can* add a feature to a language doesn't mean you *should*. The concept of using models plus code generation seemed very appealing. 

I played around with zproject and zproto but there were a few drawbacks. First, navigating a project made with zproject proved difficult. It's a hassle to move between generated code (with disclaimers at the top saying not to modify anything), partially generated code (where the function signature was autogenerated but not the body), and model files. This was the primary issue. XML also isn't my favorite thing in the world, but had everything else been dandy I would have played along.

So, I ditched it, but continued writing things for fun in C on the side, with this idea of model-oriented programming floating around in the back of my brain. 

But then, I came across [ribosome](http://sustrik.github.io/ribosome/), another code generator written by the main author of ZeroMQ, Martin Sustrik. I used it for something small at work and it turned out to be a pleasure to use. Another year passed, I thought about it again, used it again, and then I was off to the races with cgen. My goal was to allow model-oriented programming to be mixed in with ordinary C code in the same file format, so that looking at the generated source code would be largely unnecessary. As a bonus, it was a natural extension to equip it with the ability to generate a Makefile (or whatever build tool I chose) from the same format.

# Future work
* Improving Makefile generation or switching to another build tool
* Better integration with external projects
* Building up a library of models and utilities
* Experimenting with alternate file formats
* Polishing rough edges around the basic API, like allowing different compiler directives for header files and .c files, making sure transitive linking dependencies work with external dependencies, etc.
* Create tooling, e.g. vim plugins
* Make sure all generator errors are easy to diagnose. (They aren't bad as is.)
* Porting cgen to use the ruby version of ribosome (for ruby enthusiasts)

# Tutorial

This tutorial runs through the core concepts of cgen and then through a series of examples. All of the below examples can be found in full in the [examples](https://www.github.com/saltzm/cgen/examples) directory.

## Core Concepts

From the perspective of cgen, a C program can be divided into a few key components, each of which can be described by a JSON object. The following list describes each component and its corresponding properties:
* **module**: Describes a .h/.c file pair. Used for code and makefile generation.
  * **executable**: Whether the module should be compiled to an executable. If true, a main()
                function will be automatically defined in the .c file. See 'entry point'
                below for more details.
  * **project\_deps**: The names of other modules defined in the project that this module
    depends on. This will automatically add #includes to the header file of the
    module (for now) and wil automatically add the necessary linking dependencies to the
    Makefile for the module.
  * **external\_deps**: The names of other header files that should be included from
    projects not written in the cgen format. (Makefile integration not fully
    implemented, i.e. this file will be #included but will not add the
    appropriate "-I" flag to the compiler command if it's necessary.)
  * **external\_libs**: The names of other libraries that need to be linked into
    this module.
* **function**: Declares/defines a function
  * **name**: The full name of the function
  * **module**: The module the function belongs to
  * **visibility**:
    * 'public': Declares the function in the .h file for the module, defines it
      in the .c file
    * 'private': Declares and defines the function in the .c file and marks it
      'static'
  * **compiler\_directives** (optional): A list of things like "inline" to precede the function signature, static should never be necessary though
  * **inp**: Object of the form ` { argName : argType, ..., lastArgName : lastArgType } `
  * **out**: Specifies the return type of the function
  * **def**: Contains the body of the function in ribosome format
* **struct**: Declares/defines a struct
  * **name**: The name of the struct
  * **module**: The module the struct belongs to
  * **visibility**:
    * 'public': Declares the struct in the .h file for the module. Also declares
      'typedef struct Foo Foo;'.
    * 'private': Declares 'typedef struct Foo Foo;' in the .h file, and
      declares/defines the struct in the .c file
  * **data**: Object of the form ` { fieldName : fieldType, ..., lastFieldName : lastFieldType } ` defining the fields of the struct.
* **entry point**: A function to be called directly from main(). This is required
  for a module that's executable. The function currently must accept no
  arguments and return void but later on should accept command line arguments
  and return an int. (I also don't love this interface so it may change).
  * **name**: The name of the function (should be defined elsewhere separately).
  * **module**: The module for which this function is an entry point. This module definition must have specified "executable: true".
* **enum**: Declares/defines an enum
  * **name**: The name of the enum
  * **module**: The module the enum belongs to
  * **visibility**: 'public'/'private'
  * **entries**: An object of the format ` { name : value, ... } ` for each enum entry. Also
             accepts a list of names if the default values may be used.
* **type**: An alias in JavaScript for a type in C. When a type is defined with
  defineType, the type will be accessible from anywhere as 't.MyAlias'. This 
  currently does **not** define a typedef in C.
  * **name**: The alias for the type
  * **ctype**: The corresponding C type for the alias
* **metatype**: A JavaScript function that takes in a type (which is just a string) and outputs
    a new type string. When defined, the metatype will be accessible from
    anywhere as ` mt.MyMetatype(t.SomeOtherType) `. An easy example of this would
    be ` mt.Ptr(t.Int) ` as a pointer to an int. It would expand to "int \*". 
    Unions could also be defined this way.
  * **name**: The name of the metatype
  * **func**: Function from type to metatype name
* **constant**: A free-floating key-value pair to define as a constant.
  * **name**: The name of the constant
  * **module**: The module the constant belongs to
  * **visibility**: 'public'/'private'
  * **value**: The value of the constant.

## Running cgen 

See the [quick start](#quick-start) for instructions on running cgen. That's all there is to it.

## cgen Project Structure

cgen is very lightweight and doesn't require much to get it up and running. But it's helpful to know a few concepts and conventions.

First, the `package.js.dna` file is used to tell cgen which files should be included in the project. The order matters, so if a file requires something defined in another file, be sure to order them correctly.

Next, I use the convention of using the `.cdna` extension with camel-case for files containing definitions of things that will get turned into C code, and `.js.dna` with snake-case for files containing code for taking models and generating C code. 

Once your project is set up with all of your `.cdna` and `.js.dna` files listed in `package.js.dna`, several things happen when you run cgen. First, directories called `src` and `build` are created if they don't exist. Then a header and implementation file for each module is created in `src`, and a makefile with targets to build everything is created in the root directory of the project. Executables have the same name as their module and are placed in the `build` directory. The makefile generation currently is pretty rudimentary and needs a lot of work, but it works for the things I've used it for.

## Examples

In order to fully understand everything, it would be helpful to read or at
least skim the [ribosome](http://sustrik.github.io/ribosome/) documentation. If
you don't feel like doing that right now, just know that lines preceded with a
dot ('.') get sent to a file, with macros being JavaScript variables or
functions surrounded by '@{}' that get inserted in-line.

### Using the lowest level interface

Here's an example of the use of the raw interface (spoiler alert: we can make this less verbose later) to implement a basic array class for integers:

```c
defineType({
  name: "Int",
  ctype: "int",
})

defineType({
  name: "Size",
  ctype: "int",
})

defineType({
  name: "Nothing",
  ctype: "void",
})

defineMetatype({
  name: "Ptr",
  def: function(T) { return T + "* "; }
})

defineModule({
  name: "IntArray",
  executable: false,
  project_deps: [],
  external_deps: ["assert", "stdio", "stdlib"],
  external_libs: []
})

defineType({name: "IntArray", ctype: "IntArray"})

defineStruct({
  name: "IntArray",
  module: "IntArray",
  visibility: "private",
  data: {
    // The size of the array
    size: t.Size,
    // The data in the array
    data: mt.Ptr(t.Int)
  }
})

defineFunction({
  name: "IntArray_Create",
  module: "IntArray",
  visibility: "public",
  inp: { size: t.Size, init_value: t.Int },
  out: mt.Ptr(t.IntArray),
  def: () => {
.    IntArray *self = malloc(sizeof(IntArray));
.    assert(self);
.    self->size = size;
.    self->data = malloc(self->size * sizeof(int));
.    assert(self->data);
.    for (size_t i = 0; i < self->size; ++i) {
.      self->data[i] = init_value;
.    }
.    return self;
   }
})

defineFunction({
  name: "IntArray_Destroy",
  module: "IntArray",
  visibility: "public",
  inp: { self_ptr: mt.Ptr(mt.Ptr(t.IntArray)) },
  out: t.Nothing,
  def: () => {
.    assert(self_ptr);
.    assert(*self_ptr);
.    IntArray *self = *self_ptr;
.    free(self->data);
.    free(self);
.    *self_ptr = NULL;
  }
})

// Gets the size of the array
defineFunction({
  name: "IntArray_GetSize",
  module: "IntArray",
  visibility: "public",
  inp: { self: mt.Ptr(t.IntArray) },
  out: t.Size, 
  def: () => {
.    return self->size;
  }
})

// Gets the value at an index of the array
defineFunction({
  name: "IntArray_Get",
  module: "IntArray",
  visibility: "public",
  inp: { self: mt.Ptr(t.IntArray), idx: t.Size },
  out: t.Int, 
  def: () => {
.    assert(idx < self->size);
.    return self->data[idx];
  }
})

// Sets the value at an index of the array
defineFunction({
  name: "IntArray_Set",
  module: "IntArray",
  visibility: "public",
  inp: { self: mt.Ptr(t.IntArray), idx: t.Size, val : t.Int },
  out: t.Nothing, 
  def: () => {
.    assert(idx < self->size);
.    self->data[idx] = val;
  }
})
```

### Defining an executable

Let's add a test executable for our IntArray class:
```c

defineModule({
  name: "IntArrayTest",
  executable: true, // This should be executable, yay. See entry point below.
  project_deps: ["IntArray"], // Need to include *and* link to IntArray
  external_deps: ["assert", "stdio", "stdlib"],
  external_libs: []
})

defineEntryPoint({
  name: "RunTests",
  module: "IntArrayTest"
})
```

And then define a function to run tests:
```c
defineFunction({
  name: "RunTests",
  module: "IntArray",
  visibility: "private",
  inp: {},
  out: t.Nothing, 
  def: () => {
.   // Test that IntArray_Create creates an array with the correct size
.   {
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     assert(IntArray_GetSize(arr) == size);
.     IntArray_Destroy(&arr);
.   }
.   // Test that IntArray_Create correctly initializes all values
.   {
.     // Let's use the macro now
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     for (size_t i = 0; i < IntArray_GetSize(arr); ++i) {
.       assert(IntArray_Get(arr, i) == 0);
.     }
.     // Yay more macros
.     IntArray_Destroy(&arr);
   }
  }
})
```

### Getting fancy with macros

But what if we got FANCY!? Let's play with macros. The way ribosome works is
that you can define a JavaScript function that contains in its body lines
beginning with a '.' which will be sent to output. Then, in any line preceded
with a dot you can call that function like '@{myMacro()}' and it will
substitute the text from that function in place.

As an example, let's write a macro for setting up a test, tearing down a test, and running a test. To run a test, we can pass in the just the body of the test as a function, and automatically do setup and teardown. I'm not even arguing that this is the best way to do anything, but it's so plastic and fun it makes me happy to play with:

```c
var setUpIntArrayTest = function(size, initVal) {
.  size_t size = @{size};
.  int init_val = @{initVal};
.  IntArray* arr = IntArray_Create(size, init_val);
}

// This isn't that useful but let's do it for fun anyways
var tearDownIntArrayTest = function() {
.  IntArray_Destroy(&arr);
}

function defineIntArrayTest(size, initVal, testBody) {
.  {
.    @{setUpIntArrayTest(size, initVal)}
.    @{testBody()}
.    @{tearDownIntArrayTest()}
.  }
}

var testThatCreateCreatesAnArrayOfTheCorrectSize = {
  size: 3,
  init_val: 0,
  body: () => {
.   assert(IntArray_GetSize(arr) == size);
  }
}

var testThatSetCorrectlySetsTheValueAtTheRightIndex = {
  size: 3,
  init_val: 0,
  body: () => {
.   size_t idx = 0;
.   int val = 3;
.   IntArray_Set(arr, idx, val);
.   assert(IntArray_Get(arr, idx) == val);
  }
}

var tests = [testThatCreateCreatesAnArrayOfTheCorrectSize, 
             testThatSetCorrectlySetsTheValueAtTheRightIndex]

defineFunction({
  name: "RunTests",
  module: "IntArrayTest",
  visibility: "private",
  inp: {},
  out: t.Nothing, 
  def: () => {
    tests.forEach(function(test) {
.     @{defineIntArrayTest(test.size, test.init_val, test.body)} 
    })
.   printf("All tests passed!\n");
  }
})
```

### Building a class abstraction

The beauty of having a program represented as a bunch of JSON objects is that it's really easy to build higher and higher level abstractions. For example, it's easy to recreate the above IntArray class in the following (less verbose) way:

```c
// This is needed just so we can use t.IntArray below. Somewhat unfortunate.
defineType({name: "IntArray", ctype: "IntArray"})

defineClass({
  name: "IntArray", // This will define a module called IntArray for us
  struct: { // This will be called IntArray and will be private
    // The size of the array
    size: t.Size,
    // The data in the array
    data: mt.Ptr(t.Int)
  },
  metadata: {
    project_deps: [],
    external_deps: ["assert", "stdio", "stdlib"],
    external_libs: []
  },
  api: {
    Create: {
      inp: { size: t.Size, init_value: t.Int },
      out: mt.Ptr(t.IntArray),
      def: () => {
.       IntArray *self = malloc(sizeof(IntArray));
.       assert(self);
.       self->size = size;
.       self->data = malloc(self->size * sizeof(int));
.       assert(self->data);
.       for (size_t i = 0; i < self->size; ++i) {
.         self->data[i] = init_value;
.       }
.       return self;
      }
    },
    Destroy: {
      inp: { self_ptr: mt.Ptr(mt.Ptr(t.IntArray)) },
      out: t.Nothing,
      def: () => {
.       assert(self_ptr);
.       assert(*self_ptr);
.       IntArray *self = *self_ptr;
.       free(self->data);
.       free(self);
.       *self_ptr = NULL;
      }
    },
    GetSize: {
      inp: { self: mt.Ptr(t.IntArray) },
      out: t.Size,
      def: () => {
.       return self->size;
      }
    },
    Get: {
      inp: { self: mt.Ptr(t.IntArray), idx: t.Size },
      out: t.Int, 
      def: () => {
.       assert(idx < self->size);
.       return self->data[idx];
      }
    },
    Set: {
      inp: { self: mt.Ptr(t.IntArray), idx: t.Size, val: t.Int },
      out: t.Nothing, 
      def: () => {
.       assert(idx < self->size);
.       self->data[idx] = val;
      }
    }
  },
  tests: {
    "IntArray_Create creates an array with the correct size": () => {
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     assert(IntArray_GetSize(arr) == size);
.     IntArray_Destroy(&arr);
    },
    "IntArray_Create correctly initializes all values": () => {
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     for (size_t i = 0; i < IntArray_GetSize(arr); ++i) {
.       assert(IntArray_Get(arr, i) == 0);
.     }
.     IntArray_Destroy(&arr);
    }
  }
})
```

This generates the files IntArray.h/c, IntArrayTest.h/c, and a makefile with a target for building all modules and a target for running the tests (`make test`).

### Getting more opinionated with our class design

The above example (and the corresponding examples in examples/class_interface) gives a good example of a basic class abstraction. However, there are a few more things that we could make our class abstraction do for us. 

First, behind the scenes, the generator for the above class abstraction checks that functions called Create/Destroy exist, and that Create returns a pointer to the class type and that Destroy takes in a pointer to a pointer of the class type and returns void. If I'm going to enforce this anyway, I might as well not require the user to type those signatures in every time. 

Second, we have to include a pointer to self as the first input parameter of every member function. We can remove that and have it added behind the scenes. Yay for less typing!

There's also some things in the constructor that we'll need to do for every class we write - allocate memory for the object itself, assert that exists, and return self. Similarly for the destructor. 

And, while we're at it, what if we want every object of all of our classes to have a reference count, and then have the destructor decrement the refcount before freeing memory, and then define a function IncRefCount to increment the refcount? Let's do it. 

This is the result: 

```c
defineClass({
  name: "IntArray", // This will define a module called IntArray for us
  struct: { // This will be called IntArray and will be private
    // The size of the array
    size: t.Size,
    // The data in the array
    data: mt.Ptr(t.Int)
  },
  metadata: {
    project_deps: [],
    external_deps: ["assert", "stdio", "stdlib"],
    external_libs: []
  },
  api: {
    Create: {
      inp: { size: t.Size, init_value: t.Int },
      def: () => {
.       self->size = size;
.       self->data = malloc(self->size * sizeof(int));
.       assert(self->data);
.       for (size_t i = 0; i < self->size; ++i) {
.         self->data[i] = init_value;
.       }
      }
    },
    Destroy: {
      def: () => {
.       free(self->data);
      }
    },
    GetSize: {
      inp: {}, // self will be defined as a param behind the scenes
      out: t.Size,
      def: () => {
.       return self->size;
      }
    },
    Get: {
      inp: { idx: t.Size },
      out: t.Int, 
      def: () => {
.       assert(idx < self->size);
.       return self->data[idx];
      }
    },
    Set: {
      inp: { idx: t.Size, val: t.Int },
      out: t.Nothing, 
      def: () => {
.       assert(idx < self->size);
.       self->data[idx] = val;
      }
    }
  },
  tests: {
    "IntArray_Create creates an array with the correct size": () => {
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     assert(IntArray_GetSize(arr) == size);
.     IntArray_Destroy(&arr);
    },
    "IntArray_Create correctly initializes all values": () => {
.     size_t size = 3;
.     int init_val = 0;
.     IntArray* arr = IntArray_Create(size, init_val);
.     for (size_t i = 0; i < IntArray_GetSize(arr); ++i) {
.       assert(IntArray_Get(arr, i) == 0);
.     }
.     IntArray_Destroy(&arr);
    }
  }
})
```

And let's check out some generated code for once (comments added post
generation for demonstrative purposes):

```c
#include "IntArray.h"

#include "assert.h"
#include "stdio.h"
#include "stdlib.h"

struct 
IntArray { 
    int size;
    int* data;
    int ref_count; // YAY!
};

IntArray*
IntArray_Create(int size, int init_value) {
    // Check out all this stuff that was automatically added!
    IntArray* self = malloc(sizeof(IntArray));
    assert(self);
    self->ref_count = 1;
    self->size = size;
    self->data = malloc(self->size * sizeof(int));
    assert(self->data);
    for (size_t i = 0; i < self->size; ++i) {
      self->data[i] = init_value;
    }
    return self;
}

void
IntArray_Destroy(IntArray**  self_ptr) {
    assert(self_ptr);
    assert(*self_ptr);
    IntArray* self = *self_ptr;
    // Come on... This is cool, right??
    self->ref_count--;
    if (self->ref_count == 0) {
      free(self->data);
      free(self);
      *self_ptr = NULL;
    }
}

int
IntArray_GetSize(IntArray*  self) {
    return self->size;
}

int
IntArray_Get(IntArray*  self, int idx) {
    assert(idx < self->size);
    return self->data[idx];
}

void
IntArray_Set(IntArray*  self, int idx, int val) {
    assert(idx < self->size);
    self->data[idx] = val;
}

// HERE COMES THE DYNAMITE
void
IntArray_IncRefCount(IntArray*  self) {
    self->ref_count++;
}
```

### Generic containers

"But wait! How can we implement containers that are GENERIC?! Surely it must be nigh
IMPOSSIBLE!"

Don't worry dear reader, I have heard your voice through the ether and thus, I
give you the magic codez for implementing generic containers....!: 

```c
// Nothing to see here.
```

What??? No extra code required???

Yes, yes. Remember those things called metatypes that we used to define mt.Ptr?
They're just a name and a function that returns a string. It's perfectly valid
if, from inside that function, we define a totally new class! This also means
we can create new classes from a template simply by using the metatype
anywhere, much as you would in C++. Check it out:

```c
defineMetatype({
  name: "Array", 
  def: function(T) { 
    // The name of the class that will be generated from this template
    var ElementArray = capitalize(T) + "Array"
    var ElementType = T

    defineType({
      name: ElementArray,
      ctype: ElementArray
    })

    defineClass({
      name: ElementArray,
      metadata: {
        project_deps: [],
        external_deps: ["assert", "stdio", "stdlib"],
        external_libs: []
      },
      struct: { 
        // The size of the array
        size: t.Size,
        // The data in the array
        data: mt.Ptr(ElementType)
      },
      api: {
        Create: {
          inp: { size: t.Size, init_value: ElementType },
          def: () => {
.           self->size = size;
.           self->data = malloc(self->size * sizeof(@{ElementType}));
.           assert(self->data);
.           for (size_t i = 0; i < self->size; ++i) {
.             @{ElementArray}_Set(self, i, init_value);
.           }
          }
        },
        Destroy: {
          def: () => {
.           free(self->data);
          }
        },
        /* ... */
        Get: {
          inp: { idx: t.Size },
          out: ElementType, 
          def: () => {
.           assert(idx < self->size);
.           return self->data[idx];
          }
        },
        /* ... */
      },
      // We can't really define tests in the Array metaclass since we don't know 
      // necessarily how to e.g. initialize elements without knowing what their
      // type will be. We'll have to create a separate class for testing (see
      // below).
      tests: {}
    })
    return ElementArray
  }
})
```

But what if we want our Array to also be able to hold objects and destroy the objects it contains when it's destroyed? This will also let us see the refcounting in action. We can modify the above as follows:

```c
defineMetatype({
  name: "Array", 
  def: function(T, elements_are_objects = false) { 
    // The name of the class that will be generated from this template
    var ElementArray = capitalize(T) + "Array"
    // We're doing some funny stuff here so I'll explain. I want to have this Array
    // work if I'm storing primitive types AND if I'm storing objects of a class.
    // So I have this boolean parameter elements_are_objects that the caller
    // of defineMetaclass can use to say if this array is storing objects or not.
    // If we're storing objects, longernally we'll store a pointer to the object
    // type, and inside Destroy we'll iterate through them and call
    // ElementType_Destroy on each of them to clean up the memory. Since our
    // objects are all refcounted, and we increment the refcount when we add an
    // element to the array, this should be fine! 
    if (elements_are_objects) {
      var ElementType = mt.Ptr(T)
      var ElementClass = T
      var project_deps = [ ElementClass ]
    } else {
      var ElementType = T
      var project_deps = []
    }

    defineType({
      name: ElementArray,
      ctype: ElementArray
    })

    defineClass({
      name: ElementArray,
      metadata: {
        project_deps: project_deps,
        external_deps: ["assert", "stdio", "stdlib"],
        external_libs: []
      },
      struct: { 
        // The size of the array
        size: t.Size,
        // The data in the array
        data: mt.Ptr(ElementType)
      },
      api: {
        Create: {
          inp: { size: t.Size, init_value: ElementType },
          def: () => {
.           self->size = size;
.           self->data = malloc(self->size * sizeof(@{ElementType}));
.           assert(self->data);
.           for (size_t i = 0; i < self->size; ++i) {
.             // While it would be silly to pass in a pointer to an existing
.             // object as the initial value, we'll pass it to the Set path
.             // to make sure the refcount is incremented enough times
.             @{ElementArray}_Set(self, i, init_value);
.           }
          }
        },
        Destroy: {
          def: () => {
            // If the elements are objects we want to destroy them all. For our
            // non-object arrays (like arrays of primitive types) this code won't
            // be generated!
            if (elements_are_objects) {
.             for (long i = 0; i < self->size; ++i) {
.               // We can do this because our class abstraction enforces
.               // the same destructor API across all objects.
.               @{ElementClass}_Destroy(&(self->data[i]));
.             }
            }
.           free(self->data);
          }
        },
        /* ... */
        Set: {
          inp: { idx: t.Size, val: ElementType },
          out: t.Nothing, 
          def: () => {
.           assert(idx < self->size);
            if (elements_are_objects) {
.             if (val) {
.               @{ElementClass}_IncRefCount(val);
.             }
            }
.           self->data[idx] = val;
          }
        }
      },
      tests: {}
    })
    return ElementArray
  }
})
```

Okay, that's all well and good, but how is it *used*? Let's look at the tests:

```c
// See the examples/opinionated_class_interface directory for the full code,
// including the definition of TestObj
defineClass({
  name: "ArrayTest",
  metadata: {
    // Just by using mt.Array we're already generating files LongArray.h/c and
    // TestObjArray.h/c!
    project_deps: [ mt.Array(t.Long), mt.Array(TestObj) ],
    external_deps: ["assert", "stdio", "stdlib"],
    external_libs: []
  },
  struct: {},
  api: {},
  tests: {
    "LongArray_Create creates an array with the correct size": () => {
.     size_t size = 3;
.     long init_val = 0;
.     LongArray* arr = LongArray_Create(size, init_val);
.     assert(LongArray_GetSize(arr) == size);
.     LongArray_Destroy(&arr);
    },
    "LongArray_Create correctly initializes all values": () => {
.     size_t size = 3;
.     long init_val = 0;
.     LongArray* arr = LongArray_Create(size, init_val);
.     for (size_t i = 0; i < LongArray_GetSize(arr); ++i) {
.       assert(LongArray_Get(arr, i) == 0);
.     }
.     LongArray_Destroy(&arr);
    },
    "TestObjArray_Destroy calls destructor of TestObj": () => {
.     long size = 1;
.     TestObj* init_val = NULL;
.     TestObjArray* arr = TestObjArray_Create(size, init_val);
.
.     long test_long = 42;
.     TestObj* test_obj = TestObj_Create(&test_long);
.
.     TestObjArray_Set(arr, 0, test_obj);
.     // Since we're refcounting, we need to destroy this first
.     TestObj_Destroy(&test_obj);
.     TestObjArray_Destroy(&arr);
.     // If we correctly destroyed the TestObj in the destructor, 
.     // this long should be set to 0
.     assert(test_long == 0);
    }, 
    "TestObjArray_Set increments refcount": () => {
.     long size = 1;
.     TestObj* init_val = NULL;
.     TestObjArray* arr = TestObjArray_Create(size, init_val);
.
.     long test_long = 42;
.     TestObj* test_obj = TestObj_Create(&test_long);
.
.     TestObjArray_Set(arr, 0, test_obj);
.     TestObjArray_Destroy(&arr);
.     // Refcount should be non-zero so this shouldn't have been affected
.     assert(test_long == 42);
.
.     TestObj_Destroy(&test_obj);
    }
  }
})
```

Instead of having a boolean parameter we could also just have a global variable
'cl' or something that we can use to check if a given type is a class. We could
also just have separate PrimitiveArray and ObjectArray templates.

# Closing thoughts

What I find really cool is that to build the raw interface took only about 200 lines of JavaScript (with the help of ribosome), and then building a layer on top of it (completely separate!) to implement the above class abstractions (plus templated classes) only took an extra 150 or so lines of JavaScript! 

If I wanted to implement a way to enforce inheriting interfaces described by a JSON object (I don't, yet), it would probably only be an additional 50 lines or so. Inheritance of fields in a struct, or actual function definitions? Easy. Enforcing a consistent style across files in general becomes much easier.

Another nice thing is that if you don't like writing raw JSON like this, it's fairly easy to create your own file format (or use an existing format like YAML) and convert it to JSON before running the generator. Building extra tooling around it shouldn't be difficult either.


