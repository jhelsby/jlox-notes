# 5. Statements and State

Now we've established the foundations of our interpreter, we can move quickly into some more advanced concepts. In this section, we'll implement:

* expression statements - an expression followed by a semicolon. This allows us to evaluate expressions with side effects, such as `foo();`.

* `print`, a dedicated print statement that can produce output.

* state, using `var`.

* expressions to access and assign to variables.

* blocks and local scope.

## Statements

We have three types of statements:

* Expression statements - an expression followed by a semicolon.

* Print statements, to print output. These is included by Nystrom for didactic purposes - it would be more normal to omit this rule and implement `print` as a library function instead.

* Declaration statements. These assign identifiers to variables, functions, or classes. For example, `var foo = "bar"`.

Adding syntax trees and parsing these follows many of the same principles as for expressions, which we covered in [Representing Code](/sections/2_representing-code.md) and [Parsing Expressions](/sections/3_parsing-expressions.md). I'll be accordingly brief, and expand on details only when they differ significantly from our expression implementations.

Here are our new rules:

```
program     -> declaration* EOF ;
declaration -> varDecl | statement ;
varDecl     -> "var" IDENTIFIER ("=" expression)? ";" ;
statement   -> exprStmt | printStmt ;
exprStmt    -> expression ";"
printStmt   -> "print" expression ";"
```

* A program is a list of declarations followed by an "end of file" token. This token means the parser must consume the entire input.

* Declaration statements can either be variable declarations (of the form `var foo` or `var foo = "bar"`), or statements. We'll add function and class declarations in later sections.

* `statement` can either be an expression statement, of the form `expr;`, or a print statement, of the form `print expr;`.

We must then change our expression rules to allow for variables, adding `IDENTIFIER`:

```
expression -> assignment;
assignment -> IDENTIFIER "=" assignment | equality
          [...]
primary    -> NUMBER | STRING | "true" | "false"
             | "nil" | "(" expression ")" | IDENTIFIER ;
```

Some implementation details:

* We change our parsing function to `List<Stmt> parse()`. It reads a program and loads its statements into an ArrayList.

* _Environment_: we store our variable identifiers and their associated values in memory using a hash map - `Map<String, Object> = new HashMap<>()`. We add our `Environment` class to our `Interpreter`.

  * The statement `var foo = "bar";` will `put` this in our map.

  * `var foo;` will put the key-value pair `(foo, nil)` in the map.

  * Using a variable retrieves it from the map it has been defined, and throws a runtime error if not.

    We could make it a syntax error, but this causes problems with recursion. We want to be able to refer to functions that haven't been defined yet - they might be defined later on in our code.

    Making it a runtime error means that an error is only thrown if we try to _evaluate_ a variable and can't find it.

* Subtleties arise when trying to parse assignments, and distinguishing between l-values and r-values. In short, we parse the left-hand side, which can be any expression. If we find a `=`, we parse the right-hand side. Finally, we check that the left-hand side is a variable that can have values assigned to it. If so, we combine these into an assignment expression node.

## Scope