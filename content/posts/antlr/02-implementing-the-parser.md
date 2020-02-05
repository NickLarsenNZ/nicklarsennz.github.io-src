---
title: "2. Implementing the Parser"
series: ["Writing a Parser for a Small Language"]
date: 2020-01-22T15:22:22+13:00
draft: true
tags: [antlr, kotlin, parser, code]
---

Now that we have the gernerated code, and have verified the parse tree, we can now move on to writing code to interpret out StupidLang script.

There are two ways to implement the parser from the generated code:

- The Listener Pattern: _Optionally override listeners for each rule, and as the parser parses the token stream, the applicable Listener will be called._
- The Visitor Pattern: _Optionally override the Base Visitor for each rule, and starting with the first rule, control which branch of the tre to visit next._

I'll be implementing the Visitor pattern as I'll need control over when to visit children nodes later on when we implement branching (if/else). If you were writing a code translation tool, you might prefer the Lstener pattern and let ANTLR call then listeners.

{{< notice tip >}}
Because the Visitor pattern gives you complete control over the parse tree, you must override methods for all rules up to the root. You cannot simply override methods you are interested in.
{{< /notice >}}

{{< notice tip >}}
- IDEA code gen for antlr (right click the g4, select "Generate ANTLR Recognizer", code placed in `src/main/gen`)
- gradle modification   
- tests/tdd
{{< /notice >}}

## Setup

- Gradle

## Helper functions

```kotlin
fun getParser(code: String): StupidLangParser {
    val lexer = StupidLangLexer(CharStreams.fromString(code))
    return StupidLangParser(CommonTokenStream(lexer))
}

fun visit(context: ParserRuleContext): String {
    return StupidVisitor().visit(context)

}
```

## Print statements

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

```kotlin
class StupidVisitor : StupidLangParserBaseVisitor<String>() {
    override fun visitPrint(ctx: StupidLangParser.PrintContext?): String {
        super.visitPrint(ctx)
        val arg = visit(ctx?.string())
        return "$arg\n"
    }

    override fun visitString(ctx: StupidLangParser.StringContext?) = ctx!!.text
}
```

## Repeat Statement

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

## Multiple Statements

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

## A whole file

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