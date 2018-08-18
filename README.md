
Key entities:
* **module**: Describes a .h/.c file pair. Used for code and makefile generation.
  * executable: Whether the module should be compiled to an executable. If true, a main()
                function will be automatically defined in the .c file. See 'entry point'
                below for more details.
  * project\_deps: The names of other modules defined in the project that this module
    depends on. This will automatically add #includes to the header file of the
    module (for now) and wil automatically add the necessary linking dependencies to the
    Makefile for the module.
  * external\_deps: The names of other header files that should be included from
    projects not written in the cdna format. 
  * external\_libs: The names of other libraries that need to be linked into
    this module.
* **function**:
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
* **struct**:
  * name: The name of the struct
  * module: The module the struct belongs to
  * visibility:
    * 'public': Declares the struct in the .h file for the module. Also defines
      a 'typedef struct Foo Foo;'.
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
* **type**: An alias in javascript for a type in C. When a type is defined with
  defineType, the type will be accessible from anywhere as 't.MyAlias'. This 
  currently does **not** define a typedef in C.
  * ctype: The corresponding C type for the alias
  * alias: The alias for the type
* **metatype**: A javascript function that takes in a type (which is just a string) and outputs
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

Example of the use of the raw interface to implement a basic array class for integers:

```c
defineType({
  ctype: "int",
  alias: "Int"
})

defineType({
  ctype: "size_t",
  alias: "Size"
})

defineType({
  ctype: "void",
  alias: "Nothing"
})

defineMetatype({
  name: "Ptr",
  func: function(T) { return T + "* "; }
})

defineModule({
  name: "IntArray",
  executable: false,
  project_deps: [],
  external_deps: ["assert", "stdio", "stdlib"],
  external_libs: []
})

defineStruct({
  name: "IntArray",
  module: "IntArray",
  visibility: "private",
  data: {
    // The size of the array
    size: t.Size,
    // The data in the array
    data: mt.Ptr(t.Int);
  }
})

defineFunction({
  name: "IntArray_Create",
  module: "IntArray",
  visibility: "public",
  inp: { size: t.Size, init_value: t.Int },
  out: mt.Ptr(t.IntArray), // Structs automatically get their own type
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
  inp: { self_ptr: mt.Ptr(mt.Ptr(IntArray)) },
  out: t.Nothing, // Structs automatically get their own type
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
  inp: { self: mt.Ptr(IntArray) },
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
  inp: { self: mt.Ptr(IntArray), idx: t.Size },
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
  inp: { self: mt.Ptr(IntArray), idx: t.Size, val : t.Int },
  out: t.Int, 
  def: () => {
.    assert(idx < self->size);
.    self->data[idx] = val;
  }
})
```

And perhaps a test:
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

// Let's get fancy and define a macro for setting up a test
var setUpIntArrayTest = function(size, initVal) {
.  size_t size = @{size};
.  int init_val = @{initVal};
.  IntArray* arr = IntArray_Create(size, init_val);
}

// This isn't that useful but let's do it for fun anyways
var tearDownIntArrayTest = function() {
.  IntArray_Destroy(&arr);
}

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
.     @{setUpIntArrayTest(3, 0)}
.     for (size_t i = 0; i < IntArray_GetSize(arr); ++i) {
.       assert(IntArray_Get(arr, i) == 0);
.     }
.     // Yay more macros
.     @{tearDownIntArrayTest()}
   }
  }
})
```

But what if we got even FANCIER!? We can pass in the body of the test as a function. I'm not even arguing that this is the best way to do anything, but it's so plastic and fun it makes me happy to play with:
```c
var defineIntArrayTest(size, initVal, testBody) {
.  @{setUpIntArrayTest(size, initVal)}
.  @{testBody()}
.  @{tearDownIntArrayTest()}
}

var testThatCreateCreatesAnArrayOfTheCorrectSize = {
  array_size: 3,
  init_val: 0,
  body: () => {
.   assert(IntArray_Size(arr, idx) == val);
  }
}

var testThatSetCorrectlySetsTheValueAtTheRightIndex = {
  array_size: 3,
  init_val: 0,
  body: () => {
.   size_t idx = 0;
.   int val = 3;
.   IntArray_Set(arr, idx, val);
.   assert(IntArray_Get(arr, idx) == val);
  }
}

var tests = [testThatCreateCreatesAnArrayOfTheCorrectSize, testThatSetCorrectlySetsTheValueAtTheRightIndex]

defineFunction({
  name: "RunTests",
  module: "IntArray",
  visibility: "private",
  inp: {},
  out: t.Nothing, 
  def: () => {
    for (test in tests) {
.     @{defineIntArrayTest(test.array_size, test.init_val, test.body)} 
    }
  }
})
```
Then define a file `package.js.dna` as follows: 
```
./!include("IntArray.js.dna")
```
and run 
```
node ribosome.js cgen.js.dna
```

What do we get as a result? A header and implementation file for each module, and a makefile with targets to build everything and run the tests. The makefile generation currently is pretty rudimentary but it works.
