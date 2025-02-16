# 2. Representing Code

In this section, we:

* write syntax trees which define the following [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar for Lox expressions:

    ```
    expression -> literal | unary | binary | grouping ;
    literal    -> NUMBER | STRING | "true" | "false" | "nil" ;
    grouping   -> "(" expression ")" ;
    unary      -> ( "-" | "!" ) expression ;
    binary     -> expression operator expression
    operator   -> "==" | "!=" | "<" | "<=" | ">"
                | ">=" | "+"  | "-" | "*"  | "/" ;
    ```

* establish an extensible way to define operations on these expressions, using the Visitor design pattern.

* implement an operation which pretty-prints a syntax tree using our Visitor approach.

## Implementing Syntax Trees

> `abstract class Expr`

* This is a base class for Lox expressions. We define our expressions `Literal`, `Unary`, `Binary`, and `Grouping` as subclasses of `Expr`.

* These subclasses are simple boilerplate classes which implement the grammar above. For example, Binary looks like this:

    ```java
    static class Binary extends Expr {
        final Expr left;
        final Token operator;
        final Expr right;

        Binary(Expr left, Token operator, Expr right) {
            this.left = left;
            this.operator = operator;
            this.right = right;
        }
    }
    ```

### AST Code Generator

Rather than manually implement all of these subclasses, Nystrom suggests using a script to generate `Expr.java` for us. This will be particularly useful as we add more expression types throughout our implementation.

Inside the package `com.craftinginterpreters.tool`, we define:

> `public class defineAst GenerateAst`

* We implement a `main` method here so that, when we run `java GenerateAst.java outputDir`, it generates our complete `Expr.java` file at the specified path `outputDir` by calling:
    ```java
    defineAst(outputDir, "Expr", Arrays.asList(
        "Binary   : Expr left, Token operator, Expr right",
        "Grouping : Expr expression",
        "Literal  : Object value",
        "Unary    : Token operator, Expr right"
    ))
    ```

    This can easily be updated to add more expressions in future.

> `void defineAst(String outputDir, String baseName, List<String> types)` and `void defineType(PrintWriter writer, String baseName, String className, String fieldList)`

* These methods simply convert the list of types given above into our desired boilerplate and write it to our `Expr.java` file, using string manipulation and `PrintWriter`.

## Visitors for Expressions

As we build our language, we'll want to operate on these expressions in various ways. These operators will need to behave in different ways for different expressions. We could use a long if-then chain to implement this:

```java
if (expr instanceof Expr.Binary) {
    // ...
} else if (expr instanceof Expr.Grouping) {
    // ...
} else if // ...
```

This isn't very efficient, though. Even worse, expressions further down the chain would take longer to execute - we'd need to check more `if` cases before reaching them.

An object-oriented approach could be to add an abstract method to our `Expr` class for each operation, and override them for each subclass. However, this isn't very extensible: each time we wanted to add a new operation, we'd need to manually add a new method to each subclass. Sometimes, the behaviour we want to add will have nothing to do with how our expressions are implemented, violating the separation of concerns principle.

It turns out there's no perfect solution here - we've run into a well-known, unsolved issue called the [Expression Problem](www.wikipedia.com/wiki/Expression_problem). Instead, we can work around it using the Visitor design pattern, which lets us add new operations without having to add to our `Expr` class. Here's how it works:

1. We add a `Visitor` interface to our `Expr` class.  It will contain abstract `visitExpr` methods for each of our expressions.

    ```java
    interface Visitor<R> {
        R visitLiteralExpr(Literal literal);
        R visitUnaryExpr(Unary Unary);
        // ...continue for every expression
    }
    ```

    Every operation we want to define on our expressions should implement this interface. Note that we use generic types so that these operations can return different types as required.

2. We add an abstract `accept` method to our `Expr` class:
    ```java
    abstract <R> R accept(Visitor<R> visitor);
    ```


  Next, we override this `accept` method in each of our expression subclasses, using the corresponding `visitExpr` method defined above. This will allow us to take an operator and apply it to our expression. Here's what it looks like for `Binary`:

    ```java
    class Binary extends Expr {
        @Override
         <R> R accept(Visitor<R> visitor) {
            visitor.visitBinaryExpr(this)
        }
    }
    ```

### Visitor Code Generator

Since this is all boilerplate code which needs to be added to `Expr.java`, we can generate it automatically. To do this, we can modify `defineAst` and `defineType`, and add one more method:

> `void defineVisitor(writer, baseName, types)`

* This also uses string manipulation and `PrintWriter`.

We're now ready to add our operations by implementing `Visitor`. Thanks to this design pattern, we can define these in separate classes, as the next section will show.

## Example: A Syntax Tree Pretty Printer

To illustrate the Visitor concept, we now use it to implement a debugging utility operation which will pretty-print a syntax tree. For example:
```
    *
  /   \
 -    ( )
 |     |
123  45.67
```
will be printed as `(* (- 123) (group 45.67))`.

In `AstPrinter.java`:

```java
package com.craftinginterpreters.lox;

class AstPrinter implements Expr.Visitor<String> {
    String print(Expr expr) {
        return expr.accept(this);
    }

    @Override
    public String visitLiteralExpr(Expr.Literal expr) {
        if (expr.value == null) return "nil";
        return expr.value.toString()
    }

    @Override
    public String visitUnaryExpr(Expr.Unary expr) // ...

    // ...
}
```

We'll need to override all our operations and implement pretty-print methods for each of them. For a compound expression, we can use recursion on our `print` method and appropriate string manipulation. For example, the expression `Binary(1, + 2)` would be converted into `"+ (1 2)"`.