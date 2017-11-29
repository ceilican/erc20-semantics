ERC20: Formal Executable Specification
======================================

Author: [Grigore Rosu](http://fsl.cs.illinois.edu/grosu)

Date: November 28, 2017

**Acknowledgments:** Warm thanks to the K team who defined the 
[KEVM](https://github.com/kframework/evm-semantics) semantics
(see
[our technical report](https://www.ideals.illinois.edu/handle/2142/97207), too)
and verified smart contracts for ERC20 compliance.
It is their effort that inevitably led to the quest for a formal specification
of ERC20.
We also thank [IOHK](http://iohk.io) for their generous support
of the [KEVM](https://github.com/kframework/evm-semantics) and
[IELE](https://github.com/runtimeverification/iele-semantics)
projects; the latter, in particular, led to the question of whether
the ERC20 specification can be defined in a way that does not depend on
the EVM and thus can be used in combination with other computational
infrastructures.

## Abstract

[ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20-token-standard.md)
is one of the most important standards for the implementation of tokens
within Ethereum smart contracts.
ERC20 provides basic functionality to transfer tokens and to be approved so
they can be spent by another on-chain third party.
Here we provide a complete formalization of ERC20 in
[K](http://kframework.org).
Specifically, we provide a formal executable semantics of ERC20.
The semantics clarifies what data (accounts, allowances, etc.) is handled by
the various ERC20 functions, and the precise meaning of the functions on such
data.

## Motivation

Now that the K team has developed
[KEVM](https://github.com/kframework/evm-semantics), a formal semantics of
the Ethereum Virtual Machine, it has become possible to rigorously verify
smart contracts at the EVM level against higher level specifications.
The most requested property to verify so far was *ERC20 compliance* of
smart contracts written in languages like Solidity or Viper.
But what does that really mean?
When formal verification is sought, a property (or specification) to
verify the code against must be explicitly or implicitly provided.
Moreover, such a specification is expected to be non-ambiguous and
capable of answering all the questions regarding corner-case behaviors.
At our knowledge, there was no such formal ERC20 specification available
at the time of this writing.
Our specification below is executable, so it can also be tested for increased
confidence.
We tested it against a test-suite of more than 60 tests
(under the `tests` folder), which we believe cover all the corner cases.
Please contribute with more tests if you think that we left some behaviors
uncovered.

## Formal Specification

Below we describe our ERC20 formal specification in K.
Since it uses a small fragment of K and since not everybody knows K,
we will also explain K as needed.

Our objective is to define the ERC20 specification in such a way that
it can be imported by an arbitrary programming language, whose programs
can then make use of ERC20 tokens.
Our immediate reason for doing so is to test our ERC20 specification
programmatically.
But this design choice can have other advantages, too.
For example, it can be seen as a way to *plug-and-play* ERC20 support
to your favorite language.

We start by defining a module called `ERC20`:

```{.k}
module ERC20
```

### Syntax

A K definition consists of three major parts, often mixed:
syntactic definitions, configuration definitions, and semantic
rules.
We start by defining the ERC20 syntax.

#### Values and Addresses

ERC20 refers to two major types of data: `Value` and `Address`.
We assume that any implementation provides an additional `Bool`
type and avoid including Booleans as values here.
It is not difficult to see that although the original ERC20
standard is presented using specific types for values and
addresses, namely `uint256` and `address`, respectively, its
semantics is rather independent of these.
They can be regarded as *parameters* of the specification.
However, to avoid defining adhoc syntactic categories for these,
we here choose them to be K's unbounded integers; the semantics
is defined later on in a way that bounds on values will be
imposed and preserved.
K's (unbounded) integers live under the predefined syntactic
category `Int`:

```{.k}
  syntax Value   ::= Int  // can be easily changed
  syntax Address ::= Int  // can be easily changed
```

#### Main Functions

Below is the syntax of the main ERC20 functions:

```{.k}
  syntax AExp ::= Value | Address
                | "totalSupply" "(" ")"
                | "balanceOf" "(" AExp ")"                       [strict]
                | "allowance" "(" AExp "," AExp ")"              [strict]
  syntax BExp ::= Bool
                | "approve" "(" AExp "," AExp ")"                [strict]
                | "transfer" "(" AExp "," AExp ")"               [strict]
                | "transferFrom" "(" AExp "," AExp "," AExp ")"  [strict]
                | "throw"
```

Since we want our ERC20 specification to be easily incorporated within
programming languages, a minimal interface needs to be agreed upon.
Above, we assumed that such a language must include two syntactic categories:
`AExp` for arithmetic expressions and `BExp` for Boolean expressions.
Values and addresses are `AExp`s and `Bool` are `BExp`s.
The constructs above, except for `throw`, correspond to the ERC20 functions.
In K, grammars are defined using the BNF notation with terminals double-quoted.
The constant `throw` will be used as result of functions which perform illegal
operations and will get the computation stuck.
Note that all arguments of all functions above are `AExp`.
Also, all functions that take arguments have the attribute `strict`,
which in K means that they should first evaluate their arguments.
Their semantic rules below will not apply before their arguments are
fully evaluated to their expected types of values.
Consequently, programming language semantics importing our ERC20 semantics
will allow their programs to pass arbitrary expressions as arguments to
the ERC20 functions, yet the semantics of those functions will apply only
after their arguments are properly evaluated.

#### Events

The ERC20 functions are required, at success, to log two types of events:

```{.k}
  syntax Event ::= "Transfer" "(" Address "," Address "," Value ")"
                 | "Approval" "(" Address "," Address "," Value ")"
```

The events are then collected in a log.
The precise syntax of the log is not specified in the standard, so we take
the freedom to choose the syntax `Events: <event1> <event2> ... <eventn>`:

```{.k}
  syntax EventLog ::= "Events:"
                    | EventLog Event
```

### Configuration

Configurations hold the structure and the data on which 
the semantic rules match and apply.
At any given moment during its execution, the state of the defined
system or programming language is an instance of the defined
configuration.
A configuration consists of a set of potentially nested
*semantic cells*, each cell having a name and containing a specific
kind of semantic information.
We use an XML-like notation to say where a cell starts and where it ends.
The configuration declaration also contains an initial value for each cell.
We first show the entire ERC20 configuration and then we discuss it:

```{.k}
  configuration <ERC20>
                  <caller> 0 </caller>
                  <k> $PGM:K </k>
                  <accounts>
                    <account multiplicity="*">
                      <id> 0 </id>
                      <balance> 0 </balance>
                    </account>
                  </accounts>
                  <allowances>
                    <allowance multiplicity="*">
                      <owner> 0 </owner>
                      <spenders>
                        <allow multiplicity="*">
                          <spender> 0 </spender>
                          <amount> 0 </amount>
                        </allow>
                      </spenders>
                    </allowance>
                  </allowances>
                  <log> Events: </log>
                  <supply> 0 </supply>
                </ERC20>
```
The configuration consists or a top-level cell `<ERC20/>`, which
holds 6 other cells.

The `<caller/>` holds the initiator of the transaction, aka the sender
of the message; we initialize the configuration with caller `0`.
It is important to know the caller, because the semantics of the transfer
functions depend on it.

The `<k/>` cell holds the computation to be performed by the caller.
The `$PGM:K` variable is replaced with the input program by the `krun`
tool, that is, the program we want to execute or whose semantics we are
interested in.
In this module we define the semantics of the `ERC20` functions
when they become the first task of the computation, but each programming
language will define the semantics of its additional constructs similarly,
when they become the first task in the `<k/>` cell.
Instead of "become the first task in the `<k/>` cell" we sometimes say
"it is at the top of the `<k/>` cell".

The `<accounts/>` cell holds all the accounts.
Each account is held in a separate `<account/>` cell and has its own
unique `<id/>` and its own `<balance/>`.
The `multiplicity="*"` tag states that there could be zero, one or more
instances of that cell.

The `<allowances/>` cell holds all the allowances: for each `<owner/>`,
its `<spenders/>` cell holds an allowed `<amount/>` for each `<spender/>`.

The `<log/>` cell holds and `EventLog` term, that is, a sequence of `Event`s.
Newly generated events are appended at the end of the log.

The `<supply/>` cell holds the total token supply.

For the semantics of ERC20, we assume the configuration is already
initialized and the caller is known.
In our simple programming language on top of ERC20 and in our tests
we define macros that can be used to initialize the configuration.

### Semantics

ERC20 was originally designed for the EVM, whose integers
(of type `uint256`) have `256` bits.
The exact size of integers is important for arithmetic overflow,
which needs to be rigorously defined in the formal specification.
However, the precise size of integers can be a parameter of the
ERC20 specification.
This way, the specification can easily translate to other execution
environments, such as [eWASM](https://github.com/ewasm/) or 
[IELE](https://github.com/runtimeverification/iele-semantics).
The syntax declaration and the rule below define an integer
constant `MAXVALUE` and initialize it with the largest integer
on 256 bits (one can set it to "infinity" if one does not care about
overflows or if one's computational infrastructure supports unbounded
integers, like
[IELE](https://github.com/runtimeverification/iele-semantics) does):

```{.k}
  syntax Int ::= "MAXVALUE"  [function]
  rule MAXVALUE => 2 ^Int 256 -Int 1
```

The attribute/tag `function` above states that `MAXVALUE`
is to be reduced, or *rewritten* with its rule above, instantly
wherever it occurs (rewriting of non-function symbols is constrained
by the context).

We start with the semantics of `totalSupply()`:
```{.k}
  rule <k> totalSupply() => Total ...</k>
       <supply> Total </supply>
```
The rule above involves two cells, `<k/>` and `<supply/>`, and matches
`totalSupply()` as the first task in the `</k>` cell and the contents
of the `<supply/>` cell in the `Total` variable.
In K, variables are usually capitalized; `...` is a special variable,
called a *structural frame*, which matches the rest of a sequence, set,
map, cell, etc.  The arrow `=>` indicates that its left-hand-side term
is rewritten to its right-hand-side term.
In words, the rule says: When requested to compute, `totalSupply()`
rewrites to the `Total` contents of the `<supply/>` cell.
Note that rules only need to mention the cells they need; the rest of
the configuration is inferred automatically and rules do not change it.

We next define the semantics of `balanceOf` in a similar but slightly
more involved way:

```{.k}
  rule <k> balanceOf(Id) => Value ...</k>
       <id> Id </id>
       <balance> Value </balance>
```

Above, `balanceOf(Id)` is matched at the top of the `<k/>` cell;
As a result, variable `Id` is bound to a specific account identifier
found in the `<id/>` cell.
The `Value` of that account then becomes the result of `balanceOf(Id)`.
Note that the `<id/>` and `<balance/>` cells are *not* at the same level
in the configuration as the `<k/>` cell.
K will automatically infer the missing configuration cells as part of its
*configuration abstraction* mechanism, thus saving us from writing
verbose rules, like the following equivalent rule:

```
  rule <k> balanceOf(Id) => Value ...</k>
       <accounts>...
         <account>
           <id> Id </id>
           <balance> Value </balance>
         </account>
       ...</accounts>
```

The rule for `allowance(Owner,Spender)` makes even more aggressive use of
configuration abstraction:

```{.k}
  rule <k> allowance(Owner, Spender) => Allowance ...</k>
       <owner> Owner </owner>
       <spender> Spender </spender>
       <amount> Allowance </amount>
```

Above, the `Owner` is matched in some `<allowance/> cell, then the `Spender`
is matched inside one of owner's `<allow/>` cells, and then the spender's
allowed `Amount` is returned as the result of the `allowance` function call.

The semantics of `approve(Spender, Allowance)` is trickier.

```{.k}
  rule <k> approve(Spender, Allowance) => true ...</k>
       <caller> Owner </caller>
       <owner> Owner </owner>
       <spender> Spender </spender>
       <amount> _ => Allowance </amount>
       <log> Log => Log Approval(Owner, Spender, Allowance) </log>
    requires Allowance >=Int 0

  rule <k> approve(_, Allowance) => throw ...</k>
    requires Allowance <Int 0

  rule <k> transfer(To, Value) => true ...</k>
       <caller> From </caller>
       <account>
         <id> From </id>
         <balance> BalanceFrom => BalanceFrom -Int Value </balance>
       </account>
       <account>
         <id> To </id>
         <balance> BalanceTo => BalanceTo +Int Value </balance>
       </account>
       <log> Log => Log Transfer(From, To, Value) </log>
    requires To =/=Int From    // sanity check
     andBool Value >=Int 0
     andBool Value <=Int BalanceFrom
     andBool BalanceTo +Int Value <=Int MAXVALUE

/*
A self transfer is useless, so it will likely not happen in practice.
Yet, a complete specification of ERC20 must nevertheless consider it.
There are at least three possible behaviors to consider for self transfers:
(1) Throw.
(2) Ignore.
(3) Make the actual transfer and log the event.
The first and second cases are the easiest and cleanest to define
semantically, but they may require more code and an additional runtime check
in implementations, so they may result in more gas consumption (assuming the
code is complex enough for compilers to optimize and eliminate the check).
All ERC20 implementations that we've seen, however, do not special case
self transfers, so they implicitly have the third behavior.
Consequently, we also choose the third behavior in our specification.
However we *do* special case the semantics of self transfers because:
(1) Only one of the last two conditions in the `requires` clause above needs
    to be checked for self transfers (see the note below); 
(2) We believe it is important that developers of ERC20 implementations be
    aware of the semantic choice and program verifiers explicitly consider the
    case of self transfers.
Note: implementations of `transfer` which first add `Value` to recipient's
account and then subtract Value from sender's account will fail to satisfy the
rule below, because of the potential overflow.  To favor those implementations
we would have to change the second requires clause of the rule below to
`BalanceFrom +Int Value <=Int MAXVALUE`.
*/ 
  rule <k> transfer(From, Value) => true ...</k>
       <caller> From </caller>
       <id> From </id>
       <balance> BalanceFrom </balance>
       <log> Log => Log Transfer(From, From, Value) </log>
    requires Value >=Int 0
     andBool Value <=Int BalanceFrom

  rule <k> transfer(To, Value) => throw ...</k>
       <caller> From </caller>
       <account>
         <id> From </id>
         <balance> BalanceFrom </balance>
       </account>
       <account>
         <id> To </id>
         <balance> BalanceTo </balance>
       </account>
    requires To =/=Int From   // sanity check
     andBool (Value <Int 0
      orBool Value >Int BalanceFrom
      orBool BalanceTo +Int Value >Int MAXVALUE)

// self transfer; again, we assume withdrawal followed by deposit
  rule <k> transfer(From, Value) => throw ...</k>
       <caller> From </caller>
       <id> From </id>
       <balance> BalanceFrom </balance>
    requires Value <Int 0
      orBool Value >Int BalanceFrom

  rule <k> transferFrom(From, To, Value) => true ...</k>
       <caller> Caller </caller>
       <owner> From </owner>
       <spender> Caller </spender>
       <amount> Allowance => Allowance -Int Value </amount>
       <account>
         <id> From </id>
         <balance> BalanceFrom => BalanceFrom -Int Value </balance>
       </account>
       <account>
         <id> To </id>
         <balance> BalanceTo => BalanceTo +Int Value </balance>
       </account>
       <log> Log => Log Transfer(From, To, Value) </log>
    requires To =/=Int From    // sanity check
     andBool Value >=Int 0
     andBool Value <=Int BalanceFrom
     andBool Value <=Int Allowance   // `transfer` does not check allowance
     andBool BalanceTo +Int Value <=Int MAXVALUE

// self transfer; again, we assume withdrawal followed by deposit
  rule <k> transferFrom(From, From, Value) => true ...</k>
       <caller> Caller </caller>
       <owner> From </owner>
       <spender> Caller </spender>
       <amount> Allowance => Allowance -Int Value </amount>
       <id> From </id>
       <balance> BalanceFrom </balance>
       <log> Log => Log Transfer(From, From, Value) </log>
    requires Value >=Int 0
     andBool Value <=Int BalanceFrom
     andBool Value <=Int Allowance   // `transfer` does not check allowance

  rule <k> transferFrom(From, To, Value) => throw ...</k>
       <caller> Caller </caller>
       <owner> From </owner>
       <spender> Caller </spender>
       <amount> Allowance </amount>
       <account>
         <id> From </id>
         <balance> BalanceFrom </balance>
       </account>
       <account>
         <id> To </id>
         <balance> BalanceTo </balance>
       </account>
    requires To =/=Int From    // sanity check
     andBool (Value <Int 0
      orBool Value >Int BalanceFrom
      orBool Value >Int Allowance
      orBool BalanceTo +Int Value >Int MAXVALUE)

// self transfer; again, we assume withdrawal followed by deposit
  rule <k> transferFrom(From, From, Value) => throw ...</k>
       <caller> Caller </caller>
       <owner> From </owner>
       <spender> Caller </spender>
       <amount> Allowance </amount>
       <id> From </id>
       <balance> BalanceFrom </balance>
    requires Value <Int 0
      orBool Value >Int BalanceFrom
      orBool Value >Int Allowance   // `transfer` does not check allowance

endmodule
```