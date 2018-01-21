# Compile Time Function Decorator

* Authors: [Jeffrey Macko](https://github.com/mackoj)

This document is a work in progress. There is no implementation at this time.

## Introduction

This proposal aims at increasing code modularity by introducing some aspect-oriented programming (AOP) traits to the Swift language. AOP helps to unclutter code that otherwise tends to mix business logic with [cross-cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern).
Better separation could be achieved in Swift by injecting additional behavior (code to be executed before or after some functions) at compile time into the existing source code. Thus adds code that is not central to the business logic of a program without cluttering the core code.

The main difference with AOP is that AOP is dynamic and this is static at compile time.

## Motivation

Traditional code tends to mix specific business code with cross-cutting concerns like profiling, instrumentation, logging, security, etc., ending up in cluttered code that gets complicated to understand or refactor. Using AOP based injection, the code could better isolate those cross-cutting concerns from the business logic: the result is a better readable, maintainable code.  

Most of the time the cross-cutting concerns are adding performance, error logging, managing access control or integrating third-party SDKs and you want to seek separation and independence from this code.  

### Avantages

The main advantages of this feature are:

- a definite improvement for the developer is to improve focus on a single concern
- readability and maintainability due to the separation of business logic code and cross-cutting concerns code
- increased productivity because it's easier to work on code in one central place rather than scattered over the project
- easy adding of functionality like instrumentation, profiling, logging, debugging, access controls, etc. without having to dive deep into the business code
- a warranty of execution even if the augmented function has complicated flows with several exit points

Use cases:

- add debugging information
- add profiling information 
- add access control for increased security to several classes
- assert pre- and post-conditions
- add analytic metrics 
- isolate calls to third-party code in a central place
- add generic behaviour to code without modifing it like memoization

### Background

This feature is heavily inspired by [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) although I don't think this is Aspect Oriented Programming per se but it carries a lot of similarities.

This feature is built to work at compile time like Codable and Equatable and Hashable conformance([SE-0185](https://github.com/apple/swift-evolution/blob/master/proposals/0185-synthesize-equatable-hashable.md)).

[Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) is a big common problem in programming that Aspect Oriented Programming helps to reduce.

### Other contributors already express the need for this

Other contributors have already expressed the need for this kind of feature on the mailing list [Function Decorator](http://thread.gmane.org/gmane.comp.lang.swift.evolution/16011) sadly this thread doesn't get enough traction. I propose a different approach to fix the same issue. More recently [Swift request list](http://article.gmane.org/gmane.comp.lang.swift.evolution/19676/) ask for Aspect Oriented Programming too.


[Sourcery](https://github.com/krzysztofzablocki/Sourcery)(a very popular metaprogramming toolkit for Swift) provide by default a template for doing it([Decorator.swifttemplate](https://github.com/krzysztofzablocki/Sourcery/blob/master/Templates/Templates/Decorator.swifttemplate)).

### Other Languages Does It

Aspect-oriented programming is implemented in [many languages](https://en.wikipedia.org/wiki/Aspect-oriented_programming#Implementations).

Python decorator syntax https://www.python.org/dev/peps/pep-0318/ / https://wiki.python.org/moin/PythonDecorators

### Objective-C use of AOP

[Analytics](http://artsy.github.io/blog/2014/08/04/aspect-oriented-programming-and-aranalytics/) is a kind of cross-cutting concerns because an analytics strategy necessarily affects every part of the system. Analytics thereby crosscuts all classes and methods.

- http://artsy.github.io/blog/2014/08/04/aspect-oriented-programming-and-aranalytics/
- http://petersteinberger.com/blog/2014/hacking-with-aspects/


## Proposed solution

> Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

Teach the compiler how to insert code before/after and around a already existing function.

We propose that a function can have added code to it from an injector. We describe the specific conditions under which a function is augmented below, followed by the details of how the injector is implemented.

Introducing a new expression `@decorator` which inject code into a function. It can take parameter **before**, **after** and **wrap**.

### Expected behaviour

**before** | **after** | **wrap**
--- | --- | ---
inject code at the beginning of the target function | after the **before** using `defer` with injected code | be able to manipulate the input and the output of a target function by wrapping it in another function it will replace the original function with a new one that will be called instead and call the original function


The decorator function:

- can be anywhere
- can operate on any Swift function in a Class, Struct or Protocol implementation
- can only work on code that we have the source code of so it does not work on compiled library or frameworks
- cannot work for getter et setter on protocol
- we can inject **before**,**after** and **wrap** at the same time
- cannot work from a library

The decorator feature needs to work:

- need a way to operate (before|after|wrap)
- need a way to hook onto another function that exists
- need a priority of execution (optional by default the highest if more than two then propose a priority)
- need `[unowned self]` to do stuff
- need function original parameters `function-expression-parameters`
- need the final function to call in wrap mode

New Compiler error/warning :

- raise an error when there is a conflict between the `priority`s and show where it has already been used
- If the `function selector` does not found a function to inject into the compiler should emit a warning
- will emit an error if used in `wrap` mode and the function pass as the parameter is not used

The compiler will do most of the work so we will never have to see the generated version of the code unless we go into the pre-process representation in Xcode.


## Detailed design

I am not set on the syntax yet. It's pseudo code.

### Syntax 1 - More 🐍  like

Pros againts syntax 2 :

- the priority of call is handle by the call order
- simple
- we can see where we inject code

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

### Syntax 2 - More macro-ish like

Pros againts syntax 1 :

- we don't need to modify the code were we do the injection
- it's way more generic

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

This is an additive proposal, existing code will continue to work.

## Effect on ABI stability

This feature is purely additive and does not change ABI.

## Effect on API resilience

N/A.

## Alternatives considered

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
