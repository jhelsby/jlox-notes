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


The [visit method](/sections/4_evaluating-expressions.md##visitor-pattern-basics) to interpret function calls (see ) is quite straightforward too:

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

Now we've parsed the syntax, we want to interpret the function itself. This is significantly more complicated than interpreting many of the AST nodes we've seen previously, as we need to handle:

1. the function environment. To work properly, function parameters should be local to the function call itself.

2. return statements. Our functions should evaluate to a value (possibly nil), and `return` should immediately exit a function and its associated environments.

3. function closures, and the environment issues these create.

To do this, we'll implement a `LoxFunction` class. We'll start small, addressing (1) first, and then add functionality for (2) and (3) in turn.

### `LoxFunction` and `visitFunction`

Function parameters need to be local to the function. If we have some code like:

```java
var a = 1

fun foo(a) {
    print a + 1;
}
```

we definitely want `foo(10)` to print `11`, not `2`.

But they also need to be local to the function _call_. If we have some recursive function like:

```java
fun count(n) {
    if (n > 1) count(n-1);

    print n;
}
```

then `count(3)` should print `1`, `2`, `3`, rather than `3`, `3`, `3` or anything else.

To do this, calling a function object should create a new environment dynamically. We'll see this in the `call` method of our `LoxFunction` implementation below:

```java
import java.util.List;

class LoxFunction implements LoxCallable {
    private final Stmt.Function declaration;

    // Load our function AST node.
    LoxFunction(Stmt.Function declaration) {
        this.declaration = declaration
    }

    @Override
    public Object call(Interpreter interpreter, List<Object> arguments) {

        // Create a new environment for the function call.
        Environment environment = new Environment(interpreter.globals);

        // Bind each function argument to its parameter identifier.
        for (int i = 0; i < declaration.params.size(); ++i) {
            environment.define(
                declarations.params.get(i).lexeme,
                arguments.get(i));
        }

        interpreter.executeBlock(declaration.body, environment);

        // We'll update this part later, when
        // we implement our return statement.
        return null;
    }

    @Override
    public int arity() {
        return declaration.params.size();
    }

    @Override
    public String toString() {
        // e.g. for a function `foo()`,
        // this will print "<fn foo>".
        return "<fn " + declaration.name.lexeme + ">";
    }
}
```

To illustrate how `call` works, suppose we define the program:
```java
fun add(a, b, c) {
    print a + b + c;
}

add(1, 2, 3);
```

When we call `add`, the interpreter will:
* create a new environment.
* add the bindings `a -> 1`, `b -> 2`, `c -> 3` to that environment.
* begin executing `print a + b + c;`, the `add` function body.
* convert this body into `print 1 + 2 + 3;` using the environment we just created.
* evaluate the `1 + 2 + 3` expression to `6`.
* print `6`.

Finally, we'll implement our [visit method](/sections/4_evaluating-expressions.md#visitor-pattern-basics), which is much simpler:

```java
@Override
public Void visitFunctionStmt(Stmt.Function stmt) {
    LoxFunction function = new LoxFunction(stmt);
    environment.define(stmt.name.lexeme, function);
    return null;
}
```

This method:
* converts a function AST node into a `LoxFunction` object.
* binds the function object to its function name.
* stores a reference to this binding in the current environment.

### Return Statements

To get values out of our functions, we need to implement return statements. Since Lox is dynamically typed, the compiler can't stop us doing something like this:

```java
fun foo() {
    print "don't return anything";
}

fun bar() {
    return;
}

var result1 = foo();
var result2 = bar();

print result1;
print result2;
```

We need to ensure `result1` and `result2` are well-defined. To handle these cases, we treat any function without a return statement, or without a value after a return statement, as implicitly returning `nil`.

Here's the resulting grammar for `return`:

```
statement  -> exprStmt
            | forStmt
            | ifStmt
            | printStmt
            | returnStmt
            | whileStmt
            | block ;

returnStmt -> "return" expression? ";" ;
```

Here's our statement AST node:

```java
Return : Token keyword, Expr value
```

`keyword` stores the `return` keyword token for error reporting purposes. The parse method is as you'd expect:

```java
private Stmt returnStatement() {
    Token keyword = previous();
    Expr value = null;
    if (!check(SEMICOLON)) {
        value = expression();
    }

    consume(SEMICOLON, "Expect ';' after return value.");
    return new Stmt.Return(keyword, value);
}
```

#### Returning from calls

### Closures

Consider:

```java
fun makeCounter() {
    var i = 0;

    fun count() {
        i = i + 1;
        print i;
    }

    return count;

}
```

We want this to work as follows:

```java
var counter = makeCounter();
counter(); // Prints "1".
counter(); // Prints "2".
```
