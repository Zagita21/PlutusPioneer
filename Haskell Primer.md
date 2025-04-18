# Haskell Primer

## Week 2

### GHCi (REPL)

* `:i` - display the definition of a function, typeclass, or type
* `:t` - display the type of an expression
* `:k` - display the kind of a type
* `:l` - load a program (a source file)
* `:r` - reload the loaded program

### Unit

* `()` - unit type, similar to `Void` in other languages.
* The unit type can have only one value - the unit value `()` (similar to an empty tuple)

### Wildcard

`_` - wildcard character that matches any value

### Functions 

#### Declaration

```haskell
mkValidator :: Data -> Data -> Data -> ()
```

* `mkValidator` - function name
* `Data -> Data -> Data` - three parameters 
* `()` - return type
* In Haskell all functions with multiple parameters are curried, which allows [partial application](https://wiki.haskell.org/Partial_application)

#### Definition

```haskell
mkValidator _ _ _ = ()
```

* `mkValidator` - function name
* `_ _ _` - three arguments (ignored with wildcards)
* `()` - return value

### Type variables

```haskell
error :: () -> a
```

* `a` - any type, similar to generics in other languages

### Data types

Haskell data types are similar to `struct` and value types in other languages.

#### Data

```haskell
data Data =
      Constr Integer [Data]
    | Map [(Data, Data)]
    | List [Data]
    | I Integer
    | B BS.ByteString
    deriving stock (Show, Eq, Ord, Generic)
    deriving anyclass (NFData)
```

* `data` - define a new data type
* `=` - everything after `=` are constructors
* `Constr` - constructor name
* `Integer [Data]` - constructor parameters
*  `|` - separates multiple constructors
* `Map`, `List`, `I`, `B` - other constructor names
* `deriving` - see [[#Typeclasses]]

#### Newtype

```haskell
newtype MySillyRedeemer = MySillyRedeemer Integer
```

* `newtype` - special case of `data` declaration that permits only one constructor and only one parameter
* `MySillyRedeemer` - name
* `MySillyRedeemer Integer` - constructor

#### Type

```haskell
type POSIXTimeRange = Interval POSIXTime
```

* `type` - define a type synonym (i.e., alias) for the specified data type
* `Interval` - data type declared as `Interval a`
* `POSIXTime` - type argument for `a` in `Interval` declaration

### Typeclasses

Haskell `typeclass` (or the shorthand `class`) is similar to `interface`, `protocol` or `trait` in other languages.

```haskell
class ValidatorTypes (a :: Type) where
    type RedeemerType a :: Type
    type DatumType a :: Type
```

* `class` - declare a new typeclass
* `ValidatorTypes` - typeclass name
* `(a :: Type)` - type variable
* `type` - declare an associated type synonym (see [type families](https://wiki.haskell.org/GHC/Type_families))
* `RedeemerType` - type synonym name

#### Instance

```haskell
data Typed
instance Scripts.ValidatorTypes Typed where
	type instance DatumType Typed = ()
	type instance RedeemerType Typed = Integer
```

* `instance` - declare the data type as an instance of the specified typeclass, similar to `interface` or `protocol` implementation in other languages.
* `Scripts.ValidatorTypes` - typeclass
* `Typed` - type the instance is being provided for
* `where` - terminates the initial declaration and follows up with the implementation

#### Deriving

```haskell
data Data =
      Constr Integer [Data]
    | Map [(Data, Data)]
    | List [Data]
    | I Integer
    | B BS.ByteString
    deriving stock (Show, Eq, Ord, Generic)
    deriving anyclass (NFData)
```

* `deriving`  - automatically make this type an instance of the specified typeclasses
* `stock, anyclass` - [deriving strategies](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/deriving_strategies.html?highlight=deriving%20strategies)
* `(Show, Eq, Ord, Generic)` - typeclasses to be derived

### Guards

```haskell
mkValidator :: Data -> Data -> Data -> ()
mkValidator _ r _
    | r == I 42 = ()
    | otherwise = traceError "wrong redeemer"
```

* `|` - guard case
* `r == I 42` - expression used to test this case (must evaluate to `Bool`)
* `= ()` - expression to be evaluated and returned, if the test is `True`
* `otherwise` - default case which always evaluates to `True`
* Guards without pattern matching are similar to `if-else` statements in other languages.
* Guard with pattern matching are similar to `switch` statements in other languages.

### Where

```haskell
typedValidator :: Scripts.TypedValidator Typed
typedValidator = Scripts.mkTypedValidator @Typed
    $$(PlutusTx.compile [|| mkValidator ||])
    $$(PlutusTx.compile [|| wrap ||])
  where
    wrap = Scripts.wrapValidator @() @Integer
```

* `where` - allows to define local functions and constants
* `where` definitions are bound to the current function 

### Function application operator

```haskell
traceIfFalse "wrong redeemer" $ r == 42
```

Equivalent to:

```haskell
traceIfFalse "wrong redeemer" (r == 42)
```

* `$` is an infix version of function application 
* `($) :: (a -> b) -> a -> b`
* `$` allows to avoid parentheses by separating expressions and giving *precedence* to anything after it. 
* `$` has right *associativity* and the lowest *precedence* possible - 0, whereas normal function application is left *associative* and has the highest *precedence* possible - 10.

### Pragmas

```haskell
{-# INLINABLE mkValidator #-}`
```

* `{-# ... #-}` - [pragmas](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/pragmas.html) are instructions to the compiler (they affect the generated code)

### Template Haskell

[Template Haskell](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/template_haskell.html) is used for compile-time metaprogramming.

```haskell
$$(PlutusTx.compile [|| mkValidator ||])
```

* AST - Abstract Syntax Tree
* `[|| ... ||]` - typed expression quotation, quotes the expression and produces the AST of it (`Q Exp`)
* `$$(...)` - typed expression splice, converts from the AST to the expression

```haskell
PlutusTx.unstableMakeIsData ''MySillyRedeemer
```

* `''` - quote a type name, `''MySillyRedeemer` has type `Name`.
* `Name` is used to construct Template Haskell expressions, patterns, declaration, etc.

## Week 3

### Infix syntax

```haskell
$$(PlutusTx.compile [|| mkValidator ||]) `PlutusTx.applyCode` PlutusTx.liftCode p
```

Equivalent to:

```haskell
PlutusTx.applyCode $$(PlutusTx.compile [|| mkValidator ||]) (PlutusTx.liftCode p)
```

* `` `PlutusTx.applyCode` `` - infix function application

### Composition operator

```haskell
validator = Scripts.validatorScript . typedValidator
```

Equivalent to:

```haskell
validator p = Scripts.validatorScript $ typedValidator p
```

* `(.) :: (b -> c) -> (a -> b) -> a -> c` - given a function `b` to `c` and a function `a` to `b`, return a function `a` to `c`.
* `(f . g) x = f (g x)` - where `f` corresponds to `b -> c` and `g` to `a -> b`.
* `.` is an infix operator with right *associativity* and the *precedence* of 9 (out of 10) (normal function application is left *associative* and has the highest *precedence* possible - 10; therefore, composed functions should be enclosed in parentheses for application or applied with `$` operator - `(negate . sum) [1, 2, 3]` or `negate . sum $ [1,2,3]`).

## Week 4

### Typeclass constraints

```haskell
fmap :: Functor f => (a -> b) -> f a -> f b
```

* `Functor f` before `=>` is a typeclass constraint, which restricts the type of `f` to instances of the `Functor` typeclass.

### Functor

`Functor` is a typeclass that declares `fmap` function. 

The idea of a functor is to bring the convenience of `map` to non-collection types. In the same way as `map` function allows to transform the contents of a collection, `fmap` function can transform the contents of any container that implements `Functor` typeclass.

Functor laws:

* Identity: `fmap id == id`
* Composition: `fmap (f . g) == fmap f . fmap g`

#### Fmap function

```haskell
> fmap (map toUpper) getLine
Haskell
"HASKELL"
```

* `fmap` - similar to `map`, but applied to functors.
* `fmap :: Functor f => (a -> b) -> f a -> f b`
	* `(a -> b)` - transformation function from `a` to `b`
	* `f a` - functor that contains `a` 
	* `f b` - resulting functor that will contain `b` after applying the transformation function `(a -> b)`

#### Fmap operator

* `<$>` - infix version of `fmap` function
* `(<$>) :: Functor f => (a -> b) -> f a -> f b`
* `$` in the name alludes to [[#Function application operator]]

### Applicative

`Applicative` is a typeclass that declares `pure` function and `<*>` operator. Any `Applicative` instance is also `Functor` instance.

The idea of an applicative functor is to allow to apply a transformation function contained in one container to the contents of another container. The difference between `fmap` (`<$>`) and `<*>` is that `(a -> b)` transformation function is now inside a container.

#### Pure function

`pure` - lifts (i.e., wraps) a type into the applicative structure
`pure :: Applicative f => a -> f a`

#### Sequential application operator

```haskell
> pure (+1) <*> [1..3]
[2,3,4]
```

Equivalent to:

```haskell
> (+1) <$> [1, 2, 3]
[2,3,4]
> fmap (+1) [1, 2, 3]
[2,3,4]
```

* `<*>` - similar to `fmap`, but transformation function is inside an applicative functor
* `<*> :: Applicative f => f (a -> b) -> f a -> f b`
	* `f (a -> b)` - applicative functor that contains a transformation function from `a` to `b`
	* `f a` - applicative functor which contents will transformed
	* `f b` - resulting applicative functor

### Monad

`Monad` is a typeclass that declares `>>` and `>>=` operators. Any `Monad` instance is also `Applicative` and `Functor` instance.

The idea of a monad is to avoid unwrapping values inside a container before passing them to a function that takes such values and returns them after transformation inside a container of the same type.

#### Return function

* `return` - lifts a type into the monad structure, similar to `pure` function of `Applicative`.
* `return :: Monad m => a -> m a` 

#### Sequencing operator

```haskell
> putStrln "Hello" >> putStrln "World"
Hello
World
```

* `>>` - sequences two actions and discards the result of the first action
* `(>>) :: Monad m => m a -> m b -> m b`

#### Bind operator

```haskell
> getLine >>= putStrLn
Haskell
Haskell
```

* `>>=` is similar to `flatMap` function in other languages.
*  `>>=` - binds a monad containing value `a` to a function that takes value `a` and returns it transformed to `b` inside a monad of the same type.
* `(>>=) :: Monad m => m a -> (a -> m b) -> m b`

### IO

```haskell
main :: IO ()
main = putStrln "Hello World!"
```

The idea of `IO` is to have a container in which interaction with real world can happen, whereas everything outside `IO` can stay pure (i.e., side effect free).

* `IO` is a type that has `Monad` instance
* `IO` execution is allowed to have side effects
* `IO` can be executed in `main` or REPL.

### Maybe

```haskell
> readMaybe "42" :: Maybe Int
```

* `Maybe` is a data type similar to `Optional` in other languages.
* `Maybe` has `Monad` instance
* `data Maybe a = Nothing | Just a` 

### Either

```haskell
readEither :: Read a => String -> Either String a
readEither s = case readMaybe s of
	Nothing -> Left $ "can't parse: " ++ s
	Just a -> Right a
```

* `Either` is a data type similar to `Result` in other languages, but more generic.
* `Either` has `Monad` instance
* `data Either a b = Left a | Right a` 

### Anonymous functions

```haskell
bar :: IO ()
bar = getLine >>= \s -> getLine >>= \t -> putStrLn (s ++ t)
```

* `\s` - is a parameter of the anonymous function
* `\` - is an allusion to *λ* symbol in lambda abstractions

### Case expression

```haskell
foo :: String -> String -> String -> Maybe Int
foo x y z = case readMaybe x of
	Nothing -> Nothing
	Just k -> case readMaybe y of
		Nothing -> Nothing
		Just l -> case readMaybe z of
		 	Nothing -> Nothing
			Just m -> Just (k + l + m)
```

* Similar to pattern matching using `switch` expression in other languages.
* `case readMaybe x of` - `readMaybe x`  is the expression to be matched.
* `Nothing -> Nothing` - pattern to match `Nothing` in `Maybe` and return `Nothing`.
* `Just k -> case readMaybe y of` - pattern to match value `k` in `Maybe` and return another case expression.

### Pattern matching

```haskell
bindMaybe :: Maybe a -> (a -> Maybe b) -> Maybe b
bindMaybe Nothing  _ = Nothing
bindMaybe (Just x) f = f x
```

* Functions can have separate function definitions for different patterns.
* `bindMaybe Nothing _` - define a function body for the case when the first argument is matching `Nothing`.
* `bindMaybe (Just x) f` - define a function body for the case when the first argument is matching `Just x` (`Maybe` contains value `x`).

### Let

```haskell
foo :: Writer Int -> Writer Int -> Writer Int -> Writer Int
foo (Writer k xs) (Writer l ys) (Writer m zs) =
  let
    s = k + l + m
    Writer _ us = tell ["sum: " ++ show s]
  in
    Writer s $ xs ++ ys ++ zs ++ us
```

* `let ...` - define variables
* `in ...` - expression in which defined variables are used

### Do notation

```haskell
threeInts' :: Monad m => m Int -> m Int -> m Int -> m Int
threeInts' mx my mz = do
    k <- mx
    l <- my
    m <- mz
    let s = k + l + m
    return s
```

Equivalent to:

```haskell
threeInts :: Monad m => m Int -> m Int -> m Int -> m Int
threeInts mx my mz =
    mx >>= \k ->
    my >>= \l ->
    mz >>= \m ->
    let s = k + l + m in return s
```

* Do notation is a syntactic convenience for sequencing actions
* When using `let` inside a `do` block, `in` is not needed
* `k <- mx` - bind the result of `mx` to the variable `k`

```haskell
foo'' :: Writer Int -> Writer Int -> Writer Int -> Writer Int
foo'' x y z = do
    s <- threeInts x y z
    tell ["sum: " ++ show s]
    return s
```

* `tell ["sum: " ++ show s]` - execute action without binding it to a variable

### Monoid

`Monoid` is a typeclass that declares `mempty` and `mappend` functions. Any `Monoid` instance is also `Semigroup` instance. The most common example of a monoid is a list.

Monoid laws:

* Binary operation - operates on two values
* Associative - the order of evaluation doesn't matter
* Identity - there exists a neutral value (when combined with any other value, will return the other value) 

#### Mempty function

* `mempty :: a` - returns the identity value

#### Mappend function

* `mappend :: a -> a -> a` - joins two values together

### List

```haskell
element : list 
```

* `:` - add an element to the start of a list.

```haskell
list ++ [element]
```

* `++` - add an element to the end of a list.

```haskell
> head [1,2,3]
1
```

* `head` - return the first element.

```haskell
> tail [1,2,3]
[2,3]
```

* `tail` - return all elements without the first one.

```haskell
> init [1,2,3]
[1,2]
```

* `init` - return a new list without the last element.

```haskell
> null []
True
> null [1,2,3]
False
```

* `null` - return whether the list is empty.

### Void

* `Void` cannot have a value (whereas `Unit` has one value only), similar to `Never` in other languages.