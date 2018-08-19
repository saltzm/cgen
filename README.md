
# What is cgen?
At its simplest, cgen is a tool that lets you write C code mixed with metaprogramming in JavaScript.

At its finest, cgen is a tool that lets you build higher and higher level abstractions for code generation to help you write C code with less and less effort. 

[Jump to examples](#tutorial)

[Jump to the cooler example](#building-a-class-abstraction)

# Why?
C++ template metaprogramming is Turing complete... But would you ever really want to write a whole program in it?

I've been writing almost exclusively in C++ for work since 2015. Even after having read several books about C++ best practices, API design, design patterns, and so on, I still bump into parts of the language from time to time that leave me scratching my head, rummaging through StackOverflow, and asking coworkers for help. (In the latest case, I was trying and failing to pass a move-only type into a lambda capture for a lambda being passed as a function parameter. Turns out it doesn't work with the standard library - you need something like [unique_function](https://naios.github.io/function2/).) About a year and a half ago I was reading through the [ZeroMQ docs](http://zguide.zeromq.org/page:all) for fun (they're a good read, actually, written by Pieter Hintjens, one of the contributors), and came across this tidbit:
>... my preferred language for systems programming is C (upgraded to C99, with a constructor/destructor API model and generic containers). There are two reasons I like this modernized C language. First, **I'm too weak-minded to learn a big language like C++. Life just seems filled with more interesting things to understand.** Second, I find that this specific level of manual control lets me produce better results, faster.

This quote, in addition to the section on [code generation](http://zguide.zeromq.org/page:all#Code-Generation), inspired me and led me down a rabbit hole to the iMatix projects [gsl (the code generation tool described in the ZeroMQ docs)](https://github.com/imatix/gsl), [zproject](https://github.com/zeromq/zproject) and [zproto](https://github.com/zeromq/zproto). 

I was inspired by the concept of ["model-oriented programming" as described in the gsl readme](https://github.com/imatix/gsl#model-oriented-programming). I loved the idea that the programming language itself is the wrong layer of abstraction for solving certain problems, and that just because you *can* add a feature to a language doesn't mean you *should*. The concept of using models plus code generation seemed very appealing. 

I played around with zproject and zproto but there were a few drawbacks. First, navigating a project made with zproject proved difficult. It's a hassle to move between generated code (with disclaimers at the top saying not to modify anything), partially generated code (where the function signature was autogenerated but not the body), and model files. This was the primary issue. XML also isn't my favorite thing in the world, but had everything else been dandy I would have played along.

So, I ditched it, but continued writing things for fun in C on the side, with this idea of model-oriented programming floating around in the back of my brain. 

But then, I came across [ribosome](http://sustrik.github.io/ribosome/), another code generator written by the main author of ZeroMQ, Martin Sustrik. I used it for something small at work and it turned out to be a pleasure to use. Another year passed, I thought about it again, used it again, and then I was off to the races with cgen. My goal was to allow model-oriented programming to be mixed in with ordinary C code in the same file format, so that looking at the generated source code would be largely unnecessary. As a bonus, it was a natural extension to equip it with the ability to generate a Makefile (or whatever build tool I chose) from the same format.

# Core Concepts

From the perspective of cgen, a C program can be divided into a few key components, each of which can be described by a JSON object. The following list describes each component and its corresponding properties:
* **module**: Describes a .h/.c file pair. Used for code and makefile generation.
  * executable: Whether the module should be compiled to an executable. If true, a main()
                function will be automatically defined in the .c file. See 'entry point'
                below for more details.
  * project\_deps: The names of other modules defined in the project that this module
    depends on. This will automatically add #includes to the header file of the
    module (for now) and wil automatically add the necessary linking dependencies to the
    Makefile for the module.
  * external\_deps: The names of other header files that should be included from
    projects not written in the cgen format. 
  * external\_libs: The names of other libraries that need to be linked into
    this module.
* **function**: Declares/defines a function
  * name: The full name of the function
  * module: The module the function belongs to
  * visibility:
    * 'public': Declares the function in the .h file for the module, defines it
      in the .c file
    * 'private': Declares and defines the function in the .c file and marks it
      'static'
  * compiler_directives: A list of things like "inline" to precede the function signature, static should never be necessary though
  * inp: Object of the form { argName : argType, ..., lastArgName : lastArgType }
  * out: Specifies the return type of the function
  * def: Contains the body of the function in ribosome format
* **struct**: Declares/defines a struct
  * name: The name of the struct
  * module: The module the struct belongs to
  * visibility:
    * 'public': Declares the struct in the .h file for the module. Also declares
      'typedef struct Foo Foo;'.
    * 'private': Declares 'typedef struct Foo Foo;' in the .h file, and
      declares/defines the struct in the .c file
  * data:
    * Object of the form { fieldName : fieldType, ..., lastFieldName : lastFieldType }
* **entry point** (TBD): A function to be called directly from main(). This is required
  for a module that's executable. The function currently must accept no arguments and return void but 
  later on should accept command line arguments and return an int.
  * name: The name of the function (should be defined elsewhere separately).
  * module: The module for which this function is an entry point. This module definition must have specified "executable: true".
* **enum**: Declares/defines an enum
  * name: The name of the enum
  * module: The module the enum belongs to
  * visibility: 'public'/'private'
  * entries: An object of the format { name : value, ... } for each enum entry. Also
             accepts a list of names if the default values may be used.
* **type**: An alias in JavaScript for a type in C. When a type is defined with
  defineType, the type will be accessible from anywhere as 't.MyAlias'. This 
  currently does **not** define a typedef in C.
  * ctype: The corresponding C type for the alias
  * alias: The alias for the type
* **metatype**: A JavaScript function that takes in a type (which is just a string) and outputs
    a new type string. When defined, the metatype will be accessible from
    anywhere as mt.MyMetatype(t.SomeOtherType). An easy example of this would
    be mt.Ptr(t.Int) as a pointer to an int. It would expand to "int \*". 
    Unions could also be defined this way.
  * name: The name of the metatype
  * func: Function from type to metatype name
* **constant**: A free-floating key-value pair to define as a constant.
  * name: The name of the constant
  * module: The module the constant belongs to
  * visibility: 'public'/'private'
  * value: The value of the constant.
  
# Tutorial

## Using the lowest level interface

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
  compiler_directives: ["inline"],
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
  compiler_directives: ["inline"],
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
  compiler_directives: ["inline"],
  inp: { self: mt.Ptr(t.IntArray), idx: t.Size, val : t.Int },
  out: t.Nothing, 
  def: () => {
.    assert(idx < self->size);
.    self->data[idx] = val;
  }
})
```

## Defining an executable

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

## Getting fancy with macros

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

## Running cgen

In order to run cgen you'll first need to download the JavaScript version of [ribosome](http://sustrik.github.io/ribosome/).

Create a file `package.js.dna` in the same directory as ribosome.js and cgen.js.dna containing the following: 
```
./!include("IntArray.js.dna")
```
and run 
```
node ribosome.js cgen.js.dna
```

What do we get as a result? A header and implementation file for each module, and a makefile with targets to build everything and run the tests. The makefile generation currently is pretty rudimentary but it works.

## Building a class abstraction

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

## Future work
* Improving Makefile generation or switching to another build tool
* Building up a library of models and utilities
* Experimenting with alternate file formats
* Porting cgen to use the ruby version of ribosome (for ruby enthusiasts)

## Closing thoughts

What I find really cool is that to build the raw interface took only about 200 lines of JavaScript (with the help of ribosome), and then building a layer on top of it (completely separate!) to implement the above class abstraction (plus templated classes) only took an extra 100 or so lines of JavaScript! 

If I wanted to implement a way to enforce inheriting interfaces described by a JSON object (I don't, yet), it would probably only be an additional 50 lines or so. Inheritance of fields in a struct, or actual function definitions? Easy. Want to do make all of your classes add a reference count to their struct and have their destructors always check the reference count before actually freeing memory? Easy. Enforcing a consistent style across files becomes much easier.

Another nice thing is that if you don't like writing raw JSON like this, it's fairly easy to create your own file format (or use an existing format like YAML) and convert it to JSON before running the generator. Building extra tooling around it shouldn't be difficult either.
