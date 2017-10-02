# Property Behaviors

I like the inital proposal for property behaviors, and I'd like to refine the idea. Here are my thoughts on the concepts, some of them could be integrated into the current proposal?

## Concept

A behavior may be attached to any variable declaration. It provides an alternative implementation for the `get` and `set` methods provided by default from the compiler. In addition, the behavior also is given a reference to the containing scope, which allows for interesting uses such as updating related properties on demand. A behavior may additionally require accessor methods to be implemented at the use site, which provides a trivial notification interface.

Since Swift permits a variable to have it's `get` and `set` implementation altered from the compiler's default one, we are able to change the concept of what it means to store a value into a variable. That is:

    var x: Int {
        get {
            return data read from file
        }
        set {
            write data to file
        }
    }

What this implies is that we are able to provide our own definition of:
1. The assignment operator.
2. Evaluation of a particular variable.

Both are intriguing to play around with, but will refer to these concepts using traditional conventions.

## Syntax

Declare property behaviors like this:

    behavior Foo {
        ...
    }

This way:
• We aren't confused as to whether we are declaring a variable or a behavior.
• We follow the pattern of 'one word per declaration' for types. That is, it looks similar to `class`, `struct`, `enum` and `protocol`.

Use the behavior like so:

    var x: Int~Foo
    
You may like to read this as: variable x, who's type is Int, with a behavior of Foo. Only one behavior may be attached to a variable. It is done at it's declaration. The solution for stacking behaviors is solved using inheritence (have a read futher on).

The use syntax was chosen because:
1. We declare type-like things using CamelCase.
2. The concept of behaviors will be used in a very similar manner to `struct`, `class`, etc...
3. Breaking the convention of CamelCase seemed weird.
4. Using `@attribute` notation required lowerCamelCase.
5. `@attribute` was already being used for comipler attributes... let's choose a different way of notating the concept.
6. `~` represents a relationship between the two identifiers, and follows the similar use of `&` for attaching protocol requirements onto an existing type.
7. It was the neatest of all of the other syntax that I considered.

Arguable, the sigil is located in the wrong place.

## Associated Types

1. Behaviors are given access to the containing scope.
2. Variables act as a translation function of values passing in between storage and temporary memory (ish, this isn't strictly true, but traditionally, that's what variables were).
3. We live in a type safe world (aka Swift).

Thus: We need to know the type of the containing scope and the variable we are interacting with.

Protocols introduce the concept of associated types, which allows requirements to be attached to a type:

    protocol Foo {
        associatedType Bar: Sequence
    }

Behaviors will have the same mechanism for interacting with the two previously mentioned types:

    behavior Foo {
        associatedType Owner
        associatedType Value
    }

• `Owner` is the type of the containing scope. The type of the object if the variable is decalared under a `struct`, `class`, `enum` or `protocol`. `Void` for all other scopes.
• `Value` is the type of the variable that is being stored.

These associated types will be implicitly available, authors will not be required to declare them each time they would like to use them. It is a useful concept for teaching others.

    behavior Foo where Value: Sequence, Owner: Codable {
        ...
    }

Note:
    It's interesting to think about the rule placed on the type of `Owner`. What if there was a way to talk about the scope of variables in a function? Or perhaps the global scope? If we could talk about the global scope in this way, then perhaps we could attach protocol requirements on it... This would have practical use cases for plugin development, where you could require that declared in the global scope of a plugin is a variable with a particular type. The Swift Package Manager seems to need something along these lines, with the requirement of the `package` variable.

## Initalizers

Behaviors must exactly one of the following initalisers:
1. `init(initialValue: Type)`
2. `init()`

Initaliser 1 is called when the variable is first initalised with a value, the typical use case will be where `Type == Value`. In the typical case, out-of-band initalisation is allowed.  `Type != Value`  is the atypical use case, and permits a variable to be initalized with a different type.

Generic parameters are acceptable for initaliser 1 (note that in the above example that `Type` is refering to a concret object, like `String`).

Initializer 2 will be called at the declaration site of the variabile. `get` calls are permitted before a call to `set`.

## Protocols

Behaviors are not permitted to conform to protocols. It does not make sense for:

    behavior Lazy: Sequence {
        ...
    }

There is currently no way to access a behavior's variables directly from outside of the behavior. Additionally, a better reason for not permitting this is that behaviors them selves are never explicity instantiated. You can never have a reference to a behavior... but you can, because there is `self`. This needs some more thought.

When we can access some of the values stored on a behavior they will exist, but they will also have a one to one relationship with the variable's declaration. Ugh...

## `get` and `set`

In order to be evaluatable, a behavior must provide the `get` accessor method:

    behavior Foo where Value: ExpressibleByIntegerLiteral {
        get {
            return 42
        }
    }

In order for a variable to be writable, the behavior must provide a `set` accessor method:

    behavior Foo {
        set {
            ... // save `newValue` somewhere... or not
        }
    }

It is an error to provide a `set` implementation without a `get` implementation, however it is not an error to declare a behavior without a `get` accessor. This is because at the site of the variable declaration we can provide accessor implementations, therefore there is still a chance for the `set` implementation to be provided.

    // using Foo from above...
    var superDuperRandomNumber: Int~Foo {
        get {
            return 42 // chosen by a random dice roll
        }
    }

`get` and `set` are special accessors, because they have syntax provided by the language (mentioned at the start of the document).

If `get` and `set` are not provided by the behavior, then the compiler will synthesize the implementations for them by allocating storage for the variable. In this particular case, a behavior may implement `didSet` and `willSet` on behalf of the variable declaration. `value` will be the identifier for the allocated storage and will be accessable to the behavior.

    behavior Foo {
        didSet {
            print(value)
        }
    }
    
    var x: Int~Foo = 0
    var y: Int = 0 {
        didSet {
            print(y)
        }
    }
    
    x = 123 // prints '123'
    y = 456 // prints '456'

I suspect that these rules are tedios to learn. We would not have the new `value` being available if `self` refered to it, however I don't like that concept. Another alternative would be to not provide this feature, and instead require developers to implement the storage themselves if they needed access to it. Another alternative would also be to provide `Let` and `Var` as implementations of this particular functionality. Then developers could inherit from that behavior and receive a reference to the storage.

## Accessors

Behaviors may declare accessor methods. An accessor method is a closure requirement that must be provided at the declaration site of the variable. Like protocols, accessor methods may declare a default implementation, otherwise they must be provided either at the declaration site or by additonal subbehaviors.

    behavior Observable where Value: Equatable {
        accessor didChange(_ oldValue: Value)
        
        var storage: Value
        
        get {
            return storage
        }
        
        set {
            let oldValue = storage
            storage = newValue
            if storage != newValue { didChange(oldValue) }
        }
        
        init(initialValue: Value) {
            storage = initialValue
        }
    }
    
    var x: Int~Observable {
        didChange {
            print("x changed to: \(x)")
        }
    }
    
    x = 1 // calls behavior initalizer
    x = 2 // prints '2'
    x = 2 // prints nothing
    x = 1 // prints '1'
    
Accessors may be mutating or nonmutating. If mutating, then they must be attached to a `var`iable.

## Variables and Functions

Behaviors can:
- have stored variables (which themselves can have a behavior attach)
- implement methods
- have a nested enum, class or struct

Variables decalared apart of a behavior must follow the normal conventions of initalisation that classes and structures follow. Note that this concept will modify the conventions of initalisation becausue a behavior may not nessesarily need to be initalised before it is read.

## Access Modifiers

Access modifiers are different to accessors. An accessor is a closure that is attached to a behavior while access modifiers indicate the API visibility of the code.

Behaviors are apparently fragile, thus, they must always be explicitly internal. Additionally, because access modifiers have not yet been defined for behaviors, they are not permitted on any declarations inside of the behavior. By always annotating a behavior as `internal`, they will be source compatible if the semantics of access modifiers changes for them in the future.

### Suggestions for access modifiers meaning

    [private..<open] behavior Foo where Value: Observable {
        [internal..<open] accessor didChange(_ oldValue: Value)
        [private..<open] var observers: [Observer]
        [private..<open] func postNotification() { // ... }
        [internal..<open] init() { // ... }
        [internal..<open] init(initialValue: Value) { // ... }
    }

## Generic Paramaters or Associated Types

Not thought about yet.

## KeyPaths

It would be nice to tell the behavior what it's `KeyPath` is. This perhaps could be exposed with a good compile time metadata API.

## Alternative syntax

Declaraing the behavior:

    behavior Foo { ... }
    var behavior Foo { ... }
    
# Examples re-written from the proposal

## Lazy

    behavior Lazy {
        var value: Value? = nil
        var expression: () -> Value
        
        init(initialValue expr: @autoclouser @escaping () -> Value) {
            expression = expr
        }
        
        mutating get {
            if let value = self.value {
                return value
            } else {
                value = expression()
                return value!
            }
        }
        
        mutating set {
            self.value = newValue
        }
    }


