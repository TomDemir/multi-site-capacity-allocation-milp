# Multi-Site Capacity Allocation

A mixed-integer programming model.

Deciding which commercial orders to accept, and at which manufacturing site to produce them,
under machine-hour capacity constraints. Formulated as an integer program and solved with
PuLP/CBC.

> **All numerical data in this repository is synthetic and illustrative.** Capacities, margins,
> order volumes and the logistics rate were generated for this modelling exercise. They are not
> the actual figures of any organisation. The only non-generated inputs are geographic: the eight
> Arts et Métiers campus cities, and great-circle distances computed from real coordinates.

> **This is a personal modelling project.** It is inspired by my role as Business Manager leading
> the Commercial Offers workstream at AMJE (Arts et Métiers Junior-Entreprise), but it is not an
> official mandate, not a requested deliverable, and not a tool deployed anywhere.

---

## What this project does

It formulates an integer program, solves it, and then spends most of its length working out what
the solution is worth as an answer. Producing an optimal number is the easy part. The sections on
the linear relaxation, and on what shadow prices mean once integrality is imposed, are where the
work actually is.

The motivation was to close a specific gap in my own toolkit: my prior technical work is built on
machine learning, Monte Carlo simulation and heuristic decision systems, none of which involves
classical mathematical programming.

## The problem

A manufacturing network pools the capacity of eight sites across several process types: CNC
turning, CNC milling on polymers, conventional machining, sheet metal work, FDM additive
manufacturing and electrical discharge machining. Commercial orders arrive; each carries a gross
margin, consumes machine-hours on one or more processes, and originates from a client somewhere
in France.

Two decisions are made simultaneously:

1. **Which orders to accept.** Capacity is finite.
2. **Which site produces each accepted order.** The client never specifies this.

Two real constraints govern the second decision:

- **Machine compatibility, a hard constraint.** A site can only take an order if it owns every
  process the order requires. This includes an *exclusion perimeter*: two processes exist nowhere
  in the network, so orders requiring them cannot be served at all.
- **Geographic proximity, a cost rather than a constraint.** A distant site is not forbidden; it
  is more expensive. Modelling this as a cost term rather than a hard rule was a deliberate choice,
  and much of what the model can say afterwards depends on it.

## Results

Synthetic instance: 28 orders, 8 sites, 8 process types → 163 binary variables, 63 constraints.

| | |
|---|---|
| MILP optimum | €40,628.64 |
| LP relaxation bound | €45,158.07 |
| Integrality gap | 11.15% |
| Orders accepted / refused | 24 / 4 |
| Binding capacity constraints | 9 |
| Non-zero shadow prices | 13 |

All results reproduced identically on two independent solvers, CBC via PuLP and HiGHS via SciPy.
That includes the ranking of dual values, which could legitimately have diverged given the
degeneracy present in this instance.

### Four findings

**The optimum routes around congestion, and pays for it.** Thirteen of the twenty-four accepted
orders are *not* sent to their nearest compatible site, at a total logistics premium of €3,616.
Those thirteen assignments are what a greedy nearest-site rule gets wrong: it routes them to a
saturated site and then has nowhere to put the orders that follow. Run on this instance, the best
of three greedy orderings returns €38,762.90 against the model's €40,628.64.

**Refusal has two separable causes.** Three orders are refused for *capability*: no site owns the
required process, and they represent €13,100 of structurally inaccessible margin. One is refused
for *capacity*: it could be produced, but the hours are worth more elsewhere. These
demand opposite managerial responses: investment or subcontracting in one case, pricing and
arbitration in the other. Only the model separates them cleanly.

**Stranded capacity is invisible to utilisation metrics.** The refused order needs 14 hours of
electrical discharge machining. The network still holds 12 idle EDM hours, but they sit at Cluny,
two short of what an indivisible order requires, while Angers (14/14) and Metz (16/16) are
saturated. Capacity that exists, is idle, and is unusable. No utilisation percentage can show
this.

**Shadow prices do not survive the integrality constraint intact.** The dual on the scarcest
resource reads €185.26 per machine-hour. Adding capacity there and re-solving the *integer*
problem:

| hours added | realised gain | dual prediction | ratio |
|---|---|---|---|
| 1 | €0.00 | €185.26 | 0.00 |
| 4 | €0.00 | €741.04 | 0.00 |
| 12 | €0.00 | €2,223.12 | 0.00 |
| 14 | €2,326.40 | €2,593.64 | 0.90 |
| 20 | €2,326.40 | €3,705.20 | 0.63 |

The first twelve hours are worth exactly nothing. Value appears only at the 14-hour threshold
matching the indivisible requirement of the refused order, then plateaus while the linear
prediction keeps growing.

## Formulation

**Sets.** Orders $I$, sites $C$, processes $P$.

**Parameters.** $m_i$ gross margin of order $i$; $h_{i,p}$ machine-hours of process $p$ required
by order $i$; $K_{c,p}$ hours of process $p$ available at site $c$; $d_{i,c}$ client-to-site
distance; $\tau$ logistics cost per kilometre.

**Compatibility set.** $\mathcal{A} = \lbrace (i,c) : h_{i,p} > 0 \Rightarrow K_{c,p} > 0 \text{ for all } p \rbrace$.
Incompatible assignments get no variable at all, rather than being forbidden by a constraint.

**Decision variable.** $x_{i,c} \in \lbrace 0,1 \rbrace$ for $(i,c) \in \mathcal{A}$, meaning order $i$ is
accepted and assigned to site $c$.

**Objective.**

$$\max \sum_{(i,c) \in \mathcal{A}} (m_i - \tau d_{i,c}) x_{i,c}$$

Both terms are in euros. This matters: if distance were penalised in kilometres or in an
arbitrary score, the shadow price "euros per machine-hour" would lose its dimensional meaning.

**Constraints.**

$$\sum_{c} x_{i,c} \le 1 \quad \text{for all } i$$

$$\sum_{i} h_{i,p} x_{i,c} \le K_{c,p} \quad \text{for all } c, p$$

The first is an inequality, not an equality. Writing $= 1$ would force every order to be
accepted, making the problem infeasible as soon as capacity is short. Worse, it would remove the
model's ability to express a refusal. Refusing an order is a decision, not a failure.

This is a **generalised assignment problem**. The class is NP-hard, but that describes worst
cases as the problem grows and settles nothing about an instance of 28 orders, which solves in
well under a second. The case for the model is therefore made on the instance instead: the
notebook runs the greedy nearest-site rule the model replaces, and the model beats the best of
three greedy orderings by €1,865.73, or 4.81%, while placing 24 orders against the rule's 23.

### A note on the name

Every variable in the **nominal** model is binary; strictly speaking that model is a *binary
integer program*, of which mixed-integer programming is the more general family. The robust
counterpart of the final section is a genuine mixed-integer program: its Bertsimas-Sim
reformulation introduces continuous dual variables alongside the binary assignment variables.

## On duality

The scarcity information in this model, meaning where an extra machine-hour is worth most, is the
part a manager would act on. The linear relaxation is the only practicable way to get at it. Integer programs do not hand you shadow prices: strong duality does not survive integrality
constraints. The feasible region is a set of isolated lattice points rather than a polyhedron,
there is no *linear* dual program whose optimum matches it, and PuLP returns `None` for `.pi` on
a problem solved as an integer program.

So the standard route, used here, is to solve the relaxation and read its duals. Concretely, the
shadow prices are read from a model rebuilt with continuous variables in $[0,1]$. They are not
read from the integer model, and not obtained by fixing the binaries at their optimal values and
re-solving, which would price the marginal hour conditional on a fixed assignment and answer a
different question.
That route works, and the resulting ranking carries real information. But it comes at a cost
this project states rather than glosses over:

- The duals price the marginal hour *in a world where orders can be split fractionally across
  sites*, which is not the world the orders live in.
- They are valid only locally; push the right-hand side far enough and the basis changes.
- The integrality gap bounds how far they can be off.
- They are not necessarily unique. This instance is degenerate, so the ranking is read as a
  directional indication of where scarcity concentrates, not as a certified price vector.

The right use is therefore directional, and the fourth finding above tests that empirically
rather than assuming it.

## Limitations

Every order is produced entirely on one site. That is assumed rather than derived: a binary
$x_{i,c}$ assigned at most once cannot split an order between a site doing the turning and
another doing the sheet metal, which is how real multi-process work often runs. It changes
answers: the capacity-refused order needs 14 EDM hours while 12 sit idle elsewhere, and a
splittable model might well place it.

Distance is a one-way great-circle approximation, not road distance and not a return trip, and
the rate applied to it is a figure chosen for the exercise rather than quoted from anywhere.

No time dimension: capacity is a single stock of hours over one horizon, with no release dates,
due dates or sequencing. No setup times: machine-hours are consumed linearly. No uncertainty:
margins and hour requirements are point estimates. Deterministic and single-shot: all orders are
known at once, where reality is a rolling arrival process.

The formulation is the transferable part of this work. None of the numerical results describe any
real organisation.

## Running it

```bash
git clone <this-repo>
cd <this-repo>
pip install pulp pandas numpy
jupyter notebook capacity_allocation.ipynb
```

PuLP ships with the CBC solver, so no separate solver installation is needed. The notebook runs
top to bottom in about a minute and is self-contained, with all data generated inline.

---

Tom Demir, September 2026
