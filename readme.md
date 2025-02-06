# Crafting Interpreters - jlox Notes

This repository contains my notes on the _jlox_ interpreter described in [Crafting Interpreters](https://craftinginterpreters.com/) (2021), by Robert Nystrom. Once I complete them, I plan to implement my own interpreter based on _clox_, the book's more efficient C-based interpreter.

## jlox

jlox is a dynamically typed, object-oriented tree-walk interpreter built in Java. Its built-in data types are booleans, numbers, strings, and nil.

All the code described below lives inside the package `com.craftinginterpreters.lox`.

1. [Scanning](/scanning.md)
2. Representing Code
3. Parsing Expressions
4. Evaluating Expressions
5. Statements and State
6. Control Flow
7. Functions
8. Resolving and Binding
9. Classes
10. Inheritance