Upon the path to world domination, Swift will eventually need scripting support right?

## The problem.

At the moment there is very little support for dynamically loading and executing swift code from source. Consider the Swift Package
Manager and what it must do to parse the manifest file (`Package.swift`):

1. First it must first expose an API, `PackageDescription`
2. Then it compile and execute a script
3. Finally it must parse and validate the output of the script.

That is quite a lot of work to perform in each project when wanting to provide scripting support, it would be nice to have an efficient
and seemless way to load a module of code and execute parts of it at runtime. What would be even more awesome, would be to verify that
only a particular set of API calls are used within that module of code, so that you can guarentee that it's safe to execute.

There is a performance impact that occurs when transfering the description of the packgage that's in the manifest to the swift package
manager. It is most likely that the same Swift complier was used to compile both the Swift Package Manager and the manifest file, which
means that the representations of objects could be guarenteed to be the same between the two executables. If this is so, then we could
eliminate the parsing from the above steps, reducing it to just needing to expose an API and to compiling the module.

## Mechanics

To complie Swift code, we need a copy of the Swift Compiler. We could place the burden on the developer to ensure that a copy of the
Swift compiler is the exact same version as the one used to compile the project, and that the executable can run on the target
architecture. This would be provided at runtime and verified to be the same as what was used to compile the project.

    class Compiler {
        init(locatedAt path: String) throws
        func compile(_ target: Target) throws
    }

    let swift = try Compiler(locatedAt: "/usr/bin/swift")

What we want to then is provide the exposed public definitions of our API to the compiler so that it can correctly verify our module.
This is when we would also restrict what API method calls can be used from within the module. We could do this perhaps by using a
whitelist to permit known safe APIs.

    let exposedAPI = try Requirements(path: "PackageDescription.requirements")

Now declare that a particular module must contain specific declarations. Either we could use exposed runtime functions to check this,
or we could provide a forward declaration file. The forward 
    
    let target = try Target(path: "path/to/source/folder/", exposing: exposedAPI)
    let module = try swift.compile(target)
    if let package = module.runAndExtract(variable: "package") {
        // now we have our package description.
    }
    
    if let function = module.function(named: "updatePosition(_:)") as? Function<inout CGPoint, Void> {
        function.call(&playerPosition)
    }
