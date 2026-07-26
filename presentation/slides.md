---
marp: true
theme: default
paginate: false
title: Worst-Case Optimal Datalog (++)
author: Frank McSherry (presented by Moritz Hoffmann)
---

<style>
/* Fix titles at the top: top-align content slides (those whose first element
   is an h2) so the title stays put regardless of how much content follows. */
section:has(> h2:first-child) {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}
</style>

# Worst-Case Optimal Datalog (++)

<!--
    Frank McSherry
    Minnowbrook Analytic Reasoning Seminar
    June 2026
-->

Moritz Hoffmann (Materialize Inc.)
Foundations of Data Management
July 2026

Work by Frank McSherry; presented by Moritz Hoffmann.

<!--
* worst-case optimal datalog is more important than the audience might see.
* In the end, the whole computation, full, semi-naive, comes with worst-case optimal bounds. You wouldn't get them from joins alone.
* WCOJ alone doesn't compose, don't do 1x1M repeatedly
* Reinterpretation of existing work (Ammar et al), taking streaming computation for datalog
*
* DISTRIBUTION: those last two bullets are Frank's, and they are NOT for this slide any more -
* saying them here front-loads the talk and spends the payoffs early. Where they land now:
*   - "doesn't compose" -> slide 7, where the delta sum is on screen and the argument is visible.
*   - Ammar et al.      -> slide 2 in passing, then properly on slide 8 with the theorem.
* Here: credit Frank, the systems-engineer caveat, and the claim. Then stop.
-->
---

## Context: [`datatoad`](https://github.com/frankmcsherry/datatoad) framework

A { columnar, wco, ~directional, .. } DatalogZ (with integers)

- Columnar data layout and computation.
- Worst-case optimal incremental joins. ([Ammar et al., VLDB 2018](https://www.vldb.org/pvldb/vol11/p691-ammar.pdf))
- { bi / many / omni } - directional predicates.
- Interactive, Distributed, Low memory, ..

<!--
* concrete implementation
* properties 1-3 are consequential, 4 is important for system implementors, consider cutting 3, but it is interesting.
  * All the logical predicates also play nice with the framework.
* DatalogZ defence, say this: "Integers, but arrived at as *relations* rather than expressions - so this is
  DatalogZ in spirit; the mode system is what keeps it from running away, not a syntactic restriction."
  Pre-empts "which decidable fragment?" from the theory side.
-->
---

<style scoped>
section { padding: 40px 56px; }
section h2 { margin: 0 0 0.4em; }
section pre { font-size: 0.8em; line-height: 1.3; margin: 0.5em 0; }
section p { margin: 0.45em 0; }
</style>

## `datatoad`, in the browser

A graph where *every* pair-at-a-time join plan blows up:

```
arc(0, x) :- :range(1, x, 1000001).
arc(x, 0) :- :range(1, x, 1000001).
arc(x, y) :- :range(1, x, 1000001), :plus(x, 1, y).

tri(a, b, c) :- arc(a,b), arc(b,c), arc(c,a).
```

3M facts, **in your browser**: load ~120ms, all triangles ~1s.
The same query in *native* PostgreSQL: *still running.*

**▶ Try it:** `frankmcsherry.org/datatoad/demo`

<!--
* Wasm datatoad: the whole engine compiled to WebAssembly, client-side.
* SLIDE IS STATIC ON PURPOSE - shared machine, unknown network. Say the numbers, don't run anything.
* THESE NUMBERS ARE FROM THE BROWSER, single-threaded wasm - not the native build. Don't say "natively".
  That's the better story anyway: it beats native PostgreSQL from inside a browser tab.
* No pair-at-a-time order avoids ~1 trillion intermediate results here; PostgreSQL spins up helpers and maxes the CPUs.
* Backup live-demo slide is at the very end, after Thanks. Only jump to it if the network is known-good AND you have time; the wasm build is single-threaded and freezes the page while it runs.
-->

---

## Talk outline

1.  A worst-case optimal join algorithm, done columnar.

2.  WCO join bounds (can) extend to indexes, streaming, iteration.

3.  Columnar WCOJ is actually a nice **interface** to relations.

    - Disjunctive clauses (body only)
    - Directional predicates
    - FFI / Extensibility

<!--
* Can delete (1.) but: other algorithms don't talk about data structures, because the worst-case optimality doesn't come from the data structures.
* (2) restates Ammar, delta joins.
* (3) is the extensibility, potential for cutting.
* There used to be a fourth item, "Comments, provocations, and future directions". The
  provocations and future-directions slides (Conversation starters, DDIR) are cut for the
  15-min slot, and what remains is a summary, not a fourth part - so the outline says three.
  The one argument you do want to invite (how the bound compares to IVM / dynamic query
  evaluation) is already placed on the theorem slide, where it lands first.
-->

---

## Columnar WCO Joins (1/2)

An instance of GenericJoin (NPRR).

    Head(T0, T1, .. ) :- Body0(T1, ..), Body1(.., T0), .. BodyK(Tj) .

<div style="height: 0.6em"></div>

Develop assignments satisfying the body projected on increasing subsets of terms.
1.  Terms start `{ }`, trivial assignment satisfies the body (if all non-empty).
2.  Repeatedly add a new term, update satisfying assignments by that term.
3.  Project down to head terms (and as you go, if you like).

<!--
* Lookup GenericJoin NPRR, framework. We're doing breadth-first instead of depth-first.
-->

---

## Columnar WCO Joins (2/2)

An instance of GenericJoin (NPRR).

    Head(T0, T1, .. ) :- Body0(T1, ..), Body1(.., T0), .. BodyK(Tj) .

<div style="height: 0.6em"></div>

To extend an assignment `A` by a term `Ti`, each involved atom does three things
1. Each atom that mentions `Ti` **counts**
    1. for each `a` in `A` its distinct compatible `ti` values.
1. Each atom that mentions `Ti` **extends** by `ti \in Ti`
    1. each `a` in `A` for which it had the least count.
1. Each atom that mentions `Ti` **validates**
    1. each extended assignment `[a;Ti=ti]`.

<!--
* Up until here it's not novel, just an explanation of WCOJ.
-->

---

## Streaming WCO Joins (1/2)

    Head(T0, T1, .. ) :- Body0(T1, ..), Body1(.., T0), .. BodyK(Tj) .

What actually happens is that the input atoms *change*: `dBody0`, `dBody1`, ... `dBodyK`.

<div style="height: 0.6em"></div>

Could restart from scratch, but can also determine `dHead`:

    dHead = dBody0, (Body1 + dBody1), .., (BodyK + dBodyK)
          + dBody1, (Body0 + dBody0), .., (BodyK + dBodyK)
          + ..
          + dBodyK, (Body0 + dBody0), (Body1 + dBody1), ..

A sequence of *seeded* WCO joins, against maintained indexes. Constrained term order.

<!--
* what goes wrong when you just do each of these as WCOJ? Each is a join of a small term with a big term, and WCOJ gives permission to do all of it. We need to depend more strongly on the `d` term.
  * If we have multi-set semantics, we need to be careful not to mention `d`s repeatedly, but also mentioned in Ammar et al.
-->

---

## Streaming WCO Joins (2/2)

    Head(T0, T1, .. ) :- Body0(T1, ..), Body1(.., T0), .. BodyK(Tj) .

What actually happens is that the input atoms *change*: `dBody0`, `dBody1`, ... `dBodyK`.

<div style="height: 0.6em"></div>

**Theorem:** *For append-only changes, the WCO bound applies using the final atom sizes.*

For [Ammar et al]: "Streaming WCO Joins", but here "WCO Datalog".

**Corollary:** For each Datalog rule, the WCO bound applies using the final atom sizes.

---

## Columnar WCO Joins: An API

```rust
/// A type that can participate in WCO joins.
pub trait ExecAtom<T> {

    /// An initial set of facts, either the recent facts or all facts.
    fn facts(&self, recent: bool) -> Facts<T>;

    /// The number of values of `added` terms that extend each fact.
    fn count(&self, facts: &mut Facts<T>, added: &Set<T>);

    /// Join or semijoin `facts` with `self`, introducing `added`.
    fn join(&self, facts: &mut Facts<T>, added: &Set<T>);

}
```

---

## What implements `ExecAtom`?

1. Stored relations (the obvious one)
2.
3.
4.
5.

---

## What implements `ExecAtom`?

1. Stored relations (the obvious one)
2. Antijoins .. wait.
3.
4.
5.

---


## Columnar WCO Joins: Another API

```rust

/// A type that can be *planned* in WCO joins
pub trait PlanAtom<T> {

    /// Terms the atom references.
    fn terms(&self) -> Set<T>;

    /// Terms that can be made concrete from other concrete terms.
    ///
    /// The output are cardinality bounds, like Mercury's determinism levels.
    fn modes(&self, from: &Set<T>, onto: &Self) -> (usize, Option<usize>);

}
```

---

## What implements `PlanAtom + ExecAtom`?

1. Stored relations (the obvious one)
2. Antijoins.
3. Disjunctions.
4.
5.

---

## What implements `PlanAtom + ExecAtom`?

1. Stored relations (the obvious one)
2. Antijoins.
3. Disjunctions.
4. Logic (Relations)
5.

---

## Logic Atoms

Consider:

```
:plus(a, b, c) : { (a, b, c) | a + b = c }
```

<div style="height: 0.6em"></div>

Is it a predicate, or a function, or a relation?

1. As a **predicate** it can map `(a, b, c)` to present or not.
2. As a **function** it can map `(a, b)` to `c`.
3. As a **relation** it can map any two of `{ a, b, c }` to the other.

This is enough for `PlanAtom + ExecAtom`.

---

## `:plus` in action

```
tri1(a, b, c) :- arc(a, b), arc(b, c), arc(c, a).
```
```
tri2(a, b, c) :- arc(a, b), arc(b, c), arc(c, a), :plus(a, b, c).
```

The second query can have far fewer results, but do we notice in time?

|  N=1000       | `tri1` | `tri2` |
|---------------|-------:|-------:|
| Outputs       |     1B |  ~500K |
| Time          | 117.4s |  196ms |

---

## A more-WCO example

Sensors `s` with readings `r`; few are noisy, many are not.
```
data(s, r) :- :range(0, s, 256), :range(0, r, 65536).
data(s, s) :- :range(256, s, 65536).
```
Queries for sensors `s` have ranges `[l, u)`; narrow for noisy, wide for others.
```
asks(s, 0, 256) :- :range(0, s, 256).
asks(s, 0, 65536) :- :range(256, s, 65536).
```
Intersecting the query range with the sensor readings
```
query(s, r) :- asks(s, l, u), data(s, r), :range(l, r, u).
```

---

## Connecting to relational programming

Languages that have supported this for years:

- **CLP** (Prolog with `clpfd`, etc.) — via constraint solvers.
- **Mercury** — via compile-time mode declarations.
- **Kanren family** — via runtime unification + search.

Maybe new here: **the WCOJ count protocol as the mechanism**.
<!--
Generous to prior art. The "marriage of relational programming with WCOJ throughput"
is the framing that lands here.
AUDIENCE CHECK DONE: there are NO Mercury / kanren / CLP / logic-programming people on the
FDM26 in-person list. (The original Minnowbrook note said "Hemann is in the room" - not here.)
So this slide is pure prior-art citation, not a nod to anyone present. Deliver it as honest
context, don't fish for recognition, and keep "maybe new here" hedged - Suciu and Abo Khamis
are in the room and novelty claims are cheap in front of them.
-->



---


## What implements `PlanAtom + ExecAtom`?

1. Stored relations (the obvious one)
2. Antijoins.
3. Disjunctions.
4. Logic (Relations)
5. Extensibility (FFI)

---

```
┌────────────────┬────────┬────────┬────────┬────────┬─────────┐
│    problem     │  1×1   │  1×4   │  4×1   │  4×4   │ 1×1→4×4 │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ galen          │  10.10 │   5.01 │   4.65 │   3.38 │    3.0× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ tc-10k         │  19.58 │   8.94 │   7.13 │   2.74 │    7.1× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ sg-10k         │  34.63 │  14.35 │  12.01 │   4.67 │    7.4× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ cspa           │  29.47 │  11.46 │  10.18 │   4.25 │    6.9× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ dyck           │   1.48 │   0.81 │   0.75 │   0.41 │    3.6× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ csda           │   1.48 │   0.70 │   0.73 │   0.43 │    3.4× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ andersen       │   1.86 │   1.33 │   1.26 │   0.68 │    2.7× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ cvc5           │  43.51 │  15.73 │  13.70 │   8.48 │    5.1× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ z3             │  55.87 │  27.95 │  28.40 │  22.58 │    2.5× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ reach          │  46.74 │  15.25 │  11.48 │   7.70 │    6.1× │
├────────────────┼────────┼────────┼────────┼────────┼─────────┤
│ batik          │ 254.59 │ 216.67 │ 184.80 │ 187.53 │    1.4× │
└────────────────┴────────┴────────┴────────┴────────┴─────────┘
```

---

## Closing

Worst-case optimal Datalog is neat and different from WCO Join.

But the **columnar orientation** also turns out to be a uniform interface:

- Stored relations
- Antijoins
- Sums and differences
- Directional predicates
- External data

Each adds capability without new engine machinery.
Columnar WCO Datalog ends up being a substrate, not just a faster engine.

---

# Thanks

Questions?

<!--
* Cut for the 15-minute slot: Hackathon, Conversation starters, DDIR language slide.
  They live in git history (see slides.md before this commit) if a longer slot opens up.
* BACKUP SLIDE FOLLOWS. Do not advance past this one in the normal flow.
-->

---

<!-- BACKUP: live demo. Only if the network is known-good and time allows. -->

<style scoped>section { padding: 28px; }</style>

<iframe src="http://www.frankmcsherry.org/datatoad/demo/" width="1180" height="640"
        style="border:1px solid #d0d7de; border-radius:8px; display:block; margin:0 auto;"></iframe>

<!--
* Single-threaded wasm; it freezes the page while it runs. Keep any live query SMALL.
-->
