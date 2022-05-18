+++
title = "Implementing higher order contracts"
date = 2020-01-22
+++

In college I spent some time working on the internals of the [Racket contract system](https://docs.racket-lang.org/guide/contract-boundaries.html), and at the time I understood very little about it beyond reading the documentation.

Around that same time, I had started implementing [Steel](https://github.com/mattwparas/steel), which is an embedded scheme in Rust. Eventually, I was inspired to add contracts as a fundamental part of Steel, both as a quest to improve
the experience of writing Steel code, and also to gain a deeper understanding of contrats and how _blame_ is assigned when contract violations occur.

Before we begin, I'll briefly summarize contracts in Racket and how you use them (if you truly want a better understanding, I would read the Racket [guide](https://docs.racket-lang.org/guide/contract-boundaries.html)), before implementing them in Steel.

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

