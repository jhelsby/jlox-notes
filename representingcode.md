# 2. Representing Code

In this section, we write syntax trees which define the following BNF grammar for Lox expressions:

```
expression -> literal | unary | binary | grouping ;
literal    -> NUMBER | STRING | "true" | "false" | "nil" ;
grouping   -> "(" expression ")" ;
unary      -> ( "-" | "!" ) expression ;
binary     -> expression operator expression
operator   -> "==" | "!=" | "<" | "<=" | ">"
            | ">=" | "+"  | "-" | "*"  | "/" ;
```

## Implementing Syntax Trees

> `abstract class Expr`

* This is a base class for Lox expressions. We define our expressions `Literal`, `Unary`, `Binary`, and `Grouping` as subclasses `Expr`.

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

Rather than manually implement all of these subclasses, Nystrom suggests using a script to generate `Expr.java` for us. Inside the package `com.craftinginterpreters.tool`, we define:

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

> `void defineAst(String outputDir, String baseName, List<String> types)`

* This method simply converts the list of types given above into our desired boilerplate and writes it to our `Expr.java` file, using string manipulation and `PrintWriter`.

## Visitors for Expressions

As we continue to build our language, different expressions will need to behave in different ways. Perhaps the simplest approach would be in a long if-then chain:

```java
if (expr instanceof Expr.Binary {
    /...
} else if (expr instanceof Expr.Grouping {
    //
} else if // ...
```

but this is slow, and even worse, expressions further down the chain would take longer to execute as we need to check more `if` cases before we reach them.

An object-oriented approach could be to add suitable abstract methods to our `Expr` class, so that each subclass has to implement them.

However, this isn't very extensible: each time we wanted to add a new behaviour, we'd need to manually add a new method to each subclass. Sometimes the behaviour we want to add will have nothing to do with how our expressions are implemented, violating the separation of concerns principle.

It turns out there's no perfect solution here - we've run into a well-known, unsolved issue called the [Expression Problem](www.wikipedia.com/wiki/Expression_problem). Instead, we can work around it using the Visitor design pattern. Here's how it works:

* We add a `Visitor` interface to our `Expr` class. It will contain abstract `visit` methods for each of our expressions:

    ```java
    void visitLiteral(Literal literal);
    void visitUnary(Unary Unary);
    // ...
    ```