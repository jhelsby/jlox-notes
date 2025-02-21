# 8. Resolving and Binding

Recall our scope rule, introduced when we [implemented block statements in Section 5](./5_statements-and-state.md#scope):

> A variable usage refers to the preceding declaration with the same name in the innermost scope that encloses the expression where the variable is used.

Our interpreter implemented this rule correctly right up until we added closures. For example, the following code currently breaks our rule:

```java
var a = "global"
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
instead of:
```
global
global
```

When we call `showA()` for the first time, the environments look like this:

|Global|Block|showA()|
|-|-|-|
|`a -> "global"`|`showA -> <fn showA>`|Empty.|

When `showA()` tries to print `a`, it climbs environments until it finds `a` in the global scope, so prints `global`.

But when we call it for the second timee, the environments have changed:

|Global|Block|showA()|
|-|-|-|
|`a -> "global"`|`showA -> <fn showA>`|Empty.|
||`a -> "block"`||

When we climb environments to print `a` this time, we find it in the Block scope, so `block` is printed instead.

There are a few ways to approach this, but some require making extensive modifications to our existing implementation. Nystrom opts for an educational (albeit inefficient) approach which works with what we've got.

## Semantic Analysis

_Resolving_ a variable is when we determine the value associated with that variable. In the example given above, `a` resolved to `"global"` the first time we called `showA`, and resolved to `"block"` the second time.

We're now going to implement a `Resolver` class, which will conduct semantic analysis on parsed input and resolve all our variable bindings in a manner consistent with our scope rule.

We'll store the resolution in way that works with our existing code: for each variable, we'll store the number of environments we have to "hop" to find the variable's value. In our example above:

|Global|Block|showA()|
|-|-|-|
|`a -> "global"`|`showA -> <fn showA>`|Empty.|
||`a -> "block"`||

if we're in the `showA` scope, we're one hop away from `a = block`, and two hops away from `a = global`. If we want `a = global`, we'd store "two hops".

The nested environments together with the number of hops completely specify the value a variable should be resolved to.

### Our Resolver's Approach

Traditionally, we'd put this resolving information in the parser. It's a static property based on the structure of the source code. And for _clox_, this is what we'll do.

But to demonstrate another technique, Nystrom implements it as a separate pass for _jlox_ - after the parser, but before the interpreter.

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

### Implementing our Resolver

The code for our `Resolver` class is a little complicated, so I'm going to reproduce it in full with comments and annotations. It starts simply enough:

```java
package com.craftinginterpreters.lox;

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

We now need to write visit methods to resolve each statement and expression we've defined, as per the rules given above. Check [Visitor Pattern Basics](./4_evaluating-expressions.md#visitor-pattern-basics) back in Section 4 for a refresher. What we're doing here is very similar, except we want to resolve our statements and expressions instead of evaluate them.

The equivalent of our `evaluate` method is:

```java
private void resolve(Stmt stmt) {
    stmt.accept(this);
}

private void resolve(Expr expr) {
    expr.accept(this);
}
```

As before, this allows our `Interpreter` to apply our visit methods to a given AST node.

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

private void beginScope() {
    scopes.push(new HashMap<String, Boolean>());
}

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

    * Note that `scopes` only contains our local scopes, not the global scope. So if we don't find `expr`, we don't store anything - our interpreter will just assume it must be in the global scope instead.

* You may be wondering why we're using the parameter `Expr expr` instead of `Expr.Variable varExpr`, like in `visitVariableExpr`.

  The reason is that we'll reuse this method to resolve functions, and classes (including `this` and `super`).

* We'll implement the `Interpreter.resolve` method later. Rest assured that it just stores the number of hops in some sensible way.