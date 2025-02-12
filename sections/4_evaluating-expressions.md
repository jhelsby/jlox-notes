# 4. Evaluating Expressions

In this section, we implement an `Interpreter` class which can:

* evaluate ASTs constructed by our [parser](/sections/3_parsing-expressions.md). This means that we can enter expressions like
  ```
  (1+2) * 3
  ```
  into our _jlox_ implementation, and it will actually output the result!

* catch and report runtime errors.

## Representing Values

To manipulate and operate on the values in our ASTs, we must first decide how to store them. This will get more complicated once we implement classes, functions, and instances, but for now we can use a very straightforward approach:

|Lox type| Java representation|
|-|-|
|Any Lox value|`java.lang.Object`
|number|`Double`|
|string|`String`|
|Boolean|`Boolean`|
|nil|`null`|

Note that we don't need to worry about memory management of these values, since Java's garbage collector handles this all for us. In fact, the simplicity which Java's object representation and automatic memory management affords us are the main reasons why Nystrom chose Java to implement this tree-walk intepreter.

## Evaluating Values

