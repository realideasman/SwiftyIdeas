# Property Behaviors

Property Behaviors seem cool and I'd like to see them in Swift. The examples are quite
intriguing and love to be able to use them in some of my existing projects.

The rational for rejecting Property Behaviors was that the community had not yet converged
on a design for them. This design includes the functionality and the notation of the feature.

I'd like to propose an alternative syntax to the one in the proposal. I'd also like to propose an
alternative symeantic that compliments the syntax.

Lastly, to fully utilise the existing features provided by the compiler, I'd like to introduce a new
concept, conditional `abstract` variables.

## Pitfalls of the current syntax and notation method.

The property behavior declaration in the currenty proposal states that behaviors are declared
like so:

    var behavior lazy<Value>: Value {
        ...
    }

This unfortunately is extremly similar a variables declaration:

    var behavior: Value = ...
    
Given that Swift is aging towards 5 years old, I think the potential for confusion is very high.
If you would like a more compelling example, then have a look at this:

    var behavior: Int = {
        set { }
        get { return 0 }
    }
    
    var behavior foobar<X>: Value {
        set { }
        get { return 0 }
    }
    
Here are two closely related declarations, except one just adds an extra identifier after
another. Semantically, we are doing something completely different however between the two
declarations. The first one declares a variable, the second introduces a new type (of behavior).

Up until now, every new line that has started with `<whitespace>var<whitespace>` has
been always been a declaration of a variable. If the current syntax in the proposal,
`var behaviour identifier`, is accepted, then we'd ruin our assumption when scanning
code. This of course could be mitigated with syntax highlighting, although we need to consider
whether we should be relying on them, and whether there could be a more meaningful way to
annotate this.

## `initialValue`

The `initialValue` concept behaves as though it is a magic variable sythesized and made
available in every behavior declaration context. You could explain it's funcitonality by using
existing features in the language `get` like so:

    var initialValue: Value {
        get {
            return computedExpressionHere
        }
    }

One could liken it to the appearance of `newValue` inside of a variable's `set` method,
however there appears to be no obvious way to change the identifier binding. I also consider
re-evaluating expression to be a bad idea. It doesn't make sense to re-evaluate the result
from a function. Is it going to call the function twice when it's retriving it's value?

## `self`

Breaking the trend of `self` is a bad idea. It will be the only place where `self` will be
declared as an `optional` `weak` reference. There'll be more on how I'd like to use this
instead.

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
