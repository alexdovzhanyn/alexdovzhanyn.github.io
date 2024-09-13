---
layout: post
title: Implementing closures and first-class functions in WebAssembly
categories: [WebAssembly,Compilers,Theta]
---

While building [Theta](https://github.com/alexdovzhanyn/ThetaLang), a functional programming language that compiles to WebAssembly,
I came across an interesting problem. As a functional language, first-class functions and closures are essential to Theta. The
ability to pass a function as an argument to another function enables the usage of such abstractions as `map`, `filter`, and `reduce`,
which are fundamental to functional programming. 

WebAssembly, being a low-level compilation target, is minimally-typed and does not
natively support high-level features such as closures and first class functions natively (well, not without the
[GC Proposal](https://github.com/WebAssembly/gc), at least). This is where our problem starts to reveal itself.

Let's take a look at an example of a simple Theta code snippet:

```typescript
main<Function<Number>> = () -> {
    multiplyBy10(5)
}

multiplyBy10<Function<Number, Number>> = (x<Number>) -> x * 10
```

We define a `main` function which takes in no arguments, and returns a `Number`. This function just calls a `multiplyBy10` function
and implicitly returns its result. The `multiplyBy10` function accepts a number as its argument and returns a number, as we
can tell from its type signature `<Function<Number, Number>>` (the last value in the type signature of a function is
it's return type).

This is pretty simple to convert to WebAssembly directly:

```clojure
(module 
    (func $main (result i64)
        (call $multiplyBy10
            (i64.const 5)
        )
    )
    (func $multiply (param $x i64) (result i64)
        (i64.mul
            (local.get $x)
            (i64.const 10)
        )
    )
)
```

Since these two functions are each independent functions which share no state between them, we don't need to do anything fancy
to get the function call to `multiplyBy10` working -- simply using the `call` instruction works. Now, what if we want to have many
multiplyByX functions? We wouldn't want to duplicate the logic of `multiplyBy10` for each different number we want to multiply with,
right? What we can do instead is make a function-generator, which is a higher-order function that generates a function for us.

We can do this with the following Theta code:

```typescript
main<Function<Number>> = () -> {
  multiplyBy10<Function<Number, Number>> = createMultiplier(1)

  multiplyBy10(5)
}

createMultiplier<Function<Number, Function<Number, Number>>> = (factor<Number>) -> {
    scalar<Number> = 10

    // The last expression of a block is implicitly returned, so no need
    // for a return statement
    (x<Number>) -> x * factor * scalar
}
```

Just like before, we have a `main` function which doesn't accept any arguments, and just returns a `Number`. Now, though, we have a
function generator called `createMultiplier` whose job it is to generate a multiplier function for us. It takes a `Number` as an 
argument, and generates a function dynamically as its return value. The generated function takes in another `Number` as its 
argument, and it just multiplies the number that was passed in (`x`) to the number that the function generator
accepted (`factor`). To make things a bit more interesting we also threw in a `scalar`, which will help illustrate scope capturing
requirements. The scalar is just a constant value in this case, which scales the `factor` that was passed in. So if we wanted to
multiply by 10, we would just need to pass in `1` as an argument, and it would become `10` before it gets multiplied with `x`.

The above code is functionally equivalent with the code we had in the first example -- the only difference is we're now free to
generate whatever multiplier functions we want. 

How would we compile this code to WebAssembly? We're posed with a few problems:

1. In WebAssembly, all functions must be declared at the module level, they can't be defined within other functions.
2. What should the result type of `createMultiplier` be? WebAssembly only supports `i32`, `f32`, `i64`, and `f64`, but `createMultiplier` returns a function.
3. Even if we could return some sort of function type, we still don't have a way of remembering the value that was passed into `createMultiplier` to begin with, since WebAssembly is stack-based -- once that value is consumed from the stack we can't use it again.

First, we need some way to define the generated multiplier function at the top level of the WebAssembly module. We can do this by employing a technique called [lambda lifting](https://en.wikipedia.org/wiki/Lambda_lifting) to restructure our code. During the compilation
process, we can transform the above code into something like this:

```typescript
main<Function<Number>> = () -> {
    multiplyBy10<Function<Number, Number>> = createMultiplier(1)

    multiplyBy10(5)
}

createMultiplier<Function<Number, Function<Number, Number>>> = (factor<Number>) -> {
    scalar<Number> = 10
    
    // What happens here?
}

anonymous<Function<Number, Number, Number>> = (factor<Number>, x<Number>) -> {
    scalar<Number> = 10

    x * scalar * factor
}
```

Obviously, this still won't work in the context of WebAssembly, but we're closer to what we need. Instead of having an anonymous function defined within our function,
we captured all of the elements from the scope of the parent function that we need, and made a new, globally-defined function in the module called `anonymous`. Notice that
`anonymous` accepts two arguments now -- a `factor` and `x`. Also notice that it has its own copy of the `scalar` constant. We still need to figure out how to return a
function from `createMultiplier`, since it has a return type of `Function<Number, Number>`.
