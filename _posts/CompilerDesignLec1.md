# Lecture 1

## OCaml

Is a dialect of ML (Meta Language), designed for easy manipulation of ASTs.
Is type-safe, mostly pure, functional, supports polymorphic (generic) algebraic 
datatypes, modules, mutable state.

Functions are first-class entities (supports higher-order functions), results
of computation can be named with `let`. It has few side-effects.

It is strongly statically typed. Compiler typechecks every expression, issues errors
if it cannot prove type safety. Type inference is supported, so are generic types.
User-defined algebraic (tree-structured) data types with pervasive use of pattern matching.
Strong and flexible module system for large projects.

Has well-engineered compiler.

OCaml Tools:

### ocaml

Top-level interactive loop

### ocamlc

Bytecode compiler

### ocamlopt

native code compiler

### ocamldep

Dependency analyzer.

### ocamldoc

Documentation generator

### ocamllex

Lexer generator

### ocamlyacc

Parser generator

### menhir

Modern parser generator

### ocamlbuild

Compilation manager.

### utop

More fully-featured interactive top-level.

### opam

`opam` - OCaml package manager:

```sh
apt-get install opam
opam list -a
opam install lwt
opam udpate
opam upgrade
```

### Merlin

`merlin` - vim completion 

```sh
opam install merlin
opam user-setup install   # configure vim and emacs
```

In a multi-file project, have a `.merlin` file in project root :

```merlin
S src
S src/foo
S src/bar
# Or S src/* to add immediate subdirectories of src
# Or S src/** to add all subdirectories of src recursively

# B for location of cmi files of other modules
B _build/parsing
B _build/typing

# To use findlib package, use PKG:
PKG lwt lwt.unix
PKG base

# REC will force merlin tool for merlin files in parent directories.
REC
```

## Compiler

Compiler translates between programming languages.

Source code is usually expressive (natural language grammar, syntax, meaning), 
redundant (more information to help with error catching), and abstract.
Low-level code is optimized for hardware.

### Translation

Front End:

1. Source code into tokens (lexical analysis)

2. Tokens into AST (parsing)

Middle End:

3. AST into Intermediate Code (Intermediate Code Generation)

3a. Disambiguation of AST into new AST

3b. Semantic analysis, generates annotated AST (type checking, error checking)

3c. Translation of AST into Intermediate Code

Back End:

4. Intermediate Code into Assembly

4a. Intermediate Code into Control-Flow Graph via control-flow analysis

4b. Control-flow graph into interference graph via data-flow analysis

4c. Register allocation, generates assembly

4d. Code emission.


Assembler generates object code, which is passed to the linker, which might link with 
the library code, and generate executable machine code.

## Grammar

Example source code in a `simple` language:

```simple
X = 6;
ANS = 1;
whileNZ (x) {
  ANS = ANS * x;
  X = X + -1;
}
```

Concrete syntax can be expressed in a BNF. 
Abstract syntax hides irrelevant details of the BNF.
Operational semantics is the behaviour (meaning) of the program.

Grammar:

```
<exp> ::=
          |   <X>
          |   <exp> + <exp>
          |   <exp> * <exp>
          |   <exp> < <exp>
          |   <integer const>
          |   (<exp>)

<cmd>  ::=
          |   skip
          |   <X> = <exp>
          |   ifNZ <exp> { <cmd> } else { <cmd> }
          |   whileNZ <exp> { <cmd> }
          |   <cmd>; <cmd>
```

`<exp>` is nonterminal.
`::=`, `|`, and `<...>` symbols are part of the meta language.
Keywords and symbols are part of the object language.

Abstract Syntax of the language:

```
type var = string

type exp =
  | Var of var
  | Add of (exp * exp)
  | Mul of (exp * exp)
  | Lt of (exp * exp)
  | Lit of int
  
type cmd = 
  | Skip
  | Assn of var * exp
  | IfNZ of exp * cmd * cmd
  | WhileNZ of exp * cmd
  | Seq of cmd * cmd
```

Factorial source:

```simple
X = 6;
ANS = 1;
whileNZ (x) {
  ANS = ANS * x;
  X = X + -1;
}
```

AST of factorial:

```
let factorial : cmd = 
  let x = "X" in
  let ans = "ANS" in
  Seq (Assn (x, Lit 6),
       Seq (Assn (ans, Lit 1),
            WhileNZ(Var x, 
                    Seq (Assn(ans, Mul(Var ans, Var x)),
                         Assn(x, Add(Var x, Lit (-1)))))))
```

Interpreter:

```
type state = var -> int

let rec interpret_exp (s:state) (e:exp) : int =
  match e with
  | Var x -> s x
  | Add (e1, e2) -> (interpret_exp s e1) + (interpret_exp s e2)
  | Mul (e1, e2) -> (interpret_exp s e1) * (interpret_exp s e2)
  | Lt  (e1, e2) -> if (interpret_exp s e1) < (interpret_exp s e2) then 1 else 0
  | Lit n -> n

let update s x v =
  fun y -> if x = y then v else s y

let rec interpret_cmd (s:state) (c:cmd) : state =
  match c with
  | Skip -> s
  | Assn (x, e1) ->
    let v = interpret_exp s e1 in
    update s x v
  | IfNZ (e1, c1, c2) ->
    if (interpret_exp s e1) = 0 then interpret_cmd s c2 else interpret_cmd s c1
  | WhileNZ (e, c) ->
    if (interpret_exp s e) = 0 then s else interpret_cmd s (Seq(c, WhileNZ (e, c)))
  | Seq (c1, c2) ->
    let s1 = interpret_cmd s c1 in
    interpret_cmd s1 c2
  
let init_state : state = fun _ -> 0
  
let main () =
  let s_ans = interpret_cmd init_state factorial in
  Printf.printf "ANS = %d\n" (s_ans "ANS")

(* let _ = main () *) 
;; main ()
```
# Resources

1. [ETH course "Compiler Design"](https://people.inf.ethz.ch/suz/teaching/252-0210.html)

2. [Merlin configuration](https://github.com/ocaml/merlin/wiki/project-configuration)

3. [Online compiler](https://godbolt.org)
