# LL(*) the foundation of the ANTLR parser generator

## Abstract

Despite the power of Parser Expression Grammars (PEGs) and GLR, parsing is not a solved problem. Adding nondeterminism (parser speculation) to traditional $LL$ and $LR$ parsers can lead to unexpected parse-time behavior and introduces practical issues with error handling, single-step debugging, and side-effecting embedded grammar actions. This paper introduces the $LL$_(*)_ parsing strategy and an associated grammar analysis algorithm that constructs $LL$_(*)_ parsing decisions from ANTLR grammars. At parse-time, decisions gracefully throttle up from conventional fixed $k\geq 1$ lookahead to arbitrary lookahead and, finally, fail over to backtracking depending on the complexity of the parsing decision and the input symbols. $LL$_(*)_ parsing strength reaches into the context-sensitive languages, in some cases beyond what GLR and PEGs can express. By statically removing as much speculation as possible, $LL$_(*)_ provides the expressivity of PEGs while retaining $LL$'s good error handling and unrestricted grammar actions. Widespread use of ANTLR (over 70,000 downloads/year) shows that it is effective for a wide variety of applications.

## 1 Introduction

Parsing is not a solved problem, despite its importance and long history of academic study. Because it is tedious and error-prone to write parsers by hand, researchers have spent decades studying how to generate efficient parsers from high-level grammars. Despite this effort, parser generators still suffer from problems of expressiveness and usability.

When parsing theory was originally developed, machine resources were scarce, and so parser efficiency was the paramount concern. In that era, it made sense to force programmers to contort their grammars to fit the constraints of $LALR(1)$ or $LL(1)$ parser generators. In contrast, modern computers are so fast that programmer efficiency is now more important. In response to this development, researchers have developed more powerful, but more costly, nondeterministic parsing strategies following both the "bottom-up" approach ($LR$-style parsing) and the "top-down" approach ($LL$-style parsing).

In the "bottom-up" world, _Generalized LR_ (GLR) [16] parsers parse in linear to cubic time, depending on how closely the grammar conforms to classic $LR$. GLR essentially "forks" new subparsers to pursue all possible actions emanating from nondeterministic $LR$ states, terminating any subparsers that lead to invalid parses. The result is a parse forest with all possible interpretations of the input. _Elkhound_[10] is a very efficient GLR implementation that achieves _yacc_-like parsing speeds when grammars are $LALR(1)$. Programmers unfamiliar with $LALR$ parsing theory, though, can easily get nonlinear GLR parsers.

In the "top-down" world, Ford introduced _Packrat_ parsers and the associated _Parser Expression Grammars_ (PEGs) [5, 6]. PEGs preclude only the use of left-recursive grammar rules. Packrat parsers are backtracking parsers that attempt the alternative productions in the order specified. The first production that matches at an input position wins. Packrat parsers are linear rather than exponential because they memoize partial results, ensuring input states will never be parsed by the same production more than once. The _Rats_[7] PEG-based tool vigorously optimizes away memoization events to improve speed and reduce the memory footprint.

A significant advantage of both GLR and PEG parser generators is that they accept any grammar that conforms to their meta-language (except left-recursive PEGs). Programmers no longer have to wade through reams of conflict messages. Despite this advantage, neither GLR nor PEG parsers are completely satisfactory, for a number of reasons.

First, GLR and PEG parsers do not always do what was intended. GLR silently accepts _ambiguous grammars_, those that match the same input in multiple ways, forcing programmers to detect ambiguities dynamically. PEGs have no concept of a grammar conflict because they always choose the "first" interpretation, which can lead to unexpected or inconvenient behavior. For example, the second production of PEG rule $A\to a|ab$ (meaning "$A$ matches either $a$ or $ab$") will never be used. Input $ab$ never matches the second alternative since the first symbol, $a$, matches the first alternative. In a large grammar, such hazards are not always obvious and even experienced developers can miss them without exhaustive testing.

Second, debugging nondeterministic parsers can be very difficult. With bottom-up parsing, the state usually represents multiple locations within the grammar, making it difficult for programmers to predict what will happen next. Top-down parsers are easier to understand because there is a one-to-one mapping from $LL$ grammar elements to parser operations. Further, recursive-descent $LL$ implementations allow programmers to use standard source-level debuggers to step through parsers and embedded actions, facilitating understanding. This advantage is weakened significantly, however, for backtracking recursive-descent packet parsers. Nested backtracking is very difficult to follow!

Third, generating high-quality error messages in nondeterministic parsers is difficult but very important to commercial developers. Providing good syntax error support relies on parser context. For example, to recover well from an invalid expression, a parser needs to know if it is parsing an array index or, say, an assignment. In the first case, the parser should resynchronize by skipping ahead to a ] token. In the second case, it should skip to a ; token. Top-down parsers have a rule invocation stack and can report things like "_invalid expression in array index_." Bottom-up parsers,on the other hand, only know for sure that they are matching an expression. They are typically less able to deal well with erroneous input. Packrat parsers also have ambiguous context since they are always speculating. In fact, they cannot recover from syntax errors because they cannot detect errors until they have seen the entire input.

Finally, nondeterministic parsing strategies cannot easily support arbitrary, embedded grammar actions, which are useful for manipulating symbol tables, constructing data structures, _etc._ Speculating parsers cannot execute side-effecting actions like print statements, since the speculated action may never really take place. Even side-effect free actions such as those that compute rule return values can be awkward in GLR parsers [10]. For example, since the parser can match the same rule in multiple ways, it might have to execute multiple competing actions. (Should it merge all results somehow or just pick one?) GLR and PEG tools address this issue by either disallowing actions, disallowing arbitrary actions, or relying on the programmer to avoid side-effects in actions that could be executed speculatively.

### Antlr

This paper describes version 3.3 of the ANTLR parser generator and its underlying top-down parsing strategy, called _LL(*)_, that address these deficiencies. The input to ANTLR is a context-free grammar augmented with _syntactic_[14] and _semantic_ predicates and embedded actions. Syntactic predicates allow arbitrary lookahead, while semantic predicates allow the state constructed up to the point of a predicate to direct the parse. Syntactic predicates are given as a grammar fragment that must match the following input. Semantic predicates are given as arbitrary Boolean-valued code in the host language of the parser. Actions are written in the host-language of the parser and have access to the current state. As with PEGs, ANTLR requires programmers to avoid left-recursive grammar rules.

The contributions of this paper are 1) the top-down parsing strategy _LL(*)_ and 2) the associated static grammar analysis algorithm that constructs _LL(*)_ parsing decisions from ANTLR grammars. The key idea behind _LL(*)_ parsers is to use regular-expressions rather than a fixed constant or backtracking with a full parser to do lookahead. The analysis constructs a deterministic finite automata (DFA) _for each nonterminal in the grammar_ to distinguish between alternative productions. If the analysis cannot find a suitable DFA for a nonterminal, it fails over to backtracking. As a result, _LL(*)_ parsers gracefully throttle up from conventional fixed $k\geq 1$ lookahead to arbitrary lookahead and, finally, fail over to backtracking depending on the complexity of the parsing decision. Even within the same parsing decision, the parser decides on a strategy dynamically according to the input sequence. Just because a decision _might_ have to scan arbitrarily far ahead or backtrack does not mean that it _will_ at parse-time for every input sequence. In practice, _LL(*)_ parsers only look one or two tokens ahead on average despite needing to backtrack occasionally (Section 6). _LL(*)_ parsers are _LL_ parsers with supercharged decision engines.

This design gives ANTLR the advantages of top-down parsing without the downside of frequent speculation. In particular, ANTLR accepts all but left-recursive context-free grammars, so as with GLR or PEG parsing, programmers do not have to contort their grammars to fit the parsing strategy. Unlike GLR or PEGs, ANTLR can statically identify some grammar ambiguities and dead productions. ANTLR generates top-down, recursive-descent, mostly non-speculating parsers, which means it supports source-level debugging, produces high-quality error messages, and allows programmers to embed arbitrary actions. A survey of 89 ANTLR grammars [1] available from sorceforce.net and code.google.com reveals that 75% of them had embedded actions, counting conservatively, which reveals that such actions are a useful feature in the ANTLR community.

Widespread use shows that _LL(*)_ fits within the programmer comfort zone and is effective for a wide variety of language applications. ANTLR 3.x has been downloaded 41,364 (binary jar file) + 62,086 (integrated into ANTLRworks) + 31,126 (source code) = 134,576 times according to Google Analytics (unique downloads January 9, 2008 - October 28, 2010). Projects using ANTLR include Google App Engine (Python), IBM Tivoli Identity Manager, BEA/Oracle WebLogic, Yahoo! Query Language, Apple XCode IDE, Apple Keynote, Oracle SQL Developer IDE, Sun/Oracle JavaFX language, and NetBeans IDE.

This paper is organized as follows. We first introduce ANTLR grammars by example (Section 2). Next we formally define _practical grammars_ and a special subclass called _predicated_ LL-_regular_ grammars (Section 3). We then describe _LL(*)_ parsers (Section 4), which implement parsing decisions for predicated _LL_-regular grammars. Next, we give an algorithm that builds lookahead DFA from ANTLR grammars (Section 5). Finally, we support our claims regarding _LL(*)_ efficiency and reduced speculation (Section 6).

## 2 Introduction to _LL(*)_

In this section, we give an intuition for _LL(*)_ parsing by explaining how it works for two ANTLR grammar fragments constructed to illustrate the algorithm. Consider nonterminal s, which uses the (omitted) nonterminal expr to match arithmetic expressions.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120547581.png)
Nonterminal s matches an identifier (ID), an ID followed by an equal sign and then an expression, zero or more occurrences of the literal unsigned followed by the literal int followed by an ID, or zero or more occurrences of unsigned followed by two IDs. ANTLR grammars use _yacc_-like syntax with extended BNF (EBNF) operators such as Kleene star ($\star$) and token literals in single quotes.

When applied to this grammar fragment, ANTLR's grammar analysis yields the _LL(*)_ lookahead DFA in Figure 1. At the decision point for s, ANTLR runs this DFA on the input until it reaches an accepting state, where it selects the alternative for s predicted by the accepting state.

Even though we need arbitrary lookahead to distinguish between the $3^{rd}$ and $4^{th}$ alternatives, the lookahead DFA uses the minimum lookahead _per input sequence_. Upon int from input int x, the DFA immediately predicts the third alternative ($k=1$). Upon T (an ID) from T x, the DFA needs to see the $k=2$ token to distinguish alternatives 1, 2, and 4. It is only upon unsigned that the DFA needs to scan arbitrarily ahead, looking for a symbol (int or ID) that distinguishes between alternatives 3 and 4.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120548086.png)

The lookahead language for s is regular, so we can match it with a DFA. With recursive rules, however, we usually find that the lookahead language is context-free rather than regular. In this case, ANTLR fails over to backtracking if the programmer has requested this feature by adding syntactic predicates. As a convenience, option backtrack=true automatically inserts syntactic predicates into every production, which we call _"PEG mode"_ because it mimics the behavior of PEG parsers. However, before resorting to backtracking, ANTLR's analysis algorithm builds a DFA that adds a few extra states that allow it avoid backtracking for many input cases. In the following rule s2, both alternatives can start with an arbitrary number of - negation symbols; the second alternative does so using recursive rule expr.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120548007.png)
Figure 2 shows the lookahead DFA that ANTLR constructs for this input. This DFA can immediately choose the appropriate alternative upon either input x or 1; by looking at just the first symbol. Upon - symbols, the DFA matches a few - before failing over to backtracking. The number of times ANTLR unwinds the recursive rule before backtracking is controlled by an internal constant $m$, which we set to 1 for this example. Despite the possibility of backtracking, the decision will not backtrack in practice unless the input starts with "--", an unlikely expression prefix.

## 3 Predicated Grammars

To describe _LL(*)_ parsing precisely, we need to first formally define the predicated grammars from which they are derived. A predicated grammar $G=(N,T,P,S,\Pi,\mathcal{M})$ has elements:

* $N$ is the set of nonterminals (rule names)
* $T$ is the set of terminals (tokens)
* $P$ is the set of productions
* $S\in N$ is the start symbol
* $\Pi$ is a set of side-effect-free semantic predicates
* $\mathcal{M}$ is a set of actions (mutators)

Predicated grammars are written using the notation shown in Figure 3. Productions are numbered to express precedence as a means to resolve ambiguities. The first production form represents a standard context-free grammar rule. The second denotes a production gated by a _syntactic predicate_: symbol $A$ expands to $\alpha_{i}$ only if the current input also matches the syntax described by $A^{\prime}_{i}$. Syntactic predicates enable arbitrary, programmer-specified, context-free lookahead. The third form denotes a production gated by a _semantic predicate_: symbol $A$ expands to $\alpha_{i}$ only if the predicate $\pi_{i}$ holds for the state constructed so far. The final form denotes an action: applying such a rule updates the state according to mutator $\mu_{i}$.

The derivation rules in Figure 4 define the meaning of a predicated grammar. To support semantic predicates and mutators, the rules reference state $\mathbb{S}$, which abstracts user state during parsing. To support syntactic predicates, the rules reference $w_{r}$, which denotes the input remaining to be matched. The judgment form $(\mathbb{S},\alpha)\stackrel{{\vec{\lambda}}}{{\rightarrow}}( \mathbb{S}^{\prime},\beta)$, may be read: "In machine state $\mathbb{S}$, grammar sequence $\alpha$ reduces in one step to modified state $\mathbb{S}^{\prime}$ and grammar sequence $\beta$ while emitting trace $\lambda$." The judgment $(\mathbb{S},\alpha)\stackrel{{\vec{\lambda}}}{{\rightarrow}} \stackrel{{\vec{\lambda}}}{{\rightarrow}} \stackrel{{\ast}}{{\rightarrow}}(\mathbb{S}^{\prime},\beta)$ denotes repeated applications of the one-step reduction rule, accumulating all actions in the process. We omit $\lambda$ when it is irrelevant to the discussion. These reduction rules specify a leftmost derivation. A production with a semantic predicate $\pi_{i}$ can fire only if $\pi_{i}$ is true of the current state $\mathbb{S}$. A production with syntactic predicate $A^{\prime}_{i}$ can fire only if the string derived from $A^{\prime}_{i}$ in the current state is a prefix of the remaining input, written $w\preceq w_{r}$. Actions that occur during the attempt to parse $A^{\prime}_{i}$ are executed _speculatively_. They are undone whether or not $A^{\prime}_{i}$ matches. Finally, an action production uses the specified mutator $\mu_{i}$ to update the state.

Formally, the language generated by grammar sequence $\alpha$ is $L(\mathbb{S},\alpha)=\{w\,|\,(\mathbb{S},\alpha)\Rightarrow^{*}(\mathbb{S}^{ \prime},w)\}$ and the language of grammar $G$ is $L(G)=\{w\,|\,(\epsilon,S)\Rightarrow^{*}(\mathbb{S},w)\}$. Theoretically, the language class of $L(G)$ is recursively enumerable because each mutator could be a Turing machine. In practice, grammar writers do not use this generality, and so we consider the language class to be the context-sensitive languages instead. The class is context-sensitive rather than context-free because predicates can check both the left and right context.

This formalism has various syntactic restrictions not present in actual ANTLR input, for example, forcing predicates to the left-edge of rules and forcing mutators into their own rules. We can make these restrictions without loss of

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120549629.png)

generality because any grammar in the general form can be translated into this more restricted form [^1].

One of the key concepts behind parsing is the language matched by a production at a particular point in the parse.

**Definition 1**: $\mathcal{C}(\alpha)=\{w\mid(\epsilon,S)\Rightarrow^{*}(\mathbb{S},u\alpha\delta) \Rightarrow^{*}(\mathbb{S}^{\prime},uw)\}$ _is the continuation language for production $\alpha$._

Finally, grammar position $\alpha\cdot\beta$ means "_after $\alpha$ but before $\beta$ during generation or parsing._"

### 3.1 Resolving ambiguity

An ambiguous grammar is one in which the same string may be recognized in multiple ways. The rules in Figure 4 do not preclude ambiguity. However, for a practical parser, we want each input to correspond to a unique parse. To that end, ANTLR uses the order of the productions in the grammar to resolve ambiguities, with conflicts resolved in favor of the rule with the lowest production number. Programmers are instructed to make semantic predicates mutually exclusive for all potentially ambiguous input sequences, making such semantic productions unambiguous. However, that condition cannot be enforced because predicates are written in a Turing-complete language. If the programmer fails to satisfy this condition, ANTLR uses production order to resolve the ambiguity. This policy matches what is done in PEGs [5, 7] and is useful for concisely representing precedence.

### 3.2 Predicated _Ll_-regular grammars

There is one final concept that is helpful in understanding the _LL(*)_ parsing framework, namely, the notion of a _predicated_ LL-_regular grammar_. In previous work, Jarzabek and Krawczyk [8] and Nijholt [13] define _LL_-regular grammars to be a particular subset of the non-left-recursive, unambiguous CFGs. In this work, we extend the notion of _LL_-regular grammars to predicated _LL_-regular grammars for which we will construct efficient _LL(*)_ parsers. We require that the input grammar be non-left-recursive; we use rule ordering to ensure that the grammar is unambiguous.

_LL_-regular grammars differ from _LL(k)_ grammars in that, for any given nonterminal, parsers can use the entire remaining input to differentiate the alternative productions rather than just $k$ symbols. _LL_-regular grammars require the existence of a regular partition of the set of all terminal sequences for each nonterminal $A$. Each block of the partition corresponds to exactly one possible production for $A$. An _LL_-regular parser determines to which regular set the remaining input belongs and selects the corresponding production. Formally,

**Definition 2**: _Let $R=(R_{1},R_{2},\ldots,R_{n})$ be a partition of $T^{*}$ into $n$ nonempty, disjoint sets $R_{i}$. If each block $R_{i}$ is regular, $R$ is a regular partition. If $x,y\in R_{i}$, we write $x\equiv y\,(\text{mod}\ R)$._

**Definition 3**: $G$ _is predicated LL-regular if, for any two alternative productions of every nonterminal $A$ expanding to $\alpha_{i}$ and $\alpha_{j}$, there exists regular partition $R$ such that_
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120550594.png)
_always imply that $\alpha_{i}=\alpha_{j}$ and $\mathbb{S}_{i}=\mathbb{S}_{j}$.1_

[^1]: To be strictly correct, this definition technically corresponds to _Strong LL_-regular, rather than _LL_-regular as Nijholt [13] points out. _Strong LL_ parsers ignore left context when making decisions.

## 4 _LL_(\*) Parsers

Existing parsers for _LL_-regular grammars, proposed by Nijholt [13] and Poplawski [15], are linear but often impractical because they cannot parse infinite streams such as socket protocols and interactive interpreters. In the first of two passes, these parsers must read the input from right to left.

Instead, we propose a simpler left-to-right, one-pass strategy called _LL(*)_ that grafts _lookahead DFA_ onto _LL_ parsers. A lookahead DFA matches regular partition $R$ associated with a specific nonterminal and has an accept state for each $R_{i}$. At a decision point, _LL(*)_ parsers expand production $i$ if $R_{i}$ matches the remaining input. As a result, _LL(*)_ parsers are $O(n^{2})$, but in practice, they typically examine one or two tokens (Section 6). As with previous parsing strategies, an _LL(*)_ parser exists for every _LL_-regular grammar. Unlike previous work, _LL(*)_ parsers can take as input a _predicated LL_-regular grammar; they handle predicates by inserting special edges into the _lookahead DFA_ that correspond to the predicates.

**Definition 4**: _Lookahead DFA are DFA augmented with predicates and accept states that yield predicted production numbers. Formally, given predicated grammar $G=(N,T,P,S,\Pi,\mathcal{M})$, DFA $M=(\mathbb{S},Q,\Sigma,\Delta,D_{0},F)$ where:_

* $\mathbb{S}$ _is the system state inherited from surrounding parser_
* $Q$ _is the set of states_
* $\Sigma=T\cup\Pi$ _is the edge alphabet_
* $\Delta$ _is the transition function mapping_ $Q\times\Sigma\to Q$__
* $D_{0}\in Q$ _is the start state_
* $F=\{f_{1},f_{2},\ldots,f_{n}\}$ _is the set of final states, with one_ $f_{i}\in Q$ _per regular partition block_ $R_{i}$ _(production_ $i$_)_

A transition in $\Delta$ from state $p$ to state $q$ on symbol $a\in\Sigma$ has the form $p\xrightarrow{a}q$. There can be at most one such transition. Predicate transitions, written $p\xrightarrow{\pi}f_{i}$, must target a final state, but there can be more than one such transition eminating from $p$. The instantaneous configuration $c$ of the DFA is $(\mathbb{S},p,w_{r})$ where $\mathbb{S}$ is the system state and $p$ is the current state; the initial configuration is $(\mathbb{S},D_{0},w_{r})$. The notation $c\mapsto c^{\prime}$ means the DFA changes from configuration $c$ to $c^{\prime}$ using the rules in Figure 5. As with predicated grammars, the rules do not forbid ambiguous DFA paths arising from predicated transitions. In practice, ANTLR tests edges in order to resolve ambiguities.

For efficiency, lookahead DFA match _lookahead sets_ rather than continuation languages. Given $R=(\{ac^{*}\},\{bd^{*}\})$,for example, there is no point in looking beyond the first symbol. The lookahead sets are $(\{a\},\{b\})$.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120552478.png)

**Definition 5**: _Given partition $R$ distinguishing $n$ alternative productions, the lookahead set for production $i$ is the minimal-prefix set of $R_{i}$ that still uniquely predicts $i$:_
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120552226.png)

### 4.1 Erasing syntactic predicates

To avoid a separate recognition mechanism for syntactic predicates, we reduce syntactic predicates to semantic predicates that launch speculative parses. To "erase" syntactic predicate $(A^{\prime}_{i})$=>, we replace it with semantic predicate  $\{synpred(A^{\prime}_{i})\} ?$
created from $\alpha_{i}$ targets $p^{\prime}_{A}$. The language matched by the ATN is the same as the language of the original grammar.we can flip the result of calling function synpred, as suggested by Ford [6].

### 4.2 Arbitrary actions in predicate grammars

Formal predicated grammars fork new states $\mathbb{S}$ during speculation. In practice, duplicating system state is not feasible. Consequently, ANTLR deactivates mutators during specula- tion by default, preventing actions from “launching missiles” speculatively. However, some semantic predicates rely on changes made by mutators, such as the symbol table manip- ulations required to parse C. Avoiding speculation whenever possible attenuates this issue, but still leaves a semantic haz- ard. To address this issue, ANTLR supports a special kind of action, enclosed in double brackets {{...}}, that executes even during speculation. ANTLR requires the programmer to verify that these actions are either side-effect free or undoable. Luckily, symbol table manipulation actions, the most common {{...}} actions, usually get undone automati- cally. For example, a rule for a code block typically pushes a symbol scope but then pops it on exit. The pop effectively undoes the side-effects that occur during code block.

### 4.2 Arbitrary actions in predicate grammars

Formal predicated grammars fork new states $\mathbb{S}$ during speculation. In practice, duplicating system state is not feasible. Consequently, ANTLR deactivates mutators during specula- tion by default, preventing actions from “launching missiles” speculatively. However, some semantic predicates rely on changes made by mutators, such as the symbol table manip- ulations required to parse C. Avoiding speculation whenever possible attenuates this issue, but still leaves a semantic haz- ard. To address this issue, ANTLR supports a special kind of action, enclosed in double brackets {{...}}, that executes even during speculation. ANTLR requires the programmer to verify that these actions are either side-effect free or undoable. Luckily, symbol table manipulation actions, the most common {{...}} actions, usually get undone automati- cally. For example, a rule for a code block typically pushes a symbol scope but then pops it on exit. The pop effectively undoes the side-effects that occur during code block.

## 5. LL(*) Grammar Analysis

For LL(\*), analyzing a grammar means finding a lookahead DFA for each parsing decision, i.e., for each nonterminal in the grammar with multiple productions. In our discussions, we use A as the nonterminal in question and αi for i ∈ 1..n as the corresponding collection of right-hand sides. Our goal is to find for each A a regular partition $R$, represented by a DFA, that distinguishes between productions. To succeed, A must be LL-regular: partition block $R_i$ must contain every sentence in $C(\alpha_i)$, the continuation language of αi, and the $R_i$ must be disjoint. The DFA tests the remaining input for membership in each $R_i$; matching $R_i$ predicts alternative i. For efficiency, the DFA matches lookahead sets instead of partition blocks.

It is important to point out that we are not parsing with the DFA, only predicting which production the parser should expand. The continuation language $C(\alpha_i)$ is often context- free, not regular, but experience shows there is usually an ap- proximating regular language that distinguishes between the

Grammar analysis is like an inter-procedural flow analysis that statically traces an ATN-like graph representation of a program, discovering all nodes reachable from a top-level call site. The unique configuration of a program for flow purposes is a graph node and the call stack used to reach that node. Depending on the type of analysis, it might also track some semantic context such as the parameters from the top-level call site.

Similarly, grammar analysis statically traces paths through the ATN reachable from the "call site" of production $\alpha_{i}$, which is the left edge state $p_{A,i}$. The terminal edges collected along a path emanating from $p_{A,i}$ represent a lookahead sequence. Analysis continues until each lookahead sequence is unique to a particular alternative. Analysis also needs to track any semantic predicate $\pi_{i}$ from the left edge of $\alpha_{i}$ in case it is needed to resolve ambiguities. Consequently, an _ATN configuration_ is a tuple $(p,i,\gamma,\pi)$ with ATN state $p$, predicted production $i$, ATN call stack $\gamma$, and optional predicate $\pi$. We will use the notation $c.p,c.i,c.\gamma$, and $c.\pi$ to denote projecting the state, alternative, stack, and predicate from configuration $c$, respectively. Analysis ignores machine storage $\mathbb{S}$ because it is unknown at analysis time.

### 5.2 Modified subset construction algorithm

For grammar analysis purposes, we modify subset construction to process ATN not NFA configurations. Each DFA state $D$ represents the set of possible configurations the ATN could be in after matching a prefix of the remaining input starting from state $p_{A,i}$. Key modifications include:

* The _closure_ operation simulates the push and pop of ATN nonterminal invocations.
* If all the configurations in a newly discovered state predict the same alternative, the analysis does not add the state to the work list; no more lookahead is necessary.
* To resolve ambiguities, the algorithm adds predicate transitions to final states if appropriate predicates exist.

The structure of the algorithm mirrors that of subset construction. It begins by creating DFA start state $D_{0}$ and adding it to a work list. Until no work remains, the algorithm adds new DFA states computed by the _move_ and _closure_ functions, which simulate the transitions of the ATN. We assume that the ATN corresponding to our input grammar G, $M_{G}=(Q_{M},N\cup T\cup\Pi\cup\mathcal{M},\Delta_{M},E_{M},F_{M})$, and the nonterminal $A$ that we are analyzing are in scope for all the operations of the algorithm.

Function _createDFA_, shown in Algorithm 8, is the entry point: calling _createDFA_$(p_{A})$ constructs the lookahead DFA for $A$. To create start state $D_{0}$, the algorithm adds configuration $(p_{A,i},i,[],\pi_{i})$ for each production $A\rightarrow\pi_{i}\,\alpha_{i}$ and configuration $(p_{A,i},i,[],-)$ for each production $A\rightarrow\alpha_{i}$; the symbol "$-$" denotes the absence of a predicate. The core of _createDFA_ is a combined _move-closure_ operation that creates new DFA states by finding the set of ATN states directly reachable upon each input terminal symbol $a\in T$:

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120601200.png)

and then adding the closure of those configurations. Once the algorithm identifies a new state $D^{\prime}$, it invokes _resolve_ to check for and resolve ambiguities. If all of the configurations in $D^{\prime}$ predict the same alternative $j$, then $D^{\prime}$ is marked as $f_{j}$, the accept state for alternative $j$, and $D^{\prime}$ is not added to the work list: once the algorithm can uniquely identify which production to predict, there is no point in examining more of the input. This optimization is how the algorithm constructs DFA that match the minimum lookahead sets $LA_{j}$ instead of the entire remaining input. Next, the algorithm adds an edge from $D$ to $D^{\prime}$ on terminal $a$. Finally, for each configuration $c\in D$ with a predicate that resolves an ambiguity, _createDFA_ adds a transition predicated on $c.\pi$ from $D$ to the final state for alternative $c.i$. The test _wasResolved_ checks whether the _resolve_ step marked configuration $c$ as having been resolved by a predicate.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120601406.png)

to assume any production $p_{1}\xrightarrow{A}p_{2}$ in the input grammar might have invoked $A$, and so _closure_ must chase all such states $p_{2}$.

If _closure_ detects recursive nonterminal invocations (sub-machines directly or indirectly invoking themselves) in more than one alternative, it terminates DFA construction for $A$ by throwing an exception; Section 5.4 describes our fall back strategy. If _closure_ detects recursion deeper than internal constant $m$, _closure_ marks the state parameter $D$ as having _overflowed_. In this case, DFA construction for $A$ continues but _closure_ no longer pursues paths derived from $c$'s state and stack. We discuss this situation more in Section 5.3.

**DFA State Equivalence** The analysis algorithm relies on the notion of equivalence for DFA states:

Definition 6: _Two DFA states are equivalent, $D\equiv D^{\prime}$, if their configuration sets are equivalent. Two ATN configurations are equivalent, $c\equiv c^{\prime}$, if the $p$, $i$, and $\pi$ components are equal and their stacks are equivalent. Two stacks are equivalent, $\gamma_{1}\equiv\gamma_{2}$, if they are equal, if at least one is empty, or if one is a suffix of the other._

This definition of stack equivalence reflects ATN context information when _closure_ encounters a submachine stop state. Because analysis searches the ATN for all possible lookahead sequences, an empty stack is like a wildcard. Any transition $p_{1}\xrightarrow{A}p_{2}$ could have invoked $A$, so analysis must include the _closure_ of every such $p_{2}$. Consequently, a _closure_ set can have two configurations $c$ and $c^{\prime}$ with the same ATN state but with $c.\gamma=\epsilon$ and $c^{\prime}.\gamma\neq\epsilon$. This can only happen when _closure_ reaches a nonterminal's submachine stop state with an empty stack and by chasing states following references to that nonterminal, _closure_ reenters that submachine. For example, consider the DFA for $S$ in grammar $S\to a|\epsilon,A\to SS$. Start state construction computes _closure_ at positions $S\to\,\cdot\,a$ and $S\to\,\cdot\,\epsilon$ then $S\to\epsilon\,\cdot\,S$'s stop state. The state stack is empty so _closure_ chases states following references to $S$, such as position $A\to S\cdot S$. Finally, _closure_ reenters $S$, this time with a nonempty stack.

Equivalence of $\gamma_{1}\equiv\gamma_{1}\gamma_{2}$ where $\gamma_{1},\gamma_{2}\neq\epsilon$ degenerates to the previous case of $\gamma_{1}=\epsilon$. Given configurations $(p,\_,\gamma_{1},\_)$ and $(p,\_,\gamma_{1}\gamma_{2},\_)$ in $D$, _closure_ reaches $p$ following the same sequence of most recent submachine invocations, $\gamma_{1}$. Once $\gamma_{1}$ pops off, _closure_ has configurations $(p,\_,\|,\_)$ and $(p,\_,\gamma_{2},\_)$.

**Resolve**  One of benefits of static analysis is that it can sometimes detect and warn users about ambiguous nonterminals. After _closure_ finishes, the _resolve_ function (Algorithm 10) looks for _conflicting configurations_ in its argument state $D$. Such configurations indicate that the ATN can match the same input with more than one production.

Definition 7: _If DFA state $D$ contains configurations $c=(p,i,\gamma_{i},\pi_{i})$ and $c^{\prime}=(p,j,\gamma_{j},\pi_{j})$ such that $i\neq j$ and $\gamma_{i}\equiv\gamma_{j}$, then $D$ is an ambiguous DFA state and $c$ and $c^{\prime}$ are conflicting configurations. The set of all alternative numbers that belong to a conflicting configuration of D is the conflict set of D._

For example, the ATN for subrule $(a|a)$ in $A\to(a|a)\,b$ merges back together and so analysis reaches the same state from both alternatives with the same (empty) stack context:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120602173.png)

$D_{0}=\{(p_{2},1),(p_{5},2)\}$, where we abbreviate $(p_{2},1,\|,-)$ as $(p_{2},1)$ for clarity. $D_{1}=\{(p_{3},1),(p_{4},1),(p_{6},2),(p_{4},2)\}$, reachable with symbol $a$, has conflicting configurations $(p_{4},1)$ and $(p_{4},2)$. No further lookahead will resolve the ambiguity because $ab$ is in the continuation language of both alternatives.

If _resolve_ detects an ambiguity, it calls _resolveWithPredicate_ (Algorithm 11) to see if the conflicting configurations have predicates that can resolve the ambiguity. For example, a predicated version of the previous grammar, $A\to(\{\pi_{1}\}?\,a\,|\,\{\pi_{2}\}?\,a)\,b$, yields ATN:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120602087.png)

$D_{0}$ for decision state $p_{1}$ starts with $\{(p_{2},1,\|,\pi_{1}),(p_{6},2,\|,\pi_{2})\}$ to which we add $\{(p_{3},1,\|,\pi_{1}),(p_{7},2,\|,\pi_{2})\}$ for _closure_. $D_{1}$ has conflicting configurations as before, but now predicates can resolve the issue at runtime with DFA $D_{0}\xrightarrow{a}D_{1}$, $D_{1}\xrightarrow{\pi_{1}}f_{1}$, $D_{1}\xrightarrow{\pi_{2}}f_{2}$.![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120603940.png)
If _resolve_ found predicates, it returns without emitting a warning, leaving _createDFA_ to incorporate the predicates into the DFA. Without predicates, there is no way to resolve the issue at runtime, so _resolve_ statically removes the ambiguity by giving precedence to $A$'s lowest conflicting alternative by removing configurations associated with higher-numbered conflicting alternatives. For example, in the unpredicted grammar for $A$ above, the ressulting DFA is $D_{0}\xrightarrow{}f_{1}$ because the analysis resolves conflicts by removing configurations not associated with highest precedence production $1$, leaving $\{(p_{3},1),(p_{4},1)\}$.

If _closure_ tripped the recursion overflow alarm, _resolve_ may not see conflicting configurations in $D$, but $D$ might still predict more than one alternative because the analysis terminated early to avoid nontermination. The algorithm can use predicates to resolve the potential ambiguity at runtime, if they exist. If not, the algorithm again resolves in favor of the lowest alternative number and issues a warning to the user.

### 5.3 Avoiding analysis intractability

Because the _LL_-regular condition is undecidable, we expect a potential infinite loop somewhere in any lookahead DFA construction algorithm. Recursive rules are the source of nontermination. Given configuration $c=(p,i,\gamma)$ (we omit $\pi$ from configurations for brevity in the following) the _closure_ of $c$ at transition $p\xrightarrow{A}p^{\prime}$ includes $(p_{A},i,p^{\prime}\gamma)$. If _closure_ reaches $p$ again, it will include $(p_{A},i,p^{\prime}\gamma)$. Ultimately, _closure_ will "pump" the recursive rule forever, leading to stack explosion. For example, for the ATN with recursive nonterminal $A$ given in Figure 6, the DFA start state $D_{0}$ for $S\to Ac\,|\,Ad$ is:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120603723.png)

Function $\mathit{move}(D_{0},a)$ reaches $(p_{8},1,p_{2})$ to create $D_{1}$ via $(p_{7},1,p_{2})$ in $D_{0}$. The _closure_ of $(p_{8},1,p_{2})$ traverses the implied $\epsilon$ edge to $p_{A}$, adding three new configurations: $(p_{A},1,p_{9}p_{2})$, $(p_{7},1,p_{9}p_{2})$, $(p_{10},1,p_{9}p_{2})$. $D_{1}$ has the same configuration as $D_{0}$ for $p_{7}$ but with a larger stack. The configuration stacks grow forever as $p_{9}^{m}p_{2}$ for recursion depth $m$, yielding an ever larger DFA path: $D_{0}\xrightarrow{\alpha}D_{1}\xrightarrow{\alpha}\ldots\xrightarrow{ \alpha}D_{m}$.

There are two solutions to this problem. Either we forget all but the top $m$ states on the stack (as Bermudez and Schimpf [2] do with $LAR(m)$) or simply avoid computing _closure_ on configurations with $m$ recursive invocations to any particular submachine start state. We choose the latter because it guarantees a strict superset of $\mathit{LL(k)}$ (when $m\geq k$). We do not have to worry about an approximation introducing invalid sequences in common between the alternatives. In contrast, Bermudez and Schimpf give a family of $LALR(1)$ grammars for which there is no fixed $m$ that gives a valid $\mathit{LAR(m)}$ parser for every grammar in the family. Hard-limiting recursion depth is not a serious restriction in practice. Programmers are likely to use (regular) grammar $S\to a^{*}bc\,|\,a^{*}bd$ instead of the version in Figure 6.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120605923.png)

### 5.4 Aborting DFA construction

As a heuristic, we terminate DFA construction for nonterminal $A$ upon discovering recursion in more than one alternative. (Analysis for $S$ in Figure 6 would actually terminate before triggering recursion overflow.) Such decisions are extremely unlikely to have exact regular partitions and, since our algorithm does not approximate lookahead, there is no point in pursuing the DFA in wain. ANTLR's implementation falls back on $\mathit{LL(1)}$ lookahead for $A$, with backtracking or other predicates if _resolve_ detects recursion in more than one alternative.

### 5.5 Hoisting Predicates

This algorithm and the formal predicated grammar semantics in Section 3 require that predicates appear at production left edges. This restriction is cumbersome in practice and can force users to duplicate predicates. The full algorithm in ANTLR automatically discovers and _hoists_ all predicates visible to a decision even from productions further down the derivation chain. (See [1] for full algorithm and discussion.) ANTLR's analysis also handles EBNF operators in the right-hand side of productions, _e.g._$A\to a^{*}b$ by adding cycles to the ATN.

## 6 Empirical Results

This paper makes a number of claims about the suitability and efficiency of our analysis algorithm and the _LL(*)_ parsing strategy. In this section, we support these claims with measurements obtained by running ANTLR 3.3 on six large, real-world grammars, described in Figure 12, and profiling the resulting parsers on large sample input sentences, described in Figure 13. We include two PEG-derived grammars to show ANTLR can generate valid parsers from PEGs. (Readers can examine the non-commercial grammars, sample input, test methodology, and raw analysis results [1].)

### 6.1 Static grammar analysis

We claim that our grammar analysis algorithm statically optimizes away backtracking from the vast majority of parsing decisions and does so in a reasonable amount of time. Table 1 summarizes ANTLR's analysis of the six grammars. ANTLR processes each of them in a few seconds except for the 8,231 line-SQL grammar, which takes 13.1 seconds. Analysis times include input grammar parsing, analysis (with EBNF construct processing and predicate hoisting [1]), and parser generation. (As with the exponentially-complex classic subset construction algorithm, ANTLR's analysis can hit a "land-mine" in rare cases; ANTLR provides a means to isolate the offending decisions and manually set their lookahead parameters. None of these grammars hits land-mines.)
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120607580.png)
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310120607927.png)

All the grammars use backtracking to some extent, providing evidence that it is not worth contorting a large grammar to make it _LL(k)_. The first three grammars use PEG mode, in which ANTLR automatically puts a syntactic predicate on the left edge of every production. Unlike a PEG parser, however, ANTLR can statically avoid backtracking in many cases. For example, ANTLR strips away syntactic predicates from all but 11.8% of the decisions in Javai.5. As expected, the Rats grammar has the highest ratio of backtracking decisions at 22.4% because C variable and function declarations and definitions look the same from the left edge. The author of the three commercial grammars manually-specified syntactic predicates, which reduced lookahead requirements. In 4 out of 6 grammars, ANTLR was able to construct cyclic DFA to avoid backtracking.

Table 1 shows that the vast majority of decisions in the sample grammars are fixed _LL(k)_ for some $k$. Table 2 reports the number of decisions per lookahead depth, showing that most decisions are in fact $LL(1)$. The table also shows that ANTLR is able to statically determine $k$ almost all the time, even though this problem is undecidable in general.

### 6.2 Parser runtime profiling

At runtime, we claim that _LL(*)_ parsing decisions use only a few tokens of lookahead on average and parse with reasonable speed. Table 3 shows that for each sample input file, the average lookahead depth per decision event is roughly one token, with PEG-mode parsers requiring almost two. The average parsing speed is about 26,000 lines / second for the first three grammars and 9,000 for the other grammars, which have tree-building overhead.

The average lookahead depth for just the backtracking decisions is less than six tokens, highlighting the fact that although backtracking can scan far ahead, usually it doesn't. For example, a parser might need to backtrack to match a C type specifier, but most of the time such specifiers look like int or int * and so they can be identified quickly. The grammars not derived from PEGs have much smaller maximum lookahead depths, indicating the authors did some restructuring to take advantage of $LL(k)$ efficiency. The RatsC grammar, in contrast, backtracks across an entire function (looking ahead 7,968 tokens in one decision event) instead of looking ahead just enough to distinguish declarations from definitions, _i.e._, between int f(); and int f() {...}.

Although we derived two sample grammars from _Rats1_, we did not compare parser execution times nor memory utilization as one might expect. Both ANTLR and _Rats1_ are practical parser generators with parsing speed and memory footprints suitable to most applications. _Rats1_ is also scannerless, unlike ANTLR, which makes memoization caches hard to compare.

ANTLR statically removes backtracking from most parser decisions as shown in Table 1. At runtime, ANTLR backtracks even less than predicted by static analysis. For example, statically we find an average of 10.9% of the decisions backtrack for our sample grammars but Table 4 shows that the generated parsers backtrack in only 6.8% of the decision events on average. The non-PEG-derived grammars backtrack only about 2.5% of the time. This is partly because some of the backtracking decisions manage uncommon grammar constructs. For example, there are 1,120 decisions of any kind in the TSQL grammar but the sample input exercises only 309 of them.

Most importantly, just because a decision _can_ backtrack, does not mean it will. The last column in Table 4 shows that the potentially backtracking decisions (PBDs) only backtrack about half the time on average across the sample grammars. The commercial VB.NET and TSQL grammars yield PBDs that backtrack in only about 30% of the decision events. Some PBDs never trigger backtracking events. Subtracting the first two columns in Table 4 gives the number of PBDs that avoid backtracking altogether.

We should point out that without memoization, backtracking parsers are exponentially complex in the worst case. This matters for grammars that do a lot of nested backtracking. For example, the RatsC grammar appears not to terminate if we turn off ANTLR memoization support. In constrast, the VB.NET and C# parsers are fine without it. Packrat parsing [5] achieve linear parsing results at the cost of the increased memory for the memoization cache. In the worst case, we need to squirrel away $O(|N|*n)$ decision event results (one for each nonterminal decision at each input position). The less we backtrack, the smaller the cache since ANTLR only memoizes while speculating.

## 7 Related work

Many parsing techniques exist, but currently the two dominant strategies are Tomita's bottom-up GLR [16] and Ford's top-down packrat parsing [5], commonly referred to by its associated meta-language PEG [6]. Both are nondeterministic in that parsers can use some form of speculation to make decisions. _LL(*)_ is an optimization of packrat parsing just as GLR is an optimization of Earley's algorithm [4]. The closer a grammar conforms to the underlying _LL_ or _LR_ strategy, the more efficient the parser in time and space. _LL(*)_ ranges from $O(n)$ to $O(n^{2})$ whereas GLR ranges from $O(n)$ to $O(n^{3})$. Surprisingly, the $O(n^{2})$ potential comes from cyclic lookahead DFA not backtracking (assuming we memoize). ANTLR generates _LL(*)_ parsers that are linear in practice and that greatly reduce speculation, reducing memoization overhead over pure packrat parsers.

GLR and PEG tend be scannerless, which is necessary if a tool needs to support _grammar composition_. Composition means that programmers can easily integrate one language into another or create new grammars by modifying and composing pieces from existing grammars. For example, the _Rats!_ Jeannie grammar elegantly composes all of C and Java.

The ideas behind _LL(*)_ are rooted in the 1970s. _LL(*)_ parsers without predicated lookahead DFA edges implement _LL_-regular grammars, which were introduced by Jarzabek and Krawczyk [8] and Nijholt [13]. Nijholt [13] and Poplawski [15] gave linear two-pass _LL_-regular parsing strategies that had to parse from right-to-left in their first pass, requiring finite input streams (_i.e._, not sockets or interactive streams). They did not consider semantic predicates.

Milton and Fischer [11] introduced semantic predicates to $LL(1)$ grammars but only allowed one semantic predicate per production to direct the parse and required the user to specify the lookahead set under which the predicates should be evaluated. Parr and Quong [14] introduced syntactic predicates and semantic predicate hoisting, the notion of incorporating semantic predicates from other nonterminals into parsing decisions. They did not provide a formal predicated grammar specification or an algorithm to discover visible predicates. In this paper, we give a formal specification and demonstrate limited predicate discovery during DFA construction. Grimm supports restricted semantic predicates in his PEG-based _Rats!_[7] and arbitrary actions but relies on programmers to avoid side-effects that cannot be undone. Recently, Jim _et al_ added semantic predicates to transducers capable of handling all CFGs with Yakker [9].

ANTLR 1.x introduced syntactic predicates as a manually-controlled backtracking mechanism. The ability to specify production precedence came as a welcome, but unanticipated side-effect of the implementation. Ford formalized the notion of ordered alternatives with PEGs. Backtracking in versions of ANTLR prior to 3.x suffered from exponential time complexity without memoization. Ford also solved this problem by introducing packrat parsers. ANTLR 3.x users can turn on memoization with an option.

_LL_-regular grammars are the analog of _LR_-regular grammars [3]. Bermudez and Schimpf [2] provided a parsing strategy for _LR_-regular grammars called $LAR(m)$. Parameter $m$ is a stack governor, similar to ours, that prevents analysis algorithm nontermination. $LAR(m)$ builds an $LR(0)$ machine and grafts on lookahead DFA to handle nondeterministic states. Finding a regular partition for every _LL_-regular or _LR_-regular parsing decision is undecidable. Analysis algorithms for _LL(*)_ and $LAR(m)$ that terminate construct a valid DFA for a subset of the _LL_-regular or _LR_-regular grammar decisions, respectively. In the natural language community, Nederhof [12] uses DFA to approximate entire CFGs, presumably to get quicker but less accurate language membership checks. Nederhof inflines rule invocations to a specific depth, $m$, effectively mimicking the constant from $LAR(m)$.

## 8 Conclusion

_LL(*)_ parsers are as expressive as PEGs and beyond due to semantic predicates. While GLR accepts left-recursive grammars, it cannot recognize context-sensitive languages as _LL(*)_ can. Unlike PEGs or GLR, _LL(*)_ parsers enable arbitrary action execution and provide good support for debugging and error handling. The _LL(*)_ analysis algorithm constructs cyclic lookahead DFA to handle non-_LL(k)_ constructs and then fails over to backtracking via syntactic predicates when it fails to find a suitable DFA. Experiments reveal that ANTLR generates efficient parsers, eliminating almost all backtracking. ANTLR is widely used in practice, indicating _LL(*)_ hits a sweet spot in the parsing spectrum.

Finally, we would like to thank Sriram Srinivasan for contributing to and coining the term _LL(*)_.

## References

* [1]Paper Appendix. [http://antlr.org/papers/LL-star/index.html](http://antlr.org/papers/LL-star/index.html).
* [2]Bermudez, M. E., and Schimpf, K. M. Practical arbitrary lookahead LR parsing. _JCSS 41_, 2 (1990).
* [3]Cohen, R., and Culik, K. LR-regular grammars: An extension of LR(k) grammars. In _SWAT '71_.
* [4]Earley, J. An efficient context-free parsing algorithm. _CACM 13_, 2 (1970), 94-102.
* [5]Ford, B. Packrat parsing: Simple, powerful, lazy, linear time. In _ICFP'02_, pp. 36-47.
* [6]Ford, B. Parsing expression grammars: A recognition-based syntactic foundation. In _POPL'04_, pp. 111-122.
* [7]Grimm, R. Better extensibility through modular syntax. In _PLDI'06_, pp. 38-51.

* [9]Jim, T., Mandelbaum, Y., and Walker, D. Semantics and algorithms for data-dependent grammars. In _POPL '10_.

* [10]McPeak, S., and Necula, G. C. Elkhound: A fast, practical GLR parser generator. In _CC'04_.
* [11]Milton, D. R., and Fischer, C. N. LL(k) parsing for attributed grammars. In _ICALP_ (1979).
* [12]Nederhof, M.-J. Practical experiments with regular approximation of context-free languages. _Comput. Linguist. 26_, 1 (2000), 17-44.
* [13]Nijholt, A. On the parsing of LL-regular grammars. In _MFCS_ (1976), Springer Verlag, pp. 446-452.
* [14]Parr, T. J., and Quong, R. W. Adding semantic and syntactic predicates to $LL(k)$--_pred-$LL(k)$_. In _CC'94_.

* [16]Tomita, M. _Efficient Parsing for Natural Language_. Kluwer Academic Publishers, 1986.
* [17]Woods, W. A. Transition network grammars for natural language analysis. _CACM 13_, 10 (1970), 591-606.
