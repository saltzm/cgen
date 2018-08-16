
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
  * inp: Object of the form { argName : argType, ..., lastArgName : lastArgType }
  * out: Specifies the return type of the function
  * def: Contains the body of the function in ribosome format
* **struct**:
  * module: The module the struct belongs to
  * visibility:
    * 'public': Declares the struct in the .h file for the module. Also defines
      a 'typedef struct Foo Foo;'.
    * 'private': Declares 'typedef struct Foo Foo;' in the .h file, and
      declares/defines the struct in the .c file
  * data:
    * Object of the form { fieldName : fieldType, ..., lastFieldName : lastFieldType }
* **entry point** (TBD): A function to be called directly from main(). This is required
  for a module that's executable. This could change to be a normal private
  function, with the module object specifying the name of the function to be
  used as the entry point in the case where the module is executable. This
  probably makes more sense.
* **enum**: Declares/defines an enum
  * name: The name of the enum
  * module: The module the enum belongs to
  * visibility: 'public'/'private'
  * entries: An object of the format { name : value, ... } for each enum entry. Also
             accepts a list of names if the default values may be used.
* **type**: An alias in javascript for a type in C. When a type is defined with
  defineType, the type will be accessible from anywhere as 't.MyAlias'
  * ctype: The corresponding C type for the alias
  * alias: The alias for the type
* **metatype**: 
  * A javascript function that takes in a type (which is just a string) and outputs
    a new type string. When defined, the metatype will be accessible from
    anywhere as mt.MyMetatype(t.SomeOtherType). An easy example of this would
    be mt.Ptr(t.Int) as a pointer to an int. It would expand to "int \*". Enums and 
    Unions could also be defined this way (and may be in the future).
* **constant**: A free-floating key-value pair to define as a constant.
  * name: The name of the constant
  * module: The module the constant belongs to
  * visibility: 'public'/'private'
  * value: The value of the constant.

