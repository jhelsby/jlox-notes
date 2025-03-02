# Inheritance

In this section, we'll implement class inheritence in Lox by:

* parsing, resolving, and interpreting our inheritance syntax:
  ```java
  class SomeSubclass < SomeSuperclass {}
  ```

* Implementing method inheritance.

* Calling superclass methods with the keyword `super`.

## Inheritance Syntax

Here's our new inheritance grammar:

```
classDecl -> "class" IDENTIFIER ( "<" IDENTIFIER )? 
             "{" function* "}" ;
```

* Note the superclass clause is optional - classes don't have to have a superclass in Lox.

Here's the new AST node:

```java
Class : Token name, Expr.Variable superclass, List<Stmt.Function> methods
```

* `superclass` will be `null` if there isn't one. 
* `superclass` is an `Expr.Variable` so we can easily store the relevant superclass as a variable, resolve it, and look it up at runtime.

The implementation looks as you would expect. Brief notes:

* We store the superclass of a class as a field in `LoxClass`.

* We add some semantic error handling in the resolver to prohibit a class inheriting from itself:

    ```java
    class Foo < Foo {} // Forbidden.
    ```

    Allowing this would do nothing useful, and create a cycle in the inheritance chain, breaking the method inheritance code we'll write in a moment.

* We add some runtime error handling in the interpreter to ensure that the superclass is a class:
    ```java
    var NotAClass = "Not a class."
    
    class Subclass < NotAClass {} // Forbidden.
    ```

## Method Inheritance

Thanks to the inheritance implementation above, this couldn't be easier. We add the following recursive lookup to `LoxClass.findMethod()`:

```java
LoxFunction findmethod(String name) {
    // Old code looking up the name in 
    // the current instance and class.
    // ...

    // If we can't find it in the instance,
    // look in the superclass chain.
    if (superclass != null) {
        return superclass.findMethod(name);
    }

    // Otherwise, the method doesn't exist.
    return null;
}
```

Note that this approach means a subclass method takes precedence (or _overrides_) the superclass method. Because of this, there's currently no way to access the superclass method. We'll fix this now.

## Calling Superclass Methods

Sometimes we want to replace the superclass method entirely, but very often we want to refine it instead - do what it does already, plus a few modifications. We now add a `super` keyword for this purpose, which can only be used in a class that inherits from a superclass:

```java
class SomeClass < SomeSuperclass {
    method() {
        // Access the superclass field 
        // SomeSuperclass.someField.
        super.someField;

        // Call the superclass method 
        // SomeSuperclass.someMethod().
        super.someMethod();

        // Hoist SomeSuperclass.someMethod into foo.
        var foo = super.someMethod;

        // Call the superclass method 
        // SomeSuperclass.someMethod().
        foo();
    }
}
```

It's similar to `this`, except it accesses the superclass of the current class, instead of the current class instance.

* Note that it doesn't access the superclass of the current class _instance_ - this introduces some undesirable behaviour such as the following:

    ```java
    class A {
        method() {
            print "A";
        }
    }

    class B {
        method() {
            print "B";
        }

        test() {
            super.method();
        }
    }

    class C < B {}

    C().test(); // We want this to print "A", not "B".
    ```

Here's the grammar for `super`:

```
primary -> [...] | "super" "." IDENTIFIER ;
```

Here's the AST node:

```java
Super : Token keyword, Token method
```

The parsing code is as you would expect. To implement the functionality discussed above, we can use a similar approach to how we implemented `this` in the [previous section](./9_classes.md#this).

In our resolver, if a class declaration has a superclass, we create a new scope around its methods containing the name "super". Our visit method resolves `super` just like any other variable:

```java
@Override
public Void visitSuperExpr(Expr.Super expr) {
    resolveLocal(exp, expr.keyword);
    return null;
}
```

As with `this`, we must adapt the interpreter's environments to match the resolver's scopes. In the interpreter's `visitClassStmt`, if `superclass != null`: 
* create a new environment containing the binding `"super" -> superclass`.
* create the LoxFunctions for each class method, so they can access `super`.
* once all the methods have been created, pop the `super` environment.

The interpreter's visit method for `super` looks like this:

```java
@Override
public Object visitSuperExpr(Expr.Super expr) {
    // Get the number of hops to the 
    // environment where "super" is bound.
    int distance = locals.get(expr);

    // Use distance to access the relevant superclass.
    LoxClass superclass = (LoxClass)environment.getAt(distance, "super");

    // Get the object to which "this" is bound. 
    // Because "this" is always right inside
    // the "super" environment, the required
    // distance is one less than "super".
    LoxClass object = (LoxClass)environment.getAt(distance - 1, "this");

    // Retrieve the desired method from the superclass.
    LoxFunction method = superclass.findMethod(expr.method.lexeme);

    // Error handling if the method can't be found.
    if (method == null) {
        throw new RuntimeError(expr.method, 
            "Undefined property '" + expr.method.lexeme + "'.")
    }

    // Recall from Section 9: method.bind(object)
    // creates a LoxFunction with the body of 
    // method and an environment containing the
    // binding "this" -> object.
    return method.bind(object);
}
```

* The `distance - 1` logic isn't very elegant, we have no other convenient way for the resolver to get the correct environment containing `this`.

Finally, we can add some semantic error handling to ensure `super` is only used inside subclasses. To do this, we update our resolver by:
* adding a new enum val, `ClassType.SUBCLASS` set in `visitClassStmt` if `stmt.superclass != null`.

* updating `visitSuperExpr` so report an error if used when `currentClass == ClassType.NONE` or `currentClass != ClassType.SUBCLASS`.

That's it! We've worked through the entire _jlox_ implementation. For me, this served as a detailed, high-level introduction to many core concepts involved in designing and implementing programming languages.

However, there's a lot more we've yet to cover. While _jlox_ is a fully functioning implementation of the Lox programming language, it is extremely slow and much of the hard work (such as automatic memory management or use of `instanceof`) is handled for us by Java and the JVM.

For the final step of this project, I will implement _clox_ - a much more efficient and low-level implementation of Lox, built in C from the ground up. My _clox_ repository can be found [here](https://github.com/jhelsby/clox).