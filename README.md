
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
.   assert(self);
.    self->size = size;
.    self->data = malloc(self->size * sizeof(int));
.    assert(self->data);
.    for (size_t i = 0; i < self->size; ++i) {
.      self->data[i] = init_value;
.    }
.    return self;
.  }
})
```
