Though we've only just begun our dive into *Bananas, Lenses, Envelopes, and Barbed Wire*, the next natural step in understanding recursion schemes brings us outside its purview. We must turn our attention to a paper written seven years later--- [*Primitive(Co)Recursion and Course-of-Value (Co)Iteration, Categorically*](http://cs.ioc.ee/~tarmo/papers/inf99.pdf), by Tarmo Uustalu and Varmo Vene. *Primitive (Co)Recursion* explores and formalizes the definition of apomorphisms (introduced first by Meijer et. al, and which we discussed, briefly, in the [previous installment](http://blog.sumtypeofway.com/recursion-schemes-part-iii-folds-in-context/)) and describes two new recursion schemes, the *histomorphism* and the *futumorphism*.

*Primitive (Co)Recursion* is a wonderful and illuminating paper, but it is dense in its concepts for those unfamiliar with category theory, and uses the semi-scrutable bracket syntax introduced by *Bananas*. But there's no need for alarm if category theory isn't your cup of tea: Haskell allows us, once again, to express elegantly the new recursion schemes defined in *Primitive (Co)Recursion*. Guided by Uustalu and Vene's work, we'll derive these two new recursion schemes and explore their ways in which they simplify complicated folds and unfolds. Though these new morphisms are, definition-wise, simple variations on paramorphisms and apomorphisms, in practice they provide surprising power and clarity, as Uustalu and Vene assert:

> \[We\] argue that even these schemes are helpful for a declaratively thinking programmer and program reasoner who loves languages of programming and program reasoning where programs and proofs of properties of programs are easy to write and read.

That sure sounds like us. Let's get going. This article is literate Haskell; you can find the source code [here](https://github.com/patrickt/recschemes/blob/master/src/Part4.lhs).

### A Brief Recap

In our first entry, we defined `Term`, the fixed-point of a Haskell `Functor`, with an `In` constructor that wraps one level of a structure and an `out` destructor to perform the corresponding unwrap[^1].

    newtype Term f = In { out :: f (Term f) }

Given an algebra --- a folding function that collapses a Functor `f` containing `a`'s into a single `a`---

    type Algebra f a = f a -> a

we use the catamorphism `cata` to apply a leaf-to-root[^2] fold over any recursively-defined data structure. `cata` travels to the most deeply-nested point in the data structure by `fmap`ing itself, recursively, into the next level of the stucture. When `fmap cata x` returns an unchanged `x`, we cease recursing (because we have hit the most-deeply-nested point); we can then begin constructing the return value by passing each node to the algebra, leaf-to-root, until all the recursive invocations have finished.

    cata :: (Functor f) => Algebra f a -> a -> Term f
    cata f = out >>> fmap (cata f) >>> f

But the catamorphism has its limits: as it is applied to each level of the structure, it can only examine the current carrier value from which it is building. Given the F-algebra `f a -> a`, each of the structure's children---the `a` values contained in the `f` container---has already been transformed, thus losing information about the original structure. To remedy this, we introduced `para`, the paramorphism, and an R-algebra to carry the original structure with the accumulator:

    type RAlgebra f a = f (Term f, a) -> a

    para :: Functor f => RAlgebra f a -> Term f -> a
    para f = out >>> fmap (id &&& para f) >>> f

### Running a Course with Histomorphisms

Paramorphisms allow us, at each stage of the fold, to view the original structure of the examined node before the fold began. Though this is more powerful than the catamorphism, in many cases it does not go far enough: many useful functions are defined not just in terms of the original argument to the function, but in terms of previous computed values. The classic[^3] example is the Fibonacci function, the general case of which is defined in terms of two previous invocations:

``` {.sourceCode .literate .haskell}
fib :: Int -> Int
fib 0 = 0
fib 1 = 1
fib n = fib (n-1) + fib (n-2)
```

We could express this function using a catamorphism---though one of the carrier values (`fib (n-1)`) would be preserved, as the accumulator of our fold, we would need another explicit recursive call to `cata` to determine the historical value of `fib (n-2)`. This is a bummer, both in terms of efficiency---we're recalculating values we've already calculated---and in terms of beauty: a function so fundamental as `fib` deserves a better implementation, especially given the expressive power of recursion schemes.

The imperative programmers among us will have a solution to this inefficiency: "iterate!", they will yell, or perhaps they will clamor "introduce a cache!" in a great and terrible voice. And it's true: we could compute `fib` with a for-loop or by memoizing the recursive call. But the former approach entails mutable state---a very big can of worms to open for such a simple problem---and the latter leaves us [with two problems](https://twitter.com/importantshock/status/241173326846898176). Uustalu and Vene's histomorphism provides a way out: we will *preserve the history* of the values our fold computes, so that further recursive calls to compute past values become unnecessary. This style of recursion is called *course-of-value recursion*, since we record the *values* evaluated as our fold function *courses* through the structure.

Rather than operate on an `f a`, a data structure in the process of being folded, we'll operate on a more sophisticated structure, so that the argument to our fold function contains the history of all applications of the fold itself. Instead of a just a carrier value `a`, our `f` will contain a carrier value and a recursive, unrollable record of past invocations, to wit:

``` {.sourceCode .literate .haskell}
data Attr f a = Attr
              { attribute :: a
              , hole      :: f (Attr f a)
              }
```

We'll call this `Attr`, since it's an 'attributed' version of `Term`.

An `Attr f a` contains an `a`---a carrier value, storing the in-progress value of the fold---as well as a fixed-point value (analogous to `Term`) at each level of recursion. Thanks to the fixed-point `hole` within the `f`, further `Attr` items are preserved, each of which contains the shape of the folded functor `f`. And within the `f` there lie further `Attr` values, each of which contains a carrier value yielded by *their* application in their `attribute` slot. And those `Attr` values in turn contain further `hole`s, which contain the historical records pertaining to *their* childrens' history, and so on and so forth until the bottom of the data structure has been reached. As such, the entire history of the fold is accessible to us: the `holes` preserve the shape of the data structure (which was lost during `cata`), and the `attribute` holds the record of applying the fold to each entity in said data structure.

We have a word for preserving a record of the past, of course---*history*[^4]. A fold operation that uses `Attr` to provide both an accumulator and a record of prior invocations is known as a *histomorphism*---a shape-changing (*morpho*) fold with access to its history (*histo*).

Let's define the histomorphism. It will, like its cousins `cata` and `para`, use an algebra for its fold function. But unlike the F-algebra of `cata` or the R-algebra of `para`, we'll be using an algebra that operates on an `Attr f a`, yielding an `a` out of it. We call this a course-of-value algebra, abbreviated to a *CV-algebra*, and define a type alias for it, so we end up with a more comprehensible type signature in the histomorphism:

``` {.sourceCode .literate .haskell}
type CVAlgebra f a = f (Attr f a) -> a
```

That is, a CV-algebra maps from a container `f` containing children of type `Attr f a` (which in turn contain `f (Attr f a)` children, as far down as is needed in the nested structure), to a final result type `a`. The shape of the folded structure and the history of its applications are all contained in its `Attr` values: all you have to do is unroll the `hole` value to go back one level in history and use `attribute` to examine the stored value.

Our `histo` function will be similar to `cata` and `para` at its heart. We start by unpacking the `Term`---the initial argument must be a `Term` rather than an `Attr`, since as we haven't started the fold yet we have no value to fill in for `attribute`. We will then recurse, with `fmap`, into the thus-revealed structure until we hit its root. We then use the CV-algebra to build the value, starting at the root and continuing upwards to the topmost leaf. These steps are analogous to how we defined `cata` and `para`, so let's start defining it:

    histo :: Functor f => CVAlgebra f a -> Term f -> a
    histo h = out >>> fmap worker >>> h

But what type should the worker have? Well, we can ask GHC, thanks to one of its most useful features[^5]---type holes. By prepending an underscore to the use of `worker`, we can allow the program compilation to continue as far as is possible---however, when the compilation process has finished, GHC will remind us where we used a type hole, and inform us of the type signature it inferred for `_worker`. (As a full-time Haskell programmer, I use this feature nearly every day.)

    histo :: Functor f => CVAlgebra f a -> Term f -> a
    histo h = out >>> fmap _worker >>> h

Running this code in GHC yields the following type-hole message:

    /Users/patrick/src/morphisms/src/Main.hs:14:24: error:
        • Found hole: ‘_worker’ with type :: Term f -> Attr f a

Okay, that makes sense! We're operating on `Term f` values (lifted into this context by the `fmap` within `histo`), and we need to yield an `Attr f a`, so that the outside `Term f` can be transformed into an `f (Attr f a)` and then passed into the CV-algebra.

An `Attr f a`, as defined above, contains two values: a plain `a` type, and a recursive `f (Attr f a)` hole. Given a `Term f` and our ability to invoke both `histo` and `worker` recursively, we can build the `Attr f a` we need. Let's start by defining the skeleton of `worker`: given a `Term f`, called `t`, it constructs an `Attr`, containing two fields.

    worker t = Attr _ _

The first field, the `a`, is yielded by recursing with `histo` on the provided `Term`---easy enough. This is just like the catamorphism---indeed, a catamorphism is a histomorphism that ignores the provided history.

    worker t = Attr (histo h term) _

The second field's construction is more clever: we unwrap `term` with the `out` function, which gives us an `f (Term f)` out of a `Term f`. Since we don't know exactly what type `f` is yet, we can't extract the contained `Term f`---but we can operate on it, with `fmap`, provided by the `Functor` constraint. So, to go from an `f (Term f)` to an `f (Attr f a)`, we need a function of type `Term f -> Attr f a`... hang on, that's just `worker` itself!

    worker t = Attr (histo h term) (fmap worker (out t))

This is the heart of `histo`'s elegance: it's 'doubly recursive', in that its `worker` function invokes both `histo` and `worker` itself.

Now we have a `histo` function that passes the typechecker:

    histo :: Functor f => CVAlgebra f a -> Term f -> a
    histo h = out >>> fmap worker >>> h where
        worker t = Attr (histo h t) (fmap worker (out t))

However, this function does not share its subcomputations properly: each iteration of `worker` recomputes, rather than reuses, all the nested `hole` values within the constructed `Attr`. We can fix this by promoting `worker` to operate on `Attr` values; by recursing with `fmap worker`, placing the input and output of the CV-algebra in a tuple with `&&&`, and then unpacking the tuple into an `Attr`, we ensure that all the constructed `Attr` values share their subcomputations.

``` {.sourceCode .literate .haskell}
histo :: Functor f => CVAlgebra f a -> Term f -> a
histo h = worker >>> attribute where
  worker = out >>> fmap worker >>> (h &&& id) >>> mkAttr
  mkAttr (a, b) = Attr a b
```

But what does this function *mean*? We've filled in all these type holes, and we have a working `histo` function, but why does it work? Why does this preserve the history?

The answer lies in `worker`, in the `id` function that captures and preserves the `Attr` the worker function is operating on. If we omitted that expression, we would have a function equivalent to `cata`---one that throws all its intermediate variables away while computing the result of a fold. But our worker function ensures that the result computed at each stage is not lost: as we flow, root-to-leaf, upwards through the data structure, we construct a new `Attr` value, which in turn contains the previous result, which itself preserves the result before that, and so on. Each step yields an up-to-date snapshot of what we have computed in the past.

By *not throwing out intermediate results*, and pairing these intermediate results with the values used to calculate them, we automatically generate *and update* a cache for our fold.

Now, I may have used `fib` as an example of a course-of-value recursive function, but I won't provide an example of using `histo` to calculate the nth Fibonacci number (though it's a good exercise). Let's solve a toy problem that's slightly more interesting, one that histomorphisms make clear and pure, and one whose solution can be generalized to all other problems of its ilk.

C-C-C-Changes
-------------

The [change-making problem](https://en.wikipedia.org/wiki/Change-making_problem) is simple: given a monetary amount `N`, and a set of denominations (penny, nickel, dime, &c.), how many ways can you make change for `N`? While it's possible to write a naïve recursive solution for this problem, it becomes intolerably slow for large values of `N`: each computation for `N` entails computing the values for `N - 1`, and `N - 2`, and `N - 3`, and so forth: if we don't store these intermediate amounts in a cache, we will waste our precious time on this earth. And, though this era may be grim as all hell, slow algorithms are no way to pass the time.

We'll start by setting up a list of standard denominations. Feel free to adjust this based on the denominational amounts of your country of residence.

``` {.sourceCode .literate .haskell}
type Cent = Int

coins :: [Cent]
coins = [50, 25, 10, 5, 1]
```

So our fundamental procedure is a function `change`, that takes a cent amount and returns a count of how many ways we can make change for said cent amount:

    change :: Cent -> Int

It is here where we hit our first serious roadblock. I asserted earlier that the change-making problem, and all the other [knapsack problems](https://en.wikipedia.org/wiki/Knapsack_problem) of its ilk, are soluble with a histomorphism---a cached fold over some sort of data structure. But here we're dealing with... natural-number values. There are no lists, no vectors, no rose trees---nothing mappable (that is to say, nothing with a `Functor` instance) and therefore nothing to fold over. What are we supposed to do?

All is not lost: we can fold over the natural numbers, just as we would fold over a list. We just have to define the integers in an unconventional, but simple, way: every natural number is either zero, or 1 + the previous. We'll call this formulation of the natural numbers `Nat`--- the zero value will be `Zero`[^6], and the notion of the subsequent number `Next`. Put another way, we need to encode [Peano numerals](https://en.wikipedia.org/wiki/Peano_axioms) in Haskell[^7].

``` {.sourceCode .literate .haskell}
data Nat a
    = Zero
    | Next a
    deriving Functor
```

We use `Term` to parameterize `Nat` in terms of itself---that is to say, given `Term`, we can stuff a `Nat` into it so as to represent an arbitrarily-nested hierarchy of contained `Nat`s, and thus represent all the natural numbers:

    one, two, three :: Term Nat
    one   = In (Next (In Zero))
    two   = In (Next one)
    three = In (Next two)

For convenience's sake, we'll define functions that convert from standard `Int` values to foldable `Term Nat`s, and vice versa. Again, these do not look particularly efficient, but please give me the benefit of the doubt.

``` {.sourceCode .literate .haskell}
-- Convert from a natural number to its foldable equivalent, and vice versa.
expand :: Int -> Term Nat
expand 0 = In Zero
expand n = In (Next (expand (n - 1)))

compress :: Nat (Attr Nat a) -> Int
compress Zero              = 0
compress (Next (Attr _ x)) = 1 + compress x
```

While this is, at a glance, obviously less-efficient than using integers, it's not as bad as it seems. We only have three operations: increment, converting from zero, and converting to zero. Restricting our operations to these---rather than writing our own code for addition or subtraction, both of which are linear-time over the Peano numerals---means that operations on our `Term Nat` types are almost the same as hardware-time costs, barring GHC-specific operations. As such, the expressivity we yield with our foldable numbers is well worth the very slight costs.

Given an amount (`amt`), we solve the change-making problem by converting that amount to a `Term Nat` with `expand`, then invoking `histo` on it with a provided CV-algebra---let's call it `go`. We'll define it in a where-clause below.

``` {.sourceCode .literate .haskell}
change :: Cent -> Int
change amt = histo go (expand amt) where
```

Since we're operating on foldable natural values (`Nat`) and ultimately yielding an integral result (the number of ways it is possible to make change for a given `Nat`), we know that our CV-algebra will have as its carrier functor `Nat` and its result type `Int`.

``` {.sourceCode .literate .haskell}
  -- equivalent to Nat (Attr Nat Int) -> Int
  go :: CVAlgebra Nat Int
```

Because `histo` applies its algebra from leaf-to-root, it starts at the deepest nested position in the `Term Nat`---that is to say, `Zero`. We know that there's only one way to make change for zero coins---by giving zero coins back---so we encode our base case by explicitly matching on a Zero and returning 1.

``` {.sourceCode .literate .haskell}
  go Zero = 1
```

Now comes the interesting part---we have to match on `Next`. Contained in that `Next` value will be an `Attr Nat Int` (which we'll refer to as `attr`), containing the value yielded from applying `go` to the previous `Nat`ural number. Since we'll need to feed this function into `compress` to perform actual numeric operations on it (since we did not write the requisite boilerplate to make `Nat` an instance of the `Num` typeclass[^8]), we'll use an @-pattern to capture it under the name `curr`.

``` {.sourceCode .literate .haskell}
  go curr@(Next attr) = let
```

Because we need to find out what numeric amounts (from `coins`) are valid change-components for `curr`, we have to get an `Int` out of `curr`. We'll call this value `given`, since it's our given amount.

``` {.sourceCode .literate .haskell}
    given               = compress curr
```

Now we have to look at each value of the `coins` list. Any values greater than `given` are right out: you can't use a quarter to make change for a dime, obviously.

``` {.sourceCode .literate .haskell}
    validCoins          = filter (<= given) coins
```

Given each number in `toProcess`, we have to consider how many ways we could make change out of that number---but, since we know that that we've already calculated that result, because it's by definition less than `given`! So all we have to do is look up the cached result in our `attr`. (We'll implement the `lookup` function later on---it is two lines of code.) We'll add all these cached results together with `sum`.

``` {.sourceCode .literate .haskell}
    in sum (map (lookup attr) validCoins)
```

Let's take a look at what we've written so far.

```haskell
change :: Cent -> Int
change amt = histo go (expand amt) where
  go :: Nat (Attr Nat Int) -> Int
  go Zero = 1
  go curr@(Next attr) = let
    given = compress curr
    validCoins = filter (<= given) coins
    in sum (map (lookup attr) validCoins)
```

Wow. This is pretty incredible. Not only do we have a simple, pure, concise, and performant solution to the change-making problem, but the caching is *implicit*: we don't have to update the cache ourselves, because `histo` does it for us. We've stripped away the artifacts required to solve this problem efficiently and zeroed in on the essence of the problem. This is remarkable.

I told you I would show you how to look up the cached values, and indeed I will do so now. An `Attr Nat a` is essentially a nonempty list: if we could pluck the most-final `Attr Nat a` after `change` has finished executing, we would see the value of `change 0` stored inside the first `attribute` value, the value of `change 1` stored inside the `attribute` within the first attribute's `hole`, and the value for `change 2` inside that further `hole`. So, given an index parameter `n`, we return the `attribute` if `n` is 0, and we recurse inside the `hole` if not, with `n - 1`.

``` {.sourceCode .literate .haskell}
lookup :: Attr Nat a -> Int -> a
lookup cache 0 = attribute cache
lookup cache n = lookup inner (n - 1) where (Next inner) = hole cache
```

A Shape-Shifting Cache
----------------------

Something crucial to note is that the fixed-point accumulator---the `f (Attr f a)` parameter to our CV-algebra---*changes shape* based on the functor `f` contained therein. Given an inductive functor `Nat` that defines the natural numbers, `Nat (Attr Nat a)` is isomorphic to `[]`, the ordinary linked list: a `Zero` is the empty list, and a `Next` that contains a value (stored in `Attr`'s `attribute` field) and a pointer to the next element of the list (stored in the `hole :: Nat (Attr Nat a))` field in the given `Attr`). This is why our implementation of `lookup` is isomorphic to an implementation of `!!` over `[]`---because they're the same thing.

But what if we use a different `Functor` inside an `Attr`? Well, then the shape of the resulting `Attr` changes. If we provide the list type---`[]`---we yield `Attr [] a`, which is isomorphic to a rose tree---in Haskell terms, a `Tree a`. If we use `Either b`, then `Attr (Either b) a` is a nonempty list of computational steps, terminating in some `b` value. `Attr` is more than an "attributed `Term`"---it is an *adaptive cache* for a fold over *any type of data structure*. And that is truly wild.

Obsoleting Old Definitions
--------------------------

As with `para`, the increased power of `histo` allows us to express `cata` with new vocabulary. Every F-algebra can be converted into a CV-algebra---all that's needed is to ignore the `hole` values in the contained Functor `f`. We do this by mapping `attribute` over the functor before passing it to the F-algebra, throwing away the history contained in `hole`.

``` {.sourceCode .literate .haskell}
cata :: Functor f => Algebra f a -> Term f -> a
cata f = histo (fmap attribute >>> f)
```

Similarly, we can express `para` with `histo`, except instead of just fmapping with `attribute` we need to do a little syntactic juggling to convert an `f (Attr f a)` into an `f (Term f, a)`. (Such juggling is why papers tend to use banana-bracket notation: implementing this in an actual programming language often requires syntactic noise such as this.)

``` {.sourceCode .literate .haskell}
para :: Functor f => RAlgebra f a -> Term f -> a
para f = histo (fmap worker >>> f) where
  worker (Attr a h) = (In (fmap (worker >>> fst) h), a)
```

Controlling the Future with Futumorphisms
-----------------------------------------

Throughout this series, we can derive unfolds from a corresponding fold by "reversing the arrows"---viz., finding the function dual to the fold in question. And the same holds true for histomorphisms---the dual is very powerful. But, to find the dual of `histo`, we must first find the dual of `Attr`.

Whereas our `Attr` structure held both an `a` and a recursive `f (Attr f a)` structure, its dual---`CoAttr`---holds *either* an `a` value---we'll call that `Automatic`---or a recursive `f (CoAttr f a)` value, which we'll call `Manual`. (Put another way, since `Attr` was a product type, its dual is a sum type.) The definition follows:

``` {.sourceCode .literate .haskell}
data CoAttr f a
  = Automatic a
  | Manual (f (CoAttr f a))
```

And the dual of a CV-algebra is a CV-coalgebra:

``` {.sourceCode .literate .haskell}
type CVCoalgebra f a = a -> f (CoAttr f a)
```

So why call these `Automatic` and `Manual`? It's simple---returning a `Manual` value from our CV-coalgebra means that we specify manually how the unfold should proceed at this level, which allows us to unfold more than one level at a time into the future. By contrast, returning a `Automatic` value tells the unfold to continue automatically at this level. This is why we call them *futu*morphisms---our CV-coalgebra allows us to determine the *future* of the unfold. (The term 'futumorphism' is etymologically dubious, since the 'futu-' prefix is Latin and the '-morpho' suffix is Greek, but there are many other examples of such dubious words: 'television', 'automobile', and 'monolingual', to name but a few.)

Like its predecessor unfolds `ana` and `apo`, the futumorphism will take a coalgebra, a seed value `a`, and produce a term `f`:

    futu :: Functor f => CVCoalgebra f a -> a -> Term f

We derived the anamorphism and apomorphism by reversing the arrows in the definitions of `cata` and `para`. The same technique applies here---`>>>` becomes `<<<`, and `In` becomes `out`. And as previously, we use a type hole to derive the needed signature of the helper function.

    futu :: Functor f => CVCoalgebra f a -> a -> Term f
    futu f = In <<< fmap _worker <<< f

    /Users/patrick/src/morphisms/src/Main.hs:28:32: error:
        • Found hole: ‘_worker’ with type :: CoAttr f a -> Term f

This also makes sense! The worker function we used in `histo` was of type `Term f -> Attr f a`---by reversing the arrows in this worker and changing `Attr` to `CoAttr`, we've derived the function we need to define `futu`. And its definition is straightforward:

``` {.sourceCode .literate .haskell}
futu :: Functor f => CVCoalgebra f a -> a -> Term f
futu f = In <<< fmap worker <<< f where
    worker (Automatic a) = futu f a        -- continue through this level
    worker (Manual g) = In (fmap worker g) -- omit folding this level,
                                           -- delegating to the worker
                                           -- to perform any needed
                                           -- unfolds later on.
```

When we encounter a plain `Continue` value, we continue recursing into it, perpetuating the unfold operation. When we encounter a `Stop` value, we run one more iteration on the top layer of the in-progress fold (transforming its children from `Coattr f a` values into `Term f` values by recursively invoking `worker`), then wrap the whole item up with an `In` constructor and return a final value. The product of this nested invocation of `worker` is then similarly passed to the `In` constructor to wrap it up in a fixpoint, then returned as the final output value of `futu`.

What differentiates this from `apo`---which, if you recall, used an `Either` type to determine whether or not to continue the unfold---is that we can specify, *in each field of the functor f*, whether we want to continue the unfold or not. `apo` gave us a binary switch---either stop the unfold with a `Left` or keep going with a `Right`. `futu`, by contrast, lets us build out as many layers at a time as we desire, giving us the freedom to manually specify the shape of the structure or relegate its shape to future invocations of the unfold.

This is an interesting way to encode unfolds! A CV-coalgebra that always returns a `Continue` value will loop infinitely, such as the unfold that generates all natural numbers. This means that we can tell, visually, whether our unfold is infinite or terminating.

"But Patrick," you might say, "this looks like a cellular automaton." And you would be right---CV-coalgebras describe tree automata. And in turn, coalgebras describe finite-state automata, and R-coalgebras describe stream automata. We'll use this fact to define an example CV-coalgebra, one that grows[^9] random plant life.

### Horticulture with Futumorphisms

Let's start by defining the various parts of a plant.

``` {.sourceCode .literate .haskell}
data Plant a
  = Root a     -- every plant starts here
  | Stalk a    -- and continues upwards
  | Fork a a a -- but can trifurcate at any moment
  | Bloom      -- eventually terminating in a flower
    deriving (Show, Functor)
```

Let's define a few rules for how a plant is generated. (These should, as I mentioned above, remind us of the rules for tree automata.)

    1. Plants begin at the ground.
    2. Every plant has a maximum height of 10.
    3. Plants choose randomly whether to fork, grow, or bloom.
    4. Every fork will contain one immediate bloom and two further stems.

Rather than using integers to decide what action to take, which can get obscure very quickly, let's define another sum type, one that determines the next step in the growth of the plant.

``` {.sourceCode .literate .haskell}
data Action
  = Flower  -- stop growing now
  | Upwards -- grow up with a Stalk
  | Branch  -- grow up with a Fork
```

Because we need to keep track of the total height and a random number generator to provide randomness, we'll unfold using a data type containing an `Int` to track the height and a `StdGen` generator from `System.Random`.

``` {.sourceCode .literate .haskell}
data Seed = Seed
    { height :: Int
    , rng    :: Random.StdGen
    }
```

We'll define a function `grow` that takes a seed and returns both an randomly-chosen action and two new seeds. We'll generate an action by choosing a random number from 1 to 5: if it's 1 then we'll choose to `Flower`, if it's 2 we'll choose to `Branch`, and otherwise we'll choose to grow `Upwards`. (Feel free to change these values around and see the difference in the generated plants.) The `Int` determining the height of the plant is incremented every time `grow` is called.

``` {.sourceCode .literate .haskell}
grow :: Seed -> (Action, Seed, Seed)
grow seed@(Seed h rand) = (choose choice, left { height = h + 1}, right { height = h + 1})
  where (choice, _) = Random.randomR (1 :: Int, 5) rand
        (leftR, rightR) = Random.split rand
        left = Seed h leftR
        right = Seed h rightR
        choose 1 = Flower
        choose 2 = Branch
        choose _ = Upwards
```

And now we'll define a CV-coalgebra, one that takes a `Seed` and returns a `Plant` containing a `CoAttr` value.

``` {.sourceCode .literate .haskell}
sow :: CVCoalgebra Plant Seed
```

The definition falls out rather quickly. We'll start by growing a new seed, then examining the current height of the plant:

And now we'll define a CV-coalgebra, one that takes a `Seed` and returns a `Plant` containing a `CoAttr` value.

    sow :: CVCoalgebra Plant Seed

The definition falls out rather quickly. We'll start by growing a new seed, then examining the current height of the plant:

    sow seed =
      let (action, next) = grow seed
      in case (height seed) of

Since we'll start with a height value of 0, we'll begin by generating a root (rule 1). Because we want to immediately continue onwards with the unfold, we pass a `Continue` into this `Root`, giving it the subsequent seed (so that we get a new RNG value).

       0 -> Root (Continue next)

Rule 2 means that we must cap the height of the plant at 10. So let's do that:

       10 -> Bloom

Otherwise, the height is immaterial. We must consult the `action` variable to know what to do next.

       _  -> case action of

If the action is to `Flower`, then we again return a `Bloom`.

          Flower -> Bloom

If it's to grow `Upwards`, then we return a `Stalk`, with a contained `Continue` value to continue our fold at the top of that `Stalk`:

          Upwards -> Stalk (Continue next)

And now we handle the `Branch` case. Our rules dictate that one of the branches will stop immediately, and the other two will continue, after a given length of `Stalk`. So we return a `Fork` with one `Stop` and two `Continues`.

          Branch  -> Fork -- grow a stalk then continue the fold
                         (Stop (Stalk (Continue next)))
                         -- halt immediately
                         (Stop Bloom)
                          -- again, grow a stalk and continue
                         (Stop (Stalk (Continue next)))

Note how, even though we specify the construction of a `Stalk` in the first and third slots, we allow the fold to `Continue` afterwards. This is the power of the futumorphism: we can choose the future of our folds, layer by layer. This is not possible with an anamorphism or apomorphism.

Here's our full `sow` function, rewritten slightly to use one `case` statement:

``` {.sourceCode .literate .haskell}
sow seed =
  let (action, left, right) = grow seed
  in case (action, height seed) of
    (_, 0)       -> Root (Automatic left)
    (_, 10)      -> Bloom
    (Flower, _)  -> Bloom
    (Upwards, _) -> Stalk (Automatic right)
    (Branch, _)  -> Fork (Manual (Stalk (Automatic left)))
                         (Manual Bloom)
                         (Manual (Stalk (Automatic right)))
```

This is pretty remarkable. We've encoded a complex set of rules, one that involves both nondeterminism and strict layout requirements, into one CV-coalgebra, and it took just eleven lines of code. No mutable state is involved, no manual accumulation is required---the entire representation of this automaton can be reduced to one pure function.

Now, in our `main` function, we can grab an RNG from the global state, and call `futu` to generate a `Term Plant`.

    main :: IO ()
    main = do
      rnd <- newStdGen
      let ourPlant :: Term Plant
          ourPlant = futu sow (Seed 0 rnd)

Using a rendering function (which I have omitted for brevity's sake, though you can be assured that it is implemented using `cata` rather than explicit recursion), we can draw a picture of the plant we've just generated, with little flowers.

    ⚘
    | ⚘     ⚘          ⚘
    |⚘|     |          |
    └─┘     |         |
     |      |          |       ⚘
     |  ⚘   |          |       |
     └─────┘          |   ⚘   |
        |              └──────┘
        |        ⚘        |
        └───────────────┘
                 |
                 _

Admittedly, the vaguaries of [code page 437](https://en.wikipedia.org/wiki/Code_page_437) leave us with a somewhat unaesthetic result---but a nicer representation of `Plant`, perhaps using [gloss](https://hackage.haskell.org/package/gloss) or [Rasterific](https://hackage.haskell.org/package/Rasterific), is left as an exercise for the reader.

One final detail: just as we can use an apomorphism to express an anamorphism, we can express anamorphisms and apomorphisms with futumorphisms:

``` {.sourceCode .literate .haskell}
ana :: (Functor f) => Coalgebra f a -> a -> Term f
ana f = futu (fmap Automatic <<< f)

apo :: Functor f => RCoalgebra f a -> a -> Term f
apo f = futu (fmap (either termToCoattr Automatic) <<< f)
  where termToCoattr = Manual <<< fmap termToCoattr <<< out
```

### My God, It's Full of Comonads

Now we know what histomorphisms and futumorphisms are. Histomorphisms are folds that allow us to query any previous result we've computed, and futumorphisms are unfolds that allow us to determine the future course of the unfold, multiple levels at a time. But, as is so often the case with recursion schemes, these definitions touch on something deeper and more fundamental.

Here's the kicker: our above `CoAttr` definition is equivalent to the `Free` monad, and `Attr` (being dual to `CoAttr`) is the `Cofree` comonad.

We usually represent `Free`, aka `CoAttr`, as two constructors, one for pure values and one for effectful, impure values:

    data Free f a
        = Pure a
        | Impure (f (Free f a))

And we usually represent the cofree comonad with an infix constructor, since the cofree comonad is at its heart a glorified tuple:

    data Cofree f a = a :< (f (Cofree f a))

The various packages in the Haskell ecosystem implement `cata` and `para` in much the same way, but the same is not true of `histo` and `futu`. Edward Kmett's [recursion-schemes](https://hackage.haskell.org/package/recursion-schemes) package uses these definitions of `Free` and `Cofree` (from the [free](https://hackage.haskell.org/package/free) package). [`fixplate`](https://hackage.haskell.org/package/fixplate) uses a different definition of `Attr`: rather than being a data type in and of itself, it is defined as a `Term` over a more-general `Ann` type. [`compdata`](https://hackage.haskell.org/package/compdata)'s is slightly more complicated, as it leverages other typeclasses `compdata` provides to define attributes on nodes, but is at its heart the same thing. Each is equivalent.

The free monad, and its cofree comonad dual, lie at the heart of some of the most fascinating constructions in functional programming. I have neither the space nor the qualifications to provide a meaningful explanation of them, but I can enthusiastically recommend [Gabriel Gonzales](https://twitter.com/GabrielG439)'s blog post on [free monads](http://www.haskellforall.com/2012/06/you-could-have-invented-free-monads.html), [Dan Piponi](https://twitter.com/sigfpe)'s post on the [cofree comonad](http://blog.sigfpe.com/2014/05/cofree-meets-free.html), and (of course) Oleg Kiselyov's [groundbreaking work](http://okmij.org/ftp/Computation/free-monad.html) on the free and freer monads. But I think the fact that, as we explore as fundamental a construct as recursion, we encounter another similarly fundamental concept of the free monad, provide an argument for the beauty and unity of the category-theoretical approach to functional programming that is far more compelling than any I could ever make myself.

I'd like to thank Rob Rix, who was essential to this work's completion, and Colin Barrett, who has been an invaluable resource on the many occasions when I find myself stuck. I'd also like to thank Manuel Chakaravarty, who has done this entire series a great favor in checking it for accuracy, and Jeanine Adkisson, who found some outrageous bugs in the provided futumorphism. Greg Pfiel, Scott Vokes, and Josh Bohde also provided valuable feedback on drafts of this post. Mark Needham, Ian Griffiths, How Si Wei and Bryan Grounds found important bugs in the first published version of this post; I owe them a debt of gratitude. Next time, we'll explore one of the most compelling reasons to use recursion schemes---the laws that they follow---and after that, we'll discuss the constructs derived from combining unfolds with folds: the hylomorphism and the chronomorphism.

[^1]: Bob Harper, in *Practical Foundations for Programming Languages*, refers to `In` and `out` as "rolling" and "unrolling" operations. This is a useful visual metaphor: the progression `f (f (Term f)) -> f (Term f) -> Term f` indeed looks like a flat surface being rolled up, and its opposite `Term f -> f (Term f) -> f (f (Term f))` looks like the process of unrolling.

[^2]: Rob Rix [points out](https://twitter.com/rob_rix/status/793430628637274112) that, though catamorphisms are often described as "bottom-up", this term is ambiguous: catamorphisms' recursion occurs top-down, but the folded value is constructed bottom-up. I had never noticed this ambiguity before. (The words of Carroll come to mind: " 'When I use a word,' Humpty Dumpty said, in rather a scornful tone, 'it means just what I choose it to mean --- neither more nor less.' ")

[^3]: Unfortunately, in this context I think "classic" can be read as "hackneyed and unhelpful". I dislike using `fib()` to teach recursion schemes, as the resulting implementations are both more complicated than a straightforward implementation and in no way indicative of the power that recursion schemes bring to the table. Throughout this series, I've done my damnedest to pick interesting, beautiful examples, lest the reader end up with the gravely mistaken takeaway that recursion schemes aren't useful for any real-world purpose.

[^4]: A word with a rich pedigree---most directly from the Greek 'ἱστορία', meaning *a narration of what has been learned*, which in turn descended from 'ἱστορέω', *to learn through research*, and in turn from 'ἵστωρ', meaning *the one who knows* or *the expert*--- a term commensurate with the first histories being passed from person to person orally. And the Greek root 'ἱστο', according to the OED, can be translated as 'web': a suitable metaphor for the structural web of values that the `Attr` type generates and preserves.

[^5]: A feature taken wholesale, we must note, from dependently-typed languages like Agda and Idris.

[^6]: Natch.

[^7]: Keen-eyed readers will note that this data type is isomorphic to the `Maybe` type provided by the Prelude. We could've just used that, but I wanted to make the numeric nature of this structure as clear as possible.

[^8]: There is no reason why we couldn't do this---I just chose to omit it for the sake of brevity.

[^9]: which brings an amusing literalism to the term 'seed value'