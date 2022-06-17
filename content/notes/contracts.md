+++
title = "Implementing higher order contracts"
date = 2022-05-18
+++

In college I spent some time working on the internals of the [Racket contract system](https://docs.racket-lang.org/guide/contract-boundaries.html), and at the time I understood very little about it beyond reading the documentation.

Around that same time, I had started implementing [Steel](https://github.com/mattwparas/steel), which is an embedded scheme in Rust. Eventually, I was inspired to add contracts as a fundamental part of Steel, both as a quest to improve
the experience of writing Steel code, and also to gain a deeper understanding of contrats and how _blame_ is assigned when contract violations occur.

Before we begin, I'll briefly summarize contracts in Racket and how you use them (if you truly want a better understanding, I would read the Racket [guide](https://docs.racket-lang.org/guide/contract-boundaries.html)), before diving in to implementing them in Steel.

## Contract Basics

If you've ever used a language with a static type system, than you've already interacted with simple contracts before - they check the types of the arguments at compile time. In Racket, this would look something like this:

```racket
;; Without contracts
(define (addition x y)
    (+ x y))

;; With a contract
(define/contract (addition x y)
    ;; This is the function contract
    (-> integer? integer? integer?) 
    ;;  ^^^ x    ^^^ y    ^^^ return predicate
    (+ x y))
```

Each position in the contract is a corresponding predicate that will get checked when the function is called. In this case, `x` and `y` are required to be `integer?`s, and we're also asserting that the return value will satisfy the contract `integer?` as well.

This is trivial by inspection - we know that as long as the input arguments are integers, the result will be an integer since we're just adding them.

What happens if we make an error calling the function?

```racket
Welcome to Racket v8.4 [cs].
> (define/contract (addition x y)
      (-> integer? integer? integer?) 
      (+ x y))
> (addition 10.1 11)
addition: contract violation
  expected: integer?
  given: 10.1
  in: the 1st argument of
      (-> integer? integer? integer?)
  contract from: (function addition)
  blaming: top-level
   (assuming the contract is correct)
  at: string:1:18
 [,bt for context]
```

Perfect - the contract system blames the call site, since the caller broke the contract, they're blamed.
But now - what happens if there is an error in our implementation? Take this for example:

```racket
(define/contract (addition x y)
    (-> integer? integer? integer?) 
    (+ x y 0.1))
```

Here the return type will be coerced to a float, and as a result the contract itself is actually incorrect - the contract system will detect this at run time and report the error like so:

```racket
Welcome to Racket v8.4 [cs].
> (define/contract (addition x y)
      (-> integer? integer? integer?) 
      (+ x y 0.1))
> (addition 10 20)
addition: broke its own contract
  promised: integer?
  produced: 30.1
  in: the range of
      (-> integer? integer? integer?)
  contract from: (function addition)
  blaming: (function addition)
   (assuming the contract is correct)
  at: string:1:18
 [,bt for context]
```

This is great! We made a mistake in our implementation, and the contract system reported our error, and in fact, blamed _us_ for making a mistake.

## Introducing higher order contracts

In the previous example, we were using predicates that could immediately be checked when a function is called and when it returns. But what if you have functions as arguments? Take this for example:

```racket
(define/contract (custom-map func collection)
    (-> (-> integer? integer?) (listof integer?) (listof integer?))
    (map func collection))
```

Here we have a function contract as a precondition for another function contract - This binds `func` to be a function that accepts and returns values which satisfy the `integer?` predicate - and also we require that the collection we pass in is in fact a list of `integer?` - by extension, the resulting value is also a list of `integer?`.

What are the implications of this? Well, it means we can't actually check the contract on the function until its applied. This changes how blame is applied, and requires keeping a history of the contracts that are applied to a function:

```racket
;; Higher order contracts, check on application
(define/contract (higher-order func y)
    (->/c (->/c even? odd?) even? even?)
    (+ 1 (func y)))

(higher-order (lambda (x) (+ x 1)) 2) ;; => 4

(define/contract (higher-order-violation func y)
    (->/c (->/c even? odd?) even? even?)
    (+ 1 (func y)))

(higher-order-violation (lambda (x) (+ x 2)) 2) ;; contract violation
```

Contracts on functions do not get checked until they are applied, so a function returning a _contracted_ function won't cause a violation until that function is actually used:

```racket
;; More higher order contracts, get checked on application
(define/contract (output)
    (-> (-> string? int?))
    (lambda (x) 10))

(define/contract (accept func)
    (-> (-> string? int?) string?)
    "cool cool cool")

(accept (output)) ;; => "cool cool cool"

;; different contracts on the argument
(define/contract (accept-violation func)
    (-> (-> string? string?) string?)
    (func "applesauce")
    "cool cool cool")

(accept-violation (output)) ;; contract violation

;; generates a function
(define/contract (generate-closure)
    (-> (-> string? int?))
    (lambda (x) 10))

;; calls generate-closure which should result in a contract violation
(define/contract (accept-violation)
    (-> (-> string? string?))
    (generate-closure))

((accept-violation) "test") ;; contract violation
```

Perhaps a more nuanced case:

```racket
(define/contract (output)
    (-> (-> string? int?))
    (lambda (x) 10.2))

(define/contract (accept)
    (-> (-> string? number?))
    (output))


((accept) "test") ;; contract violation 10.2 satisfies number? but _not_ int?
```


## Details

For the sake of this post, we'll talk about two kinds of contracts: `Flat` and `Function` contracts.

`Flat` contracts are simply predicates on values with no children - meaning they're equivalent to `Atoms` in a scheme implementation.

`Function` contracts are contracts bound to a function - they have preconditions and postconditions, which are contracts themselves. This grammar would look something like this in Rust:

```rust
// Runtime representation of a function
struct Function {
    ...
}

enum Contract {
    Flat(Function)
    Function({
        pre_conditions: Vec<Contract>,
        post_condition: Box<Contract>
    })
}
```

And the equivalent construction in scheme using the `->` constructor for function contracts:

```racket
(-> predicate? predicate? predicate?)
    ^^^^^^^^^^^^^^^^^^^^  ^^^^
    pre conditions        post condition
```

At this point, we're setting up a tree, where internal nodes are `Contract::Function`'s and leaf nodes are `Contract::Flat`. If you take a close look its akin to a classic cons list, where you have either a list or an atom. The list is analagous to a function, and the atoms are flat contracts, contracts that are bound to a singular value, not a function.

So now we've decided that at runtime, we're going to represent these contracts as values. When you go to apply a contracted function, it is going to do a little bit more work than the usual function call, take this for example:

```racket
(define/contract (add2 x y)
    (-> even? odd? odd?)
    (+ x y))

(add2 10 20) ;; => 30

```

When you apply `add2`, the runtime needs to do a bit more checking than usual - its going to apply the contracts that you've given, so in this case it has to call `(even? x)` and `(odd? y)` and make sure that they hold true. Its akin to something like this:

```racket
(define (add2 x y)
    (unless (even? x) 
        (error! "Contract violation, x was required to be even"))
    (unless (even? y)
        (error! "Contract violation, y was required to be even"))
    
    (let ((result (+ x y)))
        (unless (odd? result)
            (error! "Contract violation, the add2 expected the result to be odd?"))
        result))
```



