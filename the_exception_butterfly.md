---
math: true
permalink: the-exception-butterfly
title: The Exception Butterfly
description: An overview of Haskell from values to monads for beginners.
featured_image: /blog/images/the_exception_butterfly/haskell-art.jpg
layout: page
date: 2026-04-11
---

{:refdef: style="display:flex;align-items:center;flex-direction:column;"}
![A diagram showing monad transformer operations between Maybe and Either](/blog/images/the_exception_butterfly/cover.jpg)
{:refdef}

The `Maybe` and `Either` types may be Haskell's most notable contribution to computer science, inspiring similar structures in Rust, C++, Javascript, Python, and other languages. These types not only upgrade the usual concept of `null`, but they also include composable machinery to make dealing with potentially missing data or failing computations ergonomic and informed by the compiler. These machines come with a cost, however, and oftentimes the resulting code (even with their monadic goodness) can be difficult to reason about and harkens back to similar scenarios to "Callback Hell" in early JS days before Promises existed.

Thankfully, there are a lot of extra tools we can use to make working with these types much easier. When used correctly, code involving potentially missing fields, errors, and even nested computations should read like a procedural program and still maintain all of the guarantees of Haskell's strong typing. I'm creating this post to share what I've found on my journey of using Haskell over the past three years.

# Mixing Maybe and Either

`Maybe` already contains within it the machinery to chain value dependencies in a readable and ergonomic fashion:

{% highlight Haskell %}
mTitle :: Maybe Text
mDescription :: Maybe Text
mAuthor :: Maybe Person

book :: Maybe Book
book = do
	title <- mTitle
	description <- mDescription
	author <- mAuthor
	
	pure $ Book {..}
{% endhighlight %}

When we wish to upgrade a `Maybe` to an `Either`, however, things can start to become harder to follow. Typically, you want to start using `Either` when you wish to annotate the reason for the failure (after all, `Either` is semantically compatible with `Maybe`, it just adds a value for the `Nothing` branch).

{% highlight Haskell %}
data BookError
	= NoTitle
	| NoDescription
	| NoAuthor

book :: Either Book BookError
book = do
	title <- case mTitle of
		Just title -> Right title
		Nothing -> Left NoTitle
	title <- case mDescription of
		Just description -> Right description
		Nothing -> Left NoDescription
	author <- case mAuthor of
		Just author -> Right author
		Nothing -> Left NoAuthor
	
	pure $ Book {..}
{% endhighlight %}

There is actually a nice library method for this in `Control.Error.Util` called `note` that lets you tag the `Nothing` branch and promote the `Maybe` into an `Either`:

{% highlight Haskell %}
book :: Either Book BookError
book = do
	title <- note NoTitle mTitle
	description <- note NoDescription mTitle
	author <- note NoAuthor mTitle
	
	pure $ Book {..}
{% endhighlight %}

Sometimes this is also called `maybeToEither`, but I prefer the terser `note`.

# Transformers

`Maybe` and `Either` do not support enough functionality to make practical programming in Haskell ergonomic. All Haskell programs operate inside of the `IO` monad, even if you can produce plenty of methods that operate on types like `Maybe` independently (which is often a good idea anyways to increase ease of testing). Still, you can't avoid operating inside of `IO` as that is where you can do network requests, database operations, and all side-effects. Dealing with types like `IO (Maybe a)` is often unavoidable, and produces some new challenges. `IO` and `Maybe` both have their own monadic behavior, and yet `do` blocks only support using one at a time. Things can get hairy.

{% highlight Haskell %}
getTitle :: IO (Maybe Text)
getDescription :: IO (Maybe Text)
getAuthor :: IO (Maybe Person)

getBook :: IO (Either Book BookError)
getBook = do
	mTitle <- getTitle
	mDescription <- getDescription
	mAuthor <- getAuthor

	pure $ do
		title <- note NoTitle mTitle
		description <- note NoDescription mTitle
		author <- note NoAuthor mTitle

		pure $ Book {..}
{% endhighlight %}

Transformers are built specifically to solve this problem for a grab-bag of monads. Conceptually, they let you "lift" operations of the outer monad into an upgraded one that has your nested type, along with a set of methods to make operating with the nested monad as easy as `Maybe` and `Either` when they are not nested.

{% highlight Haskell %}
getTitle :: IO (Maybe Text)
getDescription :: IO (Maybe Text)
getAuthor :: IO (Maybe Person)

getBook :: IO (Maybe Book)
getBook = runMaybeT $ do
	title <- MaybeT $ getTitle
	description <- MaybeT $ getDescription
	author <- MaybeT $ getAuthor
	
	pure $ Book {..}
{% endhighlight %}

Magic! Under the hood, this is just a `newtype` that represents the nested monads (in this case `IO (Maybe a))`). These transformers are abstract, but the outer monad is typically something derivative of `IO` and the inner monad is usually `Maybe` or `Either`. Sometimes you'll see `ListT`, but it has been more rare in my experience. Without getting into the weeds, these concepts all serve one common purpose "make dealing with potentially absent values inside of `IO` easy".

{% highlight Haskell %}
-- From the standard library:
newtype MaybeT m a = MaybeT { runMaybeT :: m (Maybe a) }
newtype ExceptT e m a = ExceptT (m (Either e a))

class MonadTrans t where
    -- | Lift a computation from the argument monad to the constructed monad.
    lift :: (Monad m) => m a -> t m a
{% endhighlight %}

Along with the `newtype` constructor/runner `MaybeT`/`runMaybeT`, there are also convenience functions. `lift` allows you to take something like `IO a` and turn it into a `MaybeT IO a` (which just wraps the result in `Just` under the hood). `hoistMaybe` allows you to take a `Maybe a` value and bring it into `MaybeT IO a` as well. Again, I'm using `IO` a lot here but the outer monad can really be anything. There is also `ExceptT`, which is very similar to the above example except it just uses `Either` instead of `Maybe`. The naming is unfortunate, as `ExceptT` has nothing to do with runtime exceptions. `ExceptT` also supports `lift` (all transformers must), as well as a similar `hoistEither` that parallels `hoistMaybe`.
# Mixing Transformers

This is where transformers alone have not done as good of a job with making code easy to write and easy to follow. Still, there are some nice utility functions akin to equivalents in the non-nested case that can clean things up nicely. Say you have the following control flow:

{% highlight Haskell %}
getTitle :: IO (Maybe Text)
getDescription :: IO (Maybe Text)

-- Author here is now nested Either
getAuthor :: IO (Either ApiError Person)

getBook :: IO (Either BookError Book)
getBook = runExceptT $ do
	mTitle <- lift getTitle
	mDescription <- lift getDescription
	
	author <- ExceptT $ getAuthor
	
	case mTitle of
		Nothing -> Left NoTitle
		Just title -> case mDescription of
			Nothing -> NoDescription
			Just description -> pure $ Book {..}
{% endhighlight %}

This is a toy example, but you can see things are starting to get unruly. In production code this can explode to a few hundred lines of nesting with sometimes up to four levels. How can we make this better? We can start by using `noteT` which is the transformer equivalent of `note` which we already encountered. There's a slight problem, though. `note` was pretty convenient in that it was a standalone method that upgraded a `Maybe` into an `Either`, but `noteT` _requires_ a `MaybeT` note an `IO (Maybe a)`. It's a bit more awkward, but we can nest the `newtype` constructor and `noteT` to clean up our code:

{% highlight Haskell %}
getBook :: IO (Either BookError Book)
getBook = runExceptT $ do
	mTitle <- noteT NoTitle $ MaybeT getTitle
	mDescription <- noteT NoDescription $ MaybeT getDescription
	author <- ExceptT $ getAuthor
	
	pure $ Book {..}
{% endhighlight %}

# A Missing Method

Compared to much of Haskell, Monad Transformers have _a lot_ of methods to memorize and keep track of. When do you `lift` vs. `hoist`? How do I promote a value into the current transformer? The intuition comes with time and use of these patterns in your code, but a diagram can be helpful. It turns out there's actually a really wonderful symmetry for these operations:

![Diagram of monad transformer operations for Maybe and Either](/blog/images/the_exception_butterfly/diagram.png)

I call this the "Exception Butterfly" and I wish I saw it a lot earlier on in my Haskell journey. There are a lot of moving pieces here, but the symmetry helps a lot and explains how (once you get to know them) these operations are not that hard to put into practice. Granted, what about the operation we just used to improve the code in the last example? You'll notice that there is no arrow from `m (Maybe a)` to `ExceptT e m a`, unless you count the composition of `MaybeT` with `noteT` (which is what we did).

As I have written more and more Haskell, that is the arrow I have been missing the most. Perhaps avoiding yet-another-function is defending the [Fairbarn Threshhold](https://wiki.haskell.org/Fairbairn_threshold){:target="_blank"} of Haskell, but it's so common that I am always reaching for it and so I'm going to give it a name here: `annotateT`. It is like an upgraded version of `noteT` that also lifts.

![The same diagram, but extended to include annotateT](/blog/images/the_exception_butterfly/diagram-extended.png)

{% highlight Haskell %}
annotateT e = (noteT e) . MaybeT
{% endhighlight %}

Ultimately though, there's not a good spot to place this concept as it has no parallel. Even if you don't end up making a utility method for it, just know that you can compose operations to produce it whenever you need it. In general, this holds. If you have some type and need to bring it somewhere else, follow arrows on the diagram and compose as you go.
