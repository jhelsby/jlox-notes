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

You'll see these type representations being used in just a moment, in our `visit` methods below.

Note that we don't need to worry about memory management of these values, since Java's garbage collector handles this all for us. In fact, the simplicity which Java's object representation and automatic memory management affords us are the main reasons why Nystrom chose Java to implement this tree-walk intepreter.

## Evaluating Values

We could add an `interpret()` method to our `Expr` class, and call it once the parser has constructed each expression, but this approach will get messy as we add more logic to our AST classes.

Instead, we'll use the Visitor pattern we discussed in [Representing Code](/sections/2_representing-code.md#visitors-for-expressions) to add a new `Interpreter` class. In that section, we implemented an example use of the Visitor pattern - a Pretty Printer class which recursively traversed our AST and concatenated the string representations of each value in order. Our `Interpreter` class will do something similar, except instead of string concatenations, it will compute values. Here it is:

> ```java
> class Interpreter implements Expr.Visitor<Object>
> ```

* This says it's a Visitor which will return `Object` - the root class of our Lox values in Java.

### Visitor Pattern Basics

To apply the Visitor design pattern to our `Interpreter`, we must now:

1. define visit methods for each of our expression tree subclasses - literal, unary, binary, and grouping.

2. use the abstract method `<R> R accept(Visitor<R> visitor)` defined in `Expr` so that we can apply our `Interpreter` operation to our expression (no matter which subclass it belongs to).

We can implement (2) immediately:

```java
private Object evaluate(Expr expr) {
    return expr.accept(this);
}
```

This sends the expression into our interpreter. If you find this confusing, revisit [our previous explanation of the Visitor pattern](/sections/2_representing-code.md#visitors-for-expressions), or consider the following example. Suppose we have some binary expression `binaryExpr` representing `1+2`, and some interpreter instance `visitingInterpreter`.
```java
BinaryExpr binaryExpr = new Expr.Binary(1, PLUS, 2)
// This isn't quite how the Binary constructor works, but just to illustrate!

Interpreter visitingInterpreter = new Interpreter()
```
Now, to make our `visitingInterpreter` act on `binaryExpr`, we call the `accept` method we overrode in `Expr.Binary`:

```java
binaryExpr.accept(visitingInterpreter)
```

This accept method in turn calls:
```java
visitingInterpreter.visitBinaryExpr(binaryExpr)
```
which is what we wanted - `visitingInterpreter` is visiting (i.e. interpreting) `binaryExpr`.

### Evaluation Visit Methods

I think the visit method code is quite self-explanatory, so I will reproduce them below with minimal notes. We'll need a couple of simple helper functions before we can implement them, though:

```java
private boolean isTruthy(Object object) {
    if (object == null) return false;
    if (object instanceof Boolean) return (boolean)object;
    return true;
}
```

* `false` and `nil` are falsey, and everything else is truthy.

```java
private boolean isEqual(Object a, Object b) {
    if (a == null && b == null) return true;
    if (a == null) return false;

    return a.equals(b);
}
```

* Except for the null case, Java's `equals()` method on Boolean, Double and String do all the work here for us.

We'll also need some runtime error reporting methods, `checkNumberOperand` and `checkNumberOperands`, which we'll use below and then discuss in the next section. Let's get into our evaluation visit methods!

```java
@Override
public Object visitLiteralExpr(Expr.Literal expr) {
    return expr.value;
}
```

* Our scanner stored the literal value in a token, which was then stored in the `expr`, so it's easy to retrieve it.

```java
@Override
public Object visitGroupingExpr(Expr.Grouping expr) {
    return expr.expression;
}
```

* Our grouping node `expr` representing the expression `( someExpr )` contains a reference `expr.expression = someExpr`, so it's easy to retrieve it.

```java
@Override
public Object visitUnaryExpr(Expr.Unary expr) {
    Object right = evaluate(expr.right);

    switch (expr.operator.type) {
        case BANG:
            return !isTruthy(right);
        case MINUS:
            checkNumberOperand(expr.operator, right);
            return -(double)right;
    }

    // Unreachable.
    return null;
}
```

* Recall that an unary expression can only look like `(!|-)expr`, hence why the final return is unreachable.

* for the `-` case, `right` must be a number. Java can't statically guarantee this, hence why we must cast to `double` before performing the operation.

  * This is an example and consequence of Lox's dynamic typing.

  * This type cast will happen at runtime when we evaluate `expr`. If `right` _isn't_ a number, a runtime error will be thrown by `checkNumberOperand`.

```java
public Object visitBinaryExpr(Expr.Binary expr) {
    Object left = evaluate(expr.left);
    Object right = evaluate(expr.right);

    switch (expr.operator.type) {
        case BANG_EQUAL: return !isEqual(left, right);
        case EQUAL_EQUAL: return isEqual(left, right);

        case GREATER:
            checkNumberOperands(expr.operator, left, right);
            return (double)left > (double)right;
        case GREATER_EQUAL:
            checkNumberOperands(expr.operator, left, right);
            return (double)left >= (double)right;
        case LESS:
            checkNumberOperands(expr.operator, left, right);
            return (double)left < (double)right;
        case LESS_EQUAL:
            checkNumberOperands(expr.operator, left, right);
            return (double)left <= (double)right;
        case MINUS:
            checkNumberOperands(expr.operator, left, right);
            return (double)left - (double)right;
        case PLUS:
            if (left instanceof Double && right instanceof Double) {
                return (double)left + (double)right;
            }
            if (left instanceof String && right instanceof String) {
                return (String)left + (String)right;
            }
            throw new RuntimeError(expr.operator,
                "Operands must be two numbers or two strings.");
        case SLASH:
            checkNumberOperands(expr.operator, left, right);
            return (double)left / (double)right;
        case STAR:
            checkNumberOperands(expr.operator, left, right);
            return (double)left * (double)right;
    }

}
```

* Note that we chose to evaluate in left-to-right order. If those expressions have side-effects, that choice is user-facing.

* We have overloaded `+`, using it for both adding numbers and concatenating strings.

* As with our unary evaluation, we're utilising dynamic type checking here.

### Runtime Errors

If any of the casts above fail - say we enter `2 * "foo"` into our interpreter, we have a runtime error on our hands. As things stand, the JVM would throw a `ClassCastException`, printing a Java stack trace and exiting the application. To make a useful interpreter, we would prefer to:

* stop evaluating the expression.

* report the error.

* if we're running a Lox script from a file, exit the process.

* if we're running the REPL, allow the user to enter new code.

This is quite straightforward, so I will be brief. We define:

> ```java
> class RuntimeError extends RuntimeException {
>     final Token token;
>
>     RuntimeError(Token token, String message) {
>         super(message);
>         this.token = token;
>     }
> }
> ```

* This tracks the token that identifies where the runtime error came from.

We now add the following to our `Interpreter` class:

> `void checkNumberOperand(Token operator, Object operand)` and `void checkNumberOperands(Token operator, Object left, Object right)`

* These throw a `RuntimeError(operator, "Operand(s) must be of type number.")` if the objects parameters aren't `instanceof Double`.

## Hooking up the Interpreter

We define the Interpreter's public API as follows:

```java
void interpret(Expr expression) {
    try {
        Object value = evaluate(expression);
        System.out.println(stringify(value));
    } catch (RuntimeError error) {
        Lox.runtimeError(error);
    }
}
```

using

> ` String stringify(Object object)`

* This just converts our Lox value into a suitable string using the Java method `object.toString()` (with special cases for null, and for printing integer float values like `1.0` as `1`).

We now update our `Lox.java` file to use our interpreter.

> `static boolean hadRuntimeError = false;`

* This tracks whether a runtime error has been thrown.

> `static void runtimeError(RuntimeError error)`

* This prints the error in a readable format and sets `hadRuntimeError = true`.

We add the line `if (hadRuntimeError) System.exit(70)` to `runFile()`, to handle runtime errors when running Lox scripts from a file.

We add the interpreter to our Lox class:
> ```java
> private static final Interpreter interpreter = new Interpreter();
> ```

* This is static so that successive calls to `run()` inside a REPL session reuse the same interpreter. This will allow us to store global variables across a REPL session when we implement them later.

And finally, we add it to our `run` method!

```java
private static void run(String source) {
    Scanner scanner = new Scanner(source);
    List<Token> tokens = scanner.scanTokens();
    Parser parser = new Parser(tokens);
    Expr expression = parser.parse();

    // Stop if there was a syntax error.
    if (hadError) return;

    interpreter.interpret(expression);

    // This is all we have so far!
}
```