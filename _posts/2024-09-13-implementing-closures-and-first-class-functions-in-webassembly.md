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
// Note: In Theta, the function signatures are represented as Function<ArgType, ReturnType>. This is
// specific to Theta's type system and might differ from the typical notation used in other
// functional languages. For instance, Function<Number, Number> means a function that takes a
// Number and returns a Number.

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

In this example, we're using `i64` for simplicity and flexibility. WebAssembly supports both `i32` and `i64`, and depending
on your use case, you might choose one over the other. Since Theta supports larger integer types, we're using `i64` here, but in
practice, you could adjust this to `i32` if your application deals with smaller numbers

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

During lambda lifting, all variables in the outer scope that the inner function relies on, like `scalar` in this case, are pulled up and passed explicitly
to the generated top-level function. This ensures that all state needed by the generated function is available, even though the original context is no longer present.

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

We can use a sort of partial function application approach to call the lambda-lifted function. Partial application is a functional programming concept where some arguments of
a function are fixed, and a new function is returned that takes the remaining arguments. In our case, we store the applied arguments in memory and track how many more are
needed before the function can be fully executed. This allows us to simulate partial application behavior in WebAssembly.

As its parent function gets called and arguments get passed in, we know we will need those parameters, in the same order, to call our lifted function. What we can do is have
the parent function "partially apply" arguments to the lifted function during its execution. There are a few things we want to store in memory so that we can correctly
implement the partial application flow:

- The table index of the function that this partial application is for.
- The remaining arity (number of arguments the function still requires) of the function.
- Each argument that has been applied to the function so far.

What we can do is provision a custom data structure, which we'll call a "closure", to store the information we need. This approach will need some sort of garbage collection.
Although we won’t touch on garbage collection in this article, it's important to note that WebAssembly lacks built-in garbage collection (outside of proposed extensions). 
This means any dynamically allocated data structures, such as closures, will require manual memory management using techniques like reference counting or manual
deallocation to avoid memory leaks.

Our closure would be laid out in memory like so:

<img src="/images/webassembly-closures/closure_memory_layout.svg" style="width: 75%;" />

We'll store the closure as contiguous chunk of bytes into memory. Here we'll store an `i32` for the function pointer, which will use 4 bytes. In the next available
memory address we'll store the remaining arity of the function -- this is how many arguments we still need to be passed in before the function can be executed. We'll
use an `i32` for this as well, so that'll take another 4 bytes. 

The next section of the closure is going to be all of the addresses of the arguments we want to store. Each argument address will be an `i32` as well, so they'll each take
up 4 bytes.

Now, instead of returning the function pointer, we can return a pointer to the closure.

Here's how it works:

When we come across a function invocation of a function that has been lambda-lifted, we provision a closure in memory for it. The function index will just be the index of the
lifted function in the function table. The arity will start at the number of parameters that function expects. Immediately, we'll add the parameters of the parent function
into the closure. We need to store the argument somewhere in memory, and then store an `i32` pointing to that memory address into the closure. 

In order to save some space, we can insert the arguments backwards in the closure, meaning the first argument address would get stored in the last argument memory address position
in the closure. We can easily calculate which position an argument should go into, by doing using the formula: `closure_addr + 8 + (rem_arity * 4)`. Whenever we add an argument
address to the closure, we decrement the remaining arity by one. Once remaining arity hits 0, we know we can execute the function!

## Putting it all together

Let's walk through what we just described within the context of the example from above.

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

gets translated into:

```clojure
(module
 (type $0 (func (param i64 i64) (result i64)))
 (type $1 (func (param i64) (result i32)))
 (type $2 (func (param i32 i32)))
 (type $3 (func (result i64)))
 (memory $0 1 10)
 (table $ThetaFunctionRefs 3 3 funcref)
 (elem $0 (i32.const 0) $main $createMultiplier $e53488d)
 (func $Theta.Function.populateClosure (param $closure_mem_addr i32) (param $param_addr i32)
  (local $arity i32)
  (local.set $arity
   (i32.load
    (i32.add
     (local.get $closure_mem_addr)
     (i32.const 4)
    )
   )
  )
  (i32.store offset=4
   (local.get $closure_mem_addr)
   (i32.sub
    (local.get $arity)
    (i32.const 1)
   )
  )
  (i32.store offset=4
   (i32.add
    (local.get $closure_mem_addr)
    (i32.mul
     (local.get $arity)
     (i32.const 4)
    )
   )
   (local.get $param_addr)
  )
 )
 (func $main (result i64)
  (local $multiplyBy10 i32)
  (local.set $multiplyBy10
   (call_indirect (type $1)
    (i64.const 1)
    (i32.const 1)
   )
  )
  (i64.store
   (i32.const 0)
   (i64.const 5)
  )
  (call $Theta.Function.populateClosure
   (local.get $multiplyBy10)
   (i32.const 0)
  )
  (if (result i64)
   (i32.eqz
    (i32.load offset=4
     (local.get $multiplyBy10)
    )
   )
   (then
    (call_indirect (type $0)
     (i64.load
      (i32.load offset=12
       (local.get $multiplyBy10)
      )
     )
     (i64.load
      (i32.load offset=8
       (local.get $multiplyBy10)
      )
     )
     (i32.load
      (local.get $multiplyBy10)
     )
    )
   )
   (else
    (i64.const -1)
   )
  )
 )
 (func $e53488d (param $factor i64) (param $x i64) (result i64)
  (i64.mul
   (i64.mul
    (local.get $x)
    (local.get $factor)
   )
   (i64.const 10)
  )
 )
 (func $createMultiplier (param $factor i64) (result i32)
  (local $scalar i32)
  (i64.store
   (i32.const 8)
   (local.get $factor)
  )
  (i32.store
   (i32.const 16)
   (i32.const 2)
  )
  (i32.store offset=4
   (i32.const 16)
   (i32.const 1)
  )
  (i32.store offset=12
   (i32.const 16)
   (i32.const 8)
  )
  (i32.const 16)
 )
)
```

It looks like a lot, so lets step through it. When we call `$main`, we store the result of `call_indirect` into
a local called `$multiplyBy10`:

```clojure
(local.set $multiplyBy10
 (call_indirect (type $1)
  (i64.const 1)
  (i32.const 1)
 )
)
```

The call_indirect instruction takes in any arguments that the function needs, and then calls the function at the specified
function index. In this case, we're passing in 1 as the argument, and calling the function at index 1 in the function table.
That function is our `$createMultiplier` function. Lets jump there now.

```clojure
 (func $createMultiplier (param $factor i64) (result i32)
  (local $scalar i32)
  (i64.store
   (i32.const 8)
   (local.get $factor)
  )
  (i32.store
   (i32.const 16)
   (i32.const 2)
  )
  (i32.store offset=4
   (i32.const 16)
   (i32.const 1)
  )
  (i32.store offset=12
   (i32.const 16)
   (i32.const 8)
  )
  (i32.const 16)
 )
```

When this function gets invoked, it immediately stores the passed `$factor` into memory -- in this case at address 8. Next,
it creates a closure, where we see the `i32.store` calls. The closure is being stored at address 16 in memory.

First, it stores a 2 at address 16. This is the function index of the function we want to call (`$e53488d`). Then, it stores the 
remaining arity as 1, also at address 16, but with an offset of 4, so this will insert into memory address 20. This is a bit 
of a shortcut, because technically `$e53488d` has an arity of 2, but the compiler knows that we're about to store an argument,
so rather than storing an arity of 2 and then immediately decrementing it to 1, we can just store 1 right away.

As expected, we next store the address of the argument that we want to pass into the closure. Notice that this is being stored
at address 16, with an offset of 12, which gives us the effective address of 28. Also notice that we skipped address 24. We're
leaving this empty for the second parameter.

The `$createMultiplier` function then just returns 16, which is the memory address at which the closure was stored. Now let's go 
back to where we called the function in the first place.

```clojure
 (func $main (result i64)
  (local $multiplyBy10 i32)
  (local.set $multiplyBy10
   (call_indirect (type $1)
    (i64.const 1)
    (i32.const 1)
   )
  ) ;; <- We are here
  (i64.store
   (i32.const 0)
   (i64.const 5)
  )
  (call $Theta.Function.populateClosure
   (local.get $multiplyBy10)
   (i32.const 0)
  )
  (if (result i64)
   (i32.eqz
    (i32.load offset=4
     (local.get $multiplyBy10)
    )
   )
   (then
    (call_indirect (type $0)
     (i64.load
      (i32.load offset=12
       (local.get $multiplyBy10)
      )
     )
     (i64.load
      (i32.load offset=8
       (local.get $multiplyBy10)
      )
     )
     (i32.load
      (local.get $multiplyBy10)
     )
    )
   )
   (else
    (i64.const -1)
   )
  )
 )
```

Okay, now the closure pointer is stored into the local `$multiplyBy10`. The next thing we do is store the 5 into memory (remember, 
our source code immediately calls the returned function with an argument of 5). Now that our argument is in memory, we can proceed
to place its address into the closure as well. I've put together a `$populateClosure` helper function, which just takes in a
`closure_mem_addr` and a `param_addr`, and inserts the `param_addr` at the appropriate place in the closure, and also decrements
the arity. We don't want to have to rewrite that logic every time.

Next, we check if the arity of our closure equals zero:

```clojure
  (if (result i64)
   (i32.eqz
    (i32.load offset=4 ;; Offset 4 is where the arity is in the closure
     (local.get $multiplyBy10)
    )
   )
```

if it does, we `call_indirect` the function at the closure pointer address:

```clojure
   (then
    (call_indirect (type $0) 
     (i64.load
      (i32.load offset=12 ;; Load the first argument. This will give us the address of where the argument was stored
       (local.get $multiplyBy10)
      )
     )
     (i64.load
      (i32.load offset=8
       (local.get $multiplyBy10)
      )
     )
     (i32.load
      (local.get $multiplyBy10)
     )
    )
   )
```

Here, we use `i32.load` to load the argument address from the closure. Remember that we stored the arguments backwards,
so the last argument in the closure is really the first argument to the parameter. Once we load the address, we need
to read the actual number from that address using `i64.load`, which will give us 1 in the case of the first argument.

Finally, we use `local.get $multiplyBy10` to get the closure pointer, and then read at that address without an offset,
to get the function index for that closure. That gets passed into the `call_indirect` as well.

This will call our function with the intended arguments, and now we have first-class functions and closures implemented
in WebAssembly!
