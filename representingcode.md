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

* This method simply converts the list of types given above into our desired boilerplate and writes it to a `Expr.java` file, using string manipulation and `PrintWriter`.
