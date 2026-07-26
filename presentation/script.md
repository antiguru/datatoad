# Speaker script — *Worst-Case Optimal Datalog (++)*

**FDM26 workshop · 20-minute slot, targeting ~15:00 · presenter: Moritz Hoffmann**
**Work by Frank McSherry; presented by Moritz Hoffmann.**

This is *one* delivery of the deck, written to react against — a concrete
alternative, not the only way. Part 1 is the mental model to have in your head
before you walk up. Part 2 is a slide-by-slide spoken script covering all 22
slides, title → "Thanks". Timings assume ~135 words per minute. **As written it
runs ~17:05 — inside the 20-minute slot, ~2:05 over the 15:00 target. See
Pacing at the end of Part 1 for the optional trims.**

---

# Part 1 — Background: what you need to know

## The one thesis (memorize this)

> Worst-case optimality for Datalog is a property of the **whole semi-naive
> computation**, not of a single join — and you get it by treating Datalog's
> fixpoint as an *insertion-only stream*. Being columnar turns the join
> machinery into an *interface* that antijoins, disjunctions, integer logic,
> and foreign relations all plug into.

Everything on the slides is in service of those two clauses: (1) the *end-to-end
bound*, (2) the *interface*.

## The 20-second elevator version

"Datalog rule bodies are multi-way joins. Worst-case optimal join algorithms
evaluate one such join within the best possible bound — but running one per
rule per iteration does **not** bound the whole fixpoint. If you notice that
semi-naive evaluation only ever *inserts* facts, you can borrow Ammar et al.'s
streaming WCO result and bound the *entire* computation by the final relation
sizes. And because we did it column-at-a-time, the same `count / extend /
validate` protocol becomes a plug-in interface for a lot more than stored
relations."

## The novelty ledger — be honest about what's new

Keep these three buckets straight; the deck itself is scrupulous about them, and
this is the question you're most likely to get.

| Bucket | What | Your line |
|---|---|---|
| **Not novel** | Columnar WCOJ as an instance of Generic Join. | "This part is just an explanation of WCOJ." |
| **Reframing** | Ammar et al.'s insertion-only streaming WCOJ → end-to-end bound for semi-naive Datalog. | "This is a reinterpretation of existing work — the contribution is *seeing* that Datalog's fixpoint is that streaming setting." |
| **Genuinely new-ish** | Columnar count-protocol as a compositional interface; omni-directional logic atoms that participate in the WCO count (→ DatalogZ). | "This is where it stops being a faster engine and becomes a substrate." |

Do **not** claim "we invented WCO Datalog." Claim: "we get a *whole-computation*
bound, cheaply, and a uniform interface out of it." Generosity to prior art
lands better with this audience than a big novelty claim.

## The crux you must be able to defend: "WCOJ alone doesn't compose"

Someone will push here. The argument:

- Semi-naive re-derives facts using *delta* rules: `dHead = dBody0·(rest) +
  dBody1·(rest) + …`, one term per body atom, each iteration.
- If you evaluate each delta term as its own independent WCO join, you're
  joining a small delta against big full relations, over and over. WCOJ
  "gives you permission" to touch all of the big side — so summed across
  iterations the cost is **not** bounded by the final sizes. (Frank's shorthand:
  *"don't do 1×1M repeatedly."*)
- Ammar's result is what ties the sequence of *seeded* joins, against maintained
  indexes, back to a single bound in the **final** atom sizes. The seeding and
  the "don't double-count deltas" discipline are the whole trick.

If pressed on multiset/duplicate handling: yes — you must avoid re-counting the
same delta across the telescoped sum; Ammar handles this and so do we.

## Glossary — be fluent, don't lecture

- **AGM bound** — Atserias–Grohe–Marx (2008): the tight upper bound on a join
  query's output given input sizes. The yardstick "worst-case optimal" is
  measured against. Triangle query on size-*N* inputs: bound is *N^1.5*.
- **Generic Join / NPRR** — NPRR (PODS 2012) was the first algorithm meeting the
  AGM bound; Generic Join (2014) is the clean *framework* it and LeapFrog
  TrieJoin are instances of. **Precision:** we're an instance of *Generic Join,
  in the NPRR lineage* — not "an instance of NPRR." Say it the careful way if a
  theorist is in the room.
- **Semi-naive evaluation** — the standard Datalog fixpoint loop that only
  re-derives using newly-derived facts (deltas), not everything, each round.
- **The count protocol** — `count → extend → validate`: every atom counts its
  compatible extensions, the *smallest* count proposes values, the rest
  validate. This *is* Generic Join, and it's the mechanism everything else hangs
  off of.
- **Modes / determinism** — an atom declares which terms it can *produce* from
  which it's *given* (à la Mercury). This is what antijoins force us to add.
- **DatalogZ** — Datalog with integers; here it falls out of logic atoms like
  `:plus` and `:range` rather than being bolted on.

## The structural spine — *why* the slides are ordered this way

The middle of the talk is a little detective story; know the beats so you can
improvise if a slide misfires:

1. Here's the interface a WCO atom needs: `ExecAtom` (`facts / count / join`).
2. Stored relations obviously implement it.
3. Antijoins... **wait.** An antijoin can *validate* a proposed value ("is `x`
   absent?") but it can't *propose* values (the complement is infinite) and it
   can never be the smallest-count proposer. The interface is incomplete.
4. So add a *planning* interface: `PlanAtom` with `modes()` — an atom declares
   which terms it can make concrete, with cardinality bounds. That's the fix.
5. Now the floodgates open: antijoins, disjunctions, **logic** (integers),
   and **FFI** all implement `PlanAtom + ExecAtom`. Same engine, no new
   machinery.

The `:plus` payoff (`tri2` = 196 ms vs `tri1` = 117 s) is the emotional peak —
land it. It shows an omni-directional logic atom *participating* in the count so
the join proposes from `:plus` and never enumerates the billion triangles.

## Attribution posture

Open by crediting Frank plainly — it's his deck and his framework, you're
carrying it to this room. Say it once, on slide 1, and move on. It buys
goodwill and it's true.

## Positioning: you are the systems person, and that is a strength

You're a systems engineer presenting theory-adjacent work to a room of database
theorists — several of whom derive these bounds for a living (see **Audience
intel** below). The failure mode is bluffing fluency you don't have; two
questions deep, it shows, and it costs you the room.

**The fix is one honest sentence, delivered early, lightly, once.** Not an
apology — a declaration of what kind of talk this is:

> "One caveat up front: I'm a systems engineer, not a theoretician. Frank built
> this and derived the bounds; I can speak to how it works, why it's fast, and
> what it's like to use. For the deep questions about the bounds themselves I'll
> take names and put you in touch with Frank — and looking at this room, several
> of you could probably derive them faster than either of us."

Why this works: it's true, it's disarming, it sets expectations *before* anyone
invests in a question you can't answer, and the self-deprecating close reads as
confidence rather than weakness. **Say it on slide 1 and never again.** Don't
re-apologize at each theory slide — that turns one honest framing into a nervous
tic.

**In Q&A, three tiers:**

1. **Systems / behavioural questions — own them.** How it's implemented, why
   columnar, what's fast, what it's like to use, what the engine does on real
   workloads. This is your ground and you're the best person in the room on
   `datatoad` specifically.
2. **Theory questions you half-know — answer the part you know, mark the edge.**
   "The bound as I understand it is X — but whether that's tight under Y I'd be
   guessing, and I'd rather not guess in this room."
3. **Deep derivation questions — hand off cleanly and without embarrassment.**
   "That's a Frank question. Give me your email and I'll connect you — or he's
   very responsive on GitHub." **This is a completely respectable answer**, and
   in a room like this it's more respectable than a confident wrong one.

Have a way to actually capture the handoffs — phone note or paper. "I'll connect
you with Frank" is worth much more if you visibly write down the name.

## Audience intel (FDM26 in-person list)

**Short answer on Mercury/kanren/CLP: there are none.** No logic-programming
people on the in-person list — no Hemann, no Mercury or Prolog contingent. The
"Connecting to relational programming" slide is therefore *pure prior-art
citation*, not a nod to anyone present. Keep it — the intellectual honesty is
still worth it — but don't play it as though someone in the room owns that
tradition, and don't expect a knowing laugh.

**The real story is who *is* there.** This is a database-theory room, and it
contains much of the worst-case-optimal-join and dynamic-query-evaluation
establishment. Directly relevant:

- **Dan Suciu** (UW) — among the most prominent database theorists working on
  join bounds and information-theoretic query optimization.
- **Mahmoud Abo Khamis** (RelationalAI) — WCOJ theory (FAQ, PANDA); co-authors
  with **Hung Ngo** — the "N" in NPRR. Your slide's lineage claim will be heard
  by one of the lineage.
- **Dan Olteanu** (UZH) — **the host**, and the factorized-databases person.
- **Ahmet Kara** (OTH Regensburg) and **Haozhe Zhang** (UZH) — dynamic query
  evaluation and incremental view maintenance with Olteanu.
- **Christoph Koch** (EPFL) — DBToaster; incremental view maintenance.
- **Altan Birler** (TUM, Umbra) and **Niko Göbel** (RelationalAI) — WCOJ in real
  systems; likely the most sympathetic to the engineering story.
- **Gonzalo Navarro** (U. Chile) — compact/succinct data structures; a natural
  source of questions about the columnar layout.

**The one you must be ready for.** Abo Khamis, Kara, Olteanu and Suciu are
co-authors on *"Insert-Only versus Insert-Delete in Dynamic Query Evaluation"*
(2024), and Kara, Ngo, Nikolic, Olteanu and Zhang won ICDT 2019 Best Paper for
*"Counting Triangles under Updates in Worst-Case Optimal Time."* Read those
titles against your own talk: **your central theorem is an append-only
(insert-only) worst-case-optimal bound, and your headline example is triangles
under updates.** Three or four people in that room have published precisely
there.

This is opportunity, not danger — they will be the most interested people
present — but walking in without acknowledging that literature would look
uninformed, especially with Olteanu hosting. **Name it before they do**, e.g.
when you land the theorem:

> "This sits right next to the dynamic-query-evaluation and IVM work several of
> you have done — insert-only versus insert-delete, triangles under updates. I'd
> genuinely like to know how our bound lines up with yours; that's a
> conversation I want to have rather than a claim I want to make."

That single sentence converts your biggest exposure into the best conversation
you'll have all day. It also gives you a legitimate, non-defensive landing spot
for hard theory questions: *comparing* rather than *defending*.

## Anticipated Q&A (crisp answers)

- **"Isn't this just Ammar et al.?"** The join maintenance is; the contribution
  is recognizing Datalog's semi-naive fixpoint *is* the insertion-only setting,
  so the bound covers the whole computation — plus the interface work on top.
- **"How does this compare to Free Join / egglog?"** Both interpolate between
  binary and WCO plans; we converged on similar territory from the columnar/
  streaming side. Honestly: "that's about all I know — I'd genuinely like to
  compare notes afterwards."
- **"Soufflé?"** Binary-join-based and heavily optimized; the WCO angle is a
  different axis — we win big on cyclic/skewed queries where binary plans blow
  up.
- **"Why columnar?"** It's what lets `count/extend/validate` batch over whole
  columns and what makes the interface cheap — the compositionality is
  downstream of the columnar choice.
- **"Demand/magic sets?"** A tension, not a fit — demand presupposes *avoiding*
  materialization; WCOJ presupposes it. (No slide for this any more — it's a
  coffee argument. Don't open it unless asked.)
- **"Is `:plus` sound over infinite domains?"** It's mode-constrained: it only
  *proposes* when enough terms are bound to make the output finite; otherwise it
  only validates. That's exactly what `modes()` encodes.
- **"How does this relate to IVM / dynamic query evaluation / factorized IVM?"**
  *(Expect this from Olteanu, Kara, Zhang, Koch, or Abo Khamis — it's their
  literature.)* Honest answer: the delta-rule machinery is the same shape as
  higher-order IVM, and the append-only restriction is exactly the insert-only
  side of the insert-only/insert-delete distinction they've studied. What's
  different is the target — a whole Datalog fixpoint rather than a single
  maintained view — and the columnar interface on top. **Do not claim priority
  or superiority here.** Ask how the bounds compare; you'll learn more than you
  could assert.
- **"Is the append-only bound tight? What happens with deletions?"** The bound
  is stated for append-only derivation, which is what monotone Datalog does. For
  deletions you're outside the theorem, and that's a Frank question — say so.
- **"In what sense is this DatalogZ? Which decidable fragment?"** Plain Datalog
  can only shuffle constants it was given, so it always terminates; DatalogZ adds
  integers, can *construct* new values, and in general gives up guaranteed
  termination. Datatoad has the integers — but arrives at them as *relations*
  (`:plus`, `:times`, `:range`, `:noteq`) rather than as expressions, so they're
  omni-directional and can *propose* values, not just filter. **Say:** "DatalogZ
  in spirit; the mode system is what keeps it from running away, not a syntactic
  restriction." Don't claim a decidable-fragment theorem — there isn't one
  attached.

## Don't overclaim (precision caveats)

- "instance of Generic Join (NPRR lineage)", not "of NPRR".
- "worst-case optimal" is up to standard log factors / data-dependent constants;
  don't say "optimal" bare-faced to a theory crowd.
- The end-to-end bound is for **append-only** derivation (monotone Datalog);
  don't imply it covers arbitrary negation/aggregation mid-fixpoint.

## Pacing plan

The slot is **20 minutes**; the target is **~15:00**, leaving real slack for
questions and overrun. Summing the per-slide timings in Part 2:

| Segment | Slides | Target |
|---|---|---|
| Framing + positioning + context + demo + outline | 1–4 | 2:45 |
| The WCO join (breadth-first) | 5–6 | 2:30 |
| Streaming → theorem + IVM acknowledgement | 7–8 | 2:50 |
| Interface: ExecAtom → antijoin → PlanAtom | 9–13 | 3:05 |
| Logic atoms + `:plus` payoff | 14–16 | 2:20 |
| Sensors example (keep) | 17 | 1:00 |
| Relational programming + FFI + benchmarks | 18–20 | 1:40 |
| Closing + Thanks | 21–22 | 0:55 |
| **Total as written** | | **~17:05** |

**Everything fits the slot as written** — 17:05 in a 20-minute room leaves
~3 minutes spare. So nothing here is forced. To reach the 15:00 target and bank
5 minutes for questions, two trims are enough:

1. **Drop slide 18, relational programming** (−0:40). Best value for money:
   there are **no Mercury / kanren / CLP people in the room** (see Audience
   intel), so it has the least purchase on this audience. The tradeoff is that
   it's your generosity-to-prior-art beat — if you cut the slide, keep one spoken
   sentence ("this tradition goes back to CLP and Mercury") so the credit lands.
2. **Compress the interface run, slides 9–13** (−0:50). Those five slides are
   quick beats — the two "what implements" builds are ten seconds each, not
   twenty, and slide 13 can be a single sentence. Take the segment to ~2:15.

That lands **~15:35**, near enough to target. If you want to be strictly under,
collapse the two "Columnar WCO Joins" slides into one pass (−0:45) → ~14:50.

**Do not cut:**

- The **streaming/theorem** beat and the **`:plus`** payoff — those are the talk.
- The **slide-1 positioning** and the **IVM acknowledgement** on slide 8. They
  cost ~0:35 between them and are the two highest-value additions for *this*
  room specifically.
- **Slide 17, the sensors example.** An earlier draft of this plan had it as the
  first cut, calling it redundant with `:plus`. That was wrong. `:plus` shows a
  logic atom cutting work; the sensors example shows the cheapest proposer
  *flipping per tuple* under skew — which is what worst-case optimality is
  actually for, and exactly what the WCOJ theorists present think about. See the
  slide-17 script for the numbers; it may be your second-strongest slide here.

Two things already work in your favour: the demo slide is **static** (no load,
no run, no recovery-from-failure), and everything from Hackathon onward is
**cut** from the deck.

## Demo: deliberately not live

You're on a **shared machine with an unknown network**, so the demo slide is now
**static** — the query and its timings are printed on the slide, and the URL is
there for the audience. Nothing loads, nothing can fail, and it costs no time.

Rationale, if anyone asks why you didn't demo: a live iframe that fails renders
a blank box mid-talk, and even when it works the WASM build is single-threaded
and *freezes the page* while it runs.

**A backup live-demo slide sits at the very end, after "Thanks."** Jump to it
only if (a) the network is known-good — test it before you start — and (b)
questions are running short. If you do, keep the query *small*; never type the
million-node triangle query live.

---

# Part 2 — Slide-accurate spoken script

> Bracketed `[…]` are stage directions, not spoken. Bold **beats** are the
> lines to land cleanly even if you improvise around them.

### Slide 1 — *Worst-Case Optimal Datalog (++)*  ·  ~0:50

"Thanks. This is work by Frank McSherry — his framework, `datatoad` — and I'm
presenting it here.

**One caveat up front: I'm a systems engineer, not a theoretician.** Frank built
this and derived the bounds. I can speak to how it works, why it's fast, and
what it's like to use — for the deep questions about the bounds themselves I'll
take names and connect you with Frank. Looking around this room, I suspect
several of you could derive them faster than either of us.

[Say this once. Do not re-apologize at each theory slide.]

One sentence on why the title is worth your next fifteen minutes. It's easy to
hear 'worst-case optimal' and think: the join algorithm. **The claim here is
bigger — the *whole* Datalog computation, the full semi-naive fixpoint, comes
with a worst-case-optimal bound.**

Why that's hard, and where it comes from, is the middle of the talk."

[**State the claim, withhold the mechanism.** Do *not* explain non-composition
or name Ammar here — both are payoffs with their own slides (7 and 8), and
spending them now defuses those slides and buys nothing: the audience can't
evaluate "doesn't compose" before they've seen the delta sum on screen.]

[advance]

### Slide 2 — Context: the `datatoad` framework  ·  ~0:45

"Concretely: `datatoad` is a Datalog — a *DatalogZ*, with integers — that is
columnar in both its data layout and its computation, does worst-case optimal
*incremental* joins following Ammar et al., and supports predicates that run in
many directions — bi-, many-, omni-directional.

It's also interactive, distributes, and runs in low memory — that last row
matters if you build systems, less so for the theory, so I'll keep moving.
**The first three properties are the consequential ones for this room.**"

[advance]

### Slide 3 — `datatoad`, in the browser (static)  ·  ~0:45

[Static slide. Nothing to click, nothing to load. Just talk over it.]

"And it's a real, public system — the whole engine compiles to WebAssembly and
runs client-side in a browser.

Here's the example I'd point you at. That graph is built by three rules — note
`:range` and `:plus` doing the building, we'll come back to those. Then the
triangle query. **The graph is constructed so that *no* pair-at-a-time join
order avoids about a trillion intermediate results.**

And these numbers are **from the browser** — single-threaded WebAssembly, in a
tab. About a tenth of a second to load three million facts, about a second to
enumerate every triangle. **The same query, in PostgreSQL, natively, was still
running when we gave up on it.**

The URL's there — it runs on your laptop, so please do try it."

[Do NOT open the browser here. The backup live slide is at the very end.]

### Slide 4 — Talk outline  ·  ~0:25

"Three parts. A WCO join algorithm, done columnar. Then how the WCO
*bound* extends to indexes, streaming, and iteration — that's the theorem. Then
the fun part: columnar WCOJ turns out to be a nice *interface* to relations —
disjunctions, directional predicates, foreign functions.

**Part two is the load-bearing one.**"

[advance]

### Slide 5 — Columnar WCO Joins (1/2)  ·  ~1:15

"So, the algorithm. A Datalog rule is a multi-way join — a head, and a body of
atoms sharing variables, or 'terms.' This is an instance of Generic Join, in the
NPRR lineage.

The idea: build up assignments that satisfy the body, over growing subsets of
the terms. Start with the empty assignment — trivially satisfying if every atom
is non-empty. Then repeatedly add one new term and update the surviving
assignments. Project down to the head terms — at the end, or as you go.

**The one thing that's different from the textbook: we do this breadth-first —
all assignments advancing one column at a time — not the usual depth-first,
tuple-at-a-time recursion.** That columnar choice will pay off later."

[advance]

### Slide 6 — Columnar WCO Joins (2/2)  ·  ~1:15

"Here's the mechanism — the part to remember. To extend an assignment by a new
term `Ti`, every atom that mentions `Ti` does three things.

First, it **counts** — for each assignment so far, how many compatible values of
`Ti` it could offer. Second, the atom with the *smallest* count gets to
**extend** — it proposes its values. Everyone else just **validates** the
extended assignments.

**That's the whole optimality trick: always propose from the smallest set,
intersect with the rest.** Count, extend, validate. Up to here nothing is novel
— this is just what a WCO join *is*. But hold onto that three-verb protocol; the
back half of the talk is about who else can implement it."

[advance]

### Slide 7 — Streaming WCO Joins (1/2)  ·  ~1:20

"Now the twist. In Datalog, the input atoms don't sit still — they *change*.
Every iteration derives new facts, so each body atom gets a delta: `dBody0`,
`dBody1`, and so on.

You could restart from scratch each round, but you don't have to — you can
compute just the change to the head. And that's this telescoped sum: `dBody0`
against everyone else updated, plus `dBody1` against everyone else, and so on —
one term per body atom.

**And here's the trap — this is the thing I said was hard.** If you run each of
those lines as an independent WCO join, each one is a tiny delta joined against
big full relations, and WCOJ happily gives you permission to touch all of the
big side. Do that every iteration and the costs don't telescope to anything
good. **This is what people mean when they say worst-case optimal joins don't
compose: optimal per join does not add up to optimal overall.**

The fix is to make each a *seeded* join against maintained indexes, with a
constrained term order — and to be careful, with multiset semantics, not to
count the same delta twice."

[This is where non-composition lands, with the sum visible above it. Don't
pre-empt it on slide 1.]

[advance]

### Slide 8 — Streaming WCO Joins (2/2) → the theorem  ·  ~1:30

"And here's the payoff.

**Theorem: for append-only changes, the WCO bound applies using the *final* atom
sizes.** That's Ammar et al.'s streaming result.

Now — semi-naive Datalog only ever *inserts* facts. So it *is* the append-only
setting. Which gives the corollary I actually care about:

**For each Datalog rule, the WCO bound applies using the final atom sizes — and
so the whole semi-naive computation is bounded by evaluating each rule once, at
its final size.**

That's the sentence from the title. It's a reframing of Ammar — 'streaming WCO
joins' becomes 'WCO Datalog' — but the reframing is the point: you get an
end-to-end bound, for free, that the joins alone never gave you.

[**Acknowledge the adjacent literature here — several of its authors are in the
room. See "Audience intel" in Part 1.**] And I should say: this sits right next
to the dynamic-query-evaluation and incremental-view-maintenance work several of
you have done — insert-only versus insert-delete, triangles under updates. I'd
genuinely like to know how our bound lines up with yours. That's a conversation
I want to have, more than a claim I want to make."

[advance]

### Slide 9 — Columnar WCO Joins: An API  ·  ~0:45

"Second half. Having gone all-in on columnar, the join machinery has a small
interface. A type that can participate in a WCO join implements `ExecAtom`: give
me your `facts` — recent or all; `count` how many extensions you offer for each
fact; and `join` — extend or semijoin the facts by a new term.

**That's it. `facts`, `count`, `join`. Those are the three verbs from before, as
a trait.** So the natural question is: who implements this?"

[advance]

### Slide 10 — What implements `ExecAtom`? (stored)  ·  ~0:20

"Stored relations, obviously — that's what the whole thing was built for. What
else?"

[advance]

### Slide 11 — What implements `ExecAtom`? (antijoins … wait)  ·  ~0:45

"Antijoins — 'give me the `x` that are *not* in `R`' — and... wait. **An
antijoin can *validate* a value — is this `x` absent from `R`? sure. But it can't
*propose* values — the set of things not in `R` is infinite — and it can never
be the smallest-count proposer.**

So `ExecAtom` alone isn't enough. The interface can't tell that this atom is a
validator, not a producer. We need to say something about *direction*."

[advance]

### Slide 12 — Columnar WCO Joins: Another API  ·  ~0:45

"So there's a second, *planning* interface: `PlanAtom`. It says which `terms` an
atom references, and — the important method — `modes`: given the terms already
made concrete, which terms can *this* atom make concrete, and with what
cardinality bounds.

**Those bounds are exactly Mercury's determinism levels** — 'this produces at
most one,' 'this produces many,' 'this only tests.' An antijoin declares 'I can
only test' — problem solved. Now the planner knows never to ask it to propose."

[advance]

### Slide 13 — What implements `PlanAtom + ExecAtom`? (+ disjunctions)  ·  ~0:30

"With planning in hand: stored relations, antijoins — now legal — and
disjunctions, 'or' in the body, which produce a union of extensions. Same two
traits. What else fits?"

[advance]

### Slide 14 — What implements `PlanAtom + ExecAtom`? (+ logic)  ·  ~0:20

"**Logic. Relations backed by *code* rather than stored data.** This is the one
I want to spend a minute on."

[advance]

### Slide 15 — Logic Atoms  ·  ~1:00

"Take `:plus` — the relation of triples where `a + b = c`. Ask yourself: is that
a predicate, a function, or a relation?

It's all three, depending on what's bound. As a **predicate**, given `a`, `b`,
`c`, it says present or absent. As a **function**, given `a` and `b`, it
produces `c`. As a **relation**, given *any two*, it produces the third.

**And that's precisely what `PlanAtom` plus `ExecAtom` asks for** — `modes` says
'give me any two terms and I'll make the third concrete, exactly one of them.'
So `:plus` is omni-directional, and it plugs straight into the count protocol."

[advance]

### Slide 16 — `:plus` in action  ·  ~1:00

"Why do we care? Triangles. `tri1` is the plain triangle query. `tri2` is the
same query plus `:plus(a, b, c)` — only keep triangles whose labels happen to
sum.

`tri2` obviously has far fewer answers. The question is whether we *notice in
time* — or whether we enumerate a billion triangles first and filter afterward.

[let the table land] **On a thousand nodes: `tri1` produces a billion outputs in
117 seconds. `tri2` — 196 milliseconds.** Because `:plus` participates in the
count, the join proposes `c` *from* `:plus` and never materializes the triangles
that couldn't sum. An omni-directional logic atom didn't just filter — it
*changed the plan*. That's the whole pitch in one table."

[advance]

### Slide 17 — A more-WCO example (sensors)  ·  ~1:00  · **KEEP**

"Same trick as `:plus`, but sharper — and this one is really about skew.

A few sensors are noisy: those have *every* reading, but we only ask a narrow
window of them. Most sensors are quiet: one reading each, but we ask a wide
window. Note the inversion — the big data has the small query, and the small
data has the big query.

So who should propose the reading? For a noisy sensor, the *range* is the cheap
way in: 256 candidates instead of 65,000. For a quiet sensor, the *data* is
cheap: one reading, where the range would hand you 65,000. **The cheapest
proposer flips per sensor — and the count protocol re-decides it for every one
of them.**

That's the point: there's no good static plan here. Always filtering by range
after the fact costs you about 130×. Always proposing from the range costs you
about 30,000×. Only deciding per tuple gets you the output size."

[**The numbers, if pushed.** Noisy: 256 sensors × 65,536 readings = 16.8M facts,
queried over [0,256). Quiet: 65,280 sensors × 1 reading, queried over [0,65536).
Output = 130,816. Proposals: adaptive 130,816 (= output); always-`data`
16,842,496 (~129×); always-`:range` 4,278,255,616 (~32,700×).]

[Why this earns its slot in *this* room: it demonstrates per-tuple plan
adaptivity under skew, which is what worst-case optimality is for. `:plus` shows
a logic atom cutting work; only this slide shows the proposer *changing*.]

[advance]

### Slide 18 — Connecting to relational programming  ·  ~0:40

"This isn't out of nowhere. Languages have supported omni-directional relations
for years — CLP and Prolog with constraint solvers, Mercury with compile-time
modes, the Kanren family with runtime unification and search.

**What might be new here is the *mechanism*: the WCO join count protocol as the
thing that makes it go** — relational programming meeting bottom-up Datalog at
worst-case-optimal throughput."

[advance]

### Slide 19 — What implements `PlanAtom + ExecAtom`? (+ FFI)  ·  ~0:20

"And the last one: extensibility — foreign functions. Anything that can count,
propose, and validate can be an atom, including code you bring from outside.
Same two traits, no new engine machinery."

[advance]

### Slide 20 — Benchmark table  ·  ~0:40

"To show it's a real system, not just a bound: this is the standard Datalog
analysis suite — GALEN, transitive closure, same-generation, points-to,
cvc5, z3, and so on — across parallel configurations, one-by-one up to
four-by-four. **It scales, and the end-to-end speedups are real** — up to seven-
odd times. So the theory and the engine agree.

[transition into Closing:] Which brings me to what I think this all *means*..."

[advance]

### Slide 21 — Closing  ·  ~0:45

"So — two takeaways.

First: **worst-case optimal *Datalog* is a genuinely different thing from a
worst-case optimal *join*.** The bound is over the whole semi-naive computation,
and you get it by seeing the fixpoint as insertion-only — not from the joins on
their own.

Second, and maybe the more useful one: **the columnar orientation turns out to
be a uniform interface.** Stored relations, antijoins, sums and differences,
directional predicates, external data — [gesture down the list] every one of
these implements the same two traits and plugs into the same count protocol.
**Each adds capability without any new engine machinery.**

So the thing I'd leave you with: columnar WCO Datalog ends up being a
*substrate*, not just a faster engine. The speed is nice — the compositionality
is the part I think is worth arguing about. And I'd love to."

[advance]

### Slide 22 — Thanks / Questions  ·  end

"Thank you — happy to take questions."

[**Stop here.** The next slide is the backup live demo; only go there
deliberately. See "Demo: deliberately not live" above.]

---

*Cut for the 15:00 target: **Hackathon**, **Conversation starters**, and the
**DDIR language** slide. They're preserved in git history if a longer slot or a
follow-up session opens up — the conversation-starter provocations (demand
transform vs. WCOJ, Free Join, factorized DBs) are still good coffee material
even though they're off the deck.*
