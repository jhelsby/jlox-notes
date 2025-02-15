# 6. Control Flow

In this section, we'll implement basic control flow:

* `if` and `if-else` statements.

* logical operators - `or` and `and`.

* `while` and `for` loops.

As with [the previous section](/sections/5_statements-and-state.md), this doesn't require many concepts that we haven't seen already in Sections 2-4. So I'll keep it brief unless there's something worth expanding on.

## Conditional Execution

Our new statement grammar for `if` and `if-else` statements:

```
statement -> exprStmt | ifStmt | printStmt | block ;

ifStmt    ->  "if" "(" expression ")" statement ( "else" statement)? ;
```

Our `If` syntax tree node stores an `Expr condition`, a `Stmt thenBranch`, and an optional `Stmt elseBranch`.

Here's our parsing method:

```java
private Stmt ifStatement() {
    consume(LEFT_PAREN, "Expect '(' after 'if'.");
    Expr condition = expression();
    consume(RIGHT_PAREN, "Expect '(' after if condition.");

    Stmt thenBranch = statement();
    Stmt elseBranch = null;
    if (match(ELSE)) {
        elseBranch = statement();
    }

    return new Stmt.If(condition, thenBranch, elseBranch);
}
```

* This binds every `else` statement to the nearest preceding `if` statement, avoiding the [dangling else](https://en.wikipedia.org/wiki/Dangling_else) problem.

Our `visitIfStmt` method is straightforward - if the condition is truthy, execute the then branch. Otherwise, execute the else branch (if it exists).

## Logical Operators

`and` and `or` are similar to our other binary operators, but they also short-circuit. Here's the updated grammar:

```
expression -> assignment ;
assignment -> IDENTIFIER "=" assignment | logic_or ;
logic_or   -> logic_and ("or" logic_and )* ;
logic_and   -> equality ("and" logic_equality )* ;
```

We add a new `Logical` syntax tree node, which is identical to the `Binary` node:
```java
Logical : Expr left, Token operator, Expr right
```

We do this so we can give `Logical` its own visit methods with short circuiting.

The parsing methods are just like the binary operator parsing methods. The visit methods are similar but with short-circuiting. Set

```java
left = evaluate(expr.left)
```

* If we have an `or` statement and `left` is truthy, return `left`.
* If we have an `and` statement and `left` is falsy, return `left`.
* Otherwise, return `evaluate(expr.right)`.

## While Loops

These are really simple as we can just use Java's own while loops to execute them. Grammar:

```
statement -> exprStmt | ifStmt | printStmt | whileStmt | block ;

whileStmt    ->  "while" "(" expression ")" statement ;
```

Syntax tree node:
```java
While : Expr condition, Stmt body
```

## For Loops

Grammar:

```
statement -> exprStmt | forStmt | ifStmt
          | printStmt | whileStmt | block ;

for    ->  "for" "(" varDecl | exprStmt | ";" )
                     expression? ";
                     expression?  ")"
            statement ;
```



Implementing this is a little more interesting than the while loop, because for loops can be seen as syntactic sugar for while loops. Consider that the Lox code:

```java
for (var i = a; someCondition(i); i = someModifier(i)) {
    doSomething(i);
}
```

can be written as:

```java
// Example (*) - we'll refer to it in forStatement() below.
{
    var i = a;
    while (someCondition(i)) {
        doSomething(i);
        someModifier(i);
    }
}
```

Because of this, we don't need to add a new syntax tree node for for loops. Instead, we can reuse our `whileStatement()` parser to parse for loops directly. We just need to transform the for loop into a suitable while loop form first - a process known as _desugaring_. Here's the for statement parser:

```java
// Parser.java

import java util.Arrays;

[...]

private Stmt forStatement() {

    // Parse the for (...) part, checking all cases.

    consume(LEFT_PAREN, "Expect '(' after 'for'.");

    Stmt initializer;
    if (match(SEMICOLON)) {
        initializer = null;
    } else if (match(VAR)) {
        initializer = varDeclaration();
    } else {
        initializer = expressionStatement();
    }

    Expr condition = null;
    if (!check(SEMICOLON)) {
        condition = expression();
    }
    consume(SEMICOLON, "Expect ';' after loop condition.");

    Expr increment = null;
    if (!check(RIGHT_PAREN)) {
        increment = expression();
    }
    consume(RIGHT_PAREN, "Expect ')' after for clauses.");

    // Parse the statement.

    Stmt body = statement();

    // If there's an increment, include it in our transformed
    // while loop, as in the example (*) given above.

    if (increment != null) {
        body = new Stmt.Block(
            Arrays.asList(
                body,
                new Stmt.Expression(increment)));
    }

    // If there's no condition, loop infinitely.
    if (condition == null) condition = new Expr.Literal(true);

    // Build the while loop.
    body = new Stmt.While(condition.body);

    // If there's an initializer, run it
    // once before executing the loop.
    if (initializer != null) {
        body = new Stmt.Block(Arrays.asList(initializer, body));
    }

    // Return the final while loop.
    return body;
}
```