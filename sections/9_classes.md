# Classes

In this section, we'll implement classes in Lox by:

* defining their grammar, adding a `LoxClass` class in our Java code, and implementing class declarations.

* exposing a constructor that lets us create and initialise instances of a class.

* providing a way to store and access fields on instances.

* defining methods, shared by all instances of a class, but which operate on each instances' state separately.

## Class Declarations

This part is quite familar - just like how we implemented expressions and statements. Here's the updated grammar:

```
declaration -> classDecl | funDecl | varDecl | statement ;

classDecl   -> "class" IDENTIFIER "(" function* ")" ;
```

Our AST node:
```java
Class : Token name, List<Stmt.Function> methods
```

The parsing method is similar to the others we've seen. The only note is we check for an end-of-file marker while looping the `function*` part of the grammar, so that the parser doesn't get stuck in an infinite loop if the user forgets to close the class body.

For now, our `Resolver` visit method just `declare`s and `define`s the class statement, using our existing methods. We'll worry about resolving the methods lately.

Our `Interpreter` visit method uses a new `LoxClass` class we define below. For now, it's just a wrapper around a name, but we'll extend it later.

```java
import java.util.List;
import java.util.Map;

class LoxClass {
    final String name;

    LoxClass(String name) {
        this.name = name;
    }

    // Useful for testing our class objects are
    // parsed, resolved, and interpreted correctly.
    @Override
    public String toString() {
        return name;
    }
}
```

* This is our runtime representation of a class.

The `Interpreter` visit method creates an Java instance of `LoxClass`.

```java
@Override
public Void visitClassStmt(Sttmt.Class classStmt) {
    environment.define(stmt.name.lexeme, null);
    LoxClass klass = new LoxClass(stmt.name.lexeme);
    environment.assign(stmt.name, klass);
    return null;
}
```
* We declare the class's name as a variable, turn the AST node into a `LoxClass`, then store our `LoxClass` in the variable.
  * This two-stage variable binding process allows references to the class inside its own methods.

* We use `klass` below to avoid clashing with the Java keyword `class`.

## Creating Instances

Lox doesn't have static methods or fields, so we can't do:

```java
class Foo {
    var bar = 0;

    someMethod() {
        print "test";
    }
}

// These won't work.
Foo.bar
Foo.someMethod()
```

To make a class do anything, we need to create an instance of it.

To avoid introducing more syntax, such as a `new` keyword, Nystrom uses our existing `LoxCallable` interface so we can create a new class like this - much like Python:

```java
var fooInstance = Foo();
```

Let's do that now. It doesn't take long. First, here's a barebones `LoxInstance` class - our runtime representation of an instance:

```java
import java.util.HashMap;
import java.util.Map;

class LoxClass {
    private LoxClass klass;

    LoxInstance(LoxClass klass) {
        this.klass = klass;
    }

    // Useful for testing our instance objects are
    // parsed, resolved, and interpreted correctly.
    @Override
    public String toString() {
        return klass.name + " instance";
    }
}
```

Now we update `LoxClass` to let us create instances by calling a class:

```java
class LoxClass implements LoxCallable {
    // ...

    // Required method for LoxCallable.
    @Override
    public Object call(Interpreter interpreter, List<Object> arguments) {
        LoxInstance instance = new LoxInstance(this);
        return instance;
    }

    // Required method for LoxCallable. We'll update
    // this later to allow >0 argument constructors.
    @Override
    public int arity() {
        return 0;
    }
}
```

This works as follows:

```java
class Foo {}
var fooInstance = Foo();

print Foo; // Prints "Foo".
print bar; // Prints "Foo instance".
```

## Properties on Instances

We add properties before methods for ease of implementation. They use the following syntax:

```java
// Getting.
var foo = someObject.someProperty

// Setting.
someObject.someProperty = "New assignment."
```

We can add this to the grammar as follows. Get:
```
call -> primary ( "(" arguments? ")" | "." IDENTIFIER )* ;
```

Set:

```
assignment -> (call "." )? IDENTIFIER "=" assignment | logic_or ;
```

As with variable access and assignment, we'll need distinct AST nodes (and accompanying visit methods) for getting and setting.

Get AST node:
```java
Get : Expr object, Token name
```

Set AST node:
```java
Set : Expr object, Token name, Expr value
```

I shan't reproduce the parsing or visit methods as they are similar to the others we have seen. Some very brief points:

* For each instance, we store our fields and methods in hash maps:
    ```java
    private final Map<String, Object>  fields = new HashMap<>();
    private final Map<String, LoxFunction>  methods = new HashMap<>();
    ```
* We throw an error if the user tries to invoke a getter on a number (e.g. `1.foo`).

* We can't chain setters. For an expression like this:

    ```java
    a.b.c.d.e.f.g = "test";
    ```
    all but the last `.` are treated like getters, i.e. like
    ```java
    (a.b.c.d.e.f).g = "test";
    ```

* Our `visitSetExpr()` method evaluates the object, raises an error if the object isn't a class, and then evalutes the value. This can be user visible, so in theory we should specify this in Lox documentation somewhere.

## Methods on Classes

We now have getters (`.`), and implemented function calls (`()`) in [Section 7](./7_functions.md). Our method calls just chain these together:

```java
foo.bar()
```

Additionally, we want to pull these expressions apart, as follows:
```java
var foo = object.method;
foo(); // This should call object.method().
```

or:

```java
class SomeClass {}

fun someFunction(argument) {
    print "Some " + argument;
}

var instance = SomeClass();
instance.function = someFunction;
instance.function("argument"); // Should call someFunction("argument").
```

There are some other edge cases to worry about, but we'll get to them in [the next subsection](#this). FOr now, we'll just get basic method calls working.

For our `Resolver`, we'll add a new `FunctionType.METHOD` enum value (to contrast with `FunctionType.FUNCTION`, used to resolve functions), then update our `visitClassStmt` accordingly:

```java
for (Stmt.Function method : stmt.methods)  {
    resolveFunction(method, FunctionType.METHOD);
}
```

We'll add a new `methods` hash map to our `LoxClass` class to store each of our methods:

```java

private final Map<String, LoxFunction> methods;

LoxClass(String name, Map<String, LoxFunction> methods) {
    this.name = name;
    this.methods = methods;
}
```

To add methods to `LoxClass` (our runtime representation of a class), we our `visitClassStmt` in our `Interpreter` as follows:

```java
@Override
public Void visitClassStmt(Sttmt.Class classStmt) {
    environment.define(stmt.name.lexeme, null);

    // New code.
    Map<String, LoxFunction> methods = new HashMap<>();
    for (Stmt.Function method : stmt.methods)  {
        LoxFunction function = new LoxFunction(method, environment);
        methods.put(method.name.lexeme, function);
    }
    LoxClass klass = new LoxClass(stmt.name.lexeme, methods);


    environment.assign(stmt.name, klass);
    return null;
}
```

Finally, we only access methods through instances (there are no static methods in Lox). This is straightforward - first, we add a lookup method in `LoxClass`:

```java
LoxFunction findMethod(String name) {
    if (methods.containsKey(name)) {
        return methods.get(name);
    }

    return null;
}
```

Then we update the `get` method in `LoxInstance` to use `findMethod`.

* Note that in this implementation, we first try to get a field with the given name. If this is unsuccessful, we try to get a method with that name. This means that fields shadow methods.

## This

We now want to implement the following behaviour:

```java
class SomeClass() {
    someMethod() {
        print this.field;
    }
}

var instance1 = Person();
instance1.field = "Field 1";

var instance2 = Person();
instance2.field = "Field 2";

var method = instance1.someMethod;
method(); // This should print "Field 1" because
          // we grabbed the method from instance1.

instance2.someMethod = instance1.someMethod;
instance2.someMethod(); // This should print "Field 1" because
                        // we first grabbed the method from instance1.
                        // (Even though it's now in instance2.)
```

In other words, we want our methods to be bound to the instance they were first accessed from. In Python, these are known as _bound methods_.

Conceptually, the following idea will give us the result we want:

* First, add a `this` keyword which evaluates to the instance the method is bound to. This is similar to `this` in Java or `self` in Python.

* Next, when we grab a method from an instance, set it up to have access to a variable `this`, which stores the instance. For example, consider:

    ```java
    class SomeClass {
        someMethod() {
            print this.field;
        }
    }
    var instance = SomeClass()
    instance.field = "Hello."

    var function = instance.someMethod
    ```

    We somehow want to make `function` work like this:
    ```java
    fun function() {
        this = instance;
        print this.field;
    }
    ```
    
It turns out this can be done very straightforwardly using our existing implementation, using closures.