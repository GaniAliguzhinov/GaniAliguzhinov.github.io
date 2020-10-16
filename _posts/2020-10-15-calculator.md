---
layout: post
title: calculator
---

In this post I will show how to write a not-so-simple calculator in C++.

## Starting out

What does a calculator do? 
It receives input, and computes output. 

What input is allowed? 
Let's start with easy ones:

> 1+1
>
> 2*3
>
> 1-3*9
>
> 10/3
>
> 3/2+5/2
>
> 2^10
>
> 9^(0.5+0)
>

We already have quite a bit here. We want to support integers, reals, basic arithmetic operations,
raising number to a real power, combining operations with parentheses. 


We would also like to do a bit more complex stuff:

> sin(pi)
>
> sqrt(cos(pi*3/4))
>
> max(1, -3)

So, we want to support built-in functions and built-in variables... 

Now, our shopping list is getting a bit long. We not only want to 
evaluate incoming expressins and print results, we also need to 
remember functions and variables. 

Since we can remember built-in variables, why not be able to define new ones?

> a <- 10
>
> 2^(a-3)

The reason we used `<-` as assignment is that we would like to have logical statements:

> 1 < 2
>
> 1 = 2
>
> ~(1 = 2)
>
> ~(1 > 0 & 2 > 3)
>
> 1 = 2 \| 2 = 2

Since we can define variables, it only makes sense to also be able to define functions.

What is a function? What do we prefer?

```c
foo (1, 3)
1 foo 3

foo 1 3

foo 1, 3

foo (1 3)

(1 3 foo)

(foo 1 3)
```

We will go with the standard `foo(1, 3)` since it is what most people expect.

To define a function, we will use syntax:

```c
foo(a, b) <- a + b
```

Now that we have an idea of what kind of input we want to receive, let's write a lexer.

## Lexer

The first piece of system is the Lexer, or Tokenizer. It takes in character input and outputs tokens.

Tokens can be variable names, function names, "(", ")", numbers, and operators.

The Lexer itself should at least save last read token, so that the parser can have access to it.

For convenience, instead of ungetting last read char, we will also store last read character.

As we don't like typing out `std::string`, and even `string` is a bit long, we will use `S` as an alias.

### Reading consecutive chars

First, we define a helper function that will read chars as long as the condition is satisfied:

```c++
template<typename T>
S consume(char& ch, T condition) {
    S r {ch};
    while (cin.get(ch) && condition(ch))
        r += ch;
    return r;
}
```
This function will overwrite `ch` no matter what, and after it is done, `ch` will hold
last read character not satisfying the condition. It also uses current `ch` value, since 
`ch` will be the yet-to-be-processed character.

### Reading next token

Function `readNextTok()` will read the next token, that starts with `ch`.

It will also discard whitespace.

Basic structure is as follows:

```cpp
Tok cur;
if (isspace(ch) && ch != '\n') eatWS();

if (isalpha(ch)) { cur.name = readName(); return CurTok = cur; }

if (isdigit(ch)) { cur.value = readDouble(); return CurTok = cur; }

if (cin.eof()) { return CurTok = cur; }

if (ch == '\n') { cin.get(ch); return CurTok = cur; }

cur.symbol = S {ch};
cin.get(ch);

return CutTok = cur;
```

I omitted setting the token code to enumeration value, and processing of `<-`
as a single token. Functions like `readName()` use consume with appropriate lambda
expression and return string or number that they read.

This concludes the lexer. We should now write a test loop that uses the lexer
to read tokens and outputs what it reads in the right format.

```cpp
Lexer lx;

do {
    Tok t = lx.readNextTok();
    if (t.code == TokCode::t_name) {
        cout << t.name;
    } else if (t.code == TokCode::t_number) {
        cout << t.value;
    } else if (t.code == TokCode::t_symbol) {
        cout << t.symbol;
    } else {
        cout << ts.to_string(t.code);
    }
    cout << " ";
} while(lx.CurTok.code != t_eof);
```
Then, we can test the lexer by feeding it some source codes,
or even the binary itself.


Here is an example of lexer parsing a sample file:

```
1 + 23 - 2 * sin ( 1.001 / cos ( 2 ) ) 
 
 a <- 10 
 
 foo ( a ) <- sin ( a ^ 0.5 ) 
 
 bar ( a , b ) <- cos ( a ) + sin ( b ) 
 
 bar ( 2 , 1 ) + foo ( a ) 
 
 
 1 < 2 | 3 > 5 & ~ ( 10 - 2 > 3 ) 
 eof
```

Lexer parsing a source file:

```
# include " lexer . hpp " 
 
 using namespace std ; 
 
 int main ( ) { 
 
 Lexer lx ; 
 
 do { 
 Tok t = lx . readNextTok ( ) ; 
 if ( t . code = = TokCode : : t _ name ) { 
 cout < < t . name ; 
 } else if ( t . code = = TokCode : : t _ number ) { 
 cout < < t . value ; 
 } else if ( t . code = = TokCode : : t _ symbol ) { 
 cout < < t . symbol ; 
 } else { 
 cout < < lx . to _ string ( t . code ) ; 
 } 
 cout < < " " ; 
 } while ( lx . CurTok . code ! = t _ eof ) ; 
 
 return 0 ; 
 } 
 eof 
```

Lexer parsing itself (beginning of file):

```
ELF               >      � $       @        ( �           @  8   @             @        
@        @        �        �
�        �                                             � "       � "                        
P        P        P                                     � l       � |       � |       �
� l       � |       � |                                      8        8        8                                     
X        X        X        D        D                S � td     8        8        8
P � td     8 P       8 P       8 P       <        <                Q � td                                                     
R � td     � l       � |       � |       �        �                / lib64 / ld - linux - x86 - 64 so . 2
GNU     �                         GNU  � � Y �   � K �  � � n ] � � \ � j �             GNU                          
)              �      )        +    � e � ms �    � C
�                        
                        �                        5                        .                        ?                        
                        �                        �                        �                                                                                                
                        �                        �                        s                        &                        
                        �                        �                        }                        �                        
                        U                        7                        z                        �                        
                        �                        �                        �                        �                        
                        X                                               �                        m                        
                        �                        D                                 

```

I added some newlines, since it would otherwise be all in one string.

## Grammar

Now that we have rough syntax in mind, why don't we write a BNF grammar?

Here is a simple grammar for scalar arithmetics:

```R
<expr>     ::= <term> "+" <expr> | <term> "-" <expr>
<term>     ::= <fact> "*" <term> | <term> "/" <fact>
<fact>      ::= "-" <fact> | <primary> | "(" <expr> ")"
<float>     ::= <integer> | <integer> "." <integer>
<integer> ::= <digit> <integer> | <digit>
<digit>     ::= "0" | "1" | ... | "9"
```
Let's also add support for variables and functions:

```R
<variable>   ::= "[a-zA-Z][a-zA-Z0-9]*"
<expr>        ::= <term> "+" <expr> | <term> "-" <expr>
<expr>           | <variable> "<-" <expr>
<expr>           | <variable>  "(" <variable>* ")"  "<-" <expr>
```

## Parser

Parser is used to process the tokens from the lexer.
It builds an AST of the input. Leaves are tokens.

Parser can detect _syntax errors_ - when we cannot build a tree.

Our CFG describes nonterminals, with start symbol `<expr>`.


## Interpreter

Interpreter relies on the AST to know what it supposed to do. 
Since we have variables, we need a _vtable_ to know what values those variables hold.
We also need an _ftable_ binding function names with ASTs of their expressions.

Interpreter takes in the sentence AST, and uses ftable and vtable to evaluate it.

The source code of this example is available on [github](https://github.com/GaniAliguzhinov/CppCalc)
