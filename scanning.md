# 1. Scanning

In this section, we:

* set up our `main` entrypoint method which allows us to execute jlox code from files or a CLI prompt.
* implement a scanner which converts this source code into tokens, which will be parsed in future sections - see [2. Representing Code](/representingcode.md).

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

A lexeme is the smallest sequence of characters that represents something. They are raw substrings, e.g. "`<=`" for less than or equal to.

A token is a lexeme bundled together with other data - see `class Token` below.

> `enum TokenType`

* Defines types for all symbols which are used by the language to evaluate code.

  * e.g. `LEFT_PAREN`, `RIGHT_PAREN`, `COMMA`, `PLUS`, `GREATER_EQUAL`, `STRING`, `AND`, `IF`, etc.

> `class Token`

* For each token, stores:
  * its token type.
  * its lexeme, e.g. "`(`", "`myVar`", `"myString"`, or "`and`".
  * its literal (if a string or a number - otherwise, stores null).
  * the line it occurred on.
* Provides a `toString()` method which returns a string representation of the token.

## Scanner

Lox's lexical grammar is simple enough that it is a regular language. There are tools, such as Lex, which can convert the regular expressions which define its grammar into a scanner for us. But for educational purposes we will do it manually.

In `class Scanner`:

> `final String source` and `final List<Token> tokens = new ArrayList<>()`

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

### Recognising Lexemes

The heart of the scanner is the `scanToken()` method. This is essentially a huge `switch` statement that systematically checks all possibities a token can be.

We scan over each lexeme using a two-pointer algorithm with two-character lookahead. There are a number of edge cases detailed in the book but omitted here.

> `void scanToken()`

* Consumes the next lexeme and converts it into a token.

  * Single character lexemes are easy.

  * Longer lexemes require a `char peek()` method which gives us one character of lookahead.

  * Whitespace is ignored.

  * Invalid characters throw an error.

* Comments scrub through the file from `//` until a new line character, discarding all these characters.

* String literals scrub through from `"` until another `"`, extracts the literal, and adds a token. If the string is unterminated, an error is thrown.

* Number literals require a `char peekNext()` method, which gives us two characters of lookahead, to handle decimal points.

* Identifiers require a hash map of all reserved words, to see if the identifier is a keyword or just a user-defined identifier like `foo`.


> `char advance()`

* Increments `current` and returns the source character this corresponds to.

> `void addToken(TokenType type)`

* Appends a token of the given type to `tokens`, setting the token literal to be null.

> `void addToken(TokenType, Object literal)`

* Appends the given token to `tokens`, setting its type, lexeme, literal, and line number.

> `boolean match(char expected)`

* Like a conditional `advance()`. We only consume the current character if it's what we're looking for.

* This helps us differentiate between e.g. `! =` and `!=`.

> `char peek()` and `char peekNext()`

* Looks ahead by one and two characters respectively, or returns `\0` if this lookahead is past the end of the file.

> `void string()`, `void number()`, and `void identifier()`

* Parses string literals, number literals, and identifiers respectively.

> `boolean isDigit(char c)`, `boolean isAlpha(char c)`, and `boolean isAlphaNumeric(char c)`

* Helper functions for parsing strings, numbers, and identifiers. Note that `_` is classed as alphabetic since identifiers can start with underscores as well as letters.

> `static final Map<String, TokenType> keywords`

* Stores a keyword hash map, mapping reserved word strings to their `TokenType`.

* For example, `keywords.get("and") -> AND`, and `keywords.get("foo") -> null`.