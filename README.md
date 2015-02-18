# NAME

Functional::Types - a Haskell-inspired type system for Perl

# SYNOPSIS

    use Functional::Types;

    sub ExampleType { newtype }
    sub MkExampleType { typename ExampleType, Record(Int,String), @_ }

    type my $v = ExampleType;
    bind $v, MkExampleType(42,"forty-two");
    say show $v;
    my $uv = untype $v;

# DESCRIPTION

Functional::Types provides a runtime type system for Perl, the main purpose is to allow type checking and have self-documenting data structures. It is strongly influenced by Haskell's type system. More details are below, but at the moment they are not up-to-date. The /t folder contains examples of the use of each type.

# AUTHOR

Wim Vanderbauwhede <Wim.Vanderbauwhede@mail.be>

# COPYRIGHT

Copyright 2015- Wim Vanderbauwhede

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

# SEE ALSO

# NEWTYPE

The function of newtype is to glue typename information together with the constructor information, and typecheck the arguments to the constructor.
I think it is best to treat all cases separately:

    - Primitive types: e.g. sub ArgType { newtype String,@_ } # Does ArgType expect a String or a bare value? I guess a bare value is better?
    - Record types: e.g. sub MkVarDecl { newtype VarDecl, Record( acc1 => ArgType, acc2 => Int), @_ }
    - Variant types: e.g. sub Just { newtype Maybe(a), Variant(a), @_ }
    - Map type: sub HashTable { newtype Map(String,Int), @_ } is a primitive type
    - Array type: sub IntList { newtype Array(Int), @_ } is a primitive type

I expect String to be called and it will return \['$',String\] so ArgType(String("str")) should typecheck

    String("str") will return {Type => ['$',String], Val =>"str"}

    MkVarDecl will return {Type => ['~',MkVarDecl,[],VarDecl,[]], Val => ...}
    Just(Int(42)) will return {Type => ['|',Just,[{a => 'any'}],Maybe,[{a => 'any'}}, Val => {Type => ['$',Int], Val => 42}}
    

To typecheck this against type Maybe(Int) will require checking the type of the Val
So maybe newtype must do this: if the typename or type ctor (yes, rather) has a variable then we need the actual type of the value

# TYPECHECKING

This is a bit rough. We should maybe just check individual types, and always we must use the constructed type as a starting point, and the declared type to check against.
We typecheck in two different contexts:

    1/ Inside the newtype() call: for every element in @_, we should check against the arguments of the type constructor. 
    2/ Inside the bind() call: this call takes a typed value. For this typed value, all we really need to check is if its typename matches with the declared name. 

I think it might be better to have the same Type record structure for every type:

    Variant, Record: ['|~:', $ctor, [@ctor_args],$typename,[@typename_args]]

    Map, Tuple, Array: ['@%*', $ctor, [@ctor_args], $typename=$ctor,[]]

    Scalar: ['$', $ctor, [@ctor_args], $typename=$ctor,[]]

# BIND

    bind():
    
    bind $scalar, Int($v);
    bind $list, SomeList($vs);
    bind $map, SomeMap($kvs);
    bind $rec, SomeRec(...); 
    bind $func, SomeFunc(...);

For functions, bind() should do:

    - Take the arguments, which should be typed, typecheck them;
    - call the original function with the typed args
    - the return value should also be typed, just return it.

So it might be very practical to have a typecheck() function

Furthermore, we can do something similar to pattern matching by using a variant() function like this:

    given(variant $t) {
          when (Just) : untype $t;
          when (Nothing) : <do something else>
    }
    

So variant() simply extracts the type constructor from a Variant type.

# PROTOTYPES

\* These are \*not\* to be called directly, only as part of a newtype call, unless you know what you're doing.

\* I realise it would be faster for sure to have numeric codes rather than strings for the different prototypes. 

The prototype call returns information on the kind of type, the type constructor and the arguments. Currently:

\* PRIM, storing untyped values:

    Scalar: ['$', $type], Val = $x => NEVER used as-is
    
    Array: ['@', $type], Val = [@xs] 
    Hash: ['%', [$ktype,$vtype]], Val = {@kvpairs}
    Tuple: ['*', [@tupletypes]], Val = [@ts]

\* PROPER, storing only typed values:

    Variant: ['|', $ctor, [@ctor_args],$typename,[@typename_args]], Val = ???
    Record: ['~', $ctor, [@ctor_args],$typename,[@typename_args]], Val = ???
    Record with fields: [':', $ctor, [@ctor_args_fields],$typename,[@typename_args]] , Val = {}

\* FUNCTION, the function can itself take typed values or untyped ones, depending on cast() or bind()
    What we store is actually a wrapper around the function, to deal with the types
    So we should somehow get the original function back. I think we can do this by calling the wrapper without any arguments,
    in which case it should return a typed value with the function's type in Type and the original function in Value
    Anyhow untype() only makes sense for a function that works on untyped values of course

    Function: ['&',[@function_arg_types]], Val = \&f

In a call to type() the argument will only return \[$typename,\[@typename\_args\]\]
For a scalar type I could just return $typename but maybe consistency? 

In a newtype call, the primitive types don't have a constructor.
There is some asymmetry in the '$' type compared to the others:

Normally the pattern is Prototype($typename) but for primitive types it is just Scalar() and the prim type's typename comes from caller()

Also, prim types are created without newtype(), I think I should hide this behaviour.

Maybe I need to distinguish between a new data and a type alias, it would certainly clarify things; 
Also, I guess for a type alias for a prim type we can feed it an untyped value.
