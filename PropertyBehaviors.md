# Property Behaviors

## Motivation... the need for better syntax

I don't like the current property syntax. It resembles too closely to variable declarations, which is
confusing to read. Am I reading the declaration of a variable or the declaration of a behavior for a variable?

Up until now, every new line that has started with `<whitespace>var<whitespace>` has been always been a declaration
of a variable. If the current syntax in the proposal, `var behaviour identifier`, is accepted, then we'd ruin our
assumption when scanning code.

Although this could be aided with the use of syntax highlighting, there are other questions that are raised with the
current syntax:
- `initialValue` is written as though it's an identifier in our program, why?
- If `initialValue` is re-evaluated everytime it's referenced, then shouldn't it be declared as a function?
- `self` is ambigious inside of the behavior, why must we break the trend of what `self` refers to in this
    specific scenario?

## My alternative, introducing a new type: Behavior

Consider now protocol and class inheritence. We shall use a similar mechanism here to describe `behavior`s.
All behaviors must inherit from either the `let` behavior or the `var` behavior. For explanitory purposes, we can
write the existing behavior of `let` using our new `behavior` as well. A behavior has two associate values. The first
is `Value`, which is the type of object that is being proxied by this behavior. The second is the `Owner`, which is the
type of the containing context. For classes and structures, it is the containing type. For scopped variables, it is the
empty tuple. 

    open behavior let {
        associatedType Value
        // The type this property is attached to.
        associatedType Owner

        // If you combine this with `owner`, then you get this
        // axiom:
        // 
        // 		owner![keyPath: keypath] == self.get()
        //
        public var keypath: KeyPath<Owenr, Value> { get }

        abstract var storage: Value
        public weak var owner: Owner?

        public init(initialValue: Value) {
            // because `storage` is used here, everytime `storage` is used in a 
            // sub-behavior of `let`, this initalizer must be used. 
            // The compiler infers that because the `storage` property exists in
            // the sub-behavior, then it must call the super-behavior's initalizer
            // that initalizes that variable.
            storage = initialValue
        }

        public init() {
            // computed variables, no access to storage is required.
        }

        open accessor get() -> Value
    }

    open behavior var: let {
        open mutating accessor set(newValue: Value)
    }

    public behavior countAccesses: var {

        public accessor didGet(value: Value, count: Int) { 
            // default implementation.
        }

        public mutating accessor didSet(value: Value, count: Int) { 
            // default implementation.
        }

        public accessor get() -> Value {
            precondition(storage != nil)
            getCount += 1
            didGet(value: storage!, count: getCount)
            return storage!
        }

        public accessor set(newValue: Value) {
            storage = newValue
            setCount += 1
            didSet(value: storage!, count: setCount)
        }

        // Used for providing an initial value at declaration, but also
        // for out-of-line assignment.
        init(initialValue: Value) {
            setCount = 1
            storage = initialValue
            super.init(initialValue: initialValue)

            // This can be called because `self` is now initalized.
            didSet(value: storage, count: setCount)
        }

        // Allows for default initialization.
        init() {
            setCount = 0
            storage = nil
            super.init()
        }

        var storage: Value?
        var getCount: Int = 0
        var setCount: Int
    }

    @countAccesses
    var soap: String {
        didGet { (value, count) in
            print("soap gotten \(count) times.")
        }

        didSet { (value, count) in
            print("soap set \(count) times.")
        }
    }

    soap // fatalError: unwrapping nil value!
    soap = "Hello, world!" // calls countAccesses.init(initialValue:), then prints "soap set 1 times."

    // Let's try again.
    soap = "a" // prints "soap set 1 times."
    soap // prints "soap gotten 1 times."
    soap = "b" // prints "soap set 2 times."
