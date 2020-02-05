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

I'll be implementing the Visitor pattern as I'll need control over which branches to visit later on when we implement branching (if/else). If you were writing a code translation tool, you might prefer the Lstener pattern and let ANTLR call then listeners.

{{< notice tip >}}
- IDEA code gen for antlr (right click the g4, select "Generate ANTLR Recognizer", code placed in `src/main/gen`)
- gradle modification   
- tests/tdd
{{< /notice >}}

## Setup

- Gradle

## Override the `print` visitor