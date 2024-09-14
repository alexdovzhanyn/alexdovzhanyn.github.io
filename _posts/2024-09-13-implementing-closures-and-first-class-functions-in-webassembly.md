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

## Lambda lifting
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

e53488d<Function<Number, Number, Number>> = (factor<Number>, x<Number>) -> {
    scalar<Number> = 10

    x * scalar * factor
}
```

Obviously, this still won't work in the context of WebAssembly, but we're closer to what we need. Instead of having an anonymous function defined within our function,
we captured all of the elements from the scope of the parent function that we need, and made a new, globally-defined function in the module called `e53488d`. Realistically,
we could have called this function whatever we want, so in this case we just used a random hash. 

In practice, in Theta, we generate a hash of the function contents and use that
as the name -- that way the function names won't ever collide, because if the names are the same, that means the content must be the same, so it must be the same function. 

Notice that `e53488d` accepts two arguments now -- a `factor` and `x`. It captured the parameters that it's parent function took in, and now needs those parameters as well. Also 
notice that it has its own copy of the `scalar` constant. We still need to figure out how to return a function from `createMultiplier`, since it has a return type 
of `Function<Number, Number>`.

## Function references
Now that we've lifted our function out of the scope of the function generator, we still need a way for the function generator to return a reference to the function -- after all,
`createMultiplier` does return a function. WebAssembly has a feature called [reference tables](https://webassembly.github.io/reference-types/core/syntax/modules.html#syntax-table),
which is basically a indexed list of homogenous opaque references (references of the same type). 

What this allows us to do, is to place a reference to a certain function into that table at a certain index. We can keep track of which function is stored at what index -- 
which is effectively the same as having a pointer to the function! The reason this is useful to us is because WebAssembly also has a 
[call_indirect](https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/Control_flow/call) instruction, which allows us to call a function from a table using its index. 
Do you see where we're going with this?

Let's add a table to our WebAssembly module. We'll have the type of the table be a `funcref`, so it'd be defined like so:

```clojure
(table $ThetaFunctionRefs 1 1 funcref)
```
Now we have an empty table that looks something like this:

<img src="/images/webassembly-closures/function_table_empty.svg" style="width: 50%;" />

Whenever we compile a function into WebAssembly, we can make sure to also add it to our function table. So our example above would fill out the function table like so:

<img src="/images/webassembly-closures/function_table_half.svg" style="width: 50%;" />

At this point in the compilation process we don't yet have a function table entry for the createMultiplier function because we haven't finished generating it yet, we've only
generated the child function.

Now, we can return the index of the lambda-lifted function as an `i32`, since the indices of WASM tables are `i32`s. Our final generated webassembly (ignoring the main
function, for now) module becomes:

```clojure
(module
 (table $ThetaFunctionRefs 2 2 funcref) ;; Provision a function table with size 2
 (elem $0 (i32.const 0) $e53488d $createMultiplier) ;; Add our 2 functions

 (func $e53488d (param $factor i64) (param $x i64) (result i64)
  (local $scalar i64) ;; scalar<Number> = 10
  (local.set $scalar
    (i64.const 10)
  )
  (i64.mul ;; x * factor * scalar
   (i64.mul
    (local.get $x)
    (local.get $factor)
   )
   (i64.const 10)
  )
 )

 (func $createMultiplier (param $factor i64) (result i32)
  (local $scalar i64) ;; scalar<Number> = 10
  (local.set $scalar
    (i64.const 10)
  )

  i32.const 1 ;; Returns the index of function $e53488d
 )
)
```

Notice that the return type of `createMultiplier` is now an `i32`. This is because it's now going to be returning a _pointer_ to a function. We're well on our way,
but we're still missing some things. Namely, the createMultiplier function doesn't actually do anything with the `$factor` that gets passed in -- it kind of just
ignores it and returns a refernce to a whole other function. 

## Closures
In order to tie these two functions together we need some way of storing the `$factor` that was passed in, for use later in our `$e53488d` function. We can't use
the stack to do this, because we don't know _when_ that value will be used (at least, not while we're compiling the `$createMultiplier` function).

We do have somewhere else we can put that value, though. WebAssembly also features a flat [memory](https://webassembly.github.io/spec/core/syntax/modules.html#memories),
which is a list of raw bytes. We can put anything we want in here. In order to use memory, we have to define it in our module:

```clojure
(memory $0 1 10)
```

Memory in WebAssembly is defined by giving it a name (`$0` in this case), followed by an _initial page size_, and then a _max page size_. A page of memory is 64kb in WASM,
which is more than enough for our purposes here. We set 1 page as the initial size, and arbitrarily set 10 as the maximum size (you'd probably want to set this to something
more suitable for your needs). Now we'll be able to add any data we want into memory, and then retrieve it when we need it.

The thing is, we can't just throw our `$factor` into some random memory address and then load it later. Well, we kind of can, and will, but it's a bit more complicated
than that. When we call `$createMultiplier`, we want it to store the `$factor` somewhere in memory so that when `$e53488d` is called, we have access to it. `$e53488d` takes
`$factor` as one of its arguments, so we need to have that loaded onto the stack before we use the `$call_indirect` instruction to execute it.

We need some way of tracking which arguments have already been passed in, and how many we are still expecting to be passed in, before we can go ahead and execute the function
-- so that we can place everything onto the stack before the `$call_indirect`.
