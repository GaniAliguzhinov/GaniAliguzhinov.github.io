---
layout: post
title: calculator
---

In this post I will show how to write a simplistic calculator in C++.

## Starting out

What does a calculator do? 

It receives input, and computes output. 

What input is allowed? 

```
1+1
2*3
3!
1-3*9
10/3
3/2 + 5/2
2^10
9^{0.5}
sin(pi)
sqrt(cos(pi*3/4))
lim_{n->infty}(1/n)
max(1, -3)
a <- 10
ln(a + 2)
[10 20 30] * 3
[2 3]{1 2 3 4 5 6} * [3 1]{1 1 1}
```

In the last expression, we introduced matrix notation: 
$$[s_1 s_2 \dots s_n]\lbrace{a_1 a_2 \dots a_m\rbrace}$$  where $$m = \prod_{i=1}^n s_i$$.
This way, we let the calcualtor know that there is a vector of shape $$s_1 \times \dots \times s_n$$, 
which is immediately defined with data list.

It makes sense to also allow:

```
[s1 s2 ... sn] b <- {a1 a2 ... am}
[s1*s2*...*sn] c <- b
```

Here, we treat `{a1 a2 ... am}` as a generic 1D vector, 
which in turn can be used to define a matrix of appropriate shape.

Then, it follows that 


```
[m] {1 2 3 ... m} = {1 2 3 ... m}
```

Where we use `=` as equivalence relation, not as assignment.


## Writing Grammar

Now that we have rough syntax in mind, why don't we write a BNF grammar?

Here is a simple grammar for scalar arithmetics:

```
<numexpr>    ::= <numterm> "+" <numexpr> | <numterm> "-" <numexpr>
<numterm>    ::= <numfact> "*" <numterm> | <numterm> "/" <numfact>
<numfact>    ::= "-" <numfact> | <primary> | "(" <numexpr> ")"
<float>         ::= <integer> | <integer> "." <integer>
<intger>         ::= <digit> <integer> | <digit>
<digit>         ::= "0" | "1" | ... | "9"
```
Let's also add support for variables:

```
<variable>        ::= "[_][a-wA-W0-9]*"
<numexpr>        ::= <variable> "<-" <numexpr>
```

How do we add support for arrays? Let's consider vector a 1D vector, 
and a matrix as arbitrarily shaped vector:

```
<matexpr>        ::= <matterm> "+" <matterm> | <matterm> "-" <matterm> | <variable> "<-" <matexpr>
<matterm>        ::= <matfact> "*" <matterm> | <matterm> "x" <matfact>
<matfact>        ::= "-" <matfact> | <matrix> | "(" <matexpr> ")" | "T" <matfact> 
<matrix>        ::= <shapev> <vector> | <vector>
<shapev>        ::= "[" <vector> "]"
<vector>        ::= "{" <numexpr>* "}" | <numexpr>*
```

This doesn't seem quite right, but we will start from here and see what is wrong. 
Basically, we copied part of scalar gramar, and made matrices explicitly state their
shape if they are not 1D.

We forgot to include `"^"`, but adding it seems to create problems: each time we add 
a binary operator, we would need to add non-terminals with rules and change rules for 
other non-terminals. Instead, we would like to try operation precedence.

And what about functions? We might think of them as binary or unary operators too, that only 
operate on factors. However, we won't write out the new grammar since it is not trivial to include
support for binary operator precedence.

## Writing Lexer

Now that we know what we might want to achieve in the future, we should start writing code.

The first piece of system is the Lexer, or Tokenizer. It takes in character input and outputs tokens.

Tokens can be variable names, function names, "(", ")", "[", "]", "{", "}", numbers, and operators.

The Lexer itself should at least save last read token, so that the parser can have access to it.

For convenience, instead of ungetting last read char, we will also store last read character.

As we don't like typing out `std::string`, and even `string` is a bit long, we will use `S` as an alias.

### Reading consecutive chars

First, we define a helper function that will read chars as long as condition is satisfied:

```c++
template<typename T>
S consume(char& ch, T condition) {  
    S r = "";
    while (cin.get(ch) && condition(ch))
        r += ch;
    return r;
}
```
This function will overwrite `ch` no matter what, and after it is done, `ch` will hold 
last read character not satisfying the condition.

This leads to our convention that `ch` is the character that is yet to be processed.

### Reading next token

Function `readNextTok()` will read the next token, that starts with `ch`.

It will also discard whitespace.

Basic structure is as follows:

```cpp
Tok cur;
if (isspace(ch) && ch != '\n') eatWS();

if (ch == '_') { cur.name = readName(); return CurTok = cur; }

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

## Writing a parser

Now that the lexer is decently tested, we can start writing a parser.

Parser should build a high-level IR (intermediate representation) of 
the input. 

Typically, it is a DAG (directed acyclic graph).

Finally, parser should evaluate input sentence according to the graph, 
and print the result.

So far, we described a pretty functional language, but we will not have 
a pure functional language, since the assignment operator
has a side-effect of saving objects to memory.

Even so, all the statements return proper results, which we can print.

Parser should ask for the first token, consider it, and try and evaluate 
it according to the grammar.

We stumble accross a problem: what is the starting symbol of the grammar?
It seems like our grammar is actually two separate grammars - one for 
arrays, and one for numbers, and it feels hard to choose when we treat 
input as `<numexr>` and when as `<matexpr>`, especially since
variables can be both.

This might be one of the reasons some array programming languages don't have 
scalars - they treat numbers as arrays of rank 0 instead.

Let's also do so.

```
<matexpr>        ::= <matterm> "+" <matterm> | <matterm> "-" <matterm> | <variable> "<-" <matexpr>
<matterm>        ::= <matfact> "*" <matterm> | <matterm> "x" <matfact> | <matterm> "/" <matfact>
<matfact>        ::= "-" <matfact> | <matrix> | "(" <matexpr> ")" | "T" <matfact> 
<matrix>        ::= <shapev> <vector> | <vector>
<shapev>        ::= "[" <matexpr1D> "]" | "[" <vector> "]"
<vector>        ::= "{" <matexpr0D>* "}" | <matexpr0D>* | "{" <float>* "}" | <float>*
<matexprND>         ::= matexpr that has rank N
<float>         ::= <integer> | <integer> "." <integer>
<intger>         ::= <digit> <integer> | <digit>
<digit>         ::= "0" | "1" | ... | "9"
<variable>        ::= "[_][a-wA-W0-9]*"
```

So, parser tries to parse input as `<matexpr>`.
When it sees a digit, it tries to parse it as a float, and when it sees a digit
after that, it tries to build a 1D matrix, If it instead sees an operator, it
declares float a 0D matrix and proceeds to parse the expression.
Operands have to be broadcastable or have the same shape.
Scalars are broadcastable to any shape, 1D matrices of size $n$ are broadcastable to 
matrices of higher rank that have their last dimension of size $n$.
1D matrices are treated as row vectors. Column vectors of size $n$ are matrices 
of rank $n$ with each dimension of size 1.
Same rank matrices are broadcastable if for each dimension, their sizes are either 
equal, or one of them has size 1.

Matrix of smaller rank will be treated to have shape `1 ... 1 s1 s2 ... sm`.

