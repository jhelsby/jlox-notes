# 5. Statements and State

Now we've established the foundations of our interpreter, we can move quickly into some more advanced concepts. In this section, we'll implement:

* expression statements - an expression followed by a semicolon. This allows us to evaluate expressions with side effects, such as `foo();`.

* `print`, a statement that can produce output.

* state, using `var`.

* expressions to access and assign to variables.

* blocks and local scope.