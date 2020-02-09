---
title: "3. Extending the Language"
series: ["Writing a Parser for a Small Language"]
date: 2020-02-08T10:31:01+13:00
draft: true
tags: [antlr, kotlin, parser, code, tdd]
---

## Adding comments

Comments throughout scripts can be quite useful, for example to show intention, or link to information.

To keep things simple, comments will be anything including and after a `#` symbol - whether on their own line, or following other statements.

```stupid

# this is a comment
print "hello"; # this is also a comment
print "world"; # so is this
```

\
Just as I do for whitespace, I would like to ignore comments so that the parser rules need no modification.

Before, we had a lexer rule to send whitespace to a hidden channel:

```antlr
// Send all whitespace to a hidden channel
WS              : [\r\n\t ]+ -> channel(HIDDEN);
```

We now want to also send comments to the same hidden channel, so we'll do the following:
1. Rename the lexer rule to `STRIP`.
2. Change the `WS` rule to a fragment.
3. Define a new fragment for `COMMENT`, which is anything beginning with a hash (`#`), up until a newline but excluding the newline.

With this:

```antlr
// Send all whitespace and comments to a hidden channel
STRIP           : (WS | COMMENT) -> channel(HIDDEN);

fragment WS     : [\r\n\t ]+;
fragment COMMENT: '#'~('\r' | '\n')*;
```

The `fragment` keyword just allows us to define patterns that are tokens in their own right, but can be used in lexer rules to define tokens.

{{<notice tip>}}
If you wanted to make use of the comments, you could send them to a different channel and pick them up in the parser code.
{{</notice>}}

## Adding variable assignment

I'd like to be able to assign and reassign values to variables. For now, variables can either contain an integer, or a string. Later we might introduce other types such as floats and booleans.

For example:

```stupid

def x = 5               # Assign a number to x
def y = "hi"            # Assign a string to y
def z = -x              # Assign the negative value of x to z

repeat x {
    y = "hello world"   # reassign y
    print y
}
```

We first need to define a lexer rule for assignment:

```antlr
```

## Adding string interpolation

Now that we can assign values to variables, it would be nice to be able to interpolate those into strings.