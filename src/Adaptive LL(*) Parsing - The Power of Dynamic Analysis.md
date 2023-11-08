# Adaptive LL(\*) Parsing - The Power of Dynamic Analysis

## Abstract

Despite the advances made by modern parsing strategies such as PEG, _LL(*)_, GLR, and GLL, parsing is not a solved problem. Existing approaches suffer from a number of weaknesses, including difficulties supporting side-effecting embedded actions, slow and/or unpredictable performance, and counter-intuitive matching strategies. This paper introduces the _ALL(*)_ parsing strategy that combines the simplicity, efficiency, and predictability of conventional top-down _LL(k)_ parsers with the power of a GLR-like mechanism to make parsing decisions. The critical innovation is to move grammar analysis to parse-time, which lets _ALL(*)_ handle any non-left-recursive context-free grammar. _ALL(*)_ is $O(n^{4})$ in theory but consistently performs linearly on grammars used in practice, outperforming general strategies such as GLL and GLR by orders of magnitude. ANTLR 4 generates _ALL(*)_ parsers and supports direct left-recursion through grammar rewriting. Widespread ANTLR 4 use (5000 downloads/month in 2013) provides evidence that _ALL(*)_ is effective for a wide variety of applications.

## 1. Introduction

Computer language parsing is still not a solved problem in practice, despite the sophistication of modern parsing strategies and long history of academic study. When machine resources were scarce, it made sense to force programmers to control their grammars to fit the constraints of deterministic $LALR(k)$ or $LL(k)$ parser generators.1 As machine resources grew, researchers developed more powerful, but more costly, nondeterministic parsing strategies following both "bottom-up" ($LR$-style) and "top-down" ($LL$-style) approaches. Strategies include GLR [26], Parser Expression Grammar (PEG) [9], _LL(*)_[20] from ANTLR 3, and recently, GLL [25], a fully general top-down strategy.

Footnote 1: We use the term _deterministic_ in the way that deterministic finite automata (_DFA_) differ from nondeterministic finite automata (_NFA_): The next symbol(s) uniquely determine action.

Although these newer strategies are much easier to use than $LALR(k)$ and $LL(k)$ parser generators, they suffer from a variety of weaknesses. First, nondeterministic parsers sometimes have unanticipated behavior. GLL and GLR return multiple parse trees (forests) for ambiguous grammars because they were designed to handle natural language grammars, which are often intentionally ambiguous. For computer languages, ambiguity is almost always an error. One can certainly walk the constructed parse forest to disambiguate it, but that approach costs extra time, space, and machinery for the uncommon case. PEGs are unambiguous by definition but have a quirk where rule $A\to a\,|\,ab$ (meaning "$A$ matches either $a$ or $ab$") can never match $ab$ since PEGs choose the first alternative that matches a prefix of the remaining input. Nested backtracking makes debugging PEGs difficult.

Second, side-effecting programmer-supplied actions (mutators) like print statements should be avoided in any strategy that continuously speculates (PEG) or supports multiple interpretations of the input (GLL and GLR) because such actions may never really take place [17]. (Though DParser [24] supports "final" actions when the programmer is certain a reduction is part of an unambiguous final parse.) Without side effects, actions must buffer data for all interpretations in immutable data structures or provide undo actions. The former mechanism is limited by memory size and the latter is not always easy or possible. The typical approach to avoiding mutators is to construct a parse tree for post-parse processing, but such artifacts fundamentally limit parsing to input files whose trees fit in memory. Parsers that build parse trees cannot analyze large data files or infinite streams, such as network traffic, unless they can be processed in logical chunks.

Third, our experiments (Section 7) show that GLL and GLR can be slow and unpredictable in time and space. Their complexities are, respectively, $O(n^{3})$ and $O(n^{p+1})$ where $p$ is the length of the longest production in the grammar [14]. (GLR is typically quoted as $O(n^{3})$ because Kipps [15] gave such an algorithm, albeit with a constant so high as to be impractical.) In theory, general parsers should handle deterministic grammars in near-linear time. In practice, we found GLL and GLR to be "135x slower than _ALL(*)_ on a corpus of 12,920 Java 6 library source files (123M) and 6 orders of magnitude slower on a single 3.2M Java file, respectively.

_LL(*)_ addresses these weaknesses by providing a mostly deterministic parsing strategy that uses regular expressions, represented as _deterministic finite automata_ (DFA), to potentially examine the entire remaining input rather than the fixed $k$-sequences of _LL(k)_. Using DFA for lookahead limits _LL(*)_ decisions to distinguishing alternatives with regular lookahead languages, even though lookahead languages (set of all possible remaining input phrases) are often context-free. But the main problem is that the _LL(*)_ grammar condition is statically undecidable and grammar analysis sometimes fails to find regular expressions that distinguish between alternative productions. ANTLR 3's static analysis detects and avoids potentially-undecidable situations, failing over to backtracking parsing decisions instead. This gives _LL(*)_ the same $a\,|\,ab$ quirk as PEGsfor such decisions. Backtracking decisions that choose the first matching alternative also cannot detect obvious ambiguities such $A\rightarrow\alpha\,|\,\alpha$ where $\alpha$ is some sequence of grammar symbols that makes $\alpha\,|\,\alpha$ non-_LL(*)_.

### 1.1 Dynamic grammar analysis

In this paper, we introduce Adaptive _LL(*)_, or _ALL(*)_, parsers that combine the simplicity of deterministic top-down parsers with the power of a GLR-like mechanism to make parsing decisions. Specifically, LL parsing suspends at each prediction decision point (nonterminal) and then resumes once the prediction mechanism has chosen the appropriate production to expand. The critical innovation is to move grammar analysis to parse-time; no static grammar analysis is needed. This choice lets us avoid the undecidability of static _LL(*)_ grammar analysis and lets us generate correct parsers (Theorem 6.1) for any non-left-recursive context-free grammar (CFG). While static analysis must consider all possible input sequences, dynamic analysis need only consider the finite collection of input sequences actually seen.

The idea behind the _ALL(*)_ prediction mechanism is to launch subparsers at a decision point, one per alternative production. The subparsers operate in pseudo-parallel to explore all possible paths. Subparsers die off as their paths fail to match the remaining input. The subparsers advance through the input in lockstep so analysis can identify a sole survivor at the minimum lookahead depth that uniquely predicts a production. If multiple subparsers coalesce together or reach the end of file, the predictor announces an ambiguity and resolves it in favor of the lowest production number associated with a surviving subparser. (Productions are numbered to express precedence as an automatic means of resolving ambiguities like PEGs; Bi-son also resolves conflicts by choosing the production specified first.) Programmers can also embed _semantic predicates_[22] to choose between ambiguous interpretations.

_ALL(*)_ parsers memoize analysis results, incrementally and dynamically building up a cache of DFA that map lookahead phrases to predicted productions. (We use the term _analysis_ in the sense that _ALL(*)_ analysis yields lookahead DFA like static _LL(*)_ analysis.) The parser can make future predictions at the same parser decision and lookahead phrase quickly by consulting the cache. Unfamiliar input phrases trigger the grammar analysis mechanism, simultaneously predicting an alternative and updating the DFA. DFA are suitable for recording prediction results, despite the fact that the lookahead language at a given decision typically forms a context-free language. Dynamic analysis only needs to consider the finite context-free language subsets encountered during a parse and any finite set is regular.

To avoid the exponential nature of nondeterministic subparsers, prediction uses a _graph-structured stack_ (GSS) [25] to avoid redundant computations. GLR uses essentially the same strategy except that _ALL(*)_ only predicts productions with such subparsers whereas GLR actually parses with them. Consequently, GLR must push terminals onto the GSS but _ALL(*)_ does not.

_ALL(*)_ parsers handle the task of matching terminals and expanding nonterminals with the simplicity of $LL$ but have $O(n^{4})$ theoretical time complexity (Theorem 6.3) because in the worst-case, the parser must make a prediction at each input symbol and each prediction must examine the entire remaining input; examining an input symbol can cost $O(n^{2})$. $O(n^{4})$ is in line with the complexity of GLR. In Section 7, we show empirically that _ALL(*)_ parsers for common languages are efficient and exhibit linear behavior in practice.

The advantages of _ALL(*)_ stem from moving grammar analysis to parse time, but this choice places an additional burden on grammar functional testing. As with all dynamic approaches, programmers must cover as many grammar position and input sequence combinations as possible to find grammar ambiguities. Standard source code coverage tools can help programmers measure grammar coverage for _ALL(*)_ parsers. High coverage in the generated code corresponds to high grammar coverage.

The _ALL(*)_ algorithm is the foundation of the ANTLR 4 parser generator (ANTLR 3 is based upon _LL(*)_). ANTLR 4 was released in January 2013 and gets about 5000 downloaded/smooth (source, binary, or ANTLRworks2 development environment, counting non-robot entries in web logs with unique IP addresses to get a lower bound.) Such activity provides evidence that _ALL(*)_ is useful and usable.

The remainder of this paper is organized as follows. We begin by introducing the ANTLR 4 parser generator (Section 2) and discussing the _ALL(*)_ parsing strategy (Section 3). Next, we define _predicated grammars_, their _augmented transition network_ representation, and lookahead DFA (Section 4). Then, we describe _ALL(*)_ grammar analysis and present the parsing algorithm itself (Section 5). Finally, we support our claims regarding _ALL(*)_ correctness (Section 6) and efficiency (Section 7) and examine related work (Section 8). Appendix A has proofs for key _ALL(*)_ theorems, Appendix B discusses algorithm pragmatics, Appendix C has left-recursion elimination details.

## 2.  ANTLR 4

ANTLR 4 accepts as input any context-free grammar that does not contain indirect or hidden left-recursion.2 From the grammar, ANTLR 4 generates a recursive-descent parser that uses an _ALL(*)_ production prediction function (Section 3). ANTLR currently generates parsers in Java or C#. ANTLR 4 grammars use _yacc_-like syntax with extended BNF (EBNF) operators such as Kleene star (\*) and token literals in single quotes. Grammars contain both lexical and syntactic rules in a combined specification for convenience. ANTLR 4 generates both a lexer and a parser from the combined specification. By using individual characters as input symbols, ANTLR 4 grammars can be scannerless and composable because _ALL(*)_ languages are closed under union (Theorem 6.2), providing the benefits of modularity described by Grimm [10]. (We will henceforth refer to ANTLR 4 as ANTLR and explicitly mark earlier versions.)

Programmers can embed side-effecting actions (_mutators_), written in the host language of the parser, in the grammar. The actions have access to the current state of the parser. The parser ignores mutators during speculation to prevent actions from "launching missiles" speculatively. Actions typically extract information from the input stream and create data structures.

ANTLR also supports _semantic predicates_, which are side-effect free Boolean-valued expressions written in the host language that determine the semantic viability of a particular production. Semantic predicates that evaluate to false during the parse render the surrounding production nonviable, dynamically altering the language generated by the grammar at parse-time.3 Predicates significantly increase the strength of a parsing strategy because predicates can examine the parse stack and surrounding input context to provide an informal context-sensitive parsing capability. Semantic actions and predicates typically work together to alter the parse based upon previously-discovered information. For example, a C grammar could have embedded actions to define type symbols from constructs, like typedef int i32;, and predicates to distinguish type names from other identifiers in subsequent definitions like i32 x;.

Footnote 3: Previous versions of ANTLR supported _syntactic predicates_ to disambiguate cases where static grammar analysis failed; this facility is not needed in ANTLR4 because of _ALL(*)_’s dynamic analysis.

### 2.1 Sample grammar

Figure 1 illustrates ANTLRs yacc-like metalanguage by giving the grammar for a simple programming language with assignment and expression statements terminated by semicolons. There are two grammar features that render this grammar non-_LL(*)_ and, hence, unacceptable to ANTLR 3. First, rule expr is left recursive. ANTLR 4 automatically rewrites the rule to be non-left-recursive and unambiguous, as described in Section 2.2. Second, the alternative productions of rule stat have a common recursive prefix (expr), which is sufficient to render stat undecidable from an _LL(*)_ perspective. ANTLR 3 would detect recursion on production left edges and fail over to a backtracking decision at runtime.

$\text{Predicate \{!enum\_is\_keyword\}?}$ in rule id allows or disallows enum as a valid identifier according to the predicate at the moment of prediction. When the predicate is false, the parser treats id as just id : ID ; disallowing enum as an id as the lexer matches enum as a separate token from ID. This example demonstrates how predicates allow a single grammar to describe subsets or variations of the same language.

### 2.2 Left-recursion removal

The _ALL(*)_ parsing strategy itself does not support left-recursion, but ANTLR supports direct left-recursion through grammar rewriting prior to parser generation. Direct left-recursion covers the most common cases, such as arithmetic expression productions, like $E\to E\cdot\mathbf{id}$, and C declarators. We made an engineering decision not to support indirect or hidden left-recursion because these forms are much less common and removing all left recursion can lead to exponentially-big transformed grammars. For example, the C11 language specification grammar contains lots of direct left-recursion but no indirect or hidden recursion. See Appendix 2.2 for more details.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090557572.png)

### 2.3 Lexical analysis with _ALL(*)_

ANTLR uses a variation of _ALL(*)_ for lexing that fully matches tokens instead of just predicting productions like _ALL(*)_ parsers do. After warm-up, the lexer will have built a DFA similar to what regular-expression based tools such as lex would create statically. The key difference is that _ALL(*)_ lexers are predicated context-free grammars not just regular expressions so they can recognize context-free tokens such as nested comments and can gate tokens in and out according to semantic context. This design is possible because _ALL(*)_ is fast enough to handle lexing as well as parsing.

_ALL(*)_ is also suitable for scannerless parsing because of its recognition power, which comes in handy for context-sensitive lexical problems like merging C and SQL languages. Such a union has no clear lexical sentinels demarcating lexical regions:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090558283.png)
See grammar code/extras/CSQL in [19] for a proof of concept.

## 3. Introduction to _All(*)_ parsing

In this section, we explain the ideas and intuitions behind _ALL(*)_ parsing. Section 5 will then present the algorithm more formally. The strength of a top-down parsing strategy is related to how the strategy chooses which alternative production to expand for the current nonterminal. Unlike _LL(k)_ and _LL(*)_ parsers, _ALL(*)_ parsers always choose the first alternative that leads to a valid parse. All non-left-recursive grammars are therefore _ALL(*)_.

Instead of relying on static grammar analysis, an _ALL(*)_ parser adapts to the input sentences presented to it at parse-time. The parser analyzes the current decision point (nonterminal with multiple productions) using a GLR-like mechanism to explore all possible decision paths with respect to the current "call" stack of in-process nonterminals and the remaining input _on-demand_. The parser incrementally and dynamically builds a lookahead DFA per decision that records a mapping from lookahead sequence to predicted production number. If the DFA constructed to date matches the current lookahead, the parser can skip analysis and immediately expand the predicted alternative. Experiments in Section 7 show that _ALL(*)_ parsers usually get DFA cache hits and that DFA are critical to performance.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090559905.png)
Because _ALL(*)_ differs from deterministic top-down methods only in the prediction mechanism, we can construct conventional recursive-descent _LL_ parsers but with an important twist. _ALL(*)_ parsers call a special prediction function, _adaptivePredict_, that analyzes the grammar to construct lookahead DFA instead of simply comparing the lookahead to a statically-computed token set. Function _adaptivePredict_ takes a nonterminal and parser call stack as parameters and returns the predicted production number or throws an exception if there is no viable production. For example, rule stat from the example in Section 2.1 yields a parsing procedure similar to Figure 2.

_ALL(*)_ prediction has a structure similar to the well-known NFA-to-DFA _subset construction_ algorithm. The goal is to discover the set of states the parser could reach after having seen some or all of the remaining input relative to the current decision. As in subset construction, an _ALL(*)_ DFA state is the set of parser configurations possible after matching the input leading to that state. Instead of an NFA, however, _ALL(*)_ simulates the actions of an _augmented recursive transition network_ (ATN) [27] representation of the grammar since ATNs closely mirror grammar structure. (ATNs look just like syntax diagrams that can have actions and semantic predicates.) _LL(*)_'s static analysis also operates on an ATN for the same reason. Figure 3 shows the ATN submachine for rule stat.

An _ATN configuration_ represents the execution state of a subparser and tracks the ATN state, predicted production number, and ATN subparser call stack: tuple $(p,i,\gamma)$.[^4] Configurations include production numbers so prediction can identify _which_ production matches the current lookahead. Unlike static _LL(*)_ analysis, _ALL(*)_ incrementally builds DFA considering just the lookahead sequences it has seen instead of all possible sequences.

[^4]: Component $i$ does not exist in the machine configurations of GLL, GLR, or Earley [8].
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090600994.png)
When parsing reaches a decision for the first time, _adaptivePredict_ initializes the lookahead DFA for that decision by creating a DFA start state, $D_{0}$. $D_{0}$ is the set of ATN subparser configurations reachable without consuming an input symbol starting at each production left edge. For example, construction of $D_{0}$ for nonterminal stat in Figure 3 would first add ATN configurations $(p,1,[])$ and $(q,2,[])$ where $p$ and $q$ are ATN states corresponding to production 1 and 2's left edges and $[]$ is the empty subparser call stack (if stat is the start symbol).

Analysis next computes a new DFA state indicating where ATN simulation could reach after consuming the first lookahead symbol and then connects the two DFA states with an edge labeled with that symbol. Analysis continues, adding new DFA states, until all ATN configurations in a newly-created DFA state predict the same production: $(-,i,-)$. Function _adaptivePredict_ marks that state as an accept state and returns to the parser with that production number. Figure 3(a) shows the lookahead DFA for decision stat after _adaptivePredict_ has analyzed input sentence x=y;. The DFA does not look beyond = because = is sufficient to uniquely distinguish expr's productions. (Notation :1 means "predict production 1.")

In the typical case, _adaptivePredict_ finds an existing DFA for a particular decision. The goal is to find or build a path through the DFA to an accept state. If _adaptivePredict_ reaches a (non-accept) DFA state without an edge for the current lookahead symbol, it reverts to ATN simulation to extend the DFA (without rewinding the input). For example, to analyze a second input phrase for stat, such as f(x) ; _adaptivePredict_ finds an existing ID edge from the $D_{0}$ and jumps to $s_{1}$ without ATN simulation. There is no existing edge from $s_{1}$ for the left parenthesis so analysis simulates the ATN to complete a path to an accept state, which predicts the second production, as shown in Figure 3(b). Note that because sequence ID(ID) predicts both productions, analysis continues until the DFA has edges for the = and ; symbols.

If ATN simulation computes a new target state that already exists in the DFA, simulation adds a new edge targeting the existing state and switches back to DFA simulation mode starting at that state. Targeting existing states is how cycles can appear in the DFA. Extending the DFA to handle unfamiliar phrases empirically decreases the likelihood of future ATN simulation, thereby increasing parsing speed (Section 7).

### 3.1 Predictions sensitive to the call stack

Parsers cannot always rely upon lookahead DFA to make correct decisions. To handle all non-left-recursive grammars, _ALL(*)_ prediction must occasionally consider the parser call stack available at the start of prediction (denoted $\gamma_{0}$ in Section 5). To illustrate the need for stack-sensitive predictions, consider that predictions made while recognizing a Java method definition might depend on whether the method was defined within an interface or class definition. (Java interface methods cannot have bodies.) Here is a simplified grammar that exhibits a stack-sensitive decision in nonterminal $A$:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090601264.png)
Without the parser stack, no amount of lookahead can uniquely distinguish between $A$'s productions. Lookahead $ba$ predicts $A\to b$ when $B$ invokes $A$ but predicts $A\rightarrow\epsilon$ when $C$ invokes $A$. If prediction ignores the parser call stack, there is a prediction conflict upon $ba$.

Parsers that ignore the parser call stack for prediction are called _Strong LL_ (_SLL_) parsers. The recursive-descent parsers programmers build by hand are in the _SLL_ class. By convention, the literature refers to _SLL_ as _LL_ but we distinguish the terms since "real" _LL_ is needed to handle all grammars. The above grammar is $LL(2)$ but not $SLL(k)$ for any $k$, though duplicating $A$ for each call site renders the grammar $SLL(2)$.

Creating a different lookahead DFA for each possible parser call stack is not feasible since the number of stack permutations is exponential in the stack depth. Instead, we take advantage of the fact that most decisions are not stack-sensitive and build lookahead DFA ignoring the parser call stack. If _SLL_ ATN simulation finds a prediction conflict (Section 5.3), it cannot be sure if the lookahead phrase is ambiguous or stack-sensitive. In this case, _adaptivePredict_ must re-examine the lookahead using the parser stack $\gamma_{0}$. This hybrid or _optimized_ LL _mode_ improves performance by caching stack-insensitive prediction results in lookahead DFA when possible while retaining full stack-sensitive prediction power. Optimized _LL_ mode applies on a per-decision basis, but _two-stage parsing_, described next, can often avoid _LL_ simulation completely. (We henceforth use _SLL_ to indicate stack-insensitive parsing and _LL_ to indicate stack-sensitive.)

### 3.2 Two-stage _All(*)_ parsing

_SLL_ is weaker but faster than _LL_. Since we have found that most decisions are _SLL_ in practice, it makes sense to attempt parsing entire inputs in "_SLL_ only mode," which is stage one of the two-stage _ALL(*)_ parsing algorithm. If, however, _SLL_ mode finds a syntax error, it might have found an _SLL_ weakness or a real syntax error, so we have to retry the entire input using optimized _LL_ mode, which is stage two. This counter-intuitive strategy, which potentially parses entire inputs twice, can dramatically increase speed over the single-stage optimized _LL_ mode stage. For example, two-stage parsing with the Java grammar (Section 7) is 8x faster than one-stage optimized _LL_ mode to parse a 123M corpus. The two-stage strategy relies on the fact that _SLL_ either behaves like _LL_ or gets a syntax error (Theorem 6.5). For invalid sentences, there is no derivation for the input regardless of how the parser chooses productions. For valid sentences, _SLL_ chooses productions as _LL_ would or picks a production that ultimately leads to a syntax error (_LL_ finds that choice nonviable). Even in the presence of ambiguities, _SLL_ often resolves conflicts as _LL_ would. For example, despite a few ambiguities in our Java grammar, _SLL_ mode correctly parses all inputs we have tried without failing over to _LL_. Nonetheless, the second (_LL_) stage must remain to ensure correctness.

## 4 Predicated grammars, ATNs, and DFA

To formalize _ALL(*)_ parsing, we first need to recall some background material, specifically, the formal definitions of predicate grammars, ATNs, and Lookahead DFA.

### Predicated grammars

To formalize _ALL(*)_ parsing, we first need to formally define the predicated grammars from which they are derived. A predicated grammar $G=(N,T,P,S,\Pi,\mathcal{M})$ has elements:

$\bullet$$N$ is the set of nonterminals (rule names)
$\bullet$$T$ is the set of terminals (tokens)
$\bullet$$P$ is the set of productions
$\bullet$$S\in N$ is the start symbol
$\bullet$$\Pi$ is a set of side-effect-free semantic predicates
$\bullet$$\mathcal{M}$ is a set of actions (mutators)

Predicated _ALL(*)_ grammars differ from those of _LL(*)_[20] only in that _ALL(*)_ grammars do not need or support syntactic predicates. Predicated grammars in the formal sections of this paper use the notation shown in Figure 5. The derivation rules in Figure 6 define the meaning of a predicated grammar. To support semantic predicates and mutators, the rules reference state $\mathbb{S}$, which abstracts user state during parsing. The judgment form $(\mathbb{S},\alpha)\Rightarrow(\mathbb{S}^{\prime},\beta)$ may be read: "In machine state $\mathbb{S}$, grammar sequence $\alpha$ reduces in one step to modified state $\mathbb{S}^{\prime}$ and grammar sequence $\beta$." The judgment $(\mathbb{S},\alpha)\Rightarrow^{*}(\mathbb{S}^{\prime},\beta)$ denotes repeated applications of the one-step reduction rule. These reduction rules specify a leftmost derivation. A production with a semantic predicate $\pi_{i}$ is viable only if $\pi_{i}$ is true of the current state $\mathbb{S}$. Finally, an action production uses the specified mutator $\mu_{i}$ to update the state.

Formally, the language generated by grammar sequence $\alpha$ in user state $\mathbb{S}$ is $L(\mathbb{S},\alpha)=\{w\,|\,(\mathbb{S},\alpha)\Rightarrow^{*}(\mathbb{S}^{ \prime},w)\}$ and the language of grammar $G$ is $L(\mathbb{S}_{0},G)=\{w\,|\,(\mathbb{S}_{0},S)\Rightarrow^{*}(\mathbb{S},w)\}$ for initial user state $\mathbb{S}_{0}$ ($\mathbb{S}_{0}$ can be empty). If $u$ is a prefix of $w$ or equal to $w$, we write $u\preceq w$. Language $L$ is _ALL(*) iff_ there exists an _ALL(*)_ grammar for $L$. Theoretically, the language class of $L(G)$ is recursively enumerable because each mutator could be a Turing machine. In reality, grammar writers do not use this generality so it is standard practice to consider the language class to be the context-sensitive languages instead. The class is context-sensitive rather than context-free as predicates can examine the call stack and terminals to the left and right.

This formalism has various syntactic restrictions not present in actual ANTLR grammars, for example, forcing mutators into their own rules and disallowing the common Extended BNF (EBNF) notation such as $\alpha^{*}$ and $\alpha^{+}$ closures. We can make these restrictions without loss of generality because any grammar in the general form can be translated into this more restricted form.

### 4.2 Resolving ambiguity

An ambiguous grammar is one in which the same input sequence can be recognized in multiple ways. The rules in Figure 6 do not preclude ambiguity. However, for a practical programming language parser, each input should correspond to a unique parse. To that end, _ALL(*)_ uses the order of the productions in a rule to resolve ambiguities in favor of the production with the lowest number. This strategy is a concise way for programmers to resolve ambiguities automatically and resolves the well-known _if-then-else_ ambiguity in the usual way by associating the _else_ with the most recent _if_. PEGs and Bison parsers have the same resolution policy.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090603335.png)

To resolve ambiguities that depend on the current state $\mathbb{S}$, programmers can insert semantic predicates but must make them mutually exclusive for all potentially ambiguous input sequences to render such productions unambiguous. Statically, mutual exclusion cannot be enforced because predicates are written in a Turing-complete language. At parse-time, however, _ALL(*)_ evaluates predicates and dynamically reports input phrases for which multiple, predicated productions are viable. If the programmer fails to satisfy mutual exclusivity, _ALL(*)_ uses production order to resolve the ambiguity.

### 4.3 Augmented transition networks

Given predicated grammar $G=(N,T,P,S,\Pi,\mathcal{M})$, the corresponding ATN $M_{G}=(Q,\Sigma,\Delta,E,F)$ has elements [20]:

* $Q$ is the set of states
* $\Sigma$ is the edge alphabet $N\cup T\cup\Pi\cup\mathcal{M}$
* $\Delta$ is the transition relation mapping $Q\times(\Sigma\cup\epsilon)\to Q$
* $E\in Q=\{p_{A}\mid A\in N\}$ is set of submachine entry states
* $F\in Q=\{p^{\prime}_{A}\mid A\in N\}$ is set of submachine final states

ATNs resemble the syntax diagrams used to document programming languages, with an ATN submachine for each non-terminal. Figure 7 shows how to construct the set of states $Q$ and edges $\Delta$ from grammar productions. The start state for $A$ is $p_{A}\in Q$ and targets $p_{A,i}$, created from the left edge of $\alpha_{i}$, with an edge in $\Delta$. The last state created from $\alpha_{i}$ targets $p^{\prime}_{A}$. Nonterminal edges $p\xrightarrow{A}q$ are like function calls. They transfer control of the ATN to $A$'s submachine, pushing return state $q$ onto a state call stack so it can continue from $q$ after reaching the stop state for $A$'s submachine, $p^{\prime}_{A}$. Figure 8 gives the ATN for a simple grammar. The language matched by the ATN is the same as the language of the original grammar.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090604359.png)

### 4.4 Lookahead DFA

_ALL(*)_ parsers record prediction results obtained from ATN simulation with _lookahead DFA_, which are DFA augmented with accept states that yield predicted production numbers. There is one accept state per production of a decision.
**Definition 4.1**. Lookahead DFA _are DFA with augmented accept states that yield predicted production numbers. For predicated grammar $G=(N,T,P,S,\Pi,\mathcal{M})$, DFA $M=(Q,\Sigma,\Delta,D_{0},F)$ where:_

* $Q$ is the set of states
* $\Sigma=T$ is the edge alphabet
* $\Delta$ is the transition function mapping $Q\times\Sigma\to Q$
* $D_{0}\in Q$ is the start state
* $F\in Q=\{f_{1},f_{2},\ldots,f_{n}\}$ final states, one $f_{i}$ per prod. $i$
A transition in $\Delta$ from state $p$ to state $q$ on symbol $a\in\Sigma$ has the form $p\xrightarrow{a}q$ and we require $p\xrightarrow{a}q^{\prime}$ implies $q=q^{\prime}$.

## 5.  _ALL(*)_ parsing algorithm

With the definitions of grammars, ATNs, and lookahead DFA formalized, we can present the key functions of the _ALL(*)_ parsing algorithm. This section starts with a summary of the functions and how they fit together then discusses a critical graph data structure before presenting the functions themselves. We finish with an example of how the algorithm works.

Parsing begins with function _parse_ that behaves like a conventional top-down _LL(k)_ parse function except that _ALL(*)_ parsers predict productions with a special function called _adaptivePredict_, instead of the usual "switch on next $k$ token types" mechanism. Function _adaptivePredict_ simulates an ATN representation of the original predicated grammar to choose an $\alpha_{i}$ production to expand for decision point $A\rightarrow\alpha_{1}\,|\,...\,|\,\alpha_{n}$

Conceptually, prediction is a function of the current parser call stack $\gamma_{0}$, remaining input $w_{r}$, and user state $\mathbb{S}$ if $A$ has predicates. For efficiency, prediction ignores $\gamma_{0}$ when possible (Section 3.1) and uses the minimum lookahead from $w_{r}$.

To avoid repeating ATN simulations for the same input and nonterminal, _adaptivePredict_ assembles DFA that memoize input-to-predicted-production mappings, one DFA per nonterminal. Recall that each DFA state, $D$, is the set of ATN configurations possible after matching the lookahead symbols leading to that state. Function _adaptivePredict_ calls _startState_ to create initial DFA state, $D_{0}$, and then _SL4predict_ to begin simulation.

Function _SLLpredict_ adds paths to the lookahead DFA that match some or all of $w_{r}$ through repeated calls to _target_. Function _target_ computes DFA target state $D^{\prime}$ from current state $D$ using _move_ and _closure_ operations similar to those found in _subset construction_. Function _move_ finds all ATN configurations reachable on the current input symbol and _closure_ finds all configurations reachable without traversing a terminal edge. The primary difference from subset construction is that _closure_ simulates the call and return of ATN submachines associated with nonterminals.

If _SLL_ simulation finds a conflict (Section 5.3), _SLLpredict_ rewinds the input and calls _LLpredict_ to retry prediction, this time considering $\gamma_{0}$. Function _LLpredict_ is similar to _SLLpredict_ but does not update a nonterminal's DFA because DFA must be stack-insensitive to be applicable in all stack contexts. Conflicts within ATN configuration sets discovered by _LLpredict_ represent ambiguities. Both prediction functions use _getConficSetsPerLoc_ to detect conflicts, which are configurations representing the same parser location but different productions. To avoid failing over to _LLpredict_ unnecessarily, _SLLpredict_ uses _getProdSetsPerState_ to see if a potentially non-conflicting DFA path remains when _getConflictSetsPerLoc_ reports a conflict. If so, it is worth continuing with _SLLpredict_ on the chance that more lookahead will resolve the conflict without recourse to full _LL_ parsing.

Before describing these functions in detail, we review a fundamental graph data structure that they use to efficiently manage multiple call stacks _a la_ GLL and GLR.

### 5.1 Graph-structured call stacks

The simplest way to implement _ALL(*)_ prediction would be a classic backtracking approach, launching a subparser for each $\alpha_{i}$. The subparsers would consume all remaining input because backtracking subparsers do not know when to stop parsing--they are unaware of other subparsers' status. The independent subparsers would also lead to exponential time complexity. We address both issues by having the prediction subparsers advance in lockstep through $w_{r}$. Prediction terminates after consuming prefix $u\preceq w_{r}$ when all subparsers but one die off or when prediction identifies a conflict. Operating in lockstep also provides an opportunity for subparsers to share call stacks thus avoiding redundant computations.

Two subparsers at ATN state $p$ that share the same ATN stack top, $q\gamma_{1}$ and $q\gamma_{2}$, will mirror each other's behavior until simulation pops $q$ from their stacks. Prediction can treat those subparsers as a single subparser by merging stacks. We merge stacks for all configurations in a DFA state of the form $(p,i,\gamma_{1})$ and $(p,i,\gamma_{2})$, forming a general configuration $(p,i,\Gamma)$ with _graph-structured stack_ (GSS) [25]$\Gamma=\gamma_{1}\uplus\gamma_{2}$ where $\uplus$ means graph merge. $\Gamma$ can be the empty stack $[]$, a special stack $\#$ used for _SLL_ prediction (addressed shortly), an individual stack, or a graph of stack nodes. Merging individual stacks into a GSS reduces the potential size from exponential to linear complexity (Theorem 6.4). To represent a GSS, we use an immutable graph data structure with maximal sharing of nodes. Here are two examples that share the parser stack $\gamma_{0}$ at the bottom of the stack:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090606074.png)
In the functions that follow, all additions to configuration sets, such as with operator $+$=, implicitly merge stacks.

There is a special case related to the stack condition at the start of prediction. $\Gamma$ must distinguish between an empty stack and no stack information. For _LL_ prediction, the initial ATN simulation stack is the current parser call stack $\gamma_{0}$. The initial stack is only empty, $\gamma_{0}=[]$, when the decision entry rule is the start symbol. Stack-insensitive _SLL_ prediction, on the other hand, ignores the parser call stack and uses an initial stack of $\#$, indicating no stack information. This distinction is important when computing the _closure_ (Function 7) of configurations representing submachine stop states. Without parser stack information, a subparser that returns from decision entry rule $A$ must consider all possible invocation sites; i.e., _closure_ sees configuration $(p^{\prime}_{A},-,\#)$.

The empty stack $[]$ is treated like any other node for _LL_ prediction: $\Gamma\uplus[]$ yields the graph equivalent of set $\{\Gamma,[]\}$, meaning that both $\Gamma$ and the empty stack are possible. Pushing state $p$ onto $[]$ yields $p[]$ not $p$ because popping $p$ must leave the $[]$ empty stack symbol. For _SLL_ prediction, $\Gamma\uplus\#=\#$ for any graph $\Gamma$ because $\#$ acts like a wildcard and represents the set of all stacks. The wildcard therefore contains any $\Gamma$. Pushing state $p$ onto $\#$ yields $p\#$.

### 5.2 _All(*)_ parsing functions

We can now present the key _ALL(*)_ functions, which we have highlighted in boxes and interspersed within the text of this section. Our discussion follows a top-down order and assumes that the ATN corresponding to grammar $G$, the semantic state $\mathbb{S}$, the DFA under construction, and _input_ are in scope for all functions of the algorithm and that semantic predicates and actions can directly access $\mathbb{S}$.

**Function parse**. The main entry point is function _parse_ (shown in Function 1), which initiates parsing at the start symbol, argument $S$. The function begins by initializing a simulation call stack $\gamma$ to an empty stack and setting ATN state "cursor" $p$ to $p_{S,i}$, the ATN state on the left edge of $S$'s production number $i$ predicted by _adaptivePredict_. The function loops until the cursor reaches $p^{\prime}_{S}$, the submachine stop state for $S$. If the cursor reaches another submachine stop state, $p^{\prime}_{B}$_parse_ simulates a "return" by popping the return state $q$ off the call stack and moving $p$ to $q$.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090607350.png)

For $p$ not at a stop state, _parse_ processes ATN transition $p\xrightarrow{t}q$. There can be only one transition from $p$ because of the way ATNs are constructed. If $t$ is a terminal edge and matches the current input symbol, _parse_ transitions the edge and moves to the next symbol. If $t$ is a nonterminal edge referencing some $B$, _parse_ simulates a submachine call by pushing return state $q$ onto the stack and choosing the appropriate production left edge in $B$ by calling _adaptivePredict_ and setting the cursor appropriately. For action edges, _parse_ updates the state according to the mutator $\mu$ and transitions to $q$. For predicate edges, _parse_ transitions only if predicate $\pi$ evaluates to true. During the parse, failed predicates behave like mismatched tokens. Upon an $\epsilon$ edge, _parse_ moves to $q$. Function _parse_ does not explicitly check that parsing stops at end-of-file because applications like development environments need to parse input subphrases.

**Function adaptivePredict**. To predict a production, _parse_ calls _adaptivePredict_ (Function 2), which is a function of the decision nonterminal $A$ and the current parser stack $\gamma_{0}$. Because prediction only evaluates predicates during full _LL_ simulation, _adaptivePredict_ delegates to _LLpredict_ if at least one of the productions is predicated.[^5] For decisions that do not yet have a DFA, _adaptivePredict_ creates DFA $\mathit{df}a_{A}$ with start state $D_{0}$ in preparation for _SLLpredict_ to add DFA paths. $D_{0}$ is the set of ATN configurations reachable without traversing a terminal edge. Function _adaptivePredict_ also constructs the set of final states $F_{DFA}$, which contains one final state $f_{i}$ for each production of A. The set of DFA states, $Q_{DFA}$, is the union of $D_{0}$, $F_{DFA}$, and the error state $D_{error}$. Vocabulary $\Sigma_{DFA}$ is the set of grammar terminals $T$. For unpredicated decisions with existing DFA, adaptivePredict calls _SLLpredict_ to obtain a prediction from the DFA, possibly extending the DFA through ATN simulation in the process. Finally, since _adaptivePredict_ is looking ahead not parsing, it must undo any changes made to the input cursor, which it does by capturing the input index as $start$ upon entry and rewinding to $start$ before returning.

[^5]: _SLL_ prediction does not incorporate predicates for clarity in this exposition, but in practice, ANTLR incorporates predicates into DFA accept states (Section B.2). ANTLR 3 DFA used predicated edges not predicated accept states.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090608973.png)

**Function startState**. To create DFA start state $D_{0}$, _startState_ (Function 3) adds configurations $(p_{A,i},i,\gamma)$ for each $A\rightarrow\alpha_{i}$ and $A\rightarrow\pi_{i}\alpha_{i}$, if$\pi_{i}$ evaluates to true. When called from _adaptivePredict_, call stack argument $\gamma$ is special symbol $\#$ needed by _SLL_ prediction, indicating "no parser stack information." When called from _LLpredict_, $\gamma$ is initial parser stack $\gamma_{0}$. Computing _closure_ of the configurations completes $D_{0}$.

**Function SLLpredict**. Function _SLLpredict_ (Function 4) performs both DFA and _SLL_ ATN simulation, incrementally adding paths to the DFA. In the best case, there is already a DFA path from $D_{0}$ to an accept state, $f_{i}$, for prefix $u\preceq w_{r}$ and some production number $i$. In the worst-case, ATN simulation is required for all $a$ in sequence $u$. The main loop in _SLLpredict_ finds an existing edge emanating from DFA state cursor $D$ upon $a$ or computes a new one via _target_. It is possible that _target_ will compute a target state that already exists in the DFA, $\underline{D}^{\prime}$, in which case function _target_ returns $\underline{D}^{\prime}$ because $\underline{D}^{\prime}$ may already have outgoing edges computed; it is inefficient to discard work by replacing $\underline{D}^{\prime}$. At the next iteration, _SLLpredict_ will consider edges from $\underline{D}^{\prime}$, effectively switching back to DFA simulation.

Once _SLLpredict_ acquires a target state, $D^{\prime}$, it checks for errors, stack sensitivity, and completion. If _target_ marked $D^{\prime}$ as stack-sensitive, prediction requires full _LL_ simulation and _SLLpredict_ calls _LLpredict_. If $D^{\prime}$ is accept state $f_{i}$, as determined by _target_, _SLLpredict_ returns $i$. In this case, all the configurations in $D^{\prime}$ predicted the same production $i$; further analysis is unnecessary and the algorithm can stop. For any other $D^{\prime}$, the algorithm sets $D$ to $D^{\prime}$, gets the next symbol, and repeats.

**Function target.** Using a combined _move-closure_ operation, _target_ discovers the set of ATN configurations reachable from $D$ upon a single terminal symbol $a\in T$. Function _move_ computes the configurations reachable directly upon $a$ by traversing a terminal edge:

Those configurations and their _closure_ form $D^{\prime}$. If $D^{\prime}$ is empty, no alternative is viable because none can match $a$ from the current state so _target_ returns error state $D_{error}$. If all configurations in $D^{\prime}$ predict the same production number $i$, _target_ adds edge $D\xrightarrow{a}f_{i}$ and returns accept state $f_{i}$. If $D^{\prime}$ has conflicting configurations, _target_ marks $D^{\prime}$ as stack-sensitive. The conflict could be an ambiguity or a weakness stemming from _SLL_'s lack of parser stack information. (Conflicts along with _getConflictSetsPerLoc_ and _getProdSetsPerState_ are described in Section 5.3). The function finishes by adding state $D^{\prime}$, if an equivalent state, $\underline{D}^{\prime}$, is not already in the DFA, and adding edge $D\xrightarrow{a}D^{\prime}$.

**Function LLpredict.** Upon _SLL_ simulation conflict, _SLLpredict_ rewinds the input and calls _LLpredict_ (Function 6) to get a prediction based upon _LL_ ATN simulation, which considers the full parser stack $\gamma_{0}$. Function _LLpredict_ is similar to _SLLpredict_. It uses DFA state $D$ as a cursor and state $D^{\prime}$ as the transition target for consistency but does not update $A$'s DFA so _SLL_ prediction can continue to use the DFA. _LL_ prediction continues until either $D^{\prime}=\varnothing$, $D^{\prime}$ uniquely predicts an alternative, or $D^{\prime}$ has a conflict. If $D^{\prime}$ from _LL_ simulation has a conflict as _SLL_ did, the algorithm reports the ambiguous phrase (input from $start$ to the current index) and resolves to the minimum production number among the conflicting configurations.
(Section 5.3 explains ambiguity detection.) Otherwise, cursor $D$ moves to $D^{\prime}$ and considers the next input symbol.

**Function closure.** The _closure_ operation (Function 7) chases through all $\epsilon$ edges reachable from $p$, the ATN state projected from configuration parameter $c$ and also simulates the call and return of submachines. Function _closure_ treats $\mu$ and $\pi$ edges as $\epsilon$ edges because mutators should not be executed during prediction and predicates are only evaluated during start state computation. From parameter $c=(p,i,\Gamma)$ and edge $p\xrightarrow{\epsilon}q$, _closure_ adds $(q,i,\Gamma)$ to local working set $C$. For submachine call edge $p\xrightarrow{B}q$, _closure_ adds the _closure_ of $(p_{B},i,q\Gamma)$. Returning from a submachine stop state $p^{\prime}_{B}$ adds the _closure_ of configuration $(q,i,\Gamma)$ in which case $c$ would have been of the form $(p^{\prime}_{B},i,q\Gamma)$. In general, a configuration stack $\Gamma$ is a graph representing multiple individual stacks. Function _closure_ must simulate a return from each of the $\Gamma$ stack tops. The algorithm uses notation $q\Gamma^{\prime}\in\Gamma$ to represent all stack tops $q$ of $\Gamma$. To avoid non-termination due to _SLL_ right recursion and $\epsilon$ edges in subrules such as ()+, _closure_ uses a $busy$ set shared among all closure operations used to compute the same $D^{\prime}$.

When _closure_ reaches stop state $p^{\prime}_{A}$ for decision entry rule, $A$, _LL_ and _SLL_ predictions behave differently. _LL_ prediction pops from the parser call stack $\gamma_{0}$ and "returns" to the state that invoked $A$'s submachine. _SLL_ prediction, on the other hand, has no access to the parser call stack and must consider all possible $A$ invocation sites. Function _closure_ finds $\Gamma=\#$ (and $p^{\prime}_{B}=p^{\prime}_{A}$) in this situation because _startState_ will have set the initial stack as $\#$ not $\gamma_{0}$. The return behavior at the decision entry rule is what differentiates _SLL_ from _LL_ parsing.

### 5.3 Conflict and ambiguity detection

The notion of conflicting configurations is central to _ALL(*)_ analysis. Conflicts trigger failover to full _LL_ prediction during _SLL_ prediction and signal an ambiguity during _LL_ prediction. A sufficient condition for a conflict between configurations is when they differ only in the predicted alternative: $(p,i,\Gamma)$ and $(p,j,\Gamma)$. Detecting conflicts is aided by two functions. The first, _getConflictSetsPerLoc_ (Function 8), collects the sets of production numbers associated with all $(p,-,\Gamma)$ configurations. If a $p$,$\Gamma$ pair predicts more than a single production, a conflict exists. Here is a sample configuration set and the associated set of conflict sets:

These conflicts sets indicate that location $p,\Gamma$ is reachable from productions $\{1,$ 2, 3$\}$, location $p,\Gamma^{\prime}$ is reachable from productions $\{1,$ 2$\}$, and $r,\Gamma^{\prime\prime}$ is reachable from production $\{2\}$.

The second function, _getProdSetsPerState_ (Function 9), is similar but collects the production numbers associated with just ATN state $p$. For the same configuration set, _getProdSetsPerState_ computes these conflict sets:

A sufficient condition for failing over to _LL_ prediction (_LL-predict_) from _SLL_ would be when there is at least one set of conflicting configurations: _getConflictSetsPerLoc_ returns at least one set with more than a single production number. E.g., configurations $(p,i,\Gamma)$ and $(p,j,\Gamma)$ exist in parameter $D$. However, our goal is to continue using _SLL_ prediction as long as possible because _SLL_ prediction updates the lookahead DFA cache. To that end, _SLL_ prediction continues if there is at least one nonconflicting configuration (when _getProdSetsPerState_ returns at least one set of size 1). The hope is that more lookahead will lead to a configuration set that predicts a unique production via that nonconflicting configuration. For example, the decision for $S\to a|a|a$$\bullet_{\textbf{\tiny{T}}}b$ is ambiguous upon $a$ between productions 1 and 2 but is unambiguous upon $ab$. (Location $\bullet_{p}$ is the ATN state between $a$ and $b$.) After matching input $a$, the configuration set would be $\{(p_{S}^{\prime},1,[])$, $(p_{S}^{\prime},2,[])$, $(p,3,[])\}$. Function _getConflictSetsPerLoc_ returns $\{\{1,2\}$, $\{3\}\}$. The next _move-closure_ upon $b$ leads to nonconflicting configuration set $\{(p_{S}^{\prime},3,[])\}$ from $(p,3,[])$, bypassing the conflict. If all sets returned from _getConflictSetsPerLoc_ predict more than one alternative, no amount of lookahead will lead to a unique prediction. Analysis must try again with call stack $\gamma_{0}$ via _LLpredict_

Conflicts during _LL_ simulation are ambiguities and occur when each conflict set from _getConflictSetsPerLoc_ contains more than 1 production--every location in $D$ is reachable from more than a 1 production. Once multiple subparsers reach the same $(p,-,\Gamma)$, all future simulation derived from $(p,-,\Gamma)$ will behave identically. More lookahead will not resolve the ambiguity. Prediction could terminate at this point and report lookahead prefix $u$ as ambiguous but _LLpredict_ continues until it is sure for _which_ productions $u$ is ambiguous. Consider conflict sets $\{1,2,3\}$ and $\{2,3\}$. Because both have degree greater than one, the sets represent an ambiguity, but additional input will identify whether $u\preceq w_{r}$ is ambiguous upon $\{1,2,3\}$ or $\{2,3\}$. Function _LLpredict_ continues until all conflict sets that identify ambiguities are equal; condition $x=y$ and $|x|>1$$\forall\,x,y\in allsts$ embodies this test.
To detect conflicts, the algorithm compares graph-structured stacks frequently. Technically, a conflict occurs when configurations $(p,i,\Gamma)$ and $(p,j,\Gamma^{\prime})$ occur in the same configuration set with $i\neq j$ and at least one stack trace $\gamma$ in common to both $\Gamma$ and $\Gamma^{\prime}$. Because checking for graph intersection is expensive, the algorithm uses equality, $\Gamma=\Gamma^{\prime}$, as a heuristic. Equality is much faster because of the shared subgraphs. The graph equality algorithm can often check node identity to compare two entire subgraphs. In the worst case, the equality versus subset heuristic delays conflict detection until the GSS between conflicting configurations are simple linear stacks where graph intersection is the same as graph equality. The cost of this heuristic is deeper lookahead.

### 5.4 Sample DFA construction

To illustrate algorithm behavior, consider inputs $bc$ and $bd$ for the grammar and ATN in Figure 8. ATN simulation for decision $S$ launches subparsers at left edge nodes $p_{S,1}$ and $p_{S,2}$ with initial $D_{0}$ configurations $(p_{S,1},1,[])$ and $(p_{S,2},2,[])$. Function _closure_ adds three more configurations to $D_{0}$ as it "calls" $A$ with "return" nodes $p_{1}$ and $p_{3}$. Here is the DFA resulting from ATN simulation upon $bc$ and then $bd$ (configurations added by _move_ are bold):

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090614275.png)

After $bc$ prediction, the DFA has states $D_{0}$, $D^{\prime}$, and $f_{1}$. From DFA state $D^{\prime}$, _closure_ reaches the end of $A$ and pops fromthe $\Gamma$ stacks, returning to ATN states in $S$. State $f_{1}$ uniquely predicts production number 1. State $f_{2}$ is created and connected to the DFA (shown with dashed arrow) during prediction of the second phrase, $bd$. Function _adaptivePredict_ first uses DFA simulation to get to $D^{\prime}$ from $D_{0}$ upon $b$. Before having seen $bd$, $D^{\prime}$ has no $d$ edge so _adaptivePredict_ must use ATN simulation to add edge $D^{\prime}\xrightarrow{d}f_{2}$.

### 6 Theoretical results

This section identifies the key _ALLI(*)_ theorems and shows parser time complexity. See Appendix A for detailed proofs.

**Theorem 6.1**.: _(Correctness). The ALL(*) parser for non-left-recursive $G$ recognizes sentence $w$ iff $w\in L(G)$._
**Theorem 6.2**.: ALL(_) _languages are closed under union._
**Theorem 6.3**.: ALL(_) _parsing of $n$ symbols has $O(n^{4})$ time._
**Theorem 6.4**.: _A GSS has $O(n)$ nodes for $n$ input symbols._
**Theorem 6.5**.: _Two-stage parsing for non-left-recursive $G$ recognizes sentence $w$ iff $w\in L(G)$._

## 7 Empirical results

We performed experiments to compare the performance of _ALL(*)_ Java parsers with other strategies, to examine _ALL(*)_ throughput for a variety of other languages, to highlight the effect of the lookahead DFA cache on parsing speed, and to provide evidence of linear _ALL(*)_ performance in practice.

### 7.1 Comparing _ALL(*)_'s speed to other parsers

Our first experiment compared Java parsing speed across 10 tools and 8 parsing strategies: hand-tuned recursive-descent with precedence parsing, _LL(k)_, _LL(*)_, PEG, $LALR(1)$, _ALL(*)_ GLR, and GLL. Figure 9 shows the time for each tool to parse the 12,920 source files of the Java 6 library and compiler. We chose Java because it was the most commonly available grammar among tools and sample Java source is plentiful. The Java grammars used in this experiment came directly from the associated tool except for DParser and Elkhound, which did not offer suitable Java grammars. We ported ANTLR's Java grammar to the meta-syntax of those tools using unambiguous arithmetic expressions rules. We also embedded merge actions in the Elkhound grammar to disambiguate during the parse to mimic ANTLR's ambiguity resolution. All input files were loaded into RAM before parsing and times reflect the average time measured over 10 complete corpus passes, skipping the first two to ensure JIT compiler warm-up. For _ALL(*)_, we used the two-stage parse from Section 3.2. The test machine was a 6 core 3.33Ghz 16G RAM Mac OS X 10.7 desktop running the Java 7 virtual machine. Elkhound and DParser parsers were implemented in C/C++, which does not have a garbage collector running concurrently. Elkhound was last updated in 2005 and no longer builds on Linux or OS X, but we were able to build it on Windows 7 (4 core 2.67Ghz 24G RAM). Elkhound also can only read from a file so Elkhound parse times are not comparable. In an effort to control for machine speed differences and RAM vs SSD, we computed the time ratio of our Java test rig on our OS X machine reading from RAM to the test rig running on Windows pulling from SSD. Our reported Elkhound times are the Windows time multiplied by that OS X to Windows ratio.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090635672.png)

For this experiment, _ALLI(*)_ outperforms the other parser generators and is only about 20% slower than the handbuilt parser in the Java compiler itself. When comparing runs with tree construction (marked with $\dagger$ in Figure 9), ANTLR 4 is about 4.4x faster than Elkhound, the fastest GLR tool we tested, and 135x faster than GLL (Rascal). ANTLR 4's nondeterministic _ALL(*)_ parser was slightly faster than JavaCC's deterministic _LL(k)_ parser and about 2x faster than _Rats!_'s PEG. In a separate test, we found that _ALL(*)_ outperforms _Rats!_' on its own PEG grammar converted to ANTLR syntax (8.77s vs 12.16s). The $LALR(1)$ parser did not perform well against the _LL_ tools but that could be SableCC's implementation rather than a deficiency of $LALR(1)$. (The Java grammar from JavaCUP, another $LALR(1)$ tool, was incomplete and unable to parse the corpus.) When reparsing the corpus, _ALL(*)_ lookahead gets cache hits at each decision and parsing is 30% faster at 3.73s. When reparsing with tree construction (time not shown), _ALL(*)_ outperforms handbuilt Javac (4.4s vs 4.73s). Reparsing speed matters to tools such as development environments.

The GLR parsers we tested are up to two orders of magnitude slower at Java parsing than _ALL(*)_. Of the GLR tools, Elkhound has the best performance primarily because it relies on a linear $LR(1)$ stack instead of a GSS whenever possible. Further, we allowed Elkhound to disambiguate during the parse like _ALL(*)_. Elkhound uses a separate lexer, unlike JSGLR and DParser, which are scannerless. A possible explanation for the observed performance difference with _ALL(*)_ is that the Java grammar we ported to Elkhound and DParser is biased towards _ALL(*)_, but this objection is not well-founded. GLR should also benefit from highly-deterministic and unambiguous grammars. GLL has the slowest speed in this test perhaps because Rascal's team ported SDF's GLR Java grammar, which is not optimized for GLL (Grammar variations can affect performance.) Rascal is also scannerless and is currently the only available GLL tool.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090617145.png)
The biggest issue with general algorithms is that they are highly unpredictable in time and space, which can make them unsuitable for some commercial applications. Figure 10 summarizes the performance of the same tools against a single 3.2M Java file. Elkhound took 7.65s to parse the 123M Java corpus, but took 3.35 minutes to parse the 3.2M Java file. It crashed (out of memory) with parse forest construction on. DParser's time jumped from a corpus time of 98s to 10.5 hours on the 3.2M file. The speed of Rascal and JSGLR scale reasonably well to the 3.2M file, but use 2.6G and 1G RAM, respectively. In contrast, _ALL(*)_ parses the 3.2M file in 360ms with tree construction using 8M. ANTLR 3 is fast but is slower and uses more memory (due to backtracking memoization) than ANTLR 4.

### 7.2 _All(*)_ performance across languages

Figure 11 gives the bytes-per-second throughput of _ALL(*)_ parsers for 8 languages, including Java for comparison. The number of test files and file sizes vary greatly (according to the input we could reasonably collect); smaller files yield higher parse-time variance.

* **C** Derived from C11 specification; has no indirect left-recursion, altered stack-sensitive rule to render _SLL_ (see text below): 813 preprocessed files, 159.8M source from postgres database.
* **Verilog2001** Derived from Verilog 2001 spec, removed indirect left-recursion: 385 files, 659k from [3] and web.
* **JSON** Derived from spec. 4 files, 331k from twitter.
* **DOT** Derived from spec. 48 files 19.5M collected from web.
* **Lua**: Derived from Lua 5.2 spec. 751 files, 123k from github.
* **XML** Derived from spec. 1 file, 117M from XML benchmark.
* **Erlang** Derived from _LALR(1)_ grammar. 500 preproc'd files, 8M.
*

![](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090616559.png)
Some of these grammars yield reasonable but much slower parse times compared to Java and XML but demonstrate that programmers can convert a language specification to ANTLR's meta-syntax and get a working grammar without major modifications. (In our experience, grammar specifications are rarely tuned to a particular tool or parsing strategy and are often ambiguous.) Later, programmers can use ANTLR's profiling and diagnostics to improve performance, as with any programming task. For example, the C11 specification grammar is _LL_ not _SLL_ because of rule declarationSpecifiers, which we altered to be _SLL_ in our C grammar (getting a 7x speed boost).

### 7.3 Effect of lookahead DFA on performance

The lookahead DFA cache is critical to _ALL(*)_ performance. To demonstrate the cache's effect on parsing speed, we disabled the DFA and repeated our Java experiments. Consider the 3.73s parse time from Figure 9 to reparse the Java corpus with pure cache hits. With the lookahead DFA cache disabled completely, the parser took 12 minutes (717.6s). Figure 10 shows that disabling the cache increases parse time from 203ms to 42.5s on the 3.2M file. This performance is in line with the high cost of GLL and GLR parsers that also do not reduce parser speculation by memoizing parsing decisions. As an intermediate value, clearing the DFA cache before parsing each corpus file yields a total time of 34s instead of 12 minutes. This isolates cache use to a single file and demonstrates that cache warm-up occurs quickly even within a single file.

DFA size increases linearly as the parser encounters new lookahead phrases. Figure 12 shows the growth in the number of DFA states as the (slowest four) parsers from Figure 11 encounter new files. Languages like C that have constructs with common left-prefixs require deep lookahead in _LL_ parsers to distinguish phrases; e.g., struct {...} x; and struct {...} f(); share a large left-prefix. In contrast, the Verilog2001 parser uses very few DFA states (but runs slower due to a non-_SLL_ rule). Similarly, after seeing the entire 123M Java corpus, the Java parser uses just 31,626 DFA states, adding an average of ~2.5 states per file parsed. DFA size does, however, continue to grow as the parser encounters unfamiliar input. Programmers can clear the cache and _ALL(*)_ will adapt to subsequent input.

### 7.4 Empirical parse-time complexity

Given the wide range of throughput in Figure 11, one could suspect nonlinear behavior for the slower parsers. To investigate, we plotted parse time versus file size in Figure 13 and drew least-squares regression and LOWESS [6] data fitting curves. LOWESS curves are parametrically unconstrained (not required to be a line or any particular polynomial) and they virtually mirrors each regression line, providing strong evidence that the relationship between parse time and input size is linear. The same methodology shows that the parser generated from the non-_SLL_ grammar (not shown) taken from the C11 specification is also linear, despite being much slower than our _SLL_ version.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090618647.png)

We have yet to see nonlinear behavior in practice but the theoretical worst-case behavior of _ALL(*)_ parsing is $O(n^{4})$. Experimental parse-time data for the following contrived worst-case grammar exhibits quartic behavior for input $a$, $aa$, $aaa$,..., $a^{n}$ (with $n\leq 120$ symbols we could test in a reasonable amount of time). $S\to A\,\$$, $A\to aAA\,|\,aA\,|\,a$. The generated parser requires a prediction upon each input symbol and each prediction must examine all remaining input. The _closure_ operation performed for each input symbol must examine the entire depth of the GSS, which could be size $n$. Finally, merging two GSS can take $O(n)$ in our implementation, yielding $O(n^{4})$ complexity.

From these experiments, we conclude that shifting grammar analysis to parse-time to get _ALL(*)_ strength is not only practical but yields extremely efficient parsers, competing with the hand-tuned recursive-descent parser of the Java compiler. Memoizing analysis results with DFA is critical to such performance. Despite $O(n^{4})$ theoretical complexity, _ALL(*)_ appears to be linear in practice and does not exhibit the unpredictable performance or large memory footprint of the general algorithms.

## 8. Related work

For decades, researchers have worked towards increasing the recognition strength of efficient but non-general _LL_ and _LR_ parsers and increasing the efficiency of general algorithms such as Earley's $O(n^{3})$ algorithm [8]. Parr [21] and Charles [4] statically generated _LL(k)_ and _LR(k)_ parsers for $k>1$. Parr and Fisher's _LL(*)_[20] and Bermudez and Schimpf's $LAR(m)$[2] statically computed _LL_ and _LR_ parsers augmented with cyclic DFA that could examine arbitrary amounts of lookahead. These parsers were based upon _LL_-regular [12] and _LR_-regular [7] parsers, which have the undesirable property of being undecidable in general. Introducing backtracking dramatically increases recognition strength and avoids static grammar analysis undecidability issues but is undesirable because it has embedded mutators issues, reduces performance, and complicates single-step debugging. Packrat parsers (PEGs) [9] try decision productions in order and pick the first that succeeds. PEGs are $O(n)$ because they memoize partial parsing results but suffer from the $a\,|\,ab$ quirk where $ab$ is silently unmatchable.

To improve general parsing performance, Tomita [26] introduced GLR, a general algorithm based upon _LR(k)_ that conceptually forks subparsers at each conflicting _LR(k)_ state at parse-time to explore all possible paths. Tomita shows GLR to be 5x-10x faster than Earley. A key component of GLR parsing is the graph-structured stack (GSS) [26] that prevents parsing the same input twice in the same way. (GLR pushes input symbols and _LR_ states on the GSS whereas _ALL(*)_ pushes ATN states.) Elkhound [18] introduced hybrid GLR parsers that use a single stack for all $LR(1)$ decisions and a GSS when necessary to match ambiguous portions of the input. (We found Elkhound's parsers to be faster than those of other GLR tools.) GLL [25] is the _LL_ analog of GLR and also uses subparsers and a GSS to explore all possible paths; GLL uses $k=1$ lookahead where possible for efficiency. GLL is $O(n^{3})$ and GLR is $O(n^{p+1})$ where $p$ is the length of the longest grammar production.

Earley parsers scale gracefully from $O(n)$ for deterministic grammars to $O(n^{3})$ in the worst case for ambiguous grammars but performance is not good enough for general use. _LR(k)_ state machines can improve the performance of such parsers by statically computing as much as possible. LRE [16] is one such example. Despite these optimizations, general algorithms are still very slow compared to deterministic parsers augmented with deep lookahead.

The problem with arbitrary lookahead is that it is impossible to compute statically for many useful grammars (the _LL_-regular condition is undecidable.) By shifting lookahead analysis to parse-time, _ALL(*)_ gains the power to handle any grammar without left recursion because it can launch subparsers to determine which path leads to a valid parse. Unlike GLR, speculation stops when all remaining subparsers are associated with a single alternative production, thus, computing the minimum lookahead sequence. To get performance, _ALL(*)_ records a mapping from that lookahead sequence to the predicted production using a DFA for use by subsequent decisions. The context-free language subsets encountered during a parse are finite and, therefore, _ALL(*)_ lookahead languages are regular. Ancona _et al_[1] also performed parse-time analysis, but they only computed fixed _LR(k)_ lookahead and did not adapt to the actual input as _ALL(*)_ does. Perlin [23] operated on an RTN like _ALL(*)_ and computed $k=1$ lookahead during the parse.

_ALL(*)_ is similar to Earley in that both are top-down and operate on a representation of the grammar at parse-time, but Earley is parsing not computing lookahead DFA. In that sense, Earley is not performing grammar analysis. Earley also does not manage an explicit GSS during the parse. Instead, items in Earley states have "parent pointers" that refer to other states that, when threaded together, form a GSS. Earley's _SCANNER_ operation correspond to _ALL(*)_'s _move_ function. The _PREDIC

Figure 13: **Linear parse time vs file size**. Linear regression (dashed line) and LOWESS unconstrained curve coincide, giving strong evidence of linearity. Curves computed on (bottom) 99% of file sizes but zooming in to show detail in the bottom 40% of parse times.

_TOR_ and _COMPLETER_ operations correspond to push and pop operations in _ALL(_)'s _closure_ function. An Earley state is the set of all parser configurations reachable at some absolute input depth whereas an _ALL(_)_ DFA state is a set of configurations reachable from a lookahead depth relative to the current decision. Unlike the completely general algorithms, _ALL(*)_ seeks a single parse of the input, which allows the use of an efficient _LL_ stack during the parse.

Parsing strategies that continuously speculate or support ambiguity have difficulty with mutators because they are hard to undo. A lack of mutators reduces the generality of semantic predicates that alter the parse as they cannot test arbitrary state computed previously during the parse. _Rats!_[10] supports restricted semantic predicates and Yakker [13] supports semantic predicates that are functions of previously-parsed terminals. Because _ALL(*)_ does not speculate during the actual parse, it supports arbitrary mutators and semantic predicates. Space considerations preclude a more detailed discussion of related work here; a more detailed analysis can be found in reference [20].

## 9. Conclusion

ANTLR 4 generates an _ALL(*)_ parser for any CFG without indirect or hidden left-recursion. _ALL(*)_ combines the simplicity, efficiency, and predictability of conventional top-down _LL(k)_ parsers with the power of a GLR-like mechanism to make parsing decisions. The critical innovation is to shift grammar analysis to parse-time, caching analysis results in lookahead DFA for efficiency. Experiments show _ALL(*)_ outperforms general (Java) parsers by orders of magnitude, exhibiting linear time and space behavior for 8 languages. The speed of the _ALL(*)_ Java parser is within 20% of the Java compiler's hand-tuned recursive-descent parser. In theory, _ALL(*)_ is $O(n^{4})$, inline with the low polynomial bound of GLR. ANTLR is widely used in practice, indicating that _ALL(*)_ provides practical parsing power without sacrificing the flexibility and simplicity of recursive-descent parsers desired by programmers.

## 10. Acknowledgments

We thank Elizabeth Scott, Adrian Johnstone, and Mark Johnson for discussions on parsing algorithm complexity. Eelco Visser and Jurgen Vinju provided code to test isolated parsers from JS-GLR and Rascal. Etienne Gagnon generated SableCC parsers.

## References

* [1]Ancona, M., Dodero, G., Gianuzzi, V., and Morgavi, M. Efficient construction of $LR(k)$ states and tables. _ACM Trans. Program. Lang. Syst. 13_, 1 (Jan. 1991), 150-178.
* [2]Bermudez, M. E., and Schimpf, K. M. Practical arbitrary lookahead $LR$ parsing. _Journal of Computer and System Sciences 41_, 2 (1990).
* [3]Brown, S., and Vranesic, Z. _Fundamentals of Digital Logic with Verilog Design_. McGraw-Hill series in ECE. 2003.
* [4]Charles, P. _A Practical Method for Constructing Efficient LALR(k) Parsers with Automatic Error Recovery_. PhD thesis, New York University, New York, NY, USA, 1991.
* [5]Clarke, K. The top-down parsing of expressions. Unpublished technical report, Dept. of Computer Science and Statistics, Queen Mary College, London, June 1986.
* [6]Cleveland, W. S. Robust Locally Weighted Regression and Smoothing Scatterplots. _Journal of the American Statistical Association 74_ (1979), 829-836.
* [7]Cohen, R., and Culik, K. $LR$-Regular grammars--an extension of $LR(k)$ grammars. In _SWAT '71_ (Washington, DC, USA, 1971), IEEE Computer Society, pp. 153-165.
* [8]Earley, J. An efficient context-free parsing algorithm. _Communications of the ACM 13_, 2 (1970), 94-102.
* [9]Ford, B. Parsing Expression Grammars: A recognition-based syntactic foundation. In _POPL_ (2004), ACM Press, pp. 111-122.
* [10]Grimm, R. Better extensibility through modular syntax. In _PLDI_ (2006), ACM Press, pp. 38-51.
* [11]Hopcroft, J., and Ullman, J. _Introduction to Automata Theory, Languages, and Computation_. Addison-Wesley, Reading, Massachusetts, 1979.

* [13]Jim, T., Mandelbaum, Y., and Walker, D. Semantics and algorithms for data-dependent grammars. In _POPL_ (2010).
* [14]Johnson, M. The computational complexity of GLR parsing. In _Generalized LR Parsing_, M. Tomita, Ed. Kluwer, Boston, 1991, pp. 35-42.
* [15]Kipps, J. _Generalized LR Parsing_. Springer, 1991, pp. 43-59.
* [16]Mclean, P., and Horspool, R. N. A faster Earley parser. In _CC_ (1996), Springer, pp. 281-293.
* [17]McPeak, S. Elkhound: A fast, practical GLR parser generator. Tech. rep., University of California, Berkeley (EECS), Dec. 2002.
* [18]McPeak, S., and Necula, G. C. Elkhound: A fast, practical GLR parser generator. In _CC_ (2004), pp. 73-88.
* [19]Parr, T. _The Definitive ANTLR Reference: Building Domain-Specific Languages_. The Pragmatic Programmers, 2013. ISBN 978-1-93435-699-9.
* [20]Parr, T., and Fisher, K. $LL(*)$: The Foundation of the ANTLR Parser Generator. In _PLDI_ (2011), pp. 425-436.
* [21]Parr, T. J. _Obtaining practical variants of $LL(k)$ and $LR(k)$ for $k>1$ by splitting the atomic $k$-tuple_. PhD thesis, Purdue University, West Lafayette, IN, USA, 1993.
* [22]Parr, T. J., and Quong, R. W. Adding Semantic and Syntactic Predicates to $LL(k)$--_pred-$LL(k)$_. In _CC_ (1994).
* [23]Perlin, M. LR recursive transition networks for Earley and Tomita parsing. In _Proceedings of the 29th Annual Meeting on Association for Computational Linguistics_ (1991), ACL '91, pp. 98-105.
* [24]Plevyak, J. DParser: GLR parser generator, Visited Oct. 2013.
* [25]Scott, E., and Johnstone, A. GLL parsing. _Electron. Notes Theor. Comput. Sci. 253_, 7 (Sept. 2010), 177-189.
* [26]Tomita, M. _Efficient Parsing for Natural Language_. Kluwer Academic Publishers, 1986.
* [27]Woods, W. A. Transition network grammars for natural language analysis. _Comm. of the ACM 13_, 10 (1970), 591-606.

### A Correctness and complexity analysis

**Theorem A.1**.: ALL(*) _languages are closed under union._

Proof.: Let predicated grammars $G_{1}$ = $(N_{1},T,P_{1},S_{1},\Pi_{1},\mathcal{M}_{1})$ and $G_{2}$ = $(N_{2},T,P_{2},S_{2},\Pi_{2},\mathcal{M}_{2})$ describe $L(G_{1})$ and $L(G_{2})$, respectively. For applicability to both parsers and scannerless parsers, assume that the terminal space $T$ is the set of valid characters. Assume $N_{1}\cap N_{2}=\varnothing$ by renaming nonterminals if necessary. Assume that the predicates and mutators of $G_{1}$ and $G_{2}$ operate in disjoint environments, $\mathbb{S}_{1}$ and $\mathbb{S}_{2}$. Construct:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090633020.png)
**Lemma A.1**.: _The ALL(*) parser for non-left-recursive $G$ with lookahead DFA deactivated recognizes sentence $w$ iff $w\in L(G)$._

Proof.: The ATN for $G$ recognizes $w$_iff_$w\in L(G)$. Therefore, we can equivalently prove that _ALL(*)_ is a faithful implementation of an ATN. Without lookahead DFA, prediction is a straightforward ATN simulator: a top-down parser that makes accurate parsing decisions using GLR-like subparsers that can examine the entire remaining input and ATN submachine call stack.

**Theorem A.2**.: _(Correctness). The ALL(*) parser for non-left-recursive $G$ recognizes sentence $w$ iff $w\in L(G)$._

Proof.: Lemma A.1 shows that an _ALL(*)_ parser correctly recognizes $w$ without the DFA cache. The essence of the proof then is to show that _ALL(*)_'s adaptive lookahead DFA do not break the parse by giving different prediction decisions than straightforward ATN simulation. We only need to consider the case of unpredicted _SLL_ parsing as _ALL(*)_ only caches decision results in this case.

_if case:_ By induction on the state of the lookahead DFA for any given decision $A$. _Base case._ The first prediction for $A$ begins with an empty DFA and must activate ATN simulation to choose alternative $\alpha_{i}$ using prefix $u\preceq w_{r}$. As ATN simulation yields proper predictions, the _ALL_(*) parser correctly predicts $\alpha_{i}$ from a cold start and then records the mapping from $u:i$ in the DFA. If there is a single viable alternative, $i$ is the associated production number. If ATN simulation finds multiple viable alternatives, $i$ is the minimum production number associated with alternatives from that set.

_Induction step._ Assume the lookahead DFA correctly predicts productions for every $u$ prefix of $w_{r}$ seen by the parser at $A$. We must show that starting with an existing DFA, _ALL(*)_ properly adds a path through the DFA for unfamiliar $u$ prefix of $w^{\prime}_{r}$. There are several cases:

1. $u\preceq w^{\prime}_{r}$ and $u\preceq w_{r}$ for a previous $w_{r}$. The lookahead DFA gives the correct answer for $u$ by induction assumption. The DFA is not updated.
2. $w^{\prime}_{r}=bx$ and all previous $w_{r}=ay$ for some $a\neq b$. This case reduces to the cold-start base case because there is no $D_{0}\xrightarrow{b}D$ edge. ATN simulation predicts $\alpha_{i}$ and adds path for $u\preceq w^{\prime}_{r}$ from $D_{0}$ to $f_{i}$.
3. $w^{\prime}_{r}=vax$ and $w_{r}=vby$ for some previously seen $w_{r}$ with common prefix $v$ and $a\neq b$. DFA simulation reaches $D$ from $D_{0}$ for input $v$. $D$ has an edge for $b$ but not $a$. ATN simulation predicts $\alpha_{i}$ and augments the DFA, starting with an edge on $a$ from $D$ leading eventually to $f_{i}$.

_only if case_: The _ALL(*)_ parser reports a syntax error for $w\notin L(G)$. Assume the opposite, that the parser successfully parses $w$. That would imply that there exists an ATN configuration derivation sequence $(\mathbb{S},p_{S},[\![,w)\mapsto^{*}(\mathbb{S}^{\prime},p^{\prime}_{S},[\![, \epsilon)$ for $w$ through $G$'s corresponding ATN. But that would require $w\in L(G)$, by Definition 1. Therefore the _ALL(*)_ parser reports a syntax error for $w$. The accuracy of the _ALL(*)_ lookahead cache is irrelevant because there is no possible path through the ATN or parser.

**Lemma A.2**.: _The set of viable productions for LL is always a subset of SLL's viable productions for a given decision $A$ and remaining input string $w_{r}$._

Proof.: If the key _move-closure_ analysis operation does not reach stop state $p^{\prime}_{A}$ for submachine $A$, _SLL_ and _LL_ behave identically and so they share the same set of viable productions.

If _closure_ reaches the stop state for the decision entry rule, $p^{\prime}_{A}$, there are configurations of the form $(p^{\prime}_{A},-,\gamma)$ where, for convenience, the usual GSS $\Gamma$ is split into single stacks, $\gamma$. In _LL_ prediction mode, $\gamma=\gamma_{0}$, which is either a single stack or empty if $A=S$. In _SLL_ mode, $\gamma=\#$, signaling no stack information. Function _closure_ must consider all possible $\gamma_{0}$ parser call stacks. Since any single stack must be contained within the set of all possible call stacks, _LL_ closure operations consider at most the same number of paths through the ATN as _SLL_.

**Lemma A.3**.: _For $w\not\in L(G)$ and non-left-recursive $G$, SLL reports a syntax error._

Proof. As in the _only if case_ of Theorem 6.1, there is no valid ATN configuration derivation for $w$ regardless of how _adaptivePredict_ chooses productions.

**Theorem A.3**.: _Two-stage parsing for non-left-recursive $G$ recognizes sentence $w$ iff $w\in L(G)$._

Proof. By Lemma A.3, _SLL_ and _LL_ behave the same when $w\not\in L(G)$. It remains to show that _SLL_ prediction either behaves like _LL_ for input $w\in L(G)$ or reports a syntax error, signalling a need for the _LL_ second stage. Let $\mathcal{V}$ and $\mathcal{V}^{\prime}$ be the set of viable production numbers for $A$ using _SLL_ and _LL_, respectively. By Lemma A.2, $\mathcal{V}^{\prime}\subseteq\mathcal{V}$. There are two cases to consider:

1. If $min(\mathcal{V})=min(\mathcal{V}^{\prime})$, _SLL_ and _LL_ choose the same production. _SLL_ succeeds for $w$. E.g., $\mathcal{V}=\{1,2,3\}$ and $\mathcal{V}^{\prime}=\{1,3\}$ or $\mathcal{V}=\{1\}$ and $\mathcal{V}^{\prime}=\{1\}$.
2. If $min(\mathcal{V})\neq min(\mathcal{V}^{\prime})$ then $min(\mathcal{V})\not\in\mathcal{V}^{\prime}$ because _LL_ finds $min(\mathcal{V})$ nonviable. _SLL_ would report a syntax error. E.g., $\mathcal{V}=\{1,2,3\}$ and $\mathcal{V}^{\prime}=\{2,3\}$ or $\mathcal{V}=\{1,2\}$ and $\mathcal{V}^{\prime}=\{2\}$.

In all possible combinations of $\mathcal{V}$ and $\mathcal{V}^{\prime}$, _SLL_ behaves like _LL_ or reports a syntax error for $w\in L(G)$.

**Theorem A.4**.: _A GSS has $O(n)$ nodes for $n$ input symbols._

Proof.: For nonterminals $N$ and ATN states $Q$, there are $|N|\times|Q|\ p\xrightarrow{A}q$ ATN transitions if every every grammar position invokes every nonterminal. That limits the number of new GSS nodes to $|Q|^{2}$ for a closure operation (which cannot transition terminal edges). _ALL(*)_ performs $n+1$ closures for $n$ input symbols giving $|Q|^{2}(n+1)$ nodes or $O(n)$ as $Q$ is not a function of the input.

**Lemma A.5**.: _Examining a lookahead symbol has $O(n^{2})$ time._

Proof.: Lookahead is a _move-closure_ operation that computes new target DFA state $D^{\prime}$ as a function of the ATN configurations in $D$. There are $|Q|\times m\approx|Q|^{2}$ configurations of the form $(p,i,\_)\in D$ for $|Q|$ ATN states and $m$ alternative productions in the current decision. The cost of _move_ is not a function of input size $n$. Closure of $D$ computes _closure(c)_$\forall\ c\in D$ and _closure(c)_ can walk the entire GSS back to the root (the empty stack). That gives a cost of $|Q|^{2}$ configurations times $|Q|^{2}(n+1)$ GSS nodes (per Theorem A.4) or $O(n)$ add operations to build $D^{\prime}$. Adding a configuration is dominated by the graph merge, which (in our implementation) is proportional to the depth of the graph. The total cost for _move-closure_ is $O(n^{2})$.

**Theorem A.5**.: $\text{ALL(*)}$ _parsing of $n$ input symbols has $O(n^{4})$ time._

Proof. Worst case, the parser must examine all remaining input symbols during prediction for each of $n$ input symbols giving $O(n^{2})$ lookahead operations. The cost of each lookahead operation is $O(n^{2})$ by Lemma A.4 giving overall parsing cost $O(n^{4})$.

### B Pragmatics

This section describes some of the practical considerations associated with implementing the _ALLI(*)_ algorithm.

#### B.1 Reducing warm-up time

Many decisions in a grammar are $LL(1)$ and they are easy to identify statically. Instead of always generating "switch on _adaptivePredict_" decisions in the recursive-descent parsers, ANTLR generates "switch on token type" decisions whenever possible. This $LL(1)$ optimization does not affect the size of the generated parser but reduces the number of lookahead DFA that the parser must compute.

Originally, we anticipated "training" a parser on a large input corpus and then serializing the lookahead DFA to disk to avoid re-computing DFA for subsequent parser runs. As shown in the Section 7, lookahead DFA construction is fast enough that serializing and deserializing the DFA is unnecessary.

#### B.2 Semantic predicate evaluation

For clarity, the algorithm described in this paper uses pure ATN simulation for all decisions that have semantic predicates on production left edges. In practice, ANTLR uses lookahead DFA that track predicates in accept states to handle semantic-context-sensitive prediction. Tracking the predicates in the DFA allows prediction to avoid expensive ATN simulation if predicate evaluation during _SLL_ simulation predicts a unique production. Semantic predicates are not common but are critical to solving some context-sensitive parsing problems; e.g., predicates are used internally by ANTLR to encode operator precedence when rewriting left-recursive rules. So it is worth the extra complexity to evaluate predicates during _SLL_ prediction. Consider the predicated rule from Section 2.1:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090627119.png)
The second production is viable only when!enum_is_keyword evaluates to true. In the abstract, that means the parser would need two lookahead DFA, one per semantic condition. Instead, ANTLR's _ALL(*)_ implementation creates a DFA (via _SLL_ prediction) with edge $D_{0}\xrightarrow{\texttt{enum}}f_{2}$ where $f_{2}$ is an augmented DFA accept state that tests!enum_is_keyword. Function _adaptivePredict_ returns production 2 upon enum if!enum_is_keyword else throws a no-viable-alternative exception.

The algorithm described in this paper also does not support semantic predicates outside of the decision entry rule. In practice, _ALL(*)_ analysis must evaluate all predicates reachable from the decision entry rule without stepping over a terminal edge in the ATN. For example, the simplified _ALL(*)_ algorithm in this paper considers only predicates $\pi_{1}$ and $\pi_{2}$ for the productions of $S$ in the following (ambiguous) grammar.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090626470.png)
Input $ab$ matches either alternative of $S$ and, in practice, ANTLR evaluates "$\pi_{1}$ and ($\pi_{3}$ or $\pi_{4}$)" to test the viability of $S$'s first production not just $\pi_{1}$. After simulating $S$ and $A$'s ATN submachines, the lookahead DFA for $S$ would be $D_{0}\xrightarrow{a}D^{\prime}\xrightarrow{b}f_{1,2}$. Augmented accept state $f_{1,2}$ predicts productions 1 or 2 depending on semantic contexts $\pi_{1}\wedge(\pi_{3}\vee\pi_{4})$ and $\pi_{2}\wedge(\pi_{3}\vee\pi_{4})$, respectively. To keep track of semantic context during _SLL_ simulation, ANTLR ATN configurations contain extra element $\pi$: $(p,i,\Gamma,\pi)$. Element $\pi$ carries along semantic context and ANTLR stores predicate-to-production pairs in the augmented DFA accept states.

#### B.3 Error reporting and recovery

_ALL(*)_ prediction can scan arbitrarily far ahead so erroneous lookahead sequences can be long. By default, ANTLR-generated parsers print the entire sequence. To recover, parsers consume tokens until a token appears that could follow the current rule. ANTLR provides hooks to override reporting and recovery strategies.

ANTLR parsers issue error messages for invalid input phrases and attempt to recover. For mismatched tokens, ANTLR attempts single token insertion and deletion to resynchronize. If the remaining input is not consistent with any production of the current nonterminal, the parser consumes tokens until it finds a token that could reasonably follow the current nonterminal. Then the parser continues parsing as if the current nonterminal had succeeded. ANTLR improves error recovery over ANTLR 3 for EBNF subrules by inserting synchronization checks at the start and at the "loop" continuation test to avoid prematurely exiting the subrule. For example, consider the following class definition rule.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090625755.png)

An extra semicolon in the member list such as int i;; int j; should not force surrounding rule classdef to abort. Instead, the parser ignores the extra semicolon and looks for another member. To reduce cascading error messages, the parser issues no further messages until it correctly matches a token.

#### B.4 Multi-threaded execution

Applications often require parallel execution of multiple parser instances, even for the same language. For example, web-based application servers parse multiple incoming XML or JSON data streams using multiple instances of the same parser. For memory efficiency, all _ALL(*)_ parser instances for a given language must share lookahead DFA. The Java code that ANTLR generates uses a shared memory model and threads for concurrency, which means parsers must update shared DFA in a thread-safe manner. Multiple threads can be simulating the DFA while other threads are adding states and edges to it. Our goal is thread safety, but concurrency also provides a small speed up for lookahead DFA construction (observed empirically).

The key to thread safety in Java while maintaining high throughput lies in avoiding excessive locking (synchronized blocks). There are only two data structures that require locking: $Q$, the set of DFA states, and $\Delta$, the set of edges. Our implementation factors state addition, $Q$ += $D^{\prime}$, into an _addDFAstate_ function that waits on a lock for $Q$ before testing a DFA state for membership or adding a state. This is not a bottleneck as DFA simulation can proceed during DFA construction without locking since it traverses edges to visit existing DFA states without examining $Q$.

Adding DFA edges to an existing state requires fine-grained locking, but only on that specific DFA state as our implementation maintains an edge array for each DFA state. We allow multiple readers but a single writer. A lock on testing the edges is unnecessary even if another thread is racing to set that edge. If edge $D\xrightarrow{a}D^{\prime}$ exists, the simulation simply transitions to $D^{\prime}$. If simulation does not find an existing edge, it launches ATN simulation starting from $D$ to compute $D^{\prime}$ and then sets element $edge[a]$ for $D$. Two threads could find a missing edge on $a$ and both launch ATN simulation, racing to add $D\xrightarrow{a}D^{\prime}$. $D^{\prime}$ would be the same in either case so there is no hazard as long as that specific edge array is updated safely using synchronization. To encounter a contested lock, two or more ATN simulation threads must try to add an edge to the same DFA state.

### C Left-recursion elimination

ANTLR supports directly left-recursive rules by rewriting them to a non-left-recursive version that also removes any ambiguities. For example, the natural grammar for describing arithmetic expression syntax is one of the most common (ambiguous) left-recursive rules. The following grammar supports simple modulo and additive expressions.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090625288.png)
$E$ is _directly_ left-recursive because at least one production begins with $E$ ($\exists\,E\Rightarrow E\alpha$), which is a problem for top-down parsers.

Grammars meant for top-down parsers must use a cumbersome non-left-recursive equivalent instead that has a separate nonterminal for each level of operator precedence:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090624679.png)
The deeper the rule invocation, the higher the precedence. At parse-time, matching a single identifier, $\mathbf{a}$, requires $l$ rule invocations for $l$ precedence levels.

$E$ is easier to read than $E^{\prime}$, but the left-recursive version is ambiguous as there are two interpretations of input a+b+c: (a+b)+c and a+(b+c). Bottom-up parser generators such as bison use operator precedence specifics (e.g., %left 'Y') to resolve such ambiguities. The non-left-recursive grammar $E^{\prime}$ is unambiguous because it implicitly encodes precedence rules according to the nonterminal nesting depth.

Ideally, a parser generator would support left-recursion and provide a way to resolve ambiguities implicitly with the grammar itself without resorting to external precedence specifiers. ANTLR does this by rewriting nonterminals with direct left-recursion and inserting semantic predicates to resolve ambiguities according to the order of productions. The rewriting process leads to generated parsers that mimic Clarke's [5] technique.

We chose to eliminate just direct left-recursion because general left-recursion elimination can result in transformed grammars orders of magnitude larger than the original [11] and yields parse trees only loosely related to those of the original grammar. ANTLR automatically constructs parse trees appropriate for the original left-recursive grammar so the programmer is unaware of the internal restructuring. Direct left-recursion also covers the most common grammar cases (from long experience building grammars). This discussion focuses on grammars for arithmetic expressions, but the transformation rules work just as well for other left-recursive constructs such as C declarators: $D\rightarrow*$$D$, $D\to D\;[\;]$, $D\to D\;(\;)$, $D\rightarrow\mathbf{id}$.

Eliminating direct left-recursion without concern for ambiguity is straightforward [11]. Let $A\to\alpha_{j}$ for $j=1..s$ be the non-left-recursive productions and $A\to A\beta_{k}$ for $k=1..r$ be the directly left-recursive productions where $\alpha_{j},\beta_{k}\not\Rightarrow^{*}\epsilon$. Replace those productions with:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090624339.png)
The transformation is easier to see using EBNF:

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090623658.png)
or just $A\to(\alpha_{1}|...|\alpha_{s})(\beta_{1}|...|\beta_{r})^{*}$. For example, the left-recursive $E$ rule becomes:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090623142.png)
This non-left-recursive version is still ambiguous because there are two derivations for a+b+c. The default ambiguity resolution policy chooses to match input as soon as possible, resulting in interpretation (a+b)+c.

The difference in associativity does not matter for expressions using a single operator, but expressions with a mixture of operators must associate operands and operators according to operator precedence. For example, the parser must recognize a%b+c as (a%b)+c not a%(b+c).The two interpretations are shown in Figure 14.

To choose the appropriate interpretation, the generated parser must compare the previous operator's precedence to the current operator's precedence in the $(\%\;E\,|+E)^{*}$ "loop." In Figure 14, $\underline{E}$ is the critical expansion of $E$. It must match just $\mathbf{id}$ and return immediately, allowing the invoking $E$ to match the $+$ to form the parse tree in **(a)** as opposed to **(b)**.

To support such comparisons, productions get precedence numbers that are the reverse of production numbers. The precedence of the $i^{th}$ production is $n-i+1$ for $n$ original productions of $E$. That assigns precedence 3 to $E\to E\,\%\,E$, precedence 2 to $E\to E+E$, and precedence 1 to $E\to\mathbf{id}$.

Next, each nested invocation of $E$ needs information about the operator precedence from the invoking $E$. The simplest mechanism is to pass a precedence parameter, $pr$, to $E$ and require: _An expansion of $E[pr]$ can match only those subexpressions whose precedence meets or exceeds $pr$_.

To enforce this, the left-recursion elimination procedure inserts predicates into the $(\%\;E\,|+E)^{*}$ loop. Here is the transformed unambiguous and non-left-recursive rule:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090623325.png)
References to $E$ elsewhere in the grammar become $E[0]$; e.g., $S\to E$ becomes $S\to E[0]$. Input a%b+c yields the parse tree for $E[0]$ shown in **(a)** of Figure 15.

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090622597.png)
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090621926.png)

Production "$\{3\geq pr\}$? $\%E[4]$" is viable when the precedence of the modulo operation, 3, meets or exceeds parameter $pr$. The first invocation of $E$ has $pr=0$ and, since $3\geq 0$, the parser expands "$\%\;E[4]$" in $E[0]$.

When parsing invocation $E[4]$, predicate $\{2\geq pr\}$? fails because the precedence of the $+$ operator is too low: $2\not\geq 4$. Consequently, $E[4]$ does not match the $+$ operator, deferring to the invoking $E[0]$.

A key element of the transformation is the choice of $E$ parameters, $E[4]$ and $E[3]$ in this grammar. For left-associative operators like $\%$ and $+$, the right operand gets one more precedence level than the operator itself. This guarantees that the invocation of $E$ for the right operand matches only operations of higher precedence.

For right-associative operations, the $E$ operand gets the same precedence as the current operator. Here is a variation on the expression grammar that has a right-associative assignment operator instead of the addition operator:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090621651.png)
where notation $=_{right}$ is a shorthand for the actual ANTLR syntax "$|$<assoc=right>$E\;=^{right}E$." The interpretation of a=b=c should be right associative, a=(b=c). To get that associativity, the transformed rule need differ only in the right operand, $E[2]$ versus $E[3]$:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090621952.png)
The $E[2]$ expansion can match an assignment, as shown in **(b)** of Figure 15, since predicate $2\geq 2$ is true.

Unary prefix and suffix operators are hardwired as right- and left-associative, respectively. Consider the following with negation prefix and "not" suffix operators.
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090620138.png)

Prefix operators are not left recursive and so they go into the first subrule whereas left-recursive suffix operators go into the predicated loop like binary operators:
![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090620081.png)
Figure 16 illustrates the rule invocation tree (a record of the call stacks) and associated parse trees resulting from an ANTLR-generated parser. Unary operations in contiguous productions all have the same relative precedence and are, therefore, "evaluated" in the order specified. E.g., $E\to-E\,|\,+E\,|\,\mathbf{id}$ must interpret $-+a as -(+a)$ not $+(-a)$.

. Nonconforming left-recursive productions $E\to E$ or $E\to\epsilon$ are rewritten without concern for ambiguity using the typical elimination technique.

Because of the need to resolve ambiguities with predicates and compute $A$ parameters,

#### C.1 Left-Recursion Elimination Rules

To eliminate direct left-recursion in nonterminals and resolve ambiguities, ANTLR looks for the four patterns:

![image.png](https://blog-1314253005.cos.ap-guangzhou.myqcloud.com/202310090619579.png)
The subscript on productions, $A_{i}$, captures the production number in the original grammar when needed. Hidden and indirect left-recursion results in static left-recursion errors from ANTLR. The transformation procedure from $G$ to $G^{\prime}$ is:

1. Strip away directly left-recursive nonterminal references
2. Collect prefix, primary productions into newly-created $A^{\prime}$
3. Collect binary, ternary, and suffix productions into newly-created $A^{\prime\prime}$
4. Prefix productions in $A^{\prime\prime}$ with precedence-checking semantic predicate $\{pr(i){>}=\mathtt{pr}\}$? where $pr(i)=\{n-i+1\}$
5. Rewrite $A$ references among binary, ternary, and prefix productions as $A[nextpr(i,assoc)]$ where $nextpr(i,assoc)=\{assoc==left?i+1:i\}$
6. Rewrite any other $A$ references within any production in $P$ (including $A^{\prime}$ and $A^{\prime\prime}$) as $A[0]$
7. Rewrite the original $A$ rule as $A[\mathtt{pr}]\to A^{\prime}A^{\prime\prime*}$
In practice, ANTLR uses the EBNF form rather than $A^{\prime}A^{\prime\prime*}$.
