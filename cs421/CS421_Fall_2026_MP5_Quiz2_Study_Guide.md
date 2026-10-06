---
title: "CS 421 FALL 2026 — Quiz 2 / MP5 Study Guide"
subtitle: "From zero background in CPS to algebraic data types, abstract syntax, free variables, and the automatic CPS transformation"
date: "Prepared 6 October 2026"
---

> **Scope.** Built from the MP5 PrairieLearn problem statements (MP5.1–MP5.5) supplied by the student, the instructor's message that Quiz 2 is *"MP5 as given"*, the student's MP2 / Quiz 1 study guide, and the standard CS 421 lecture sequence on continuations, CPS, and user-defined types. The Fall 2026 course site (`courses.grainger.illinois.edu/cs421/fa2026/`) could not be reached from the environment used to build this guide, so **re-check the current lecture slides for notation and quiz logistics** (length, date, room). The MP5 handout is the authority: wherever this guide and the handout differ, follow the handout.
>
> **Verification.** Every OCaml block in this guide was compiled with OCaml 4.14 against a stand-in `common.ml` with the exact MP5 type definitions. Every sample output from the MP5 handout was reproduced exactly. The CPS translator was also checked by running transformed programs and comparing their values with the originals.
>
> **Academic integrity.** Use this as a learning and self-testing guide. The course policy permits help for understanding but requires you to write and test your own assignment code. Do not paste solutions from this guide into a graded submission. Cover each solution, derive it yourself, and type it from memory.

# How to use this guide

The professor's instruction is **"DO MP5 AS GIVEN."** Quiz 2 uses the original MP5 questions, just as Quiz 1 used the original MP2 questions. That fact shapes the whole plan:

1. **Parts I–VIII** teach every idea MP5 assumes, starting from the beginning of the CPS week. If you skipped everything from week 4 on, read these in order. Each part ends with what it buys you on the quiz.
2. **Part IX** solves MP5.1–MP5.5 clause by clause. For each one it shows how to derive the code from the types and the handout's rules, then gives the code, tests, and the mistakes that cost points.
3. **Parts X–XII** are drills: hand-tracing, a timed quiz procedure, and fresh practice problems with answers.
4. **Part XIII** is a one-page reference to read the night before.

> **The exam standard.** You must produce **compiling** code from a blank skeleton under time pressure. Non-compiling code scores zero, and you normally get one submission. For every MP5 function you should be able to: state its type; name every constructor of `exp` and what it carries; write one clause per constructor without looking; and predict the exact `string_of_exp_cps` output of the handout examples.

# Assessment map for Quiz 2

| Problem | Name | Skill tested | Points / notes |
|---|---|---|---|
| MP5.1 | `import_list` | *Building* an `exp` value by recursion over an OCaml list | Short; one recursive function |
| MP5.2 | `pair_sums` | Writing a whole PicoML program **by hand** as an AST | No recursion in *your* code; just a big constant |
| MP5.3 | `count_const_in_exp` | *Consuming* an `exp` by recursion, one clause per constructor | 9 clauses |
| MP5.4 | `freeVarsInExp` | Free vs. bound variables; set operations on lists | 16 pts total: 1+1+2+2+2+2+3+3, **+4 extra credit** for `let rec` |
| MP5.5 | `cps_exp` | The CPS transformation: build an `exp_cps` from an `exp` | 8 cases **+ extra credit** for `let rec`; uses `freshFor` and free variables |

What carries over from Quiz 1: same PrairieLearn/VSCode workspace, `make` then `./grader`, and one submission. Your function's **name** must match. The parameter names and the shape of the arguments may differ (the handout says so explicitly).

**Dependency chain.** MP5.4 is *used inside* MP5.5. The freshness side conditions in `cps_exp` call `freeVarsInExp`. If MP5.5 is on your quiz, you need a working `freeVarsInExp` in the same file. Practise writing both together.

# Part I — Lists: the minimum you need

MP5 uses OCaml lists in two ways. MP5.1's input is an OCaml list, and MP5.4/5.5 represent *sets of variable names* as `string list`.

## 1. Building and taking apart lists

A list is either empty, `[]`, or an element consed onto a list, `x :: xs`. `[1; 2; 3]` is only sugar for `1 :: (2 :: (3 :: []))`. The operator `::` associates **to the right**. All elements must have the same type.

```ocaml
let l1 = [1; 2; 3]                 (* int list *)
let l2 = 0 :: l1                   (* [0; 1; 2; 3] : O(1), adds at the front *)
let l3 = [1; 2] @ [3; 4]           (* append: [1; 2; 3; 4] : O(length of left) *)
let ps = [(7, 1); (4, 2)]          (* (int * int) list : a list of pairs *)
```

> **Separator trap.** List elements are separated by `;`, not `,`. `[1, 2]` is a list containing **one** pair `(1, 2)`, so its type is `(int * int) list`.

## 2. Pattern matching on lists

Every list function has the same skeleton: one clause for `[]` and one for `x :: xs`. Patterns can be nested, so `(a, b) :: rest` takes apart the first pair *and* the list in one go. This is exactly what MP5.1 needs.

```ocaml
let rec length lst =
  match lst with
  | [] -> 0                          (* base case *)
  | _ :: xs -> 1 + length xs         (* recursive case on the tail *)

let rec sum_pairs lst =
  match lst with
  | [] -> 0
  | (a, b) :: rest -> a + b + sum_pairs rest   (* nested pattern *)
```

## 3. Library functions you will actually use

| Function | Type | Use in MP5 |
|---|---|---|
| `l1 @ l2` | `'a list -> 'a list -> 'a list` | Set **union** (duplicates allowed; the grader tolerates them) |
| `List.filter p l` | `('a -> bool) -> 'a list -> 'a list` | Set **difference**: `List.filter (fun y -> y <> x) l` removes every `x` |
| `List.mem x l` | `'a -> 'a list -> bool` | Membership test (used inside `freshFor`) |
| `List.map f l` | `('a -> 'b) -> 'a list -> 'b list` | Transform each element |
| `List.fold_right f l b` | `('a -> 'b -> 'b) -> 'a list -> 'b -> 'b` | General recursion; `import_list` *could* be a fold |

```ocaml
let remove x l = List.filter (fun y -> y <> x) l
(* remove "x" ["x"; "y"; "x"; "z"] = ["y"; "z"] *)
```

> **Use `<>` and `=`.** As in MP1/MP2, use structural `=` and `<>`, never `==` or `!=`.

## 4. Tail recursion and accumulators (a bridge to CPS)

A call is a **tail call** if its result is returned directly, with no work left to do afterwards. `1 + length xs` is *not* a tail call, because the `+` is still waiting. An accumulator moves the pending work into an argument:

```ocaml
let rec sum_tr acc lst =
  match lst with
  | [] -> acc
  | x :: xs -> sum_tr (acc + x) xs   (* tail call: nothing left to do *)
(* sum_tr 0 [1; 2; 3; 4] = 10 *)
```

CPS, introduced in Part VI, generalises this idea. Instead of accumulating a *number*, you accumulate a *function* that describes the rest of the work.

**What this part buys you:** MP5.1 entirely, plus the union/difference idioms for MP5.4.

# Part II — Algebraic data types (user-defined types)

## 5. Declaring a variant type

A `type` declaration with `|` creates a new type whose values are built with **constructors**. Constructor names start with a capital letter. `of` says what data a constructor carries; a constructor with several pieces of data uses `*`.

```ocaml
type shape =
  | Circle of float                 (* carries one float *)
  | Rect of float * float           (* carries two floats *)
  | Dot                             (* carries nothing: a constant constructor *)

let s1 = Circle 2.0
let s2 = Rect (3.0, 4.0)            (* parentheses + comma: one tuple of arguments *)
let s3 = Dot
```

> **Constructor syntax rules.**
>
> - A constructor is **not** a curried function. Write `Rect (3.0, 4.0)`, never `Rect 3.0 4.0`.
> - Parenthesise a compound argument: `Circle (r *. 2.0)` and `ConstExp (IntConst 7)`. `ConstExp IntConst 7` is a type error.
> - Constructors are case-sensitive. `Intconst` is not `IntConst`, so copy names from the handout.

## 6. Pattern matching on variants

You consume a variant by matching it, with one clause per constructor. Patterns bind the carried data to names. Use `_` to ignore data you do not need.

```ocaml
let area s =
  match s with
  | Circle r -> 3.14159 *. r *. r
  | Rect (w, h) -> w *. h
  | Dot -> 0.0
```

If you forget a constructor, OCaml prints **Warning 8: this pattern-matching is not exhaustive** and gives an example of a missing case. A warning still compiles, but a missing case raises `Match_failure` at run time. That makes the grader's test fail. Read that warning: it is a free checklist.

## 7. Recursive and mutually recursive types

A type can mention itself. This is how trees and abstract syntax are represented.

```ocaml
type tree = Leaf of int | Node of tree * tree

let rec sum_tree t =
  match t with
  | Leaf n -> n
  | Node (l, r) -> sum_tree l + sum_tree r    (* recurse on every sub-tree *)
```

Two types that refer to each other are declared together with `and`. MP5's CPS types work this way: `cps_cont` mentions `exp_cps` (through `FnContCPS`), and `exp_cps` mentions `cps_cont` (in almost every constructor). Functions over them are written with `let rec ... and ...`, as in the supplied `freeVarsInContCPS` / `freeVarsInExpCPS`.

## 8. Built-in variants you already know

`bool` is a variant (`true | false`), and so is `'a list` (`[] | ::`). `'a option = None | Some of 'a` is the standard way to represent "maybe a value". Everything you learned about matching lists carries over to every other variant type.

**What this part buys you:** you can read the `exp`, `const`, `mon_op`, `bin_op`, `cps_cont` and `exp_cps` declarations, build their values, and match on them.

# Part III — Abstract syntax: programs as data

## 9. Concrete vs. abstract syntax

*Concrete syntax* is the text you type, such as `x + 1`. *Abstract syntax* is the tree a parser produces after it has resolved precedence, parentheses, and sugar. In MP5 there is no parser: **you are the parser**. You write values of type `exp` directly.

## 10. The PicoML `exp` type, constructor by constructor

| Source form | AST constructor | Example |
|---|---|---|
| variable `x` | `VarExp "x"` | `VarExp "lst"` |
| constant | `ConstExp c` | `ConstExp (IntConst 7)`, `ConstExp NilConst` for `[]`, `ConstExp (BoolConst true)` |
| unary op `⊕ e` | `MonOpAppExp (op, e)` | `hd lst` → `MonOpAppExp (HdOp, VarExp "lst")` |
| binary op `e1 ⊕ e2` | `BinOpAppExp (op, e1, e2)` | `lst = []` → `BinOpAppExp (EqOp, VarExp "lst", ConstExp NilConst)` |
| `if e1 then e2 else e3` | `IfExp (e1, e2, e3)` | |
| application `e1 e2` | `AppExp (e1, e2)` | `f x` → `AppExp (VarExp "f", VarExp "x")` |
| `fun x -> e` | `FunExp ("x", e)` | |
| `let x = e1 in e2` | `LetInExp ("x", e1, e2)` | |
| `let rec f x = e1 in e2` | `LetRecInExp ("f", "x", e1, e2)` | |

## 11. PicoML facts that trip people up

> **`hd`, `tl`, `fst`, `snd` and unary minus are operators, not functions.** In PicoML they are **`mon_op`s**, so `hd lst` is `MonOpAppExp (HdOp, VarExp "lst")`. It is **not** `AppExp (VarExp "hd", VarExp "lst")`. This is the most common MP5.2 mistake.

- **Tuples are a binary operator.** `(7, 1)` is `BinOpAppExp (CommaOp, ConstExp (IntConst 7), ConstExp (IntConst 1))`.
- **Lists have no literal form.** `[]` is the constant `ConstExp NilConst`. `[a; b]` must be written out as `ConsOp` applications, nested to the right: `BinOpAppExp (ConsOp, a, BinOpAppExp (ConsOp, b, ConstExp NilConst))`.
- **Comparison is a binary operator.** `lst = []` uses `EqOp`; `x > y` uses `GreaterOp`. There is no `<`; write `y > x`.
- **Integer vs. float operators are different constructors:** `IntPlusOp` vs. `FloatPlusOp`.
- **Multi-argument functions are curried.** `fun x y -> e` is `FunExp ("x", FunExp ("y", e))`, and `f a b` is `AppExp (AppExp (f, a), b)`, because application associates left.
- **`let f x = e1 in e2` (non-recursive) is sugar** for `LetInExp ("f", FunExp ("x", e1), e2)`.
- **Strings name variables.** The `string` in `VarExp`, `FunExp`, `LetInExp` and `LetRecInExp` is the variable's name. It is an ordinary OCaml string, so it needs quotation marks.

## 12. A method for translating any program into an AST

1. **Find the outermost construct.** `let ... in` and `if` extend as far right as possible, so the outermost construct is usually the first keyword. Otherwise it is the operator with the *lowest* precedence. In order from loosest to tightest: `,` then `::` / `@` then `=` / `>` then `+` / `-` / `^` then `*` / `/` then application then `hd` / `fst` / etc.
2. **Write that constructor with holes**, for example `LetInExp ("x", _, _)`.
3. **Fill each hole by repeating** the method on the sub-expression.
4. **Count parentheses as you close each constructor.** Line breaks and indentation that mirror the tree make this much easier.

Worked example: `let f = fun y -> y * 2 in f (f 3)`

```ocaml
LetInExp ("f",
  FunExp ("y", BinOpAppExp (IntTimesOp, VarExp "y", ConstExp (IntConst 2))),
  AppExp (VarExp "f", AppExp (VarExp "f", ConstExp (IntConst 3))))
```

`string_of_exp` turns an AST back into text, so use it to check your tree. Its output fully parenthesises sub-expressions, for example `((fst x) + (snd x)) :: (pair_sums (tl lst))`. Compare it with the handout's expected string.

**What this part buys you:** MP5.2 entirely, and the ability to read every test case in MP5.3–5.5.

# Part IV — Recursion over an AST (the universal template)

## 13. One clause per constructor, recurse on every sub-expression

Almost every function on `exp` has this shape. Look at what each constructor carries. Recurse on every field of type `exp`, ignore or inspect the other fields (strings, operators, constants), and combine the results.

```ocaml
let rec f e =
  match e with
  | VarExp x -> (* leaf *) ...
  | ConstExp c -> (* leaf *) ...
  | MonOpAppExp (op, e1) -> ... f e1 ...
  | BinOpAppExp (op, e1, e2) -> ... f e1 ... f e2 ...
  | IfExp (e1, e2, e3) -> ... f e1 ... f e2 ... f e3 ...
  | AppExp (e1, e2) -> ... f e1 ... f e2 ...
  | FunExp (x, e1) -> ... f e1 ...
  | LetInExp (x, e1, e2) -> ... f e1 ... f e2 ...
  | LetRecInExp (g, x, e1, e2) -> ... f e1 ... f e2 ...
```

Memorise the **sub-expression count** for each constructor: Var 0, Const 0, MonOp 1, BinOp 2, If 3, App 2, Fun 1, Let 2, LetRec 2. Missing a sub-expression is the classic bug, for example forgetting the `e1` of a `let`.

## 14. Or-patterns save typing

When several constructors are handled identically, one clause can match them all. Every alternative must bind the **same names with the same types**:

```ocaml
let rec count_var_occurrences e =
  match e with
  | VarExp _ -> 1
  | ConstExp _ -> 0
  | MonOpAppExp (_, e1) | FunExp (_, e1) -> count_var_occurrences e1
  | BinOpAppExp (_, e1, e2) | AppExp (e1, e2)
  | LetInExp (_, e1, e2) | LetRecInExp (_, _, e1, e2) ->
      count_var_occurrences e1 + count_var_occurrences e2
  | IfExp (e1, e2, e3) ->
      count_var_occurrences e1 + count_var_occurrences e2 + count_var_occurrences e3
```

> **Quiz advice.** Or-patterns are optional. Writing all nine clauses separately is safer under stress, and it is only a few more lines.

**What this part buys you:** MP5.3 entirely, and the skeleton of MP5.4 and MP5.5.

# Part V — Scope, binding, and free variables

## 15. Binding occurrences, scope, and bound occurrences

A **binding occurrence** introduces a name: the `x` in `fun x ->`, `let x =`, or `let rec f x =`. Its **scope** is the region of code where uses of that name refer to it. A use of the name inside that scope is a **bound occurrence**. A use that is not inside the scope of any binder for that name is a **free occurrence**. Free variables are the ones that must already have a value in the environment before the expression can run.

| Construct | Binds | Scope of the binding |
|---|---|---|
| `fun x -> e` | `x` | `e` |
| `let x = e1 in e2` | `x` | **only `e2`**, not `e1` |
| `let rec f x = e1 in e2` | `f` and `x` | `f`: **both** `e1` and `e2`; `x`: **only** `e1` |

The `let` row is the subtle one. In `let x = x + 1 in x`, the `x` in `x + 1` is **free**, because it refers to some outer `x`. The final `x` is bound.

## 16. The definition of FV, case by case

These equations are the precise rules the MP5.4 handout gives, written as math. Each is a direct recipe for one `match` clause.

```text
FV(x)                       = {x}
FV(c)                       = { }
FV(⊕ e)                     = FV(e)
FV(e1 ⊕ e2)                 = FV(e1) ∪ FV(e2)
FV(if e1 then e2 else e3)   = FV(e1) ∪ FV(e2) ∪ FV(e3)
FV(e1 e2)                   = FV(e1) ∪ FV(e2)
FV(fun x -> e)              = FV(e) − {x}
FV(let x = e1 in e2)        = FV(e1) ∪ (FV(e2) − {x})
FV(let rec f x = e1 in e2)  = (FV(e1) − {f, x}) ∪ (FV(e2) − {f})
```

Sets are represented as lists. `∪` becomes `@`, and `S − {x}` becomes `List.filter (fun y -> y <> x) S`. The grader accepts duplicates and any order.

## 17. Hand examples (do these before coding)

| Expression | Free variables | Why |
|---|---|---|
| `let x = y in fun s -> a x s` | `{y, a}` | `x` is bound by `let`, `s` by `fun` (handout example) |
| `fun x -> x + y` | `{y}` | `x` removed |
| `let x = x in x + z` | `{x, z}` | The **first** `x` (in `e1`) is free |
| `(fun x -> x) x` | `{x}` | The argument `x` lies outside the function's scope |
| `let rec g n = g m in g n` | `{m, n}` | `n` is bound only in `e1`; in `e2` it is free |
| `let f = fun a -> a b in f f` | `{b}` | |
| `let rec f x = f x in f y` | `{y}` | Handout extra-credit example |

**What this part buys you:** MP5.4 entirely, and the `freshFor` arguments in MP5.5.

# Part VI — Continuations and continuation-passing style (the week 4 material)

## 18. What a continuation is

At any point during evaluation there is "the rest of the computation", meaning what will be done with the value currently being computed. In `1 + f 3`, when `f 3` returns a value `r`, the rest of the computation is "add 1 to `r`, then give that to whoever asked". A **continuation** is that "rest of the computation" written as a function: `fun r -> ... (1 + r) ...`.

## 19. Direct style vs. continuation-passing style

In **direct style** a function *returns* its result to its caller. In **continuation-passing style** a function takes an extra argument `k`, the continuation, and *passes* its result to `k` instead of returning it.

```ocaml
(* Direct style *)
let add (a, b) = a + b

(* CPS: take a continuation k and pass the result to it *)
let addk (a, b) k = k (a + b)
let subk (a, b) k = k (a - b)
let timesk (a, b) k = k (a * b)
let eqk (a, b) k = k (a = b)
let gtk (a, b) k = k (a > b)
let consk (x, xs) k = k (x :: xs)

let report n = print_string "Result: "; print_int n; print_newline ()
(* addk (2, 3) report   prints   Result: 5 *)
```

Three properties define CPS code. The course lists them as the reasons CPS matters:

1. **Every call is a tail call.** Nothing waits after a call returns, so no stack frame needs to survive. All the "waiting work" lives inside continuations.
2. **Every intermediate result gets a name.** It is the parameter of some continuation (`fun r -> ...`).
3. **Evaluation order becomes explicit.** The nesting shows exactly what is computed first.

## 20. Converting expressions into CPS by hand

Recipe: **evaluate innermost first, name the result with a continuation parameter, and put the rest inside.** PicoML evaluates **right to left**: the right operand before the left, and the argument before the function. MP5.5 uses that order, so practise it here too.

```ocaml
(* direct: (x + 2) * (y - 1)          right operand (y - 1) first *)
let exprk x y k =
  subk (y, 1) (fun b ->               (* b = y - 1 *)
  addk (x, 2) (fun a ->               (* a = x + 2 *)
  timesk (a, b) k))                   (* pass a * b to k *)
(* exprk 3 5 report  prints  Result: 20 *)
```

## 21. Converting recursive functions into CPS

Add a `k` parameter. In a base case, **pass** the answer to `k`. In a recursive case, make the recursive call with a **new** continuation that does the leftover work and then calls the original `k`.

```ocaml
let rec fact n = if n = 0 then 1 else n * fact (n - 1)

let rec factk n k =
  eqk (n, 0) (fun b ->
    if b then k 1                                  (* base: pass 1 to k *)
    else subk (n, 1) (fun m ->                     (* m = n - 1 *)
           factk m (fun r ->                       (* r = fact m *)
             timesk (n, r) k)))                    (* pass n * r to k *)
(* factk 5 report   prints   Result: 120 *)

let rec sumk lst k =
  match lst with
  | [] -> k 0
  | x :: xs -> sumk xs (fun r -> addk (x, r) k)
(* sumk [1; 2; 3; 4] report   prints   Result: 10 *)

let rec lengthk lst k =
  match lst with
  | [] -> k 0
  | _ :: xs -> lengthk xs (fun r -> addk (1, r) k)
```

With **two** recursive calls (Fibonacci), nest the second call inside the continuation of the first:

```ocaml
let rec fibk n k =
  gtk (2, n) (fun small ->
    if small then k 1
    else subk (n, 1) (fun n1 -> fibk n1 (fun r1 ->
         subk (n, 2) (fun n2 -> fibk n2 (fun r2 ->
         addk (r1, r2) k)))))
(* fibk 7 report   prints   Result: 21 *)
```

A higher-order function in CPS takes a CPS function as its argument (`fk : 'a -> ('b -> 'r) -> 'r`):

```ocaml
let rec mapk fk lst k =
  match lst with
  | [] -> k []
  | x :: xs -> mapk fk xs (fun rest -> fk x (fun y -> consk (y, rest) k))
```

## 22. Tracing a CPS call

```text
sumk [1; 2] report
= sumk [2] (fun r1 -> addk (1, r1) report)
= sumk []  (fun r2 -> addk (2, r2) (fun r1 -> addk (1, r1) report))
= (fun r2 -> addk (2, r2) (...)) 0
= addk (2, 0) (fun r1 -> addk (1, r1) report)
= addk (1, 2) report
= report 3                        prints Result: 3
```

The continuation **grows** as recursion goes down, and it then runs as the "unwinding". The stack has become a data structure, a chain of closures.

**What this part buys you:** the intuition for every rule in MP5.5. The automatic transformation does to *any* PicoML expression exactly what you just did by hand.

# Part VII — The CPS abstract syntax in MP5

MP5.5 does not produce OCaml code. It produces a value of the type `exp_cps`, which is abstract syntax for CPS programs. You need to read three types and one printed notation.

## 23. The types

```ocaml
type cont_var = Kvar                      (* the one continuation variable, printed _k *)

type cps_cont =
   External                               (* top-level "print the answer" continuation *)
 | ContVarCPS of cont_var                 (* _k : the continuation parameter of a function *)
 | FnContCPS of string * exp_cps          (* FN x -> exp_cps : a continuation we build *)

and exp_cps =
   VarCPS of cps_cont * string                          (* K x *)
 | ConstCPS of cps_cont * const                         (* K c *)
 | MonOpAppCPS of cps_cont * mon_op * string            (* K (% x) *)
 | BinOpAppCPS of cps_cont * bin_op * string * string   (* K (x % y) *)
 | IfCPS of string * exp_cps * exp_cps                  (* IF x THEN e1 ELSE e2 *)
 | AppCPS of cps_cont * string * string                 (* x y K *)
 | FunCPS of cps_cont * string * cont_var * exp_cps     (* K (FUN x _k -> e) *)
 | FixCPS of cps_cont * string * string * cont_var * exp_cps
                                          (* K (FIX f. FUN x _k -> e) *)
```

## 24. How to read the types

> **Read "K x" as "hand x to continuation K".** Every `exp_cps` constructor except `IfCPS` starts with a `cps_cont`. That is the continuation that receives the result of this step. Where `exp` had sub-expressions, `exp_cps` has only **variable names** (`string`). In CPS, every operand must already be computed and named before an operation runs. The two exceptions keep a full `exp_cps` for a reason. `IfCPS` must not evaluate either branch before testing. `FunCPS` / `FixCPS` must not run the body before the function is called.

| Printed form | Constructor | Meaning |
|---|---|---|
| `_k` | `ContVarCPS Kvar` | The continuation passed into the current function |
| `(FN a -> body)` | `FnContCPS ("a", body)` | A continuation: "call the incoming value `a`, then do `body`" |
| `_k x` | `VarCPS (ContVarCPS Kvar, "x")` | Pass `x` to `_k` |
| `(FN a -> ...) 1` | `ConstCPS (FnContCPS ("a", ...), IntConst 1)` | Pass the constant 1 to that continuation, which binds `a = 1` |
| `_k(b + a)` | `BinOpAppCPS (ContVarCPS Kvar, IntPlusOp, "b", "a")` | Compute `b + a`, pass it to `_k` |
| `(b a _k)` | `AppCPS (ContVarCPS Kvar, "b", "a")` | Call function `b` on argument `a` **and** continuation `_k` |
| `_k (FUN x _k -> body)` | `FunCPS (ContVarCPS Kvar, "x", Kvar, body)` | Pass a CPS function value to `_k` |
| `(FN f -> e2)(FIX f. FUN x _k -> e1)` | `FixCPS (FnContCPS ("f", e2), "f", "x", Kvar, e1)` | Build a recursive function and pass it to `FN f -> e2` |

**FN vs. FUN.** `FN` (`FnContCPS`) is a **continuation**: it takes one value and never takes a `_k`. `FUN` (`FunCPS`) is a **CPS-converted user function**: it takes an argument *and* a continuation `_k`. When a test prints `(FN a -> ...)`, you need `FnContCPS`. When it prints `FUN x _k`, you need `FunCPS`.

## 25. The helper functions in `common.ml`

| Helper | Type | Use |
|---|---|---|
| `freshFor` | `string list -> string` | Returns a name **not** in the list. In practice it returns `"a"`, `"b"`, `"c"`, … in order, skipping names in the list. |
| `freeVarsInContCPS` | `cps_cont -> string list` | Free variables of a continuation (`External` and `_k` have none) |
| `freeVarsInExpCPS` | `exp_cps -> string list` | Free variables of a CPS expression |
| `string_of_exp_cps` | `exp_cps -> string` | Printer used in all the handout examples |
| `string_of_exp`, `print_exp` | `exp -> string`, `exp -> unit` | Printers for the source AST |

The handout's outputs, such as `(FN a -> (FN b -> (b a _k)) f) x`, show `freshFor` returning `a` and then `b`. Your names do not have to match the reference solution's names. They only have to satisfy the freshness constraints (handout: *"the names you use for these fresh variables do not have to be the same as the ones we use"*). If you use `freshFor` exactly as in Part IX, you will usually get identical output anyway, which makes comparing with the examples easy.

# Part VIII — The CPS transformation, rule by rule

## 26. Notation

`[[e]]κ` means "the CPS form of expression `e`, which delivers its value to continuation `κ`". In code this is `cps_exp e k`, where `k : cps_cont`. `κ` is not a program variable. It is a whole continuation: `_k`, `External`, or a built-up `FN v -> ...`. The variables `v`, `v1`, `v2` are **new** names introduced to carry intermediate values. The variable `x` is an existing program variable.

## 27. The nine rules

```text
[[x]]κ  = κ x
[[c]]κ  = κ c
[[⊕ e]]κ = [[e]]_(FN v -> κ (⊕ v))                         v fresh for κ
[[e1 ⊕ e2]]κ = [[e2]]_(FN v2 -> [[e1]]_(FN v1 -> κ (v1 ⊕ v2)))
                                     v2 fresh for e1, κ;  v1 fresh for κ and v2
[[if e1 then e2 else e3]]κ
        = [[e1]]_(FN v -> IF v THEN [[e2]]κ ELSE [[e3]]κ)     v fresh for e2, e3, κ
[[e1 e2]]κ = [[e2]]_(FN v2 -> [[e1]]_(FN v1 -> v1 v2 κ))
                                     v2 fresh for e1, κ;  v1 fresh for κ and v2
[[fun x -> e]]κ = κ (FUN x _k -> [[e]]_k)
[[let x = e1 in e2]]κ = [[e1]]_(FN x -> [[e2]]κ)
[[let rec f x = e1 in e2]]κ = (FN f -> [[e2]]κ) (FIX f. FUN x _k -> [[e1]]_k)
```

## 28. Reading each rule

- **Variable / constant.** Already a value, so hand it straight to `κ`.
- **Unary operator.** First compute `e`, catching its value as `v`. Then hand `⊕ v` to `κ`.
- **Binary operator / application.** Evaluate **right to left**: first `e2`, caught as `v2`; then `e1`, caught as `v1`; then do the operation. Notice that the transformation **recurses on `e2` on the outside**. The *first* thing evaluated is the *outermost* call, because its continuation contains everything that happens later.
- **If.** Evaluate the guard `e1`, catching the boolean as `v`. Then `IF v THEN ... ELSE ...`, where each branch is transformed with the **original** `κ`. That is why `IfCPS` carries no continuation of its own: the continuation is "buried one step down in each branch".
- **Function.** Do not evaluate the body. Hand `κ` a CPS function that takes `x` **and** a continuation `_k`. Its body is transformed with `_k`, because when the function is called its result goes to *the caller's* continuation, which arrives as `_k`.
- **Let.** The continuation that receives `e1`'s value **is** the binding: `FN x -> [[e2]]κ`. No fresh name is needed, because `x` is the program's own variable.
- **Let rec.** As for `fun`, transform the body `e1` with `_k`. Wrap it with `FIX` so that it can refer to itself, then hand it to the continuation `FN f -> [[e2]]κ`.

## 29. Why freshness matters (variable capture)

A continuation `FN v -> body` **binds** `v` inside `body`. If `body` mentions a program variable that happens to have the same name, the new binding **captures** it, and the program silently computes the wrong thing. Take `[[a b]]_k`. If we chose `v2 = "a"`, the result would be `(FN a -> [[a]]_(FN v1 -> v1 a _k)) b`. The inner `[[a]]` now reads the *argument's* value instead of the function `a`. That is wrong. `freshFor (freeVarsInExp e1 @ freeVarsInContCPS k)` avoids every name that could be captured. For `a b` it picks `v2 = "b"`. That shadows the program's `b` only *after* `b` has already been evaluated, which is harmless:

```text
[[a b]]_k = (FN b -> (FN a -> (a b _k)) a) b
```

Why the side conditions are what they are:

| Fresh variable | Must avoid | Because its scope covers |
|---|---|---|
| `v` (if) | FV(e2), FV(e3), FV(κ) | Both transformed branches, which contain κ |
| `v2` (binop/app) | FV(e1), FV(κ) | `[[e1]]...` and κ |
| `v1` (binop/app) | `v2`, FV(κ) | Only `κ (v1 ⊕ v2)`; `e1` has already been evaluated, so FV(e1) does not matter |
| `v` (unary) | FV(κ) | Only `κ (⊕ v)` |

> **DO MP5 AS GIVEN: an example of why.** The `let` rule binds the program variable `x` around `[[e2]]κ` without checking whether `x` is free in `κ`. In `let x = 5 in x + (let x = 1 in x)` the outer `x` is pending inside κ, and the rule captures it. Running the CPS output gives 2 where the original gives 6. This is a known simplification of the rule. **Do not "fix" it.** The grader compares your output with the reference solution's output, so implement each rule exactly as stated.

# Part IX — MP5 solved, problem by problem

> **Before you start any MP5 problem.** Keep the skeleton's first lines (typically `open Common`) so that `exp`, `freshFor`, etc. are in scope. Run `make` once before editing to confirm the workspace builds. Save, `make`, and `./grader` after each clause. A clause that compiles is worth more than a perfect function that does not.

## MP5.1 — `import_list : (int * int) list -> exp`

**Contract.** Turn an OCaml list of integer pairs into the PicoML expression that *denotes* that list.

**Derivation.** Recurse on the OCaml list (Part I). The empty list becomes the PicoML constant `[]`, which is `ConstExp NilConst`. A pair `(x, y)` at the head becomes the PicoML tuple `x , y`, which is `BinOpAppExp (CommaOp, ...)`. That tuple is consed onto the translation of the rest with `ConsOp`. Integers must be wrapped twice: `ConstExp (IntConst x)`.

```ocaml
let rec import_list lst =
  match lst with
  | [] -> ConstExp NilConst
  | (x, y) :: rest ->
      BinOpAppExp (ConsOp,
        BinOpAppExp (CommaOp, ConstExp (IntConst x), ConstExp (IntConst y)),
        import_list rest)
```

**Tests.** `import_list [(7,1);(4,2);(6,3)]` gives exactly the handout's tree. `string_of_exp` prints it as `((7,1)) :: (((4,2)) :: (((6,3)) :: []))`. `import_list []` = `ConstExp NilConst`.

> **Mistakes to avoid.** Writing `IntConst x` without `ConstExp` (type error: `const` is not `exp`). Putting `ConstExp NilConst` anywhere except the very end. Swapping `x` and `y`. Forgetting `rec`.

## MP5.2 — `pair_sums : exp`

**Contract.** A *value* (not a function) that is the AST of:

```ocaml
let rec pair_sums lst =
      if lst = [] then []
      else let x = (hd lst) in (fst x + snd x) :: (pair_sums (tl lst))
 in pair_sums [(7,1);(4,2);(6,3)]
```

**Derivation using the Part III method.**

1. Outermost: `let rec pair_sums lst = BODY in CALL`, so `LetRecInExp ("pair_sums", "lst", BODY, CALL)`.
2. `BODY` = `if lst = [] then [] else REST`, so `IfExp (BinOpAppExp (EqOp, VarExp "lst", ConstExp NilConst), ConstExp NilConst, REST)`.
3. `REST` = `let x = hd lst in CONS`, so `LetInExp ("x", MonOpAppExp (HdOp, VarExp "lst"), CONS)`.
4. `CONS` = `SUM :: REC`. The loosest operator is `::`, so `BinOpAppExp (ConsOp, SUM, REC)`.
5. `SUM` = `fst x + snd x`, so `BinOpAppExp (IntPlusOp, MonOpAppExp (FstOp, VarExp "x"), MonOpAppExp (SndOp, VarExp "x"))`.
6. `REC` = `pair_sums (tl lst)`, so `AppExp (VarExp "pair_sums", MonOpAppExp (TlOp, VarExp "lst"))`.
7. `CALL` = `pair_sums [(7,1);(4,2);(6,3)]`, so `AppExp (VarExp "pair_sums", L)`, where `L` is exactly `import_list [(7,1);(4,2);(6,3)]` from MP5.1, written out by hand.

```ocaml
let pair_sums =
  LetRecInExp ("pair_sums", "lst",
    IfExp (BinOpAppExp (EqOp, VarExp "lst", ConstExp NilConst),
           ConstExp NilConst,
           LetInExp ("x", MonOpAppExp (HdOp, VarExp "lst"),
             BinOpAppExp (ConsOp,
               BinOpAppExp (IntPlusOp,
                 MonOpAppExp (FstOp, VarExp "x"),
                 MonOpAppExp (SndOp, VarExp "x")),
               AppExp (VarExp "pair_sums", MonOpAppExp (TlOp, VarExp "lst"))))),
    AppExp (VarExp "pair_sums",
      BinOpAppExp (ConsOp,
        BinOpAppExp (CommaOp, ConstExp (IntConst 7), ConstExp (IntConst 1)),
        BinOpAppExp (ConsOp,
          BinOpAppExp (CommaOp, ConstExp (IntConst 4), ConstExp (IntConst 2)),
          BinOpAppExp (ConsOp,
            BinOpAppExp (CommaOp, ConstExp (IntConst 6), ConstExp (IntConst 3)),
            ConstExp NilConst)))))
```

**Check.** `string_of_exp pair_sums` must read `let rec pair_sums lst = if lst = [] then [] else let x = hd lst in ((fst x) + (snd x)) :: (pair_sums (tl lst)) in pair_sums (((7,1)) :: (((4,2)) :: (((6,3)) :: [])))`. The handout's line breaks are only display. With `eval.cmo` you can run `print_eval pair_sums` and expect the list `[8; 6; 9]`.

> **Mistakes to avoid.** `AppExp (VarExp "hd", ...)` instead of `MonOpAppExp (HdOp, ...)`. `FloatPlusOp` instead of `IntPlusOp`. Swapping the order in `lst = []`: the AST must be `EqOp, VarExp "lst", ConstExp NilConst`. Missing a closing parenthesis: count them per line. Writing `let rec pair_sums = ...` (it is not a function, so no `rec` and no parameter). Typos in the strings `"pair_sums"`, `"lst"`, `"x"`: they are compared exactly.

## MP5.3 — `count_const_in_exp : exp -> int`

**Contract.** The number of `ConstExp` nodes in the tree. Every occurrence counts, including repeats.

**Derivation.** Use the Part IV template. Leaves: `ConstExp` counts 1 (whatever constant it holds: `NilConst` and `UnitConst` count too), and `VarExp` counts 0. Every other constructor adds up the counts of its sub-expressions. The strings in `FunExp` / `LetInExp` / `LetRecInExp` are names, not constants, so they contribute nothing.

```ocaml
let rec count_const_in_exp exp =
  match exp with
  | VarExp _ -> 0
  | ConstExp _ -> 1
  | MonOpAppExp (_, e) -> count_const_in_exp e
  | BinOpAppExp (_, e1, e2) -> count_const_in_exp e1 + count_const_in_exp e2
  | IfExp (e1, e2, e3) ->
      count_const_in_exp e1 + count_const_in_exp e2 + count_const_in_exp e3
  | AppExp (e1, e2) -> count_const_in_exp e1 + count_const_in_exp e2
  | FunExp (_, e) -> count_const_in_exp e
  | LetInExp (_, e1, e2) -> count_const_in_exp e1 + count_const_in_exp e2
  | LetRecInExp (_, _, e1, e2) -> count_const_in_exp e1 + count_const_in_exp e2
```

**Tests.** The handout example gives 7: `7.3`, `()`, `4`, `"a"`, `6`, `"b"`, `[]`. `count_const_in_exp pair_sums` = 9: the two `[]` in the body, then `7 1 4 2 6 3` and the final `[]`. `count_const_in_exp (VarExp "x")` = 0.

> **Mistakes to avoid.** Forgetting the third branch of `IfExp`. Returning 0 for `NilConst` or `UnitConst`: they are constants. Forgetting `e1` in `LetInExp` or `LetRecInExp`.

## MP5.4 — `freeVarsInExp : exp -> string list`

**Contract.** The names of the free variables, as a list. Duplicates and order do not matter. Translate each equation from §16 into one clause.

```ocaml
let rec freeVarsInExp exp =
  match exp with
  | VarExp x -> [x]                                         (* 1 pt *)
  | ConstExp _ -> []                                        (* 1 pt *)
  | MonOpAppExp (_, e) -> freeVarsInExp e                   (* 2 pts *)
  | BinOpAppExp (_, e1, e2) -> freeVarsInExp e1 @ freeVarsInExp e2   (* 2 pts *)
  | IfExp (e1, e2, e3) ->                                   (* 2 pts *)
      freeVarsInExp e1 @ freeVarsInExp e2 @ freeVarsInExp e3
  | AppExp (e1, e2) -> freeVarsInExp e1 @ freeVarsInExp e2  (* 2 pts *)
  | FunExp (x, e) ->                                        (* 3 pts *)
      List.filter (fun y -> y <> x) (freeVarsInExp e)
  | LetInExp (x, e1, e2) ->                                 (* 3 pts *)
      freeVarsInExp e1 @ List.filter (fun y -> y <> x) (freeVarsInExp e2)
  | LetRecInExp (f, x, e1, e2) ->                           (* 4 pts extra credit *)
      List.filter (fun y -> y <> f && y <> x) (freeVarsInExp e1)
      @ List.filter (fun y -> y <> f) (freeVarsInExp e2)
```

**Tests (all from the handout).**

| Call | Expected |
|---|---|
| `freeVarsInExp (VarExp "x")` | `["x"]` |
| `List.length (freeVarsInExp (ConstExp (StringConst "hi")))` | `0` |
| `freeVarsInExp (MonOpAppExp (IntNegOp, VarExp "x"))` | `["x"]` |
| `freeVarsInExp (BinOpAppExp (IntTimesOp, VarExp "x", ConstExp (IntConst 3)))` | `["x"]` |
| `freeVarsInExp (IfExp (ConstExp (BoolConst true), VarExp "x", VarExp "y"))` | `["x"; "y"]` |
| `freeVarsInExp (AppExp (VarExp "f", VarExp "y"))` | `["f"; "y"]` |
| `freeVarsInExp (FunExp ("x", VarExp "x"))` | `[]` |
| `freeVarsInExp (LetInExp ("x", VarExp "y", VarExp "x"))` | `["y"]` |
| `freeVarsInExp (LetRecInExp ("f", "x", AppExp (VarExp "f", VarExp "x"), AppExp (VarExp "f", VarExp "y")))` | `["y"]` |

> **Mistakes to avoid.**
>
> - **`let`: removing `x` from `e1` as well.** `let x = x in ...` has `x` free.
> - **`let rec`: removing `x` from `e2`.** The parameter's scope is only the function body.
> - **`let rec`: forgetting to remove `f` from `e1`.** The function is recursive, so `f` is bound in its own body.
> - **Filtering with a single `<>` but two names.** The condition must be `y <> f && y <> x`, not `||`.
> - **Using `List.filter (fun y -> y = x)`.** That *keeps* only `x`, the opposite of what you want.

## MP5.5 — `cps_exp : exp -> cps_cont -> exp_cps`

**Contract.** Implement the nine rules of §27 literally. In each rule, `[[e]]κ` becomes `cps_exp e k`. A continuation `FN v -> ...` becomes `FnContCPS (v, ...)`. `κ v` becomes `VarCPS (k, v)`, `ConstCPS (k, c)`, `MonOpAppCPS (k, op, v)` or `BinOpAppCPS (k, op, v1, v2)`, depending on what is being handed over.

```ocaml
let rec cps_exp e k =
  match e with
  (* [[x]]k = k x *)
  | VarExp x -> VarCPS (k, x)

  (* [[c]]k = k c *)
  | ConstExp c -> ConstCPS (k, c)

  (* [[if e1 then e2 else e3]]k = [[e1]]_(FN v -> IF v THEN [[e2]]k ELSE [[e3]]k) *)
  | IfExp (e1, e2, e3) ->
      let v = freshFor (freeVarsInExp e2 @ freeVarsInExp e3 @ freeVarsInContCPS k) in
      cps_exp e1 (FnContCPS (v, IfCPS (v, cps_exp e2 k, cps_exp e3 k)))

  (* [[e1 e2]]k = [[e2]]_(FN v2 -> [[e1]]_(FN v1 -> v1 v2 k)) *)
  | AppExp (e1, e2) ->
      let v2 = freshFor (freeVarsInExp e1 @ freeVarsInContCPS k) in
      let v1 = freshFor (v2 :: freeVarsInContCPS k) in
      cps_exp e2 (FnContCPS (v2,
        cps_exp e1 (FnContCPS (v1, AppCPS (k, v1, v2)))))

  (* [[e1 op e2]]k = [[e2]]_(FN v2 -> [[e1]]_(FN v1 -> k (v1 op v2))) *)
  | BinOpAppExp (op, e1, e2) ->
      let v2 = freshFor (freeVarsInExp e1 @ freeVarsInContCPS k) in
      let v1 = freshFor (v2 :: freeVarsInContCPS k) in
      cps_exp e2 (FnContCPS (v2,
        cps_exp e1 (FnContCPS (v1, BinOpAppCPS (k, op, v1, v2)))))

  (* [[op e]]k = [[e]]_(FN v -> k (op v)) *)
  | MonOpAppExp (op, e1) ->
      let v = freshFor (freeVarsInContCPS k) in
      cps_exp e1 (FnContCPS (v, MonOpAppCPS (k, op, v)))

  (* [[fun x -> e]]k = k (FUN x _k -> [[e]]_k) *)
  | FunExp (x, e1) ->
      FunCPS (k, x, Kvar, cps_exp e1 (ContVarCPS Kvar))

  (* [[let x = e1 in e2]]k = [[e1]]_(FN x -> [[e2]]k) *)
  | LetInExp (x, e1, e2) ->
      cps_exp e1 (FnContCPS (x, cps_exp e2 k))

  (* [[let rec f x = e1 in e2]]k = (FN f -> [[e2]]k)(FIX f. FUN x _k -> [[e1]]_k) *)
  | LetRecInExp (f, x, e1, e2) ->
      FixCPS (FnContCPS (f, cps_exp e2 k), f, x, Kvar,
              cps_exp e1 (ContVarCPS Kvar))
```

**Tests (all from the handout, reproduced exactly).**

| Input (with `k = ContVarCPS Kvar`) | `string_of_exp_cps` output |
|---|---|
| `VarExp "x"` | `_k x` |
| `ConstExp (BoolConst true)` | `_k true` |
| `IfExp (VarExp "b", ConstExp (IntConst 2), ConstExp (IntConst 5))` | `(FN a -> IF a THEN _k 2 ELSE _k 5) b` |
| `AppExp (VarExp "f", VarExp "x")` | `(FN a -> (FN b -> (b a _k)) f) x` |
| `BinOpAppExp (IntPlusOp, ConstExp (IntConst 5), ConstExp (IntConst 1))` | `(FN a -> (FN b -> _k(b + a)) 5) 1` |
| `MonOpAppExp (HdOp, ConstExp NilConst)` | `(FN a -> _k(hd a)) []` |
| `FunExp ("x", VarExp "x")` | `_k (FUN x _k -> _k x)` |
| `LetInExp ("x", ConstExp (IntConst 2), VarExp "x")` | `(FN x -> _k x) 2` |
| `LetRecInExp ("f", "x", VarExp "x", ConstExp (IntConst 4))` | `(FN f -> _k 4)(FIX f. FUN x _k -> _k x)` |

### Rule-by-rule checklist for the clauses that lose the most points

- **App / BinOp order.** The **outer** recursive call is on **`e2`** (the right operand, or the argument), and the inner one is on `e1`. In the final node the names are in the order **`v1, v2`**: `AppCPS (k, v1, v2)` means "call `v1` (the function) on `v2` (the argument)". `BinOpAppCPS (k, op, v1, v2)` computes `v1 op v2`. In the printed example `(FN a -> (FN b -> _k(b + a)) 5) 1`: `a` = `v2` holds 1 (the right operand) and `b` = `v1` holds 5. The output `b + a` is `5 + 1`, which is correct.
- **`v1`'s freshness list is `v2 :: freeVarsInContCPS k`.** If you forget `v2`, then `freshFor` may return the same name twice, giving `(FN a -> (FN a -> (a a _k)) f) x`. That is wrong and fails the test.
- **If.** The continuation `k` goes into **both** branches, and the `IfCPS` itself gets no continuation. The freshness list contains `e2`, `e3` and `k`. It does *not* contain `e1`: `e1` is evaluated before `v` exists.
- **Fun.** The body is transformed with **`ContVarCPS Kvar`**, not with `k`. The original `k` receives the function value. The third field of `FunCPS` is the `cont_var` **`Kvar`** itself, not `ContVarCPS Kvar`.
- **Let.** No `freshFor` is needed. The bound variable `x` is the parameter of the new `FnContCPS`.
- **Let rec.** The **first** field of `FixCPS` is the continuation `FnContCPS (f, cps_exp e2 k)`. The body `e1` is transformed with `ContVarCPS Kvar`. The field order is `(cont, f, x, Kvar, body)`.

> **Type-error decoder for MP5.5.**
>
> - *"This expression has type cont_var but an expression was expected of type cps_cont"*: you wrote `Kvar` where a continuation is needed. Use `ContVarCPS Kvar`.
> - The opposite error, in `FunCPS`/`FixCPS`: the 4th (resp. 5th) argument wants plain `Kvar`.
> - *"has type exp but ... exp_cps"*: you forgot to call `cps_exp` on a sub-expression, for example in an `IfCPS` branch.
> - *"has type exp_cps but ... cps_cont"*: you passed a CPS expression where a continuation goes. Wrap it: `FnContCPS (v, ...)`.
> - *"Unbound value freeVarsInExp"*: MP5.5 needs your MP5.4 function in the same file (if the skeleton does not provide it), defined **above** `cps_exp`.

# Part X — Hand-tracing drills (cover the right column)

Each output below was produced by the solution above and the course's `freshFor` naming. Work each one by hand from the rules first. Being able to do this is what proves you understand MP5.5, and it is also the most likely form of a written exam question on CPS.

## 30. Step-by-step: `[[x + 1]]_k`

```text
BinOp: e1 = x, e2 = 1.
  v2 = freshFor (FV(x) ∪ FV(_k)) = freshFor ["x"] = a
  v1 = freshFor ([a] ∪ FV(_k))   = freshFor ["a"] = b
[[x + 1]]_k = [[1]]_(FN a -> [[x]]_(FN b -> _k(b + a)))
            = [[1]]_(FN a -> (FN b -> _k(b + a)) x)
            = (FN a -> (FN b -> _k(b + a)) x) 1
```

## 31. Step-by-step: `[[f (x + 1)]]_k` (nesting)

```text
App: e1 = f, e2 = x + 1.  v2 = freshFor ["f"] = a,  v1 = freshFor ["a"] = b
[[f (x + 1)]]_k = [[x + 1]]_κ'   where κ' = FN a -> [[f]]_(FN b -> (b a _k))
                                       = FN a -> (FN b -> (b a _k)) f
Now [[x + 1]]_κ'.  FV(κ') = {f}, so v2 = freshFor ["x"; "f"] = a,  v1 = b
  = (FN a -> (FN b -> κ'(b + a)) x) 1
  = (FN a -> (FN b -> (FN a -> (FN b -> (b a _k)) f)(b + a)) x) 1
```

The inner `a`/`b` shadow the outer ones. That is safe because the outer values have already been used by the time the inner ones are bound.

## 32. Drill table

| Source expression | `[[e]]_k` |
|---|---|
| `~ x` | `(FN a -> _k(~ a)) x` |
| `s ^ "!"` | `(FN a -> (FN b -> _k(b ^ a)) s) "!"` |
| `g 4` | `(FN a -> (FN b -> (b a _k)) g) 4` |
| `a b` | `(FN b -> (FN a -> (a b _k)) a) b` |
| `if p then 1 else a` | `(FN b -> IF b THEN _k 1 ELSE _k a) p` (`a` is free in a branch, so `v` = `b`) |
| `let y = 3 in y * y` | `(FN y -> (FN a -> (FN b -> _k(b * a)) y) y) 3` |
| `let x = y in fun z -> x` | `(FN x -> _k (FUN z _k -> _k x)) y` |
| `fun n -> n + 1` | `_k (FUN n _k -> (FN a -> (FN b -> _k(b + a)) n) 1)` |
| `fun x -> fun y -> x` | `_k (FUN x _k -> _k (FUN y _k -> _k x))` |
| `fst (a, b)` | `(FN b -> (FN a -> (FN a -> _k(fst a))(a , b)) a) b` |
| `f (if b then x else y)` | `(FN a -> IF a THEN (FN a -> (FN b -> (b a _k)) f) x ELSE (FN a -> (FN b -> (b a _k)) f) y) b` |
| `f x y` | `(FN a -> (FN b -> (FN c -> (c b (FN b -> (b a _k)))) f) x) y` |
| `let rec f n = if n = 0 then 1 else n * f (n - 1) in f 3` | `(FN f -> (FN a -> (FN b -> (b a _k)) f) 3)(FIX f. FUN n _k -> (FN a -> (FN b -> (FN a -> IF a THEN _k 1 ELSE (FN a -> (FN b -> (FN a -> (FN b -> (b a (FN a -> (FN b -> _k(b * a)) n))) f)(b - a)) n) 1)(b = a)) n) 0)` |

Note that `f (if ...)` **duplicates** the continuation into both branches. This is why `IfCPS` needs no continuation of its own.

## 33. Free-variable drill

Compute FV by hand, then check with §17. Do five more of your own by covering the middle column of the table in §17.

# Part XI — Quiz-day procedure

The quiz length was not available when this guide was built; Quiz 1 was 25 minutes. If you get a similar window with all five problems, the priority below protects the most points per minute. Scale it to the actual length and selection.

| Share of time | Action |
|---|---|
| ~5% | Read every selected problem. Note which ones are *build* (5.1, 5.2), *consume* (5.3, 5.4), and *transform* (5.5). |
| ~15% | **MP5.1 and MP5.3.** Short, mechanical, and nearly impossible to get wrong if they compile. `make` after each. |
| ~20% | **MP5.4.** Write all nine clauses. Compile. Run the grader. Fix `let` / `let rec` scoping. |
| ~35% | **MP5.5.** Write Var/Const/Fun/Let first (no freshness, quick points). Then BinOp, App, MonOp, If. LetRec last (extra credit). Compile after every two clauses. |
| ~15% | **MP5.2.** Build the tree top-down from §MP5.2 step 1–7. Compare `string_of_exp` with the handout string. |
| ~10% | Final `make` + `./grader`; confirm the saved file compiles; submit. |

> **Compiling beats complete.** If time runs out in the middle of a function, make it compile: end the `match` with `| _ -> failwith "todo"` (or `raise (Failure "todo")`). The finished clauses still earn their points, while a syntax error scores zero for the whole problem. Remove the catch-all if you finish, because it would hide exhaustiveness warnings.

## 34. Fast debugging inside the workspace

```text
make                      # compile your file + tests; read the FIRST error only
./grader                  # run the tests
ocaml                     # interactive top level, then:
#load "common.cmo";;
open Common;;
#use "cpsExp.ml";;        (* or the problem's file name *)
print_string (string_of_exp_cps (cps_exp (AppExp (VarExp "f", VarExp "x")) (ContVarCPS Kvar)));;
```

Add your own lines to the `tests` file using the same format as the existing ones, then re-run `make`. This is the quickest way to try each handout example.

# Part XII — Practice set with answer key

Do these in a blank file, with no notes and a 25-minute timer. These problems are new, in the same skill family as MP5; none of them is copied from an assessment.

## Practice problems

**P1.** Write `import_int_list : int list -> exp` so that `string_of_exp (import_int_list [1;2;3])` = `1 :: (2 :: (3 :: []))`.

**P2.** Write `import_strings : string list -> exp` (use `StringConst`).

**P3.** Write the AST for `let f = fun y -> y * 2 in f (f 3)`.

**P4.** Write the AST for `let rec len l = if l = [] then 0 else 1 + len (tl l) in len [1; 2]`.

**P5.** Write the AST for `fun x y -> x + y`.

**P6.** Write `count_var_occurrences : exp -> int`, which counts `VarExp` nodes (uses of variables, not binders). On the AST from P4 it returns 4.

**P7.** Write `uses_op : bin_op -> exp -> bool`, which tests whether a binary operator appears anywhere in the expression.

**P8.** By hand: FV of `let x = x in x + z`; `(fun x -> x) x`; `let rec g n = g m in g n`.

**P9.** By hand: `[[~ x]]_k`, `[[g 4]]_k`, `[[let x = y in fun z -> x]]_k`.

**P10.** Write `prodk : int list -> (int -> 'a) -> 'a`, a CPS product of a list, using `timesk`.

## Answer key

```ocaml
let rec import_int_list lst =
  match lst with
  | [] -> ConstExp NilConst
  | n :: rest -> BinOpAppExp (ConsOp, ConstExp (IntConst n), import_int_list rest)

let rec import_strings lst =
  match lst with
  | [] -> ConstExp NilConst
  | s :: rest -> BinOpAppExp (ConsOp, ConstExp (StringConst s), import_strings rest)

(* P3 *)
let double_twice =
  LetInExp ("f", FunExp ("y", BinOpAppExp (IntTimesOp, VarExp "y", ConstExp (IntConst 2))),
    AppExp (VarExp "f", AppExp (VarExp "f", ConstExp (IntConst 3))))

(* P4 *)
let len_prog =
  LetRecInExp ("len", "l",
    IfExp (BinOpAppExp (EqOp, VarExp "l", ConstExp NilConst),
      ConstExp (IntConst 0),
      BinOpAppExp (IntPlusOp, ConstExp (IntConst 1),
                   AppExp (VarExp "len", MonOpAppExp (TlOp, VarExp "l")))),
    AppExp (VarExp "len",
      BinOpAppExp (ConsOp, ConstExp (IntConst 1),
        BinOpAppExp (ConsOp, ConstExp (IntConst 2), ConstExp NilConst))))

(* P5 *)
let add_curried = FunExp ("x", FunExp ("y", BinOpAppExp (IntPlusOp, VarExp "x", VarExp "y")))

(* P6: see §14 for the or-pattern version; the 9-clause version is equally fine *)

(* P7 *)
let rec uses_op op e =
  match e with
  | VarExp _ | ConstExp _ -> false
  | MonOpAppExp (_, e1) | FunExp (_, e1) -> uses_op op e1
  | BinOpAppExp (op', e1, e2) -> op' = op || uses_op op e1 || uses_op op e2
  | AppExp (e1, e2) | LetInExp (_, e1, e2) | LetRecInExp (_, _, e1, e2) ->
      uses_op op e1 || uses_op op e2
  | IfExp (e1, e2, e3) -> uses_op op e1 || uses_op op e2 || uses_op op e3

(* P10 *)
let rec prodk lst k =
  match lst with
  | [] -> k 1
  | x :: xs -> prodk xs (fun r -> timesk (x, r) k)
```

**P8:** `{x, z}`; `{x}`; `{m, n}`. **P9:** `(FN a -> _k(~ a)) x`; `(FN a -> (FN b -> (b a _k)) g) 4`; `(FN x -> _k (FUN z _k -> _k x)) y`.

# Part XIII — Compact reference and self-test

## 35. The whole of MP5 on one page

```text
exp sub-expression counts:  Var 0 | Const 0 | MonOp 1 | BinOp 2 | If 3 | App 2 | Fun 1 | Let 2 | LetRec 2

PicoML encoding:  hd/tl/fst/snd/~  -> MonOpAppExp (HdOp|TlOp|FstOp|SndOp|IntNegOp, e)
                  (a, b)           -> BinOpAppExp (CommaOp, a, b)
                  h :: t           -> BinOpAppExp (ConsOp, h, t)
                  []               -> ConstExp NilConst
                  n                -> ConstExp (IntConst n)
                  f a b            -> AppExp (AppExp (f, a), b)

FV:  x -> [x]   c -> []   fun x -> e : FV e − x   let x = e1 in e2 : FV e1 ∪ (FV e2 − x)
     let rec f x = e1 in e2 : (FV e1 − f − x) ∪ (FV e2 − f)      everything else: union

CPS: Var/Const  -> VarCPS (k, x) / ConstCPS (k, c)
     MonOp      -> cps e1 (FN v -> MonOpAppCPS (k, op, v))                v ∉ FV k
     BinOp/App  -> cps e2 (FN v2 -> cps e1 (FN v1 -> BinOpAppCPS (k, op, v1, v2) / AppCPS (k, v1, v2)))
                   v2 ∉ FV e1 ∪ FV k ;  v1 ∉ {v2} ∪ FV k
     If         -> cps e1 (FN v -> IfCPS (v, cps e2 k, cps e3 k))          v ∉ FV e2 ∪ FV e3 ∪ FV k
     Fun        -> FunCPS (k, x, Kvar, cps e (ContVarCPS Kvar))
     Let        -> cps e1 (FnContCPS (x, cps e2 k))
     LetRec     -> FixCPS (FnContCPS (f, cps e2 k), f, x, Kvar, cps e1 (ContVarCPS Kvar))
```

## 36. True/false drill

| Statement | Answer and reason |
|---|---|
| In PicoML, `hd lst` is an application `AppExp`. | **False.** It is `MonOpAppExp (HdOp, ...)`. |
| In `let x = e1 in e2`, occurrences of `x` in `e1` are bound by this `let`. | **False.** The scope is only `e2`. |
| In `let rec f x = e1 in e2`, `x` is bound in `e2`. | **False.** Only `f` is bound in `e2`. |
| `[[e1 e2]]κ` transforms `e2` first (outermost). | **True.** Right-to-left: the argument is evaluated first. |
| `IfCPS` carries its own continuation. | **False.** κ is copied into both branches. |
| The body of a `fun` is transformed with the outer κ. | **False.** It is transformed with `_k` (`ContVarCPS Kvar`). |
| In CPS, every function call is a tail call. | **True.** That is the defining property. |
| `freshFor ["a"; "b"]` could return `"a"`. | **False.** It returns a name not in the list. |
| Your fresh names must match the reference solution's names exactly. | **False.** They only have to satisfy freshness, though using `freshFor` as shown usually matches. |
| `List.filter (fun y -> y <> x) l` removes every `x` from `l`. | **True.** |
| A non-exhaustive match is a compile error. | **False.** It is a warning, but missing cases crash at run time, so fix it. |

## 37. Final pre-quiz self-test

1. Can I write the nine `exp` constructors and what each carries, from memory?
2. Can I write `import_list` in under 3 minutes and explain why `ConstExp` wraps `IntConst`?
3. Can I build `pair_sums` top-down, with `HdOp` / `FstOp` / `SndOp` / `TlOp` as `MonOpAppExp`, and match the handout string?
4. Can I write `count_const_in_exp` with nine clauses and no missing sub-expression?
5. Can I state the scope of each binder in `let` and `let rec`, and explain why `let x = x in x` has `x` free?
6. Can I convert `fact` and `sum` to CPS by hand with `addk`/`timesk`/`eqk`?
7. Can I explain why `FN` and `FUN` differ, and when `_k` versus `κ` is used?
8. Can I write all nine `cps_exp` clauses, including the `v1`/`v2` freshness lists and the `e2`-first order?
9. Can I produce `(FN a -> (FN b -> (b a _k)) f) x` by hand from `f x`?
10. Can I fix *cont_var vs cps_cont* and *exp vs exp_cps* type errors in seconds?

# Sources and scope notes

Primary (authoritative for this guide):

- MP5.1–MP5.5 PrairieLearn problem statements, supplied in full by the student (handout text, types, rule descriptions, and all sample outputs).
- Instructor's statement, relayed by the student: Quiz 2 is MP5 "as given"; Quiz 1 used the original MP2 questions.
- The student's *CS 421 Fall 2026 OCaml Foundations and MP2 Study Guide* (format, workflow, and Quiz 1 conventions).

Course pages (check these for current notation and logistics; they could not be fetched from the build environment):

- CS 421 Fall 2026 lecture schedule: https://courses.grainger.illinois.edu/cs421/fa2026/lectures/index.html
- Earlier CS 421 CPS lecture decks, which cover the same week 4 material: https://courses.grainger.illinois.edu/CS421/sp2023/lectures/10-continuation-passing-style.pdf, https://courses.grainger.illinois.edu/CS421/sp2023/lectures/11-CPS-Transformation.pdf, https://courses.grainger.illinois.edu/cs421/fa2019/lectures/07-08-CPS.pdf
- Earlier CS 421 user-defined types deck: https://courses.grainger.illinois.edu/cs421/fa2024/CS421C/lectures/09-10-usertypes-intro.pdf

> **Scope warning.** This guide covers what MP5 requires: lists, algebraic data types, abstract syntax, free variables, CPS, and the CPS transformation. It does not cover later topics such as type inference, unification, lexing, parsing, or operational semantics. Re-check PrairieLearn for the exact problems selected for your quiz.
