# Congruent Structures

Suppose we have three absolutely superb modules: Foo, Bar and Soap; and each of them contain the same declaration:

    struct Point {
      var x: Int
      var y: Int
    }
  
Unfortunately, when we consume our superb modules in our super functional application, we have to provide initalizers to convert
between each of different types. And if we are good little coders, then we'd provide all 6 intialisers to convert between each of the
technically different but `written the same way` structures.

How about we just tell the compiler that these two objects in different modules are the same thing, like so:

  Foo.Point ≡ Bar.Point
  Foo.Point ≡ Soap.Point
  // => Bar.Point ≡ Soap.Point

Once this has been declared, the types become synonymous to each other.

## Technical Difficulties

The above is nice in a theoretical world, however there are some limits that restrict this idea. Consider what were to happen if two
of the structures that had the same declaration were laid out using different methods (perhaps from different versions of the compiler).
Congruence would state that we can pass the wrong type to a function!

Some options here would be to either synthesis initalizers, or issue a compiler warning.
