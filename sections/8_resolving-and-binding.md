# 8. Resolving and Binding

Recall our scope rule, introduced when we [implemented block statements in Section 5](./5_statements-and-state.md#scope):

> A variable usage refers to the preceding declaration with the same name in the innermost scope that encloses the expression where the variable is used.

Our interpreter implemented this rule correctly right up until we added closures, which broke it. For example, the following code currently violates our rule:

```java
var a = "global";
{
    fun showA() {
        print a;
    }

    showA();
    var a = "block";
    showA();
}
```
It prints:
```
global
block
```
instead of the rule-specified:
```
global
global
```

When we call `showA()` for the first time, the environments look like this:

| Environment | Binding |
|------------|-------------------------|
| Global     | `a -> "global"`         |
| Block      | `showA -> <fn showA>`   |
| showA()    | Empty.                  |


When `showA()` tries to print `a`, it climbs environments until it finds `a` in the global scope, so prints `global`.

But when we call it for the second time, the environments have changed:

| Environment | Binding              |
|------------|----------------------|
| Global     | `a -> "global"`      |
| Block      | `showA -> <fn showA>` <br> `a -> "block"` |
| showA()    | Empty.               |


Now when `showA()` print `a`, it finds `a` in the Block scope, so `block` is printed.

There are a few ways to fix this, but some require extensively modifying our existing implementation. Nystrom opts for an educational - albeit inefficient - approach which works with what we've got.

## Semantic Analysis

_Resolving_ a variable is when we determine the value associated with that variable. In the example given above, `a` resolved to `"global"` the first time we called `showA`, and resolved to `"block"` the second time.

We're now going to implement a `Resolver` class, which will conduct semantic analysis on parsed input and resolve all our variable bindings in a manner consistent with our scope rule.

We'll store the resolution in way that works with our existing code:

> For each variable, store the number of environments we have to "hop" to find the variable's value.
>
> The nested environments combined with the number of hops completely specify the value a variable should be resolved to.

In our example above, if we're in the `showA` scope and want `a = global`, we'd store "two hops":

| Environment | Binding               | Hops from showA()|
|------------|------------------------|-------|
| Global     | `a -> "global"`        |2
| Block      | `showA -> <fn showA>`  <br> `a -> "block"` | 1
| showA()    | Empty.                 | 0



## Our Resolver's Approach

Traditionally, we'd put this resolving information in the parser: it's a static property based on the structure of the source code. We'll do this for _clox_.

But to demonstrate another technique, Nystrom implements the resolver as a separate pass for _jlox_ - after the parser, but before the interpreter.

Our variable resolution pass works like a mini-interpreter, but without side effects or control flow. We walk our AST and visit every node in turn - but we don't run anything. All we do is calculate the number of hops from each variable declaration to its value. For this reason, it has O(_n_) complexity where _n_ is the number of AST nodes.

Here are the rules:

* A block statement introduces a new scope for the statements it contains.

* A variable declaration adds a new variable to the current scope. This is tricky:

  * Suppose our variable is defined in terms of an expression of other variables:
    ```java
    var a = someExpr(b, c, ...)
    ```

    Here, we must resolve all the variables `b, c, ...` in `someExpr` _before_ we can resolve `a`.

  * We forbid defining a variable in terms of itself, like:
    ```java
    var a = someExpr(a, b, c, ...)
    ```
    and add error-checking for this case.

  To resolve `a` in the examples above, we first _declare_ `a`, so we know it exists within this scope and shadows any outer `a` variables. Then, we resolve `someExpr` and make sure it isn't using `a` anywhere. If this is successful, we _define_ `a`, marking it as safe to use.

  You'll see the implementation for this in a moment.

* A function declaration introduces a new scope for its body. The function's parameters are bound within that scope.

  * Unlike variables, we don't mind if a function references itself - this enables recursion. For example:
    ```java
    fun count(n) {
        if (n > 1) count(n-1);

        print n;
    }
    ```
    should work.

    Because of this, we don't bother with the _declare, resolve, define_ approach we use for variables. We just define the function name (`count` in our example), then resolve the body.

* Variable expressions (i.e. accessing a variable) and assignment expressions need to have their variables resolved.
  * This part is more straightforward, as you'll see.

## Implementing our Resolver

The code for our `Resolver` class is a little complicated, so I'm going to reproduce quite a bit of it, with comments and annotations. It starts simply enough:

```java
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Stack;

class Resolver implements Expr.Visitor<Void>, Stmt.Visitor<Void> {
    private final Interpreter interpreter;

    Resolver(Interpreter interpreter) {
        this.interpreter = interpreter;
    }
}
```

We now need to write visit methods to resolve each statement and expression we've defined, as per the rules given above. Check [Visitor Pattern Basics](./4_evaluating-expressions.md#visitor-pattern-basics) back in Section 4 for a refresher - what we're doing here is very similar, except we want to resolve our statements and expressions instead of evaluate them.

The equivalent of our `evaluate` method is:

```java
private void resolve(Stmt stmt) {
    stmt.accept(this);
}

private void resolve(Expr expr) {
    expr.accept(this);
}
```

This allows our `Resolver` to apply our visit methods to a given AST node.

Let's implement the actual visit methods now. We'll start with blocks, since they create the local scopes we need to resolve.

```java
// Store our nested local scopes in a Java stack.
// The innermost scope will be the top of the stack.
// We use the Boolean to track if a variable is
// defined (true) or just declared (false).
private final Stack<Map<String, Boolean>> scopes = new Stack<>();

// Create a new scope, resolve all of its
// statements in turn, then discard the scope.
@Override
public Void visitBlockStmt(Stmt.Block stmt) {
    beginScope();
    resolve(stmt.statements);
    endScope();
    return null;
}

// Resolve a list of statements in order.
void resolve(List<Stmt> statements) {
    for (Stmt statement : statements) {
        resolve(statement);
    }
}

// Add the block's scope to the top of the stack.
private void beginScope() {
    scopes.push(new HashMap<String, Boolean>());
}

// Remove the block's scope from the stack.
private void endScope() {
    scopes.pop();
}
```

* As explained in the comments, we store our nested local scopes in a Java stack. We push the block's scope onto the stack, resolve everything in it, then pop the stack.

* As with our `Environment`, a scope is a hash map, and its keys are variable names.

  Unlike `Environment`, however, the variable name doesn't map to the variable value - our resolver doesn't work like that. Remember, we're going to store the number of hops, instead.

  We'll explain what it maps to in just a moment.

```java
@Override
public Void visitVarStmt(Stmt.Var stmt) {
    declare(stmt.name);
    if (stmt.initializer != null) {
        resolve(stmt.initializer);
    }
    define(stmt.name);
    return null;
}

private void declare(Token name) {
    if (scopes.isEmpty()) return;

    // Get the current scope.
    Map<String, Boolean> scope = scopes.peek();

    // Mark the variable "name" as declared.
    scope.put(name.lexeme, false);
}

private void define(Token name) {
    if (scopes.isEmpty()) return;

    // Mark the variable "name" as defined.
    scopes.peek().put(name.lexeme, true)
}

// Resolve variable expresions, reporting an error
// if we come across a declared but undefined variable.
@Override
public Void visitVariableExpr(Expr.Variable varExpr) {
    if (!scopes.isEmpty() &&
            scopes.peek().get(varExpr.name.lexeme) == Boolean.FALSE) {
        Lox.error(varExpr.name,
            "Can't read local variable in its own initializer.");
    }

    resolveLocal(varExpr, varExpr.name);
    return null;
}
```

* As discussed in the rules above, variable declarations are tricky. We want to allow declarations like:
    ```java
    var a = someExpr(b, c, ...)
    ```
    but prevent definitions such as:
    ```java
    var a = someExpr(a, b, c, ...)
    ```
    as these are ambiguous and likely indicate programmer errors.

* To get around this, we first _declare_ our variable, then resolve `someExpr` (checking if this introduces an error). If this is successful, we _define_ `a` as having the value `someExpr`.

* We use the boolean in our `scopes`  `HashMap<String, Boolean>` to mark if our variable has just been declared or if it has been successfully defined without errors.

  When we _declare_ a variable, we store it in `scopes` with the key-value pair `(variable name, false)`. When we _define_ that variable, we change that mapping to `(variable name, true)`, to indicate a successful definition.

`visitVariableExpr` above used a key method, `resolveLocal`:

```java
// expr here is some object that
// can be bound to a name.
// It's not an arbitrary expression.
private void resolveLocal(Expr expr, Token name) {
    for (int i = scopes.size() - 1; i >= 0; --i) {
        if (scopes.get(i).containsKey(name.lexeme)) {

            // Store the number of hops from
            // expr to the name it's bound to.
            interpreter.resolve(expr, scopes.size() - 1 - i);
            return;
        }
    }
}
```
* We check if the expression's name is defined in the most local scope. If it is, there are zero hops from the expression to its name. We store `(expr, 0)` in the interpreter.

    If it isn't, we check the parent scope, and so on. If we eventually find the name, we store `(expr, number of hops)` in the interpreter.

    * We'll implement the `Interpreter.resolve` method later, but all it does is store the key-value pair `(expr, number of hops)` in a `HashMap<Expr, Integer>` field in `Interpreter`.

    * Note that `scopes` only contains our local scopes, not the global scope. So if we don't find `expr`, we don't store anything - our interpreter will just assume it must be in the global scope instead.

* You may be wondering why we're using the parameter `Expr expr` instead of `Expr.Variable varExpr`, like in `visitVariableExpr`.

  The reason is that we'll reuse `resolveLocal` to resolve functions, and classes (including methods, `this` and `super`).

That's the hard part done! Now we can do assignment expressions:

```java
@Override
public Void visitAssignExpr(Expr.Assign assignExpr) {
    resolve(assignExpr.value);
    resolveLocal(assignExpr, assignExpr.name);
    return null;
}
```

and function declarations:

```java
@Override
public Void visitFunctionStmt(Stmt.Function fnStmt) {
    // We declare and define functions first
    // to allow recursion, as discussed above.
    declare(fnStmt.name);
    define(fnStmt.name);

    resolveFunction(fnStmt);
    return null;
}

// A function body defines a new scope, including
// all the function's parameters. Resolve these in turn.
private void resolveFunction(Stmt.function function) {
    beginScope();
    for (Token param : function.params) {
        declare(param);
        define(param);
    }
    resolve(function.body);
    endScope();
}
```

* We made `resolveFunction` a separate method, as we'll reuse it to implement class methods later on.

#### Other Syntax Tree Nodes

These aren't too interesting - we just do the obvious thing required to resolve them. For example:

```java
@Override
public Void visitIfSmt(Stmt.If ifStmt) {
    resolve(ifStmt.condition);
    resolve(ifStmt.thenBranch);
    if (stmt.elseBranch != null) resolve(stmt.elseBranch);
    return null;
}
```

You get the idea. We resolve any and all subexpressions and sub-statements we can find, once each.

## Interpreting Resolved variables

We now adapt our `Interpreter` to use the information the resolver extracts from the AST.

Our `Resolver` stores this in the interpreter with the method `Interpreter.resolve`, which looks like this:

```java
// New field in Interpreter.
private final Map<Expr, Integer> locals = new HashMap<>();

void resolve(Expr expr, int depth) {
    locals.put(expr, depth);
}
```

Recall that the lexical scopes our `Resolver` determines statically from the code correspond directly to environments defined dynamically by our interpreter.

We can use our `locals` map to "hop up" to our required environment from the current one. The helper method `ancestor` performs a specified number of hops to retrieve this environment:

```java
Environment ancestor(int distance) {
    Environment environment = this;
    for (int i = 0; i < distance; ++i) {
        environment = environment.enclosing;
    }

    return environment;
}
```

We can now access a resolved variable in our `locals` map as follows:

```java
// Modified existing method.
public Object visitVariableExpr(Expr.Variable varExpr) {
    return lookupVariable(varExpr.name, varExpr);
}

// New helper methods:

private Object lookUpVariable(Token name, Expr expr) {
    Integer distance = locals.get(expr);
    if (distance != null) {
        return environments.getAt(distance, name.lexeme);
    } else {
        // If we can't find the variable in the local
        // scope, assume it is in the global scope.
        // (A runtime error will be thrown if it's not.)
        return globals.get(name);
    }
}

// Get the variable's value from the ancestor environment.
Object getAt(int distance, String name) {
    return ancestor(distance).values.get(name);
}
```

* As with `resolveLocal` above, we use `Expr expr` in `lookUpVariable` so we can reuse it later to resolve functions, and classes (including methods, `this` and `super`).

* Note how the interpreter assumes that the resolver got it right, and the variable is in the ancestor environment. This implies a deep coupling between `Resolver` and `Interpreter`. If one of them has made a mistake, this coupling will cause problems.

Assigning to resolved variables is similar:

```java
// Modified existing method.
public Object visitAssignExpr(Expr.Variable varExpr) {
    Object value = evaluate(expr.value);

    Integer distance = locals.get(expr);
    if (distance != null) {
        environment.assignAt(distance, expr.name, value);
    } else {
        // Again, assume the variable is in the global
        // environment if it's not in the local environments.
        globals.assign(expr.name, value);
    }

    return value;
}

// New helper method. Assign a value to the
// named variable in the ancestor environment.
Object assignAt(int distance, Token name, Object value) {
    ancestor(distance).values.put(name.lexeme, value);
}
```

To make our resolver run, we simply add the following lines to our `run()` method in `Lox.java`:

```java
Resolver resolver = new Resolver(interpreter);
resolver.resolve(statements);

// Existing code.
interpreter.interpret(statements);
```

## Resolution Errors

We can also use our resolver to statically detect and report a few semantic errors.

Lox allows declaring multiple variables in the global scope for convenience purposes, especially in the REPL. But doing so in the local scope:

```java
{
    var a = 0;
    var a = 1;
}
```
is more likely to be an error. If the user intended to reassign, they would probably have written:

```java
{
    var a = 0;
    a = 1;
}```
```

And if they forgot they used `a` already, they may not want to overwrite it. We can easily check this in our resolver by updating our `declare` method:
```java
private void declare(Token name) {
    if (scopes.isEmpty()) return;
    Map<String, Boolean> scope = scopes.peek();

    // New code checking for the error.
    if (scope.containsKey(name.lexeme)) {
        Lox.error(name,
            "Already a variable with this name in this scope");
    }

    scope.put(name.lexeme, false);
}
```

We can also check to avoid top level return statements:

```java
return "This shouldn't be allowed."
```

To do so, we will store whether we're inside a function or not as a field in our `Resolver` class:

```java
private FunctionType currentFunction = FunctionType.NONE;

private enum FunctionType {
    NONE,
    FUNCTION
}
```

* We'll extend our `FunctionType` enum soon to include classes and the like.

We'll update `currentFunction` whenever we enter or exit a function, as follows:

```java
@Override
public Void visitFunctionStmt(Stmt.Function fnStmt) {
    declare(fnStmt.name);
    define(fnStmt.name);

    // Add the second parameter here, tracking
    // that we've entered a new function.
    resolveFunction(fnStmt, FunctionType.FUNCTION);

    return null;
}

// Add the new second parameter.
private void resolveFunction(Stmt.function function, FunctionType type) {

    // Save the current FunctionType so we can
    // restore it after we've exited this function.
    FunctionType enclosingFunction = currentFunction;

    // Tell the resolver we've entered a function.
    currentFunction = type;

    // Old code.
    beginScope();
    for (Token param : function.params) {
        declare(param);
        define(param);
    }
    resolve(function.body);
    endScope();

    // Restore the old FunctionType.
    currentFunction = enclosingFunction;
}
```

Now we know if we're in a function, we can check this when we encounter a return statement:

```java
public Void visitReturnStmt(Stmt.Return stmt) {
    if (currentFunction == FunctionType.NONE) {
        Lox.error(stmt.keyword, "Can't return from top-level code.");
    }

    // Old code (which I didn't reproduce above).
    // ...
}
```

Finally - now we're using our resolver for error-checking, there's no point running the interpreter if we find an error. To do this, we can modify `run()` in `Lox.java` again:

```java
Resolver resolver = new Resolver(interpreter);
resolver.resolve(statements);

// Stop if there was a resolution error.
if (hadError) return;

interpreter.interpret(statements);
```