+++
title = "Subset of JSON parser in 150 lines of Haskell code"
description = "??"
date = "2024-05-10"
tags = ["haskell", "parsing"]
+++

## Intro

Recently I have started to learn Haskell at my spare time by solving [codewars](https://www.codewars.com/dashboard) problems
time-to-time. This post is about my solution for [this kata](https://www.codewars.com/kata/55aa170b54c32468c30000a9), since I was
impressed by power of applicative parsing and want to document things I have learned during solving.

NOTE: I am no way an expert in Haskell or FP languages, so my solution might be not the best/cleanest/etc.

## The task

The task is simple: implement a parser for a subset of JSON. The only big difference from real json is no support for exponential
numbers and unicode characters.

Language task suggest to implement has following (BNF)[https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form]:

```
string ::= "" | "chars"

chars  ::= char | char chars

char   ::= <ASCII character except for ">

number ::= int | int frac | '-' number

int    ::= digits
frac   ::= '.' digits

digits ::= digit | digit digits

digit  ::= '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9' | '0'

object  ::= "{}" | '{' members '}'

members ::= pair | pair ',' members

pair    ::= string ':' value

array   ::= "[]" | '[' elements ']'

elements ::= value | value ',' elements

value = string | number | object | array | "true" | "false" | "null"
```

White-space characters are ignored.

And it's required to implement function `f :: String -> Value`, where `Value` defined as follows:

```haskell
data Value
  = String String
  | Number Double
  | Object [(Value, Value)] -- an association list -- only a `String` is valid as the index `Value`
  | Array [Value] -- not limited to identical primitive datatypes
  | Boolean Bool -- either `True` or `False`
  | Null
  deriving (Show)
```

## Applicative Parser

Parser by its nature is a function that takes an input string and produces some value, leaving unparsed string untouched. Based on this
parser type is defined as following:

```haskell
newtype Parser a = Parser {runParser :: String -> Maybe (a, String)}
```

`runParser` is a function that consumes part of the input string and maybe produces pair of value of type `a` and rest of the unparsed string. In our case generic type
`a` will be a `Value` introduced in previous paragraph.

### Basic Parsing

Using type introduced above it's possible to start parsing right away. The easiest thing to implement is to parser for character by predicate

```haskell
parseCond :: (Char -> Bool) -> Parser Char
parseCond fn = Parser helper
  where
    helper [] = Nothing
    helper (x : xs)
      | fn x = Just (x, xs)
      | otherwise = Nothing
```

Which can be used like

```haskell
λ> runParser (parseCond (== 'a')) "abcd"
Just ('a',"bcd")
```

As you can see, parser consumed one char and returned rest of the string unchanged. If predicate return `False` then parsing fails:

```haskell
λ> runParser (parseCond (== 'a')) "bda"
Nothing
```

Since it will be required to parse single characters a lot, let's define simple wrapper for parsing one character:

```haskell
parseChar :: Char -> Parser Char
parseChar x = parseCond (== x)
```

Next thing it's required to implement is string parser. A function to parse a string has type `f :: [Char] -> Parser [Char]` and we already have `Parser Char`. 
There should be a way to convert one to another! Hoogle points out there a `traverse` function, which has following signature:

```haskell
class (Functor t, Foldable t) => Traversable t where
  traverse :: Applicative f => (a -> f b) -> t a -> f (t b)
```

Since strings are just list of `Char`, and list implements `Traversable`, this function does exactly what we want. If we change `t` to `[]`, `f` to `Parser` and
`a` and `b` to `Char`:

```haskell
  traverse :: Applicative Parser => (Char -> Parser Char) -> [Char] -> Parser ([Char])
```

Bingo! There is one small problem left -- `Parser` should implement `Applicative`.

### Making parser applicative

Let's look what `Applicative` is. Ghci says:

```haskell
λ> :info Applicative 
type Applicative :: (* -> *) -> Constraint
class Functor f => Applicative f where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
  GHC.Base.liftA2 :: (a -> b -> c) -> f a -> f b -> f c
  (*>) :: f a -> f b -> f b
  (<*) :: f a -> f b -> f a
  {-# MINIMAL pure, ((<*>) | liftA2) #-}
```

To `Parser` type to be applicative it should be a `Functor`. Ok, but what is `Functor`?

```haskell
type Functor :: (* -> *) -> Constraint
class Functor f where
  fmap :: (a -> b) -> f a -> f b
  (<$) :: a -> f b -> f a
  {-# MINIMAL fmap #-}
```

To be a `Functor` instance, `Parser` type should implement one function called `fmap`. Let's do it!

#### Functor

To better understand what specific typeclasse does, I like to see what other types implement it and play a bit with them. Lucky me, `Maybe` implements
`Functor`.

`fmap` has following signature `fmap :: (a -> b) -> f a -> f b`, which means it "injects" a function inside of a container, that contains `a` and produces
new container that contains `b`. As an example for `Maybe`:

```haskell
λ> fmap (+ 1) (Just 1)
Just 2
λ> fmap (+ 1) (Nothing)
Nothing
λ> fmap ord (Just 'a')
Just 97
```

If you are familiar with Rust, `fmap` is like [`and_then()`](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then) for Options.
For our `Parser` type it means, that it's will be possible to change produced value to another one in case of successful parsing.

`Functor` instance looks like:

```haskell
instance Functor Parser where
  fmap fn (Parser f) =
    Parser
      ( \x ->
          case f x of
            Just (val, other) -> Just (fn val, other)
            Nothing -> Nothing
      )
```

Also it can be rewritten more cleanly in `do` notation, since `Maybe` implement `Monad`

```haskell
instance Functor Parser where
  fmap fn (Parser f) =
    Parser
      ( \x ->
          do
            (v, other) <- f x
            return (fn v, other)
      )
```

Now it's possible to do following:

```haskell
λ> runParser (fmap ord (parseChar 'a')) "abc"
Just (97,"bc")
```

`ord` was "injected" into `Parser` to produce ASCII number out of parsed 'a'. Also note that fancy looking `<$>` operator is an alias for `fmap`. Haskell people
like using esoteric symbols for operators for some reason.

#### Applicative

Since `Parser` now implements `Functor`, it's possible to implement `Applicative` instance for it. Recap that it's requied to implement 2 following
functions

```
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
```

Let's start with `pure`. This function just "wraps" value into container. For our parser it means not parsing anything and just returning passed value.

```haskell
pure val = Parser (Just . (val,))
```

Next thing is more complex. `<*>` accepts function wrapped into container and applies it to another container, that holds a value. To better understand what's going on, that's
how it works with `Maybe`:

```haskell
λ> Just (+ 1) <*> Just 2
Just 3
λ> Just (+ 1) <*> Nothing
Nothing
```

Not scary at all: it's just `fmap`, but function is wrapped into container. Let's implement using `do` notation right away, since `case of` spaghetti would be unreadable. 

```haskell
  (Parser fn) <*> (Parser f) =
    Parser
      ( \x -> do
          (fn', other) <- fn x
          (v, other') <- f other
          return (fn' v, other')
      )
```

This gives our parser very powerful property -- chaining. Now it's possible to parse strings:

```haskell
λ> runParser  (fmap (\old a b -> old ++ [a] ++ [b]) (pure "" :: Parser String) <*> parseChar 'a' <*> parseChar 'b') "abc"
Just ("ab","c")
```

Using this kind of construction for parsing string would be really annoying, but this problem is already solved by `traverse` function, that was
mentioned before! So it's possible to define string parser as follows:

```haskell
parseString :: String -> Parser String
parseString = traverse parseChar
```

And test it:

```haskell
λ> runParser (parseString "hello") "hello, world!"
Just ("hello",", world!")
```

Note that applicative gives up another 2 important operators: `<*` and `*>`. I understand them as follows "chain 2 applicatives,
but pick the result of the one the arrow is pointing to.

As an example:

```haskell
λ> runParser (parseString "hello" <* parseString "world") "helloworld"
Just ("hello","")
λ> runParser (parseString "hello" *> parseString "world") "helloworld"
Just ("world","")
```

We will need it for skipping white-spaces, for instance.

#### Alternative

The only thing left is to implement `Alternative` instance. Imagine parsing bool value in json. It could be represented via "true" or "false". Current
implementation of parser cannot choose between different values, since it parses until it finds mismatch of EOF.

`Alternative` is a way to choose between 2 applicatives.

```haskell
class Applicative f => Alternative f where
  empty :: f a
  (<|>) :: f a -> f a -> f a
```

Operator `<|>` takes two applicatives and chooses one of them. For our parser this means we can try one and
if it fails we fall back to another one:

```haskell
instance Alternative Parser where
  empty = Parser (const Nothing)
  (Parser fn) <|> (Parser fn') =
    Parser
      ( \x ->
          case fn x of
            Nothing -> fn' x
            Just e -> Just e
      )
```

Now it's possible to do something like this:

```haskell
λ> runParser (parseString "hello" <|> parseString "world") "world hello"
Just ("world"," hello")
```

## Parsing Json

### Skipping spaces

Since Json is not space-sensitive format, parser should not take them into account. To skip them we need
a parser that just reads all spaces.

```haskell
skipSpace :: Parser String
skipSpace = parseWhile (parseChar ' ') <|> pure ""
```

There is a function `parseWhile` which I did not mention, but it is a parser that runs another parser
until it does not return an error. Just in case there is spaces at all, there is `<|> pure ""` fallback
which just returns an empty string, indicating that there were no spaces in input string.

### Null

The easiest one to parse is `Null` value. `Null` value is represented as simple "null" string.

```haskell
parseNull :: Parser Value
parseNull = const Null <$> parseString "null"
```

Null parser just parses "null" string and `fmaps` result to `Null` Json value.

### Boolean

Boolean is represented with either "true" or "false" strings. We can use `Alternative` property of our
parser to choose between "true" and "false" values.

```haskell
parseBool :: Parser Value
parseBool = (\x -> if x == "true" then Boolean True else Boolean False) <$> (parseString "true" <|> parseString "false")
```

### Number

Based on BNF grammar from the beginning of the article, there are 2 types of number: integers and doubles.
Let's look at their parsers implementation:

Integer parsing is quite straightforward: just read characters until they are numbers.

```haskell
    intParser :: Parser String
    intParser = parseWhile (parseCond isNumber)
```

Double parser is a bit tricky, but idea is simple. The idea is to look at the floating point number as two integers,
separated by '.' character:

```haskell
    doubleParser :: Parser String
    doubleParser = fmap concat intParser <*> parseString "." <*> intParser
```

And to handle negative numbers we need another parser that just tries to read '-' and then double or integer:

```haskell
    negativeInt :: Parser String
    negativeInt = fmap (++) (parseString "-") <*> (doubleParser <|> intParser)
```

Equipped with these 3 parsers, it's quite easy to write generic parser for any number: just chain them all with `<|>` a
pass result to `read` function which converts string to a number.

The whole number parsing function is shown below:

```haskell
parseInt :: Parser Value
parseInt =
  Number . read <$> (skipSpace *> (doubleParser <|> intParserNotFromZero <|> negativeInt))
  where
    intParser, doubleParser, negativeInt :: Parser String

    negativeInt = fmap (++) (parseString "-") <*> (doubleParser <|> intParser)
    intParserNotFromZero = test *> parseWhile (parseCond isNumber)
    intParser = parseWhile (parseCond isNumber)
    doubleParser = fmap concat intParser <*> parseString "." <*> intParser
    concat a b c = a ++ b ++ c
```

### String

In Json strings are represented as string in quotation marks. So parser should parse quotation mark and read everthing
until the next quotation mark:

```haskell
parseStringValue :: Parser Value
parseStringValue = String <$> (parseChar '\"' *> (parseWhile (parseCond (/= '\"')) <|> pure "") <* parseChar '\"')
```

Here we use `*>` and `<*` just to ignore parsed quotation marks, since the are not part of the string itself.

### Array

Arrays in Json are untyped, so they could contain any types (like tuples in statically typed languages). Array are
represented as list of valid Json object separated by ',' in '[]' brackets.

```haskell
parseArray :: Parser Value
parseArray =
  Array
    <$> ( parseChar '['
            *> ((parseSep (skipSpace *> parseJson <* skipSpace) (parseString ",")) <|> pure [])
            <* parseChar ']'
        )
```

`parseSep` is another helper parser (like `parseUntil`) that adapts 2 parsers: one for a value and one for the separator and
returns a list of parsed objects.

### Object

Json object is a pairs like collection of pair `"name" : <Json>`, wrapped with curly braces, where `Json` is a valid json object.

First of all, let's define pair parser, which will consume string, ':' and any valid json object.

```haskell
parsePair :: Parser (Value, Value)
parsePair =
  Parser
    ( \x -> do
        (str, other) <- runParser parseStringValue x
        (val, other') <- runParser (skipSpace *> parseChar ':' *> skipSpace *> parseJson) other
        return ((str, val), other')
    )
```

I am using `do` notation to make code readable, but `<*>` could be used here as well. Then parsing whole Json object is
trivial. We need to parse curly braces and parse pairs separated by ','.

```haskell
parseObject :: Parser Value
parseObject =
  Object
    <$> ( parseChar '{'
            *> ((parseSep (skipSpace *> parsePair <* skipSpace) (parseString ",")) <|> pure [])
            <* parseChar '}'
        )

```

## Conclusion

I was very impressed that whole parser is only 150 LoC. I am new to FP world, so I guess, it could be done even with less code,
but at the end I learned a lot about Haskell types and stuff.

The whole code could be found [here](https://github.com/pskrgag/haskell_training/blob/master/4kuy/json.hs)
