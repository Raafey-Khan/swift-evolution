# Module Aliasing For Disambiguation

* Proposal: [SE-NNNN](NNNN-module-aliasing-for-disambiguation.md)
* Authors: [Ellie Shin](https://github.com/elsh)
* Review Manager: TBD
* Status: **Implemented**
* Pitch: [Forum discussion](https://forums.swift.org/t/pitch-module-aliasing/51737)
* Implementation:
[apple/swift#39376](https://github.com/apple/swift/pull/39376), [apple/swift#39496](https://github.com/apple/swift/pull/39496), [apple/swift#39533](https://github.com/apple/swift/pull/39533), [apple/swift#39628](https://github.com/apple/swift/pull/39628), [apple/swift#39634](https://github.com/apple/swift/pull/39634),  [apple/swift#39705](https://github.com/apple/swift/pull/39705), [apple/swift#39929](https://github.com/apple/swift/pull/39929), [apple/swift#40064](https://github.com/apple/swift/pull/40064),  [apple/swift#40384](https://github.com/apple/swift/pull/40384), [apple/swift#40450](https://github.com/apple/swift/pull/40450), [apple/swift#40494](https://github.com/apple/swift/pull/40494), [apple/swift-driver#912](https://github.com/apple/swift-driver/pull/912), [apple/swift-driver#913](https://github.com/apple/swift-driver/pull/913)

## Introduction

Currently when there are duplicate module names, it fails to compile. The issue has become more widespread as an increasing number of libraries written in Swift has become adopted. There is no module namespace support in Swift, and introducing structural namespacing would be a non-trivial task and most likely source breaking. This proposal introduces a way to disambiguate conflicting modules without requiring source breaking changes, by means of aliasing module names. 

## Motivation

As Swift libraries and packages are more widely adopted, we are increasingly encountering module name clashes. As there’s no module namespace support in Swift, libraries are often forced to be renamed or pinned to a non-conflicting version in such scenario. This makes use cases such as the following challenging: 

* Adding a new dependency or upgrading as it can introduce a collision: A new (or upgraded) module can have the same name as another module that is already in the dependency graph. 
* Upgrading a package from a version pinned by another library in the same dependency graph: If a package is used by multiple libraries in the same graph, only one version of the package can be allowed; this prevents a library from upgrading to a newer version of the package, making migration work much harder when it finally does.  


## Proposed solution

We believe that module aliasing support will provide a more systematic means to address module name collision. By aliasing, i.e. renaming, we can disambiguate the conflicting modules without requiring manual source changes. It will be supported for pure Swift modules only (see the *Requirements / Limitations* section for more details). 

To go over how the aliasing works, let’s consider the following scenario: App depends on module Utils from package swift-game, and now wants to add another dependency, also called Utils but from another package. Currently we will get an error due to the duplicate module names. 
```
App
|— Module Utils (from package ‘swift-game’)
|— Module Utils (from package ‘swift-draw’) // want to add a new dependency
```

As an example, Utils has the following code:
```
[Module Utils] // swift-game

public struct Level { ... }
public var currentLevel: Utils.Level { ... }
```
```
[Module Utils] // swift-draw

public protocol Drawable { ... }
public class Canvas: Utils.Drawable { ... }
```

In the following steps, we will use a compiler invocation command to demonstrate how the conflicting modules are aliased. 

1. First, take the Utils module from swift-game, and rename it to be unique; let’s call it GameUtils. We will need to compile the module (a) by giving a new name GameUtils while (b) treating any references to Utils in its source files as GameUtils.  
    1. The first part (renaming) can be achieved via passing the new name (GameUtils) to `-module-name` and an output path flag, `-o`,  `-emit-module-path`, or `-emit-module-interface-path`, which are all existing flags. The resulting binary then, for example, will be `/path/to/GameUtils.swiftmodule` (instead of `Utils.swiftmodule`).
    2. The second part (treating references to Utils in source files as GameUtils) will be addressed by introducing a new compiler flag  `-module-alias [name]=[binary_name]`, where the name is the module name that appears in source files (Utils), while the binary_name is the name of the binary (GameUtils), i.e. the canonical name that is used for file contents including symbol mangling. Combined with step (a), the compiler invocation command then will be `swiftc -module-name GameUtils -emit-module-path /path/to/GameUtils.swiftmodule -module-alias Utils=GameUtils ...`.    
2. Similarly, we can rename Utils from swift-draw as DrawUtils, following the step 1.  
3. Finally, we build App (without the aliasing flag) and directly import GameUtils and/or DrawUtils. 

[App]
```
import GameUtils
import DrawUtils
```
In practice, a dependency graph will be more complex than the scenario described above. As an example, we have the following modified scenario. App imports module Game, which imports module Utils from the same package. App also imports another module called Utils from a different package.  
```
App 
|— Module Game (from package ‘swift-game’)
|— Module Utils (from package ‘swift-game’)
|— Module Utils (from package ‘swift-draw’)
```

[Module Game] // swift-game
```
import Utils  // swift-game
public func start(level: Utils.Level) { ... }
```

As with the first scenario, the Utils modules are conflicting, so we need to perform the steps 1-2 above. Then we need to build module Game by applying `-module-alias Utils=GameUtils`, so that the references to Utils in source files of module Game are compiled as GameUtils without making any source changes. The compiler invocation command to build Game then is `swiftc -module-name Game -module-alias Utils=GameUtils ...`. App can then import module Game and any of the renamed Utils as needed. 

While module aliasing can be done directly via a compiler invocation command as seen above, most users do not interact with the command directly. Thus we plan to provide an easier access via new build configs which can be adopted by any build systems; in particular, we will focus on how it can be adopted via SwiftPM, described in a later section. 

## Detailed design

### Changes to Swift Frontend

The arguments to the `-module-alias` flag will be validated against reserved names, invalid identifiers, wrong format or ordering (`-module-alias Utils=GameUtils` is correct but `-module-alias GameUtils=Utils` is not). The flag can be repeated to allow multiple aliases, e.g. `-module-alias Utils=GameUtils -module-alias Logging=SwiftLogging`, and will be checked against duplicates. Diagnostics and fix-its will contain the name Utils in the error messages as opposed to GameUtils to be consistent with the names appearing to users. 

The validated map of aliases will be stored in the AST context and used for dependency scanning/resolution and module loading; from the above scenario, if Game is built with `-module-alias Utils=GameUtils` and has `import Utils` in source code, `GameUtils.swiftmodule` should be loaded instead of `Utils.swiftmodule` during import resolution.  

While the name Utils appears in source files, the actual binary name will be used for name lookup, semantic analysis, symbol mangling (e.g. `$s9GameUtils5Level`), and serialization. Since the binary names will be stored during serialization, the aliasing flag will only be needed to build the conflicting modules and their immediate consuming modules; non-immediate consuming modules will not require the flag. 

The module alias map will also be used to disallow any references to the binary module names in source files; only the name Utils should appear in source files, not the binary name GameUtils. This is true only if the `-module-alias` was used to build the module (if the renamed modules were directly imported, the binary module names can appear in source files). This restriction is useful as it can make it easier to rename the module again later if needed, e.g. from GameUtils to SwiftGameUtils.   

Unlike source files, the generated interface module (.swiftinterface) will contain the binary module name in all its references. The binary name will also be stored for indexing and debugging, and treated as the source of truth. 

### Changes to Code Assistance / Indexing

The compiler arguments including the new flag `-module-alias` will be available to SourceKit and indexing. The aliases will be stored in the AST context and used to fetch the right results for code completion and other code assistance features. They will also be stored for indexing so features such as jump to definition can navigate to decls under the binary module names. 

Generated documentation, quick help, and other assistance features will contain the binary module names and treat them as the source of truth. 

### Changes to Swift Driver

The module aliasing arguments will be used during dependency scan for both implicit and explicit build modes; the resolved dependency graph will contain the binary names. In case of the explicit build mode, the dependency input passed to the frontend will contain the binary names in its json file. Similar to the frontend, validation of the flag arguments will be performed at the driver. 

### Changes to SwiftPM

To allow module aliasing more accessible, we will introduce new build configs which can map to the compiler flags for aliasing described above. Let’s go over how they can be adopted by SwiftPM with the second scenario (copied below). 
```
App
|— Module Game (from package ‘swift-game’)
|— Module Utils (from package ‘swift-game’)
|— Module Utils (from package ‘swift-draw’)
```
Manifest swift-game: the Utils module needs to opt in to allow module aliasing; we will introduce a new parameter called `allowModuleAliasing`. The Utils target has to meet the requirements described in the *Requirements/Limitations* section below. A package can vend multiple targets which might not meet the requirements, so this new parameter needs to be specified per target. If opted in, SwiftPM will perform validations including whether the target is a pure Swift module.

```
{
 name: "swift-game",
 dependencies: [],
 products: [
   .library(name: "GameProduct", targets: ["Game"]),
   .library(name: "UtilsProduct", targets: ["Utils"]),
 ],
 targets: [
   .target(name: "Game", targets: ["Utils"]),
   .target(name: "Utils", *allowModuleAliasing*: true, dependencies: ["Logging"])
 ]
}
```

Manifest swift-draw: similar to above, opt-in via `allowModuleAliasing`.
```
{
 name: "swift-draw",
 dependencies: [],
 products: [
   .library(name: "UtilsProduct", targets: ["Utils"]),
 ],
 targets: [
   .target(name: "Utils", *allowModuleAliasing*: true, dependencies: ["Logging"])
 ]
}
```

App manifest: needs to explicitly define unique names for conflicting modules via a new parameter called `moduleAliases`. 
```
{
 name: "App",
 dependencies: [
  .package(url: https://.../swift-game.git),
  .package(url: https://.../swift-draw.git)
 ],
 products: [
  .executable(name: "App", targets: ["App"])
 ]
 targets: [
  .executableTarget(
    name: "App",
    dependencies: [
     .product(name: "GameProduct", *moduleAliases*: {"Utils": "GameUtils"}, package: "swift-game"), 
     .product(name: "UtilsProduct", *moduleAliases*: {"Utils": "DrawUtils"}, package: "swift-draw"), 
   ])
 ]
}
```

SwiftPM will check validations on `moduleAliases`; for each entry in `moduleAliases`, check if the target indeed opts in for `allowModuleAliasing`, and trigger a build with `-module-alias` if validated. An error will be thrown if not opted in or no `moduleAliases` are declared, prompting a user to opt in to resolve duplicate modules. The aliasing option should be supported with a minor version upgrade. 

### Resources

Tools invoked by a build system to compile resources should be modified to handle the module aliasing. The module name entry should get the renamed value and any references to aliased modules in the resources should correctly map to the corresponding binary names. The resources likely impacted by this are IB, CoreData, and anything that explicitly requires module names. We will initially only allow asset catalogs and localized strings for targets to be aliased. 

### Debugging

When module aliasing is used, the binary module name will be stored in mangled symbols, e.g. `$s9GameUtils5Level` instead of `$s5Utils5Level`, which will be stored in Debuginfo.

When evaluating an expression, the name Utils as it appears in source files (that were already compiled with module aliasing) can be used; however, the result will contain the binary module name. 

If a module were to be loaded directly in lldb, the binary module name should be used, so import GameUtils instead of import Utils,  since it doesn’t have access to the aliasing flag. 

In REPL, binary module names should be used for importing or referencing; support for aliasing may be added in the future.


## Requirements / Limitations

To allow module aliasing, the following requirements need to be met.

* Only pure Swift modules allowed for aliasing: no ObjC/C/C++/Asm due to potential symbol collision. Similarly, @objc(name) is discouraged. 
* Building from source only: aliasing distributed binaries is not possible due to the impact on mangling and serialization.
* Runtime: calls to convert String to types in module, i.e direct or indirect calls to NSClassFromString(...), will fail and should be avoided.
* For resources, only asset catalogs and localized strings are allowed.
* Higher chance of running into existing issues as follows:
    * Retroactive conformance (https://forums.swift.org/t/retroactive-conformances-vs-swift-in-the-os/14393): this is already not a recommended practice and should be avoided.   
    * Extension member “leaks”: this is considered a bug (https://bugs.swift.org/browse/SR-3908) but hasn’t been fixed yet. More discussions here (https://forums.swift.org/t/pre-pitch-import-access-control-a-modest-proposal/50087). 
* Code size increase will be more implicit and requires a caution, although module aliasing will be opt-in and a size threshold could be set to provide a warning. 

## Future Directions

* Currently when a module contains a type with the same name, fully qualifying a type in the module results in an error; it treats the left most qualifier as a type instead of the module (bug (https://bugs.swift.org/browse/SR-14195), forums1 (https://forums.swift.org/t/fixing-modules-that-contain-a-type-with-the-same-name/3025), forums2 (https://forums.swift.org/t/pitch-fully-qualified-name-syntax/28482)); `XCTest` is a good example as it contains a class called `XCTest`. Trying to access a top level function `XCTAssertEqual` via `XCTest.XCTAssertEqual(...)` results in “Type 'XCTest' has no member 'XCTAssertEqual'” error. Module aliasing could mitigate this issue by renaming `XCTest` as `XCTestFramework` without requiring source changes in the `XCTest` module and allowing the function access via `XCTestFramework.XCTAssertEqual(...)` in the user code.

* Introducing new import syntax such as `import Utils as GameUtils` has been discussed in forums to improve module disambiguation. The module aliasing infrastructure described in this proposal paves the way towards introducing such syntax that can allow more explicit (in source code) aliasing.  

* Visibility change to import decl access level (from public to internal) pitched here (https://forums.swift.org/t/pre-pitch-import-access-control-a-modest-proposal/50087) could help address extension leaks issues mentioned in *Requirements / Limitations* section. 

* Swift modules can have C target dependencies. Adding module aliasing support for such modules is possible but will require changing visibility to C symbols.

* C++ interop support could potentially allow C++ modules to be aliased besides pure Swift modules.  


## Source compatibility

This is an additive feature. Currently when there are duplicate module names, it does not compile at all. This feature requires explicitly opting into allow and use module aliaisng via package manifests or compiler invocation commands and does not require source code changes. 

## Effect on ABI stability
The feature in this proposal does not have impact on the ABI or runtime.

## Effect on API resilience
This proposal does not introduce features that would be part of a public API.

## Acknowledgments
This proposal was improved with helpful suggestions and feedback along with code reviews by Becca Royal-Gordon, Alexis Laferriere, Pavel Yaskevich, Joe Groff, Mike Ash, Adrian Prantl, Artem Chikin, Boris Buegling, Anders Bertelrud, Tom Doron, and Johannes Weiss, and others.  