# Let's build a calculator

- 1. [Parsing a number](#parsing-a-number) - regex, run, map
- 2. [Multiplication](#multiplication) - seq, Error, span
- 3. [Shortcuts](#shortcuts) - string, implicit conversions
- 4. [Whitespace](#whitespace) - optional, drop
- 5. [More than two numbers](#more-than-two-numbers) - deferred resolution

Let's build a parser that can model a basic 1970s pocket calculator. It should take an expression like

    3 * 10 + 7 * 11

parse it, and evaluate it, getting the answer:

    107

We'll start from the bottom up, building simple parsers and then combining them, and in the process, get a whirlwind tour of packrattle.


## Parsing a number

The first thing to do is recognize and match numbers. If you're good at regular expressions, it's a breeze.

```javascript
$ node --harmony --harmony_arrow_functions

> var packrattle = require("packrattle");
> var number = packrattle.regex(/\d+/);
> number.run("34");
[ '34', index: 0, input: '34' ]
```

If you don't know regular expressions, `\d+` means "one or more digits". That's sufficient to match an int, which is all we need for this example. You could get fancier if you want to match against floats (the javascript `Number` type).

`regex` is the packrattle function for creating a parser based on a regular expression (or "regex"). `number`, then, is a `Parser`, and to make a Parser tackle a string, you call `run`. `run` will either return a successful match, or throw an exception.

```javascript
> number.run("a")
Error: Expected /\d+/
```

Hm. So packrattle can hold a regular expression and then execute it against a string. But you can do that easily already.

```javascript
> var number = /\d+/;
> number.exec("34")
[ '34', index: 0, input: '34' ]
```

So if that was all it could do, we might as well pack up and go home. This vacation is over.

Luckily, this is just the beginning! Parser objects have a few methods that allow us to transform the results. This is how a compiler can turn parser output into an AST, and how we can evaluate expressions as the parser works. For this parser, we want to turn the match object from the regex into a javascript `Number`, and we can get that by adding a transform with `map`.

```javascript
> var number = packrattle.regex(/\d+/).map(match => parseInt(match[0], 10));
> number.run("34")
34
```

Okay, that's a little bit cooler. `map` will call a function on a successful match, letting us change the result. In this case, we take the matched string and parse it immediately into an int. Now `number` is sort of a glorified wrapper for `parseInt` that rejects anything but positive integers.

We need to go deeper.


## Multiplication

The precedence rules of algebra say that multiplication takes priority over addition, so let's handle that case next. We want to string together a few small parsers in a sequence, like this:

```
multiply ::= number "*" number
```

which is BNF syntax for saying that a multiply operation is a number followed by a star, followed by another number. In packrattle, this is a "sequence" or `seq`.

```javascript
> var multiply = packrattle.seq(number, packrattle.string("*"), number);
> multiply.run("3*4")
[ 3, '*', 4 ]
```

As you can see, `seq` takes a list of parsers and joins them together. We didn't have to explain how to parse numbers again, either; we can just use the parser we stored in `number`. Being able to combine the parsers by name this way will help a lot as the parsers and combinations get more complex.

This new "sequenced" parser returns an array of the match results, in order. But it only succeeds if each of the inner parsers succeeds.

```javascript
> multiply.run("3@4")
Error: Expected '*'
```

That error message is correct, but unhelpful. There's more information in the `Error` object if you dig in.

```javascript
> try { multiply.run("3@4") } catch (error) { console.log(util.inspect(error)) }
{ [Error: Expected '*']
  span: { text: '3@4', start: 1, end: 2, ... } }
```

The start position (1) tells us that the error occurred at offset 1 in the string: the "@". In fact, if you're wearing fancy pants today, you can ask packrattle to show you where the parser hurt you:

```javascript
> try {
...   multiply.run("3@4")
... } catch (error) {
...   console.log(error.message);
...   error.span.toSquiggles().forEach(line => console.log(line));
... }
Expected '*'
3@4
 ~
```


## Shortcuts

You may have noticed that the "\*" parser used a new building block.

```javascript
packrattle.string("*")
```

The `string` parser is even simpler than `regex`: it matches only the exact string requested.

Because strings, regular expressions, and sequences are used all the time, shortcuts are begrudgingly allowed. Any time packrattle expects a nested parser, it will accept a string, RegExp object, or array, and implicitly convert them. (An array becomes a sequence.) So we could have defined `multiply` as:

```javascript
> var multiply = packrattle.seq(number, "*", number);
```

The packrattle module is also a function that will do these implicit conversions for you, so we could also write it like this:

```javascript
> var multiply = packrattle([ number, "*", number ]);
```


## Whitespace

It would be nice if we could add whitespace around the terms.

```javascript
> multiply.run("3 * 4")
Error: Expected '*'
```

A good regex for whitespace might be `/[ \t]+/`, or "one or more space or tab characters". But we don't care if the whitespace is there or not -- it's optional. There's a parser for that.

```javascript
> var whitespace = packrattle(/[ \t]+/).optional();
> var multiply = packrattle([ number, whitespace, "*", whitespace, number ]);
> multiply.run("3 * 4")
[ 3,
  [ ' ', index: 0, input: ' * 4' ],
  '*',
  [ ' ', index: 0, input: ' 4' ],
  4 ]
```  

Hey, pretty good! But also, we don't care about the results of the whitespace. If it's there, we just want to ignore it. So let's `drop` it.

```javascript
> var whitespace = packrattle(/[ \t]+/).optional().drop();
> var multiply = packrattle([ number, whitespace, "*", whitespace, number ]);
> multiply.run("3 * 4")
[ 3, '*', 4 ]
```

Nice! Good job, everyone!

In fact, we don't even need the `*` either. We just care about the numbers. Let's pull it out into its own parser, to make it easier to read, and make `multiply` actually do the math.

```javascript
> var star = packrattle([ whitespace, "*", whitespace ]).drop();
> var multiply = packrattle([ number, star, number ]).map(match => match[0] * match[1]);
> multiply.run("3 * 4")
12
```


## More than two numbers

We're at a crossroads now. What if we want to compute _three_ numbers multiplied?

    4 * 5 * 7

One way to handle this is through recursion. Each side of a multiplication could itself be another multiplication. We'll call that a "factor".

```javascript
> var factor = number.or(multiply);
```

But, uh... `factor` refers to `multiply`, and our new definition of `multiply` is going to need to refer to `factor`. How can that work? It's a loop!

Packrattle will let us pass in a function wherever a parser is expected, to let us "defer" resolving the parsers until they're all defined. The first time we execute a parser, it walks the tree, looking for functions and calling them, to build up the real tree, which may contain loops. So we can use functions to create this loop: `factor` will be either a number or a (deferred reference to) `multiply`.

```javascript
> var factor = number.or(() => multiply);
> var star = packrattle([ whitespace, "*", whitespace ]).drop();
> var multiply = packrattle([ factor, star, factor ]).map(match => match[0] * match[1]);
> multiply.run("3 * 4")
12
> multiply.run("4 * 5 * 7")
140
```

If you're familiar with parsers, your head may have just spun around. The parser we just built is "left-recursive", meaning that the left side of the expression for `multiply` is `factor star factor`, and `factor` is `number | multiply`, so most parser engines will go navel-gazing immediately and never return. In these engines, you need to carefully arrange the parsers so that they can only recurse on the right side, like this:

```javascript
> var multiply = packrattle([ number, star, factor ]).map(match => match[0] * match[1]);
> multiply.run("4 * 5 * 7")
140
```

You don't need to do this in a GLL engine because it effectively walks branches in parallel, memoizing loops. If this interests you, there are some papers listed at the end of packrattle's README.



----------


If you're parsing into a syntax tree, you may want to preserve the span so you can highlight errors later. For example:

```javascript
const number = packrattle.regex(/\d+/).map((match, span) => {
  return { number: parseInt(match[0], 10), span: span };
});

// later:
console.log("Everything went wrong here:");
badMatch.span.toSquiggles().forEach(line => console.log(line));
```

The result of the parser for `number` will be an object with a `number` field set to the matched value, and a `span` that covers the matching text. You can refer to it later when it turns out that number was a bad seed.



- `value`: The result of the previous parser. For simple parsers like `string` and `regex`, this will be the string literal or regex match object, respectively. For nested parsers with their own `onMatch` transforms, the parameter will be the object returned by that parser. For example, the `seq` combinator (below) returns an array of the sequence of matches. An expression parser might build up a tree of expression nodes.


-----

### Reduce

The reduce method is borrowed from scala's parser-combinator library, and is particularly useful for parsing expression trees.

- `reduce(p, separator = "", options)`

Options are:
- `min` - minimum number of matches of 'p' allowed (default: 1)
- `max` - maximun number of matches of 'p' allowed (default: Infinity)
- `first` - one-argument function to transform the initial match value (default: `x => [ x ]`)
- `next` - three-argument function to combine successive matches (default: `(sum, sep, x) => sum.push(x)`)

Like 'repeatSeparated', it attempts to match at least one 'p', separated by 'separator'. In standard grammar, it matches:

    p (separator p)*

with an optional limit on the minimum or maximum number of 'p' there can be. Two functions are called to transform the match results:

- `first(value)` is called with the first result of 'p' and can be used to transform the result, just like 'onMatch'. The default accumulator creates a new array with the match result as its only element.

- `next(total, separator, value)` is called for each subsequent match of a separator and 'p'. The first parameter is the total result so far (or the result of the 'first' function). The second is the result of the separator, and the last is the result of the current 'p'. This function should return the new 'total' that will be passed in on future matches.