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

if we're in the `showA` scope, we're one hop away from `a = block`, and two hops away from `a = global`.

Since a variable in a given environment can have at most one value, the nested environments combined with the number of hops completely specifies the value it should be resolved to.