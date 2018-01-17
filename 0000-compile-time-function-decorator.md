# Compile Time Function Decorator

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Jeffrey Macko](https://github.com/mackoj), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)

## Introduction

> A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing.

This proposal aims at increasing code modularity by introducing some aspect-oriented programming (AOP) traits to the Swift language. AOP helps to unclutter code that otherwise tends to mix business logic with [cross-cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern).
Better separation could be achieved in Swift by injecting additional behavior (code to be executed before or after some functions) at compile time into the existing source code. This adds behaviors that are not central to the business logic of a program without cluttering the core code.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

> Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Swift
developers create better Swift code.

Traditional code tends to mix specific business code with cross-cutting concerns like profiling, instrumentation, logging, security etc., ending up in cluttered code that gets complicated to understand or refactor. Using AOP based injection, the code could better isolate those cross-cutting concerns from the business logic: the result is a better readable, maintainable code.  

Most of the time the cross-cutting concerns are adding performance or error logging, manage access control or integrating third-party SDKs and you really want to seek separation and independence from those third-party vendors calls.  

### Avantages

The main advantages of this feature are:

- a clear improvement for developer to focus on a single concern,
- a better readability and maintainability due to the separation of business logic code and cross-cutting concerns code,
- increased productivity because it's easier to work on code in one central place rather than scattered over the project,
- easy adding of functionality like instrumentation, profiling, logging, debugging, access controls etc without having to dive deep into the business code,
- a warranty of execution even if the augmented function has complicated flows with several exit points. 

Implementing this feature on compiler level has a big advantage over implementing it through a framework: The injection would work on any project without requiring to depend on another framework. The native implementation will also avoid additional compile steps


Use cases:

- add debugging information
- add profiling information 
- add an access control for increased security to several classes
- assert pre- and post conditions
- add analytic metrics 
- isolate calls to third party code in a central place

### Background

This feature is heavely inspired by [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) although I don't think this is Aspect Oriented Programming per se but it carries a lot of similarities.

This feature is build to work at compile time like Codable and Equatable and Hashable conformance([SE-0185](https://github.com/apple/swift-evolution/blob/master/proposals/0185-synthesize-equatable-hashable.md)).

[Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) is a big common problem in programming that Aspect Oriented Programming help to reduce.

### Other contributors already express the need for this

Other contributors has already express the need for this kind of feature on the mailing list [Function Decorator](http://thread.gmane.org/gmane.comp.lang.swift.evolution/16011) sadly this thread don't get enough traction. I propose a different approach to fix the same issue. More recently [Swift request list](http://article.gmane.org/gmane.comp.lang.swift.evolution/19676/) ask for Aspect Oriented Programming too.


[Sourcery](https://github.com/krzysztofzablocki/Sourcery)(a very popular metaprogramming toolkit for Swift) provide by default a template for doing it([Decorator.swifttemplate](https://github.com/krzysztofzablocki/Sourcery/blob/master/Templates/Templates/Decorator.swifttemplate)).

### Other Languages Does It

Aspect oriented programming is implemented in [many languages](https://en.wikipedia.org/wiki/Aspect-oriented_programming#Implementations).

Python decorator syntax https://www.python.org/dev/peps/pep-0318/ / https://wiki.python.org/moin/PythonDecorators

### Objective-C use of AOP

[Analytics](http://artsy.github.io/blog/2014/08/04/aspect-oriented-programming-and-aranalytics/) is a kind of cross-cutting concerns because an analytics strategy necessarily affects every part of the system. Analytics thereby crosscuts all classes and methods.

- http://artsy.github.io/blog/2014/08/04/aspect-oriented-programming-and-aranalytics/
- http://petersteinberger.com/blog/2014/hacking-with-aspects/


## Proposed solution

> Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

Have a way to instruct the compiler how to inject code into an already existing function.

In general, we propose that a function can have added code to it from a injector. We describe the specific conditions under which a function is augmented below, followed by the details of how the injector is implemented.

Introducing a new expression `@decorator` which inject code into a function. It can take parameter **before**, **after** and **wrap**.

before | after | wrap
--- | --- | ---
injected code in the beggining of the target function | add a `defer` at the beginning of the function with injected code in it | replace the original function with a new one that will be called instead and call the original function by itself the return value of the original function can be use in this mode

Adding a mechanism that allow us to insert code before/after and around a function will fix this issue. 

The decorator function:

- can be anywhere
- can operate on any Swift function in a Class, Struct or Protocol
- can only work on code that we have the source code of so it does not work on compiled library or frameworks
- cannot work for getter et setter on protocol
- we can inject **before**,**after** and **wrap** at the same time
- cannot work from a library

The decorator feature need to work:

- need a way to operate (before|after|wrap)
- need a way to hook onto another function that exist
- need a priority of execution (optional by default the highest if more than two then propose a priority)
- need `[unowned self]` in order to do stuff
- need function original parameters `function-expression-parameters`
- need the final function to call in wrap mode

New Compiler error/warning :

- raise error when there is a conflict with the `priority`s and show where it has already been used
- If the `functionSelector` does not found a function to inject into the compiler should emmited a warning
- will emmit a error if used in `wrap` mode and the function pass as parameter is not used

The compiler will do most of the work so we will never have to see the generated version of the code unless we go into the pre-process representation in Xcode.


## Detailed design

> Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

Trouver un moyen de proposer plusieurs syntax.

### Syntax 1

Avantage vs Syntax 2 :

- La priorité est gérée par l'ordre d'appel
- Simple
- On voit on ou on injecte


```swift

@decorator(before|after|wrap, Class.AllFunction|Struct.AllFunction|Protocol.AllFunctionImplementation) 

/// Example

func decoratorBefore(x : Int) {
	print("before")
}

func decoratorAfter(x : Int) {
	print("after")
}

func decoratorWrap(x : Int, f : (x : Int) -> Bool) -> Bool {
	print("wrap-b")
	let val = f(x)
	print("wrap-a")
	return val
}

@decorator(before, decoratorBefore) 
@decorator(after, decoratorBefore) 
@decorator(wrap, decoratorBefore) 
func functionToDecorate(x : Int) -> Bool {
	print("toto")
	return false
}

```

### Syntax 2

Avantage vs Syntax 1 :

- laisse intoucher le code ou on inject
- est beaucoup plus generic

```swift
@decorator(before|after|wrap) where <Class.AllFunction|Struct.AllFunction|Protocol.AllFunctionImplementation> priority 1-255 { [unowned self], <function-expression-parameters>, originalFunction : (<function-expression-parameters>)->()) in
	<# Your Code #>
}

/// Example

@decorator(before) where <A.B> priority 1 { [unowned self], x : Int) in
	print("before")
}

@decorator(after) where <A.B> priority 1 { [unowned self], x : Int) in
	print("after")
}

@decorator(wrap) where <A.B> priority 1 { [unowned self], x : Int, f : (x : Int) -> Bool) in
	print("wrap-b")
	let val = f(x)
	print("wrap-a")
	return val
}

class A {
	func B(x : Int) -> Bool {
		print("toto")
		return false
	}
}
```


## Source compatibility

> Relative to the Swift 3 evolution process, the source compatibility
requirements for Swift 4 are *much* more stringent: we should only
break source compatibility if the Swift 3 constructs were actively
harmful in some way, the volume of affected Swift 3 code is relatively
small, and we can provide source compatibility (in Swift 3
compatibility mode) and migration.

> Will existing correct Swift 3 or Swift 4 applications stop compiling
due to this change? Will applications still compile but produce
different behavior than they used to? If "yes" to either of these, is
it possible for the Swift 4 compiler to accept the old syntax in its
Swift 3 compatibility mode? Is it possible to automatically migrate
from the old syntax to the new syntax? Can Swift applications be
written in a common subset that works both with Swift 3 and Swift 4 to
aid in migration?


This is an additive proposal, existing code will continue to work.

## Effect on ABI stability

> Does the proposal change the ABI of existing language features? The
ABI comprises all aspects of the code generation model and interaction
with the Swift runtime, including such things as calling conventions,
the layout of data types, and the behavior of dynamic features in the
language (reflection, dynamic dispatch, dynamic casting via `as?`,
etc.). Purely syntactic changes rarely change existing ABI. Additive
features may extend the ABI but, unless they extend some fundamental
runtime behavior (such as the aforementioned dynamic features), they
won't change the existing ABI.

> Features that don't change the existing ABI are considered out of
scope for [Swift 4 stage 1](README.md). However, additive features
that would reshape the standard library in a way that changes its ABI,
such as [where clauses for associated
types](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md),
can be in scope. If this proposal could be used to improve the
standard library in ways that would affect its ABI, describe them
here.


This feature is purely additive and does not change ABI.

## Effect on API resilience

> API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.


N/A.

## Alternatives considered

> Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

### Using `precedence` name instead of `priority`

`priority` seems to be better understood by non-english people. 

### Using `#selector` name instead of `#decorator`

Using `#selector` was my first idea and it was a bad one because this does not do the same as the `#selector` and can be very confusing on what it try to achieve.

### Not having a priority

The priority help to order the injection of those code blocks.

### Adding an @notdecorable

This keyword can be used like @objc but this keyword prevent other to decorate your code.

If someone try to decorate `importantWork()` he will receive a compiler error.

Bar.swift

	class bar {
		@notdecorable func work() {
			print("bar is working...")
		}
	}

### Adding an @decorable

This keyword can be used like @objc but this keyword is necessary to allow other to decorate your code.

Decoration only work on function with this decorator

Bar.swift

	class bar {
		@decorable func work() {
			print("bar is working...")
		}
	}

## Criticism
 
It's biggest advantage is it biggest weakness because with this feature it's harder to understand the flow of execution. Since all the injection is happening at compile time. Since it's a compile time feature (a bit like macro) it does not need to be declared within a function or a class it could increase a bit the compile time.
 `

