# 5. Statements and State

Now we've established the foundations of our interpreter, we can move quickly into some more advanced concepts. In this section, we'll implement:

* expression statements - an expression followed by a semicolon. This allows us to evaluate expressions with side effects, such as `foo();`.

* `print`, a dedicated print statement that can produce output.

* state, using `var`.

* expressions to access and assign to variables.

* blocks and local scope.

## Statements

We have three types of statements:

* Expression statements - an expression followed by a semicolon.

* Print statements, to print output. These is included by Nystrom for didactic purposes - it would be more normal to omit this rule and implement `print` as a library function instead.

* Declaration statements. These assign identifiers to variables, functions, or classes. For example, `var foo = "bar"`.

Adding syntax trees and parsing these follows many of the same principles as for expressions, which we covered in [Representing Code](/sections/2_representing-code.md) and [Parsing Expressions](/sections/3_parsing-expressions.md). I'll be accordingly brief, and expand on details only when they differ significantly from our expression implementations.

Here are our new rules:

```
program     -> declaration* EOF ;
declaration -> varDecl | statement ;
varDecl     -> "var" IDENTIFIER ("=" expression)? ";" ;
statement   -> exprStmt | printStmt ;
exprStmt    -> expression ";"
printStmt   -> "print" expression ";"
```

* A program is a list of declarations followed by an "end of file" token. This token means the parser must consume the entire input.

* Declaration statements can either be variable declarations (of the form `var foo` or `var foo = "bar"`), or statements. We'll add function and class declarations in later sections.

* `statement` can either be an expression statement, of the form `expr;`, or a print statement, of the form `print expr;`.

We must then change our expression rules to allow for variables, adding `IDENTIFIER`:

```
expression -> assignment;
assignment -> IDENTIFIER "=" assignment | equality ;
          [...]
primary    -> NUMBER | STRING | "true" | "false"
             | "nil" | "(" expression ")" | IDENTIFIER ;
```

Finally, we change our parsing function to `List<Stmt> parse()`. It reads a program and loads its statements into an ArrayList.

### Environment

The _environment_ is where we store our variable identifiers and their associated values in memory. We implement this using a hash map - `Map<String, Object> = new HashMap<>()`. We add our `Environment` class to our `Interpreter`.

* The statement `var foo = "bar";` will `put` this in our map.

* `var foo;` will put the key-value pair `(foo, nil)` in the map.

* Using a variable retrieves it from the map it has been defined, and throws a runtime error if not.

  We could make it a syntax error, but this causes problems with recursion. We want to be able to refer to functions that haven't been defined yet - they might be defined later on in our code.

  Making it a runtime error means that an error is only thrown if we try to _evaluate_ a variable and can't find it.

### Assignment

Our assignment rule is a little tricky, because we need to distinguishing between l-values and r-values in our assignments.

* We don't want to evaluate an l-value to find out what value is stored there. Rather, we want to evaluate it to find out the storage location it refers to.

* On the other hand, we want to evaluate the r-value to get its value.

* Finally, we want to store the value represented by the r-value in the storage location referred to by the l-value.

Consider the following example:
```java
makeList().head.node = 1 + 2;
```

* We don't care what is stored at the l-value `makeList().head.node` - we just want to assign the r-value `1+2` to that location.

* We need to evaluate `1+2` to find that it represents the value `3`.

* Finally, we need to store `3` at the l-value location `makeList().head.node`.

To make this work alongside our grammar - which looks like
```
expression -> assignment;
assignment -> IDENTIFIER "=" assignment | equality ;
equality   -> ...
```
- we can implement the `assignment` rule as follows:

```java
private Expr assignment() {
    Expr expr = equality();

    if (match(EQUAL)) {
        Token equals = previous();
        Expr value = assignment();

        if (expr instanceof Expr.Variable) {
            Token name = ((Expr.Variable)expr).name;
            return new Expr.Assign(name, value);
        }

        error(equals, "Invalid assignment target.");
    }

    return expr;
}
```

* If we have an expression that _isn't_ followed by an `=` assignment operator, we just parse and return it.

* If it _is_ followed by `=`, we parse the l-value, then the `=`, then the r-value (recursively, using `assignment()`).

* Now, we check if our l-value refers to a valid storage location - is it a variable?

  * If so, we can create and return the corresponding `Expr.Assign` object.

  * Otherwise, we indicate an error, and return the l-value we were able to parse successfully.


## Scope