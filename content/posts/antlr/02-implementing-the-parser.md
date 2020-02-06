---
title: "2. Implementing the Parser"
series: ["Writing a Parser for a Small Language"]
date: 2020-01-22T15:22:22+13:00
draft: true
tags: [antlr, kotlin, parser, code, tdd]
---

Now that we have the gernerated code, and have verified the parse tree, we can now move on to writing code to interpret out StupidLang script.

There are two ways to implement the parser from the generated code:

- The Listener Pattern: _Optionally override listeners for each rule, and as the parser parses the token stream, the applicable Listener will be called._
- The Visitor Pattern: _Optionally override the Base Visitor for each rule, and starting with the first rule, control which branch of the tre to visit next._

I'll be implementing the Visitor pattern as I'll need control over when to visit children nodes later on when we implement branching (if/else). If you were writing a code translation tool, you might prefer the Lstener pattern and let ANTLR call then listeners.

{{< notice tip >}}
Because the Visitor pattern gives you complete control over the parse tree, you must override methods for all rules up to the root. You cannot simply override methods you are interested in.
{{< /notice >}}

I'll be taking a test-driven-development approach so that I can writes tests for each unit without having to worry about implementing the the parser from the `file` entry point down to each leaf. This allows me to implement something simple like `print` then climibing my way to parsing a whole file.

{{< notice tip >}}
- IDEA code gen for antlr (right click the g4, select "Generate ANTLR Recognizer", code placed in `src/main/gen`)
- gradle modification   
- tests/tdd
{{< /notice >}}

## Setup

I'll quickly explain my setup, becuase it took a little while to figure out.

Firstly, my directory structure is as follows:

```bash
├── src/
│   ├── main/
│   │   ├── antlr/       # ANTLR g4 files
│   │   ├── gen/
│   │   |   └── antlr/   # Generated ANTLR code
│   │   └── kotlin/      # My implementation of the generated code
│   └── test/
│       └── kotlin/      # Unit Tests
├── build.gradle
└── ...
```

Secondly, I had to add a task to `build.gradle` to generate the ANTLR code for whenever I changed the grammar (g4) files.

Thirdly, I had to add the generated sources directory to the Java `main` source set, and configure my IDE to be aware of the generated source.

```gradle
// ... lines removed for brevity

// A task to regenerate the code from the grammar
// Note: I had trouble when using a package name other than "antlr"
generateGrammarSource {
    maxHeapSize = "64m"
    arguments += [
            "-visitor",
            "-no-listener",
            "-long-messages",
            "-package", "antlr"
    ]
    outputDirectory = new File("src/main/gen/antlr".toString())
}


// Required for compilation
def generatedSourcesDir = file('src/main/gen')
sourceSets.main.java.srcDir file(generatedSourcesDir)
idea {
    module {
        // Required for the IDE
        generatedSourceDirs += file(generatedSourcesDir)
        sourceDirs += file(generatedSourcesDir)
    }
}
```

## Helper functions

I wrote some helper functions in my implementation to make it easier visit specific parts of the tree. This makes testing a lot simpler.

**`SampleParser.kt`**
> Todo: refactor this into another file

```kotlin
fun getParser(code: String): StupidLangParser {
    val lexer = StupidLangLexer(CharStreams.fromString(code))
    return StupidLangParser(CommonTokenStream(lexer))
}

fun visit(context: ParserRuleContext): String {
    return StupidVisitor().visit(context)

}
```

So now I can simply load a snippet of code and retrieve a parser instance, and pick a context to visit from the parser.

## Empty implementation of the parser

The first step to implementing the Visitor is to create a class that extends the generated BaseVisitor.

**`StupidVisitor.kt`**
> Todo: rename to StupidLangImplementation.kt or something

```kotlin
class StupidVisitor : StupidLangParserBaseVisitor<String>()
```

This is the class we'll be overriding methods on for parser rules we are insterested in.

## Implement the `print` statement

I figured the `print` statement was the easiest to start with, as I can simply check that what I want to `print` results in just that.

I wrote two tests, one to test a normal usage of print, and another to ensure it still works even with an empty string.

Note the newline on the expected result - the implementation of `print` will always end with a newline.

```kotlin
class LanguageTests {
    @Test
    fun `print statement`() {
        val statement = """
            print "hello world";
        """.trimIndent()

        val parser = getParser(statement)
        val result = visit(parser.print())

        assertEquals("hello world\n", result)
    }

    @Test
    fun `print nothing`() {
        val statement = """
            print "";
        """.trimIndent()

        val parser = getParser(statement)
        val result = visit(parser.print())

        assertEquals("\n", result)
    }
}
```

The test of course fails because we haven't overriden any of the methods. Let's start by overriding the `visirPrint` method.

When visitPrint is called, we want to visit one of the children - specifically the "string" rule, as that contains the string we want to print. This means we also need to override `visitString` to return the contents of the string.

```kotlin
class StupidVisitor : StupidLangParserBaseVisitor<String>() {
    override fun visitPrint(ctx: StupidLangParser.PrintContext?): String {
        val arg = visit(ctx?.string())
        return "$arg\n"
    }

    override fun visitString(ctx: StupidLangParser.StringContext?) = ctx!!.text
}
```

Now we run the test, and indeed it passes.

> Todo: note on nullabiliy and `ctx!!`

## Implement the `repeat` Statement

The `repeat` test is quite similar to `print`.

```kotlin
    @Test
    fun `repeat statement`() {
        val statement = """
            repeat 2 {
                print "double";
            };
        """.trimIndent()

        val parser = getParser(statement)
        val result = visit(parser.repeat())

        assertEquals("double\ndouble\n", result)
    }


    @Test
    fun `empty repeat`() {
        val statement = """
            repeat 5 {};
        """.trimIndent()

        val parser = getParser(statement)
        val result = visit(parser.repeat())

        assertEquals("", result)
    }
```

However, the implementation is somewhat different. First we need to check how many times to repeat the statements in the block, then we need to repeat the statements that many times - unless the block is empty, in which case, do nothing (return an empty string).

```kotlin
    override fun visitRepeat(ctx: StupidLangParser.RepeatContext?): String {
        val times = Integer.valueOf(visit(ctx?.times()))
        return when {
            ctx?.statements() == null -> ""
            else -> visit(ctx.statements()).repeat(times)
        }
    }

    override fun visitTimes(ctx: StupidLangParser.TimesContext?) = ctx!!.text
```

## Handle multiple statements

```kotlin
    @Test
    fun `multiple statements`() {
        val statement = """
            print "hello world";
            print "goofy goober";
        """.trimIndent()

        val parser = getParser(statement)
        val result = visit(parser.statements())

        assertEquals("hello world\ngoofy goober\n", result)
    }
```

```kotlin
    override fun visitStatements(ctx: StupidLangParser.StatementsContext?): String {
        return ctx
            ?.statement()
            ?.map { visit(it) }
            ?.joinToString(separator = "") { it }
            .toString()
    }

    override fun visitStatement(ctx: StupidLangParser.StatementContext?) = visit(ctx!!.children.first())
```

## Handle a whole file of statements

```kotlin
@Test
    fun `whole file`() {
        val stupidScript = """
            repeat 5 {
                print "hello world";
            };
            
            repeat 3 {
                print "hi";
            };
            
            print "the end";
        """.trimIndent()

        val expected = """
            hello world
            hello world
            hello world
            hello world
            hello world
            hi
            hi
            hi
            the end
            
        """.trimIndent()

        val parser = getParser(stupidScript)
        val result = visit(parser.file())

        assertEquals(expected, result)
    }

    @Test
    fun `empty file`() {
        val stupidScript = ""

        val expected = ""

        val parser = getParser(stupidScript)
        val result = visit(parser.file())

        assertEquals(expected, result)
    }
```

```kotlin
    override fun visitFile(ctx: StupidLangParser.FileContext?): String {
        return when {
            ctx?.statements() == null -> ""
            else -> visit(ctx.statements())
        }
    }
```