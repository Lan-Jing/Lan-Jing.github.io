---
title: "Regular Expressions, Finite Automatons"
categories:
  - PL
  - Math
---

This article covers some very basic concepts and algorithms for lexical analysis.

## Regular Expressions

### Alphabet and Language

* Alphabet: an alphabet is any finite set of symbols.
* String: a string is any finite sequence of symbols from an alphabet.
* Language: a language is a subset of the closure of an alphabet.

### Language Operations

* Union: $ L \bigcup M = \{ s\vert s \in L \text{ or } s \in M \} $
* Concatenation: $ LM = \{ st \vert s \in L \text{ and } t \in M \}$
* Kleene closure: $ L^*=\bigcup_{i=0}^{\infty}L^i $
* Positive closure: $ L^+=\bigcup_{i=1}^{\infty}L^i $

### Definition of Regular Expression

It is defined in recursion. Base:

* $L(\epsilon)=\{\epsilon\}$
* $L(a)=\{a\}, a \in \Sigma$

Induction: if r and s are regular exprs,

* $L(r\vert s)=L(r) \bigcup L(s)$
* $L(rs)=L(r)L(s)$
* $L(r^{\ast})=L(r)^{\ast}$
* $L((r))=L(r)$

## Nondeterministic Finite Automaton (NFA)

### Regular Expr to NFA

A regular expression can be expressed as an NFA. An NFA is formally a 5-tuple, $(Q,\Sigma,\Delta,q_0,F)$, which:

* Q is a set of states
* $\Sigma$ is a set of input symbols
* $\Delta$ is a transition function
* $q_0$ is an initial state
* $F \subseteq Q$ is a set of final states 

From the initial state, an NFA maps each state $s$ to a set $S$ of succeeding states. One can recursively construct the corresponding NFA from a regular expression, in a bottom-up manner:

* For concatenation of two sub-expressions, link them directly.
* For union operations, set two arms for each expr, then connect them with other parts, using $\epsilon$ as input symbols.
* For Kleene closure, set a path backward from the end of the expr. And a shortcut to skip what is inside the closure(**not needed for positive closure**).

![Regular Expr to NFA]({{ site.url }}{{ site.baseurl }}/assets/images/reg-to-nfa.png){: .align-center}

### Evaluate an Expression Using NFA

For NFA, the basic idea is to maintain a set of **reachable** states $S$. In each iteration, use the current symbol $c$ to expand $S=\epsilon-closure(move(S,c))$. Finally, check whether an accepting state is inside $S$.

```c++
  S = epsilon_closure(s_0);
  c = next_char();
  while(c != eof) {
    S = epsilon_closure(move(S, c));
    c = next_char();
  }
  return S intersect F is not empty ? True : False;
```

## Deterministic Finite Automaton (DFA)

DFA is a special case of NFA. A DFA has these additional properties:

* $\epsilon$ not allowed to be an input symbol
* Exactly one state, instead of a set, can be the output of each transition. (**Deterministic**)

### Evaluate an Expression Using DFA

It is trivial to evaluate using a DFA, since the maintained state is deterministic at any time. Follow edges out from the states, until a dead state or a final state is met:

```c++
  s = s_0;
  c = next_char();
  while(c != eof) {
    s = move(s, c);
    c = next_char();
  }
  return s is in F ? True : False;
```

### NFA to DFA

We can easily construct an NFA that each node is paired with another in the expression. This is hard for DFA, however. To get an equivalent DFA, the key idea is to group all states that can be reached **at the same time** as one DFA node.

First, We have a $S=\epsilon-closure(n_0)$ that each state inside it can be the first symbol of the expression. Then, for each input symbol $c$, we compute the next group of states $S'=\epsilon-closure(move(S,c))$ and mark it as a new node(if not before). Finally, link $S$ and $S'$ with $c$.

```c++
  Dstates.insert(epsilon_closure(s_0));
  while(Dstates is not empty) {
    T = Dstates.pop();
    if(visited[T] == True)
      continue;
    else
      visited[T] = True;
    
    for(each input symbol a) {
      U = epsilon_closure(move(T, a));
      Dstates.insert(U);
      Dtran[T, a] = U;
    }
  }
```

### Minimizing a DFA

Multiple equivalent DFAs are possible, but only one is optimal. It means some nodes in a DFA can be combined: if two nodes **respond equally for any input symbol**, we can bind them together with the input edges unioned but the out edges unchanged.

Therefore, it is easy to distinguish them by repeat tests with each input symbol. Ultimately you will get groups of nodes, which nodes inside a group can be merged but are distinguishable between groups.