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

* Nystrom's `finishCall` method also reports an error if you try to pass more than 255 argument. I've omitted that here (and elsewhere in these docs) as it isn't necessary - it's just to be compatible with _clox_.

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

_Native functions_ (a.k.a. _primitives_, _external functions_, or _foreign functions_) are implementation-defined functions which provide fundamental services which programmers will often need to write programs. For example, providing native functions which can access the file system will allow programmers to read files far more easily.

Here, we implement a simple benchmarking function, `clock()`, which will returns the current system time.

Native functions live in the global scope, so we'll want to add the following fields to our `Interpreter` class:

```java
final Environment globals = new Environment();
private Environment environment = globals;
```

* `environment` tracks the _current_ environment, which will change as we enter and exit local scopes. `globals` will always hold the global environment.

Here's our clock function. It's pretty self-explanatory:

```java
Interpreter() {
    globals.define("clock", new LoxCallable() {
        @Override
        public int arity { return 0; }

        @Override
        public Object call(Interpreter interpreter, List<Object> arguments) {
            // Return time in seconds.
            return (double)System.currentTimeMillis() / 1000.0;
        }

        @Override
        public Strig toString() {return "<native fn>"; }
    });
}
```

## Function Declarations

Now, we allow the user to define their own functions, using the `fun` keyword. We'll reuse much of this functionality to implement methods in future sections. Here's the updated grammar:

```
declaration -> funDecl | varDecl | statement ;
```

```
funDecl  -> "fun" function
function -> IDENTIFIER "(" parameters? ")" block ;
```

* `function` will be reused to declare methods. In Lox, method declarations don't require a `fun` keyword - they look like:
    ```java
    class myClass {
        myMethod() {
            // Do something here.
        }
    }
    ```

Here's the AST node:
```java
Function : Token name, List<Token> params, List<Stmt> body
```

Here's the function parser:

```java
// kind will either be "function" or "method".
// This will be used for this method's error messages.
private Stmt.Function function(String kind) {

    // Parse the function/method name.
    Token name = consume(IDENTIFIER, "Expect " + kind + " name.");

    // Parse the parameters.

    consume(LEFT_PAREN, "Expect '(' after " + kind + " name.")

    List<Token> parameters = new ArrayList<>();

    if (!check(RIGHT_PAREN)) {
        do {
            parameters.add(
                consume(IDENTIFIER, "Expect parameter name."));
        } while (match(COMMA));
    }

    consume(RIGHT_PAREN, "Expect ')' after " + kind + " parameters.")

    // Parse the function/method body.

    consume(LEFT_BRACE, "Expect '{' before " + kind + " body.")

    List<Stmt> body = block();

    return new Stmt.Function(name, parameters, body);
}

```

* To use this for functions and for methods, we'll call `function("function")` and `function("method")` respectively.

## Function Objects