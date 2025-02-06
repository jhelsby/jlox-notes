# Scanning

## Interpreter Framework

In `public class Lox`:

> `public static void main(String[] args)`

* Executes directly from source: `javac Lox.java`

* If given multiple arguments, returns error.

* If given one argument, `path`, runs `runFile(String path)`.
  * Reads and executes a file.

* If given no arguments, runs `runPrompt()`. 
  * Launches a prompt in an infinite loop. Executes input code one line at a time.

> `void runFile(String path)` and `void runPrompt()`

* Loads a string `source` from the file or path and passes it to `run(String source)`.

> `void run(String source)`

* Converts the source to tokens and acts on them in some way.

> `void error(int line, String message)` and `void report(int line, String where, String message)`

* Will be used throughout the interpreter to handle errors.
* Prints an error message specifying the error and its location (line and column within the line)
* Sets the class field `static boolean hadError` to `true`. 

  * This will call `System.exit(65);` to exit when there is an error.

## Lexemes and Tokens

A lexeme is the smallest sequence of characters that represents something. They are raw substrings, e.g. "<=". 

A token is a lexeme bundled together with other data - see `class Token` below.

> `enum TokenType`

* Defines types for all symbols which are used by the language to evaluate code.

  * e.g. `LEFT_PAREN`, `RIGHT_PAREN`, `COMMA`, `PLUS`, `GREATER_EQUAL`, `STRING`, `AND`, `IF`, etc.

> `class Token`

* For each token, stores its token type, its lexeme, its literal (if any - e.g. a number could have literal 4), the line it occurred on.
* Provides a `toString()` method which returns a string representation of the token.

## Scanner

Lox's lexical grammar is simple enough that it is a regular language. There are tools, such as Lex, which can convert the regular expressions which define its grammar into a scanner for us. But for educational purposes we will do it manually.

In `class Scanner`:

> `String source` and `List<Token> tokens = new ArrayList<>()`

* Store the source we are scanning, and the ArrayList which will hold tokens we are going to convert the source into.

> `Scanner(String source)`

* Create the scanner and load the source into it.

> `List<Token> scanTokens()`

* While `!isAtEnd()` (i.e. while there are still lexemes left to scan), scan the next token with `scanToken()`.

* At the end, append an end of file token and return `tokens`.

> `int start`, `int current`, and `int line`

* Store the start of the lexeme we are scanning, the current character in that lexeme we are scanning, and the line that lexeme is on. 

> `boolean isAtEnd()`

* Tells us when we've consumed all the characters - `current >= source.length()`.


  
