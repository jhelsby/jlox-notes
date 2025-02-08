# Crafting Interpreters - jlox Notes

This repository contains my notes on the _jlox_ interpreter described in [Crafting Interpreters](https://craftinginterpreters.com/) (2021), by Robert Nystrom. Once I complete them, I plan to implement my own interpreter based on _clox_, the book's more efficient C-based interpreter.

_jlox_ and _clox_ interpret the book's language, Lox. Lox is a simple, dynamically typed, object-oriented language. Its built-in data types are booleans, numbers, strings, and nil.

## jlox

jlox is a tree-walk interpreter for Lox built in Java.

Unless otherwise stated, the code described below is defined within the package `com.craftinginterpreters.lox`.

1. [Scanning](/sections/1_scanning.md)
2. [Representing Code](/sections/2_representing-code.md)
3. [Parsing Expressions](/sections/3_parsing-expressions.md)
4. Evaluating Expressions
5. Statements and State
6. Control Flow
7. Functions
8. Resolving and Binding
9. Classes
10. Inheritance