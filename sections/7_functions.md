# 7. Functions

In this (lengthy) section, we'll implement functions. This covers:

* Function calls.
* Native functions.
* Function declarations.
* Function objects.
* Return statements.
* Local functions and closures.

## Function Calls

First, we implement the ability to call a function:

```java
foo(x1, x2, ..., xn)
```

Here, `foo` can be the name of a function. But it can also be any expression that evaluates as a function. For example, consider:

```java
fun callback() {
    return someFun();
}
```

Here, `callback()()` should do the same thing as calling `someFun()` directly.

Because of this, we want a pair of parentheses following an expression to indicate a function call. (Note that these parentheses can optionally contain arguments.)

Here's our updated grammar:

```
unary   -> ( "!" | "-" ) unary | call ;
call    -> primary ( "(" arguments? ")" )* ;
primary -> [...] ;
```

The arguments list grammar - which we'll implement inside out `call` method - looks like:

```
arguments -> expression ( "," expression )* ;
```

Finally, the call syntax tree node has the form:

```java
Call : Expr callee, Token paren, List<Expr> arguments
```

* Here, `paren` stores the token for the closing parenthesis. We'll use its location to report any runtime errors caused by a function call.

The parser implementation is:

```java
private Expr call() {
    Expr expr = primary();

    // An expression followed by '('
    // indicates a function call.
    while (match(LEFT_PAREN)) {
        expr = finishCall(expr);
    }

    return expr;
}

/**
 * Helper function to parse the argument list.
 */
private Expr finishCall(Expr callee) {
    List<Expr> arguments = new ArrayList<>();

    // If there are any arguments,
    // parse them all, one by one.
    if (!check(RIGHT_PAREN)) {
        do {
            arguments.add(expression());
        } while (match(COMMA));
    }

    // Ensure the left parenthesis is closed.
    Token paren = consume(RIGHT_PAREN, "Expect ')' after arguments.");

    return new Expr.Call(callee, paren, arguments);
}
```

* Nystrom's `finishCall` method also reports an error if you try to pass more than 255 argument. I've omitted that here as it isn't necessary - it's just to be compatible with _clox_.

We now add a new interface for our callees:

```java
interface LoxCallable {
    int arity();
    Object call(Interpreter interpreter, List<Object> arguments);
}
```

* `arity()` reports the _arity_ of the function - the number of arguments a function can take. For example, the binary operator `add(1, 2)` has an arity of 2.


The `visit` method to interpret function calls (see [Evaluation Visit Methods](/sections/4_evaluating-expressions.md#evaluation-visit-methods)) is quite straightforward too:

```java
@Override
public Object visitCallExpr(Expr.Call expr) {
    Object callee = evaluate(expr.callee);

    List<Object> arguments = new ArrayList<>();

    for (Expr argument : expr.arguments) {
        arguments.add(evaluate(argument));
    }

    if (!(callee instanceof LoxCallable)) {
        throw new RuntimeError(expr.paren, "Can only call functions and classes.");
    }

    LoxCallable function = (LoxCallable)callee;

    if (arguments.size() != function.arity()) {
        throw new RuntimeError(expr.paren, "Expected " +
            function.arity() + " arguments but got " +
            arguments.size() + ".");
    }

    return function.call(this, arguments);
}
```

* We need to cast the callee to our new `LoxCallable` interface so we can call it. We throw an error if the callee can't be called.

  For example, strings aren't callable, so `"notAFunction"()` should throw an error.

* After this, we need to check that the function's arity matches the number of arguments we're passing it. If they don't match, we throw an error.

* Note that we first evaluate the callee, then each argument in turn, _then_ check for errors.

## Native Functions