---
math: true
permalink: a-tour-of-haskell
title: A Tour of Haskell
description: An overview of Haskell from values to monads for beginners.
featured_image: /blog/images/trig_fractal/first_look.png
layout: page
date: 2024-11-10
---

# Motivation

I'm going to attempt to write the tutorial I wish I could have read when I was learning Haskell earnestly. In college, I was excited about functional programming but always stopped short of fully diving into Haskell. The type system seemed counter-intuitive and errors were hard to debug, but I thoroughly enjoyed the expressiveness of this language and the patterns that I did learn became staples of my everyday coding style. These days, Haskell is most of what I program in professionally and I finally feel comfortable sharing a post on how to learn the basics.

Before we start, it's good to get an idea of _why_ we would use a language like Haskell. Most are used to an imperative style with FP sprinkled in wherever we need declarative code, at least in the case of JS/TS, Python, and even C++ these days. If you're excited about Rust, then chances are you may not need this initial motivation. Much of Rust is inspired from Haskell, and the overlap in Rust/Haskell communities is significant. One might even say that Rust evangelism is highly comorbid with Haskell enthusiasm.

## Entertain a Hypothetical

If you have no experience with a strongly typed purely functional language like Haskell, take a moment here and try to gather up all of the programming knowledge you have learned up until this point in your life and set it down for a moment. The imperative and pure functional styles are difficult to map onto each other. It is possible, as you will see by the end of this post, but attempting to do so as a new Haskell learner can lead to false summits and counter-productive analogies.

Let's continue on and, for this post, pretend that we are showing up to our CS1 course with an open mind and an enthusiasm for learning.

## Philosophy

We want to build Haskell from scratch, at least the syntax and ergonomics. We won't need to worry about how the compiler works, or many other details under-the-hood. First off, what do we care about in a language? When we write programs, we are trying to create instructions with minimal effort that represent the steps needed to produce a desired result given arbitrary input. At the end of the day, all languages do this, but as our programs develop we run into unforseen spaghetti. Each language also balances expressiveness and dynamic syntax with coding ourselves into a corner that we must type our way out of.

The core of any non-esoteric language philosophy lie some simple facts about programming:

1. We all have finite energy
2. That energy is valuable, both in money and our own time
3. We have tools to provably prevent classes of errors

From (1) and (2) we must use (3) so that (1) can be used as much on unsolved problems and not solved problems. In addition, we should minimize the syntax needed to express the problem we are solving. Different languages take different approaches here, and it would be incorrect to assert any one language has solved the problem (for typed languages specifically, check out [the expression problem](https://en.wikipedia.org/wiki/Expression_problem)). The most common source of errors in a dynamic language are caused by evolving assumptions about state, and how it gets used/transformed. Any changes in how we label or treat our program state result in changes that ripple through our program. Without types, the responsibility of remembering where all of the state gets used falls on the programmer.

Types offer a unique advantage for preventing whole classes of errors. If we can rely on a compiler to check our assumptions and lead us to where errors exist in our code, then we can instead focus our energy on other aspects of the problems we are solving. If at all possible, we should try to surface errors at compile time. Having errors surface at runtime usually means that our programs blow up in our own faces at best, and in our users' faces at worst.

## Sources of Bugs and Complexity

All programs have an amount of [essential complexity](https://en.wikipedia.org/wiki/Essential_complexity) that we cannot program away. Like a wrinkle in fabric, we can move it around but it will never truly be absent from the topology of our program. It should then never be our goal to eliminate complexity entirely, but instead to minimize the amount of syntax required to express that complexity in a way that is readable. At the end of the day, our code suffers most from how difficult it is to understand (not things like time or space complexity, although those are important as well). A program is a living document, and how it evolves over time depends on the path of least resistance starting from the interpretation yielded by whoever is reading the code. Start with the wrong interpretation, and enable a path of least resistance to more complexity, and you will eventually get spaghetti.

### State and Side-Effects

Side-effects happen whenever we change values in a program. When reasoning about how we change values with code it is very intuitive to think about picking up a value, changing it, and setting it back down. This is the imperative style, and it is often too powerful for its own good. With all the freedom of changing values at any time, we quickly lose the ability to reason about what a given variable contains at any point in time. Reading code is a spatial experience, but debugging side-effects is a temporal one and turns our code into a branching three dimensional hydra of state. The compiler also lacks the ability to help us when we make small errors in judgement regarding these mutations.

We can't avoid side-effects, but wouldn't it be amazing if we could define classes of them and how they combine together? If we could, the compiler could be informed about certain safe-guards we want to ensure when performing those side-effects. Then, when using those effects we could focus more energy on the goal of our code rather than debugging how we get there.

# Haskell

Before we continue, I want to give an intro to Haskell syntax. I find that other tutorials trying to compare other languages to Haskell end up being distracting, so we will only be using Haskell in this tutorial. I won't go over setting up your environment, but if you'd like to follow along I highly recommend installing `ghcup` using your package manager of choice and running `ghcup tui` to install the compiler `ghc`, the language server `hls`, the project manager `stack`. The language server should let you use your IDE of choice.

## Overview

### Values

We can't do much without values and variables in a language. Values in Haskell are immutable (they cannot be changed after declaration). Aside from that, declaring variables is similar to other languages. Also, note that comments begin with `-- I'm a comment :)` and will not be interpreted as code by the compiler.

{% highlight Haskell %}
let x = 40
    y = 2
{% endhighlight %}
### Functions or Methods

Haskell is built on the foundation of [Lambda Calculus](https://en.wikipedia.org/wiki/Lambda_calculus), which is an entire computing paradigm completely built out of functions. That's a rabbit hole in itself, and it is not required reading for this post. The important takeaway is that the foundational atom of Haskell is the function, defined as an operation that takes one value as input and returns one value as output.

{% highlight Haskell %}
-- Uninteresting function, just returns what you give it
f x = x
{% endhighlight %}

A consequence of this choice is that functions of multiple arguments are actually a chain of individual functions each returning the next. For example, the operator `+` for adding numbers takes a number and returns a function taking one more value for the second number being added. This process is called Currying, and the intermediate value of something like `(+3)` ("add three") is called a partially applied function.

{% highlight Haskell %}
3 + 5  -- 8
(+3) 5 -- 8
{% endhighlight %}

When a value is partially applied in a function, it is also said to be "closed over" which means the application is now state stored in the function. This is the fundamental way of representing state in Haskell, so you'll see closures used everywhere. This is a useful concept to grok fully before moving forward.

Even though functions of multiple arguments are really nested functions, the syntax lets you define functions as if they took multiple arguments:

{% highlight Haskell %}
quadratic a b c x = a + b*x + c*x^2

let polynomial = quadratic 1 2 3
    -- The coefficients have been closed over, now we can call
    -- the quadratic with the remaining value. This pattern of
    -- placing parameters first and the input last is called
    -- "data-last" and is useful in curried languages.
    atZero = polynomial 0 -- 1
    atFour = polynomial 4 -- 57
{% endhighlight %}

One more useful piece of syntax is lambdas, which let you define functions in-place wherever you need them.

{% highlight Haskell %}
example x y = x * y + 3

-- Here is the same function as a lambda,
-- you don't even need to assign the right
-- hand expression to a variable if you like.
let theSameThing = \x y -> x * y + 3
{% endhighlight %}

### Types

Types allow us to tag values with metadata informing the compiler what subset of values is acceptable at the call site of a given function. More formally, types are a value-level semantic construct, allowing us to express statements about the values of our program.

It turns out we have already been using types in the previous examples, albeit implicitly. Haskell's compiler performs state inference in a rather clever way. Until specified, all values are assumed to be compatible with any type. The moment you saturate a value with a given value, its type becomes determined and the compiler will propagate that to any spot the variable is used. In this sense, Haskell is generic by default until we specify types. Even though you may be tempted to omit types because of this, it's generally advisable to give type signatures to all of your functions so that you can get better compiler errors. Aim to be as strict as you can be, and selectively relax your types when you need to support more generic cases.

{% highlight Haskell %}
-- The part we add on top here is the type signature
add :: a -> a -> a
add x y = x + y

-- Here we saturate `a` with `Int`
let sum = add 3 4 -- 7
    badSum = add 3 "lol" -- This will not compile
{% endhighlight %}

If you are curious to see the type of a value, in the Haskell `ghci` interactive REPL you can use the `:t` command to print the type of any variable. For instance:

{% highlight Haskell %}
ghci> :t (add 3)
(add 3) :: Num a => a -> a
{% endhighlight %}

There is a new piece of syntax here `Num a => . . .` that we will visit soon, for now just think of it as expressing "any type that is a number".  We could use a `Float`, `Int`, `Double`, etc... and still use `add 3`.

### Sum Types

Haskell uses a type system that is algebraic, which is a buzzword that leads you to believe it is much more complicated than it actually ends up being. Formally, the math that leads to this type system is [wildly complex](https://en.wikipedia.org/wiki/Product_(category_theory)) but just like in the case of Lambda Calculus it is not necessary reading for this post.

Often in our programs we want to express a value that can take on one of many values. We call this a sum type, and in Haskell you can define such a value like this:

{% highlight Haskell %}
data Theme = LightMode | DarkMode | UseSystemDefault
{% endhighlight %}

On the left, we have the type we are creating (`Theme`). On the right are data constructors, they are functions that create values of the type. They may not seem like functions at first, but it becomes more clear when we define a sum type that also has values contained inside of it:

{% highlight Haskell %}
data NotificationPreference
  = DoNotNotify
  | NotifyIntervalDays Int
  | NotifyIntervalMonths Int
  -- ^ This is Haskell style formatting for multi-line
  --   syntax. Separator first, then value.

let annoyingNotifications = NotifyIntervalDays 1
{% endhighlight %}

The type itself is also a function, surprisingly enough. In the above example, `NotificationPreference` takes no type arguments, but if we wanted to make a sum type that could take arguments we absolutely could.

{% highlight Haskell %}
data UserInput a
  = FromKeyboard a
  | FromTextToSpeech a
  | FromSiameseTwins a a
  | FromMindControl a

-- `a` is saturated with `Int`, producing `UserInput Int`
let userValue = FromKeyboard 5
{% endhighlight %}

For values, we had data constructors, and now for types we have type constructors. `UserInput` is a type constructor taking one type argument and producing a type we can use to tag a value. To review:

1. `UserInput` is a **type constructor**
2. `UserInput Int` is a **type**
3. `FromKeyboard`, `FromTextToSpeech`, `FromSiameseTwins`, and `FromMindControl` are all **data constructors**
4. These are values of a **sum type**

A very useful sum type in Haskell that replaces `NULL` in other languages is `Maybe a`:

{% highlight Haskell %}
data Maybe a = Just a | Nothing
{% endhighlight %}

This minimally represents a value that may be present, or absent.
### Product Types

Product types are even simpler, and their name also comes from the algebraic origins of this type system. In practice, they are just groupings of values of different types.

{% highlight Haskell %}
type UserNameAndAge = (String, Int)
        -- ("Leeroy Jenkins", 29)
        --    ^ a "product" of `String` and `Int`

type Color = (Int, Int, Int) -- e.g. (255, 0, 255) for purple
{% endhighlight %}

You can keep adding more types to the product until you get abominations like `(Int, String, Bool, Int, Int, String)` but at a certain point it makes more sense to use the next type construct in Haskell.
### Record Types

A record type is something like a product type, except that for a given value you are also given functions to extract one piece of the product. It's easier to give an example:

{% highlight Haskell %}
data Address = Address
 { address :: String
 , street :: String
 , city :: String
 , zipCode :: Int
 }
 -- This is kind of like:
 -- (String, String, String, Int)
 -- but actually sane.

let myAddress = Address
      { address = "42"
      , street = "Wallaby Way"
      , city = "Sydney"
      , zipCode = 12345
      }
    myCity = city myAddress
{% endhighlight %}

### Pattern Matching

A consequence of having the compiler match on values to saturate unknown types is that value definition can be structural. This is called pattern matching, and it allows for some of the most expressive definitions in any language. In Typescript, this is somewhat adjacent to pattern matching, but it's much more powerful. In a nutshell, defining types and consuming types uses the _exact same syntax_. This works for values in addition to types.

{% highlight Haskell %}
-- You can match on values for functions. `0` and `1`
-- as arguments will match before the more general pattern
-- listed last.
fib n :: a -> a
fib 0 = 1
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)

-- Alternatively, you can case match
fib n :: a -> a
fib n = case n of
  1 -> 1
  2 -> 1
  _ -> fib (n - 1) + fib (n - 2)

-- This extends to sum types as well
data ListIndex = ZeroBased Int | OneBased Int

toZeroBased :: ListIndex -> Int
toZeroBased listIndex = case listIndex of
  -- On the left-hand side, x is bound to the contained value
  ZeroBased x -> x
  OneBased x -> x - 1
{% endhighlight %}

There are so many more ways this pattern may be used, but this is probably enough to continue our path.

## Structure and Payload

So far most of the type machinery we have covered describes values, but what if we also want to describe a structure of values? Sum and product types themselves are technically structures. You can think of structure as the trunk/branches of a tree and the types as leaves. Sum types are like a tree with only one layer of branches, and product types are like conjoined pre-existing trees. What if we want to represent something more complicated, like a list of values?

A list value can be described as either an empty list `[]` or a value and the rest of the list `a : as`.  `:` here is implied to be a function taking a value of type `a` and a list of type `a` and returning a new list with all of the values of `as` but with the first value prepended to the list. We just described how the value of a list works, and the type follows the exact same format:

{% highlight Haskell %}
-- This is a type defining itself in terms of itself.
data List a = [] | a : List a
-- `:` is an infix data constructor. In Haskell, we can
-- change infix to prefix using backticks if we want:
     List a
       = []
       | `:` a (List a)

-- Alternatively, if it helps, we can avoid infix to show
-- how `List a` is defined using no new concepts:

data MyList a
  = EmptyList
  | Joined a (MyList a)
{% endhighlight %}

We can follow this same pattern for trees, matrices, or any other structures we want to represent.

## Typeclasses

### Second-Order Types

So far we have used types to describe values, but we haven't covered that weird bit of syntax we discovered earlier `Num a => . . .`. This is an example of a higher-order concept called a typeclass. Just like how types describe values, typeclasses describe types. For instance, `Num` is defined as:

{% highlight Haskell %}
class Num a where
  (+), (-), (*)       :: a -> a -> a
  negate              :: a -> a
  abs                 :: a -> a
  signum              :: a -> a
  fromInteger         :: Integer -> a
  x - y               = x + negate y
  negate x            = 0 - x
{% endhighlight %}

There is a lot going on here in the typeclass, but abstractly we are informing the compiler how a given value can be treated as a number. When we write a function definition, we can use this typeclass as a constraint on our type to support using all of the contents of the typeclass inside of our function.

{% highlight Haskell %}
product :: Num a => a -> a -> a
product x y = x * y

-- Alternatively, since we use `*` in the function body the
-- compiler infers that `x` and `y` must be constrained `Num a`.
-- The same pattern we used in our value system to saturate types
-- also works for typeclasses (many features are symmetrical in
-- Haskell between values and types).

product x y = x * y
-- ghci> :t product
-- product :: Num a => a -> a -> a
{% endhighlight %}

### Higher-Kinded Types

To break my rule for a moment, I would like to make a comparison to other languages because this is a distinction that is very easy to get wrong. So far, typeclasses look a lot like generics in other languages. For instance, in Typescript you might think of making a `Num<a>` type that contains functions to do all of the same operations. Alternatively, you could also liken this to method overloading in languages like C++ where you define multiple versions of a method like `add` so that the compiler can choose which version of the method should be called at runtime for a given value (assuming the value can take on multiple types, otherwise it will just pick the one matching version).

So, how are typeclasses different than those other forms of generics or polymorphism? The real power comes with the fact that the type system truly does mirror the value system in Haskell. Type constructors are functions, and they can take multiple arguments. What if we wanted to create a typeclass that describes a type whose sum can be computed? Let's call this `Summable`.

{% highlight Haskell %}
class Summable s where
  getSum :: s a -> a
{% endhighlight %} 

The nuance here is subtle. At first glance, this looks like a generic or a method overload, but notice how our type `s` being constrained is actually being called with a type argument in `getSum` as `s a`. In other words, `s` is a higher kinded type (a function of types). If you were to try and define `Summable<s>` in Typescript then have some `s<a>` in the definition, the compiler would explode. Type arguments are final, and cannot accept more type arguments. In another word, generics are not composable in this way, and so you cannot make assertions about types like this in any language that does not support higher kinded types such as Haskell.

We call this a "higher kinded" type because we are talking about functions of a type, instead of functions of a value. We actually glimpse into a third level of abstraction, a type of types called "kind" denoted by `*`. `s` in this example is a function from kind `*` to kind `*` also denoted as `* -> *`:

{% highlight Haskell %}
ghci> :i Summable
type Summable :: (* -> *) -> Constraint
class Summable s where
  getSum :: s a -> a
{% endhighlight %}

One more subtle nuance between something like method overloading and typeclasses is the order of definition. In overloading, you define the method for a given type. In typeclasses, you are developing a type for a given set of methods. This transpose is a direct reflection of [the expression problem](https://en.wikipedia.org/wiki/Expression_problem) (which is not required reading, but if you are curious to read more, check it out).

## A Useful Ladder

Alright, how do we actually _do_ anything in Haskell though? All of this expression has so far been quite poetic, but at the end of the day our philosophic principles are based around actually getting something done. If we can't do that, then all of this is not exactly helpful.

It turns out that typeclasses are the missing link to take all of the concepts we have produced so far and generate a useful and expressive language. Going forward, it is important to grok the difference between structure (list, sum, product, etc...) and payload (the types at the leaves). We are going to construct a three-rung ladder that lets us climb to any operation we need to perform in Haskell, but in a type-safe way that the compiler can help us with along the way.

### Functor

Function application is the fundamental unit of work in Haskell. it's so fundamental, in fact, that we even have an operator for it: `$`.

{% highlight Haskell %}
timesTwo :: Num a => a -> a
timesTwo x = 2 * x

-- ghci> :t ($)
-- ($) :: (a -> b) -> a -> b
-- ^ Take a function `f` and apply it to an argument

let aResult = timesTwo $ 5
    theSame = timesTwo 5
{% endhighlight %}

But what if we want to apply a function over more than just one value? The most basic operation we could wish to apply to a given structure of values is some transformation of the leaves, or the payload values. We could define this per-type, or we could recognize that this "mapping" is a fundamental operation on our values and create a typeclass to describe some type that supports this operation. We call this typeclass a `Functor`, and abstractly it is any type that supports mapping with a provided function. Just like `$` is an operator applying a function to a value, we call this new operator `<$>` which applies a function to a structure of values (also called `fmap`).

{% highlight Haskell %}
-- Instances get to decide what this means for the given
-- type.
class Functor f where
  <$> :: (a -> b) -> f a -> f b

-- Examples:

-- Apply a function over a list.
(+3) <$> [1, 2, 3] -- [4, 5, 6]

-- Chain two functions together:
-- The payload here is an operation, not a value.
-- "Add four after multiplying by two"
let chained = (+4) <$> (*2)
    result = chained 7 -- 7*2 + 4 = 18

-- Compare with a value that may or may not be present
let found = Just 3
    missing = Nothing
    check x = x == 3
    A = check <$> found -- Just True
    B = check <$> missing -- Nothing
{% endhighlight %}

### Applicative

What happens if we want to apply a function over multiple values containing a structure of leaves? For instance, what if we want to add two `Maybe Int` values? We can try using `fmap` first just to see:

{% highlight Haskell %}
let x = Just 4
    y = Just 8
    (+) <$> x -- Just (+4)
    -- ???
{% endhighlight %}

We get stuck, this is awkward. We want to add these values, but we can't do it directly, and `fmap` partially applies the inside and leaves us with a structure of that function partially applied. We need machinery to apply that to the next value.  We create a new typeclass, `Applicative`:

{% highlight Haskell %}
class Applicative f where
  <*> :: f (a -> b) -> f a -> f b
  pure :: a -> f a
{% endhighlight %}

Think of `<*>` like a comma in function application, only it's happening inside of the structure. For now, ignore `pure`.

{% highlight Haskell %}
-- Examples:

-- Add two `Maybe Int`s
let x = Just 4
    y = Just 8
    result = (+) <$> x <*> y -- Just 12

-- Find all sums of values from two lists
let  sums = (+) <$> [1, 2, 3] <*> [4, 5, 6]
			-- ^ [5,6,7,6,7,8,7,8,9]
			
-- Note that Cartesian product as behavior is a
-- choice, and not a requirement. We could also
-- choose to zip in place with `ZipList`.
let zipSums = (+) <$> ZipList [1, 2, 3] <*> ZipList [4, 5, 6]
			  -- ^ ZipList {getZipList = [5,7,9]}
{% endhighlight %}

`pure` is needed so that we can lift a function into this "structure of operations" concept. It's also a way to take a payload value and bring it into the `Functor` type, so we could have ostensibly chosen to introduce it there too. Don't let it confuse you from the real star of the show here `<*>`. Here's how `pure` can be used, though:

{% highlight Haskell %}
-- Pure is just a way to bring a value or operation into the application
let x = Just 4
    y = Just 8
    knownVersion = (+) <$> x <*> y -- Just 12
    pureVersion = pure (+) <*> x <*> y -- Just 12 (notice the lack of <$>)
    anotherWay = (+) <$> pure 10 <*> y -- Just 18
{% endhighlight %}

### Monad

So far, we have only been able to change payload data, leaving the structure alone. What if we want to change the structure too? If this is in `Maybe`, this means we want to be able to conditionally return `Nothing` or `Just something` depending on what the value is. For a list, we want to return a new list with combinations from multiple lists, or with the contents of two lists of identical length zipped together. Note that for the list example we have two options for monadic behavior, this is not canonical and depending on the monad you choose to use it will change the behavior.

Just like before, let's see if we can change the structure without using anything extra. To recap we have `<$>` "map over" and `<*>` "apply further with" (formally, these are known as "fmap" and "apply" but I added more words to explain further).

{% highlight Haskell %}
-- Assume these came from elsewhere, input, DB, etc...
let address = Just "123 P. Sherman Lane"
    password = Just "hunter3"
    missingName = Nothing
    presentName = Just "Leeroy Jenkins"

-- All these values are required
data User = User
  { userAddress :: String
  , userPassword :: String
  , userName :: String
  }
  deriving Show

-- This is a use of functor/applicative to construct a user
-- from optionally present values.
let badUser = User <$> address <*> password <*> missingName
        -- ^ Nothing
    myUser = User <$> address <*> password <*> presentName
        -- ^ Just (User {..})

-- But what if we want to condition on the password? We want
-- to hypothetically return `Nothing` if the password is
-- too short (<9001 characters).
validUserName :: User -> Maybe User
validUserName user =
  | length (userPassword user) < 9001 = Nothing
  | otherwise = Just user

-- We can run this easily enough
let validated = validUser myUser

-- But what if we also want to validate address length?
validUserAddress :: User -> Maybe User
validUserAddress user =
  | length (userAddress user) < 9001 = Nothing
  | otherwise = Just user

-- We can't run this anymore... Our validation expects
-- a `User` not a `Maybe User`. How do we chain these?
-- validUserAddress (validateUserAddress myUser)
--	^^^^^^^^^^^^^^^^^^^^^^^^^^^^ This is `Maybe User`
{% endhighlight %}

This is one of many motivations of chaining modification of not just payload, but structure. We call these `a -> f a` operations where you take a base value of type `a` and produce a value in the application `f a` "binding" in Haskell.

{% highlight Haskell %}
instance Monad f where
  =<< :: (a -> f b) -> f a -> f b
{% endhighlight %}

Notice how in the type signature, the first thing we pass is `a -> f a` which can be interpreted as "controlling both structure and payload of the output". The final `f b`'s structure will be determined by the operation you pass. For instance, here we can finally chain user validation:

{% highlight Haskell %}
-- Parentheses not needed, but added for clarity
let validUser = validUserAddress =<< (validUserName myUser)
{% endhighlight %}

### Sugar

If we already have established methods to chain, `=<<` works well, but what if we are doing things ad-hoc? We can use lambdas to chain operations. I'll stick to using `Maybe a` because it is honestly the easiest structure to think about with these operations (but keep in mind this reasoning applies to any `Monad` instance):

{% highlight Haskell %}
let userValue = Just 4

(\x -> Just (x * 3))
  =<< (\x -> if even x then Just x else Nothing)
  =<< userValue
  -- Just 12
{% endhighlight %}

This is starting to look pretty unreadable, especially if we're actually doing a real program with more complexity and edge cases. If you're thinking you're ready to go back to an imperative style, Haskell actually agrees with you here. We introduce a `do` syntax to convert the above code to:

{% highlight Haskell %}
let userValue = Just 4

do
  x <- userValue
  y <- if even x then Just x else Nothing
  Just (y * 3)
  -- Just 12
{% endhighlight %}

This is actually sugar, not a new operation. There is a flipped version of `=<<` that has the input on the left and the function on the right (it is called `>>=`) and if we use that operator to rewrite the operation you can see the similarity (in fact, this is roughly what the above desugars to):

{% highlight Haskell %}
let userValue = Just 4

(
  userValue >>= \x
  -> if even x then Just x else Nothing) >>= \y
  -> Just (y * 3)
{% endhighlight %}

That's pretty bad, still, so the `do` syntax really makes the usage of `Monad` shine. It reads imperative, so from an ergonomics perspective it becomes very easy to use these structure manipulations without resorting to lambda callback hell. I think that other languages disincentivize using these types of patterns because they lack `do` notation, and for almost no other reason.

Finally, since we already have a method `pure` to bring a value into the application, we can make `do` blocks look the same whether or not we are in `Maybe a`, `[a]`, or other `Monad a` instances. I also cheat a little bit and use a value `mempty` from the `Monoid a` typeclass, which indicates the "empty" value for that operation. For `Maybe a` it will be `Nothing`, for a `[a]` it will be `[]`, for `String` it will be `""`, etc...

{% highlight Haskell %}
let userValue = Just 4

do
  x <- userValue
  y <- if even x then pure x else mempty
  pure (y * 3)
  -- Just 12
{% endhighlight %}

Check it out, we can apply the same operation to a list now!

{% highlight Haskell %}
let userValues = [1, 2, 3, 4, 5]

do
  x <- userValues
  y <- if even x then pure x else mempty
  pure (y * 3)
  -- [6, 12]
{% endhighlight %}

The tricky part sometimes can be figuring out what monad we are inside of for a given do block, but each monad itself isn't that crazy. It's a structure with payload where we define how structure should be combined and how to map operations over the payload. The complexity lies with the implementer to make sure these definitions are sound (if you want to get formal, see the monad laws), but using the monads should be straightforward.

# Wrapping Up

That's most of what you'll need to know to write Haskell. To save time, I avoided talking about one of the other big parts of Haskell: Laziness. I think that if you understood everything in this post, you could easily loop back and learn the lazy parts of Haskell and it would map well onto what you have learned here.

I hope that you enjoyed this whirlwind tour of Haskell, and that the patterns here are as fascinating and useful for you as they have been for me.
