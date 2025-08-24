---
title: Visualizing Regular Expression NFAs in Lean
date: 2025-08-23 12:00:00 +0530
categories: [Visualization]
tags: [lean, regex, penrose, proofwidgets]     # TAG names should always be lowercase
---

The [lean-regex](https://github.com/pandaman64/lean-regex) project implements a
formally verified regular expression engine for the
[Lean](https://lean-lang.org/) programming language. It is authored and
maintained by [pandaman](https://github.com/pandaman64). It has a few
newcomer-friendly issues, one of which is about adding [NFA
visualization](https://github.com/pandaman64/lean-regex/issues/96). After
discussing with `pandaman` I decided to pick up this issue. Previously I had
worked on [implementing word boundary
operators](https://github.com/pandaman64/lean-regex/issues/93), and this seemed
like a nice orthogonal diversion in terms of implementation.

In the rest of this post I'll describe how I went about implementing
visualization for NFAs.

## NFAs in lean-regex

Every regular expression can be converted to an NFA (Non-deterministic finite
automaton) using standard constructions, such as [Thompson's
construction](https://en.wikipedia.org/wiki/Thompson%27s_construction).
`lean-regex` uses a similar construction where the nodes are represented as an
inductive data type with distinct constructors for each type of node. The nodes
are stored in a Lean array and each node contains the data related to the
transitions from that node.

A sample regex and its NFA representation is shown below:

```lean
def sampleRegex : Regex := re! r##"a|"##

#eval sampleRegex
-- when we put the cursor at the end of the previous line,
-- the following outpput is displayed in the Lean Infoview pane
/-
{
  nfa := {
    nodes := #[
      Regex.NFA.Node.done,
      Regex.NFA.Node.save 1 0,
      Regex.NFA.Node.char 'a' 1,
      Regex.NFA.Node.epsilon 1,
      Regex.NFA.Node.split 2 3,
      Regex.NFA.Node.save 0 4
    ],
    start := 5
  },
  wf := ⋯,
  maxTag := 1,
  optimizationInfo := { firstChar := none }
}
-/
```

We will use the `nodes` array and the `start` field to build the visualization.

## Visualization using ProofWidgets and Penrose

[ProofWidgets](https://github.com/leanprover-community/ProofWidgets4) is a
library that enables custom visualizations in the Lean InfoView pane. It has
many features and integrations but we will use its support for
[Penrose](https://penrose.cs.cmu.edu/). Penrose provides a declarative DSL for
diagramming in which we specify the types of domain objects in a Domain file,
the actual instances in a Substance file, and the drawing/styling rules in a
Style file. The Penrose engine then takes care of the layout and rendering. I
chose Penrose to avoid having to manually specifying the precise layout.

The NFA rendering for the sample regex given above is as follows: ![NFA
rendering for sample regex](/assets/sample-regex-nfa.png)

## Domain definition

An NFA consists of nodes and transitions which we can directly define in the Domain file:

```
type Node
type Transition
```

Each node is displayed at a certain position relative to other nodes. This
position is determined by its offset in the `nodes` array. We define the
constructor for the `Node` type:

```
-- offset: index of the node in the nodes array
constructor Node(String offset)
```

A `Transition` goes from a source `Node` to a target `Node`:

```
-- source: the ssource node for the transition
-- target: the target node for the transition
-- isNeighbor: True if target node is a neighbor of the source node, False otherwise
constructor Transition(Node source, Node target, Boolean isNeighbor)
```

The `isNeighbor` flag is used in the Style file to render the transitions. The
transition is drawn as a straight line between neighboring nodes, and as a
curved line between non-neighboring nodes.

We also define a few predicates that will be used in the Style file for rendering:
```
-- identifies the start node
predicate StartNode(Node)

-- identifies the done node
predicate DoneNode(Node)

-- predicate relating two successive nodes
-- this is used for drawing the nodes in the appropriate order
-- thisNode: current node
-- successorNode: the next node
predicate Successor(Node thisNode, Node successorNode)
```

## Substance definition

The Substance definition is programmatically generated from the Lean regex
representation. The Substance definition for nodes consists of the node objects,
their labels and their successor nodes.

```
Let n0 := Node("0")
Label n0 $\text{done}$
DoneNode(n0)

Let n1 := Node("1")
Label n1 $\text{save 1 0}$
Successor(n1, n0)

Let n2 := Node("2")
Label n2 $\text{char 'a' 1}$
Successor(n2, n1)

Let n3 := Node("3")
Label n3 $\text{epsilon 1}$
Successor(n3, n2)

Let n4 := Node("4")
Label n4 $\text{split 2 3}$
Successor(n4, n3)

Let n5 := Node("5")
Label n5 $\text{save 0 4}$
Successor(n5, n4)
StartNode(n5)
```

The Substance definition for transitions consists of the transition objects and
their labels.

```
Let t_1_0 := Transition(n1, n0, TRUE)
Label t_1_0 $\epsilon$

Let t_2_1 := Transition(n2, n1, TRUE)
Label t_2_1 $\text{a}$

Let t_3_1 := Transition(n3, n1, FALSE)
Label t_3_1 $\epsilon$

Let t_4_2 := Transition(n4, n2, FALSE)
Label t_4_2 $\epsilon$

Let t_4_3 := Transition(n4, n3, TRUE)
Label t_4_3 $\epsilon$

Let t_5_4 := Transition(n5, n4, TRUE)
Label t_5_4 $\epsilon$
```

## Style definition

The Style file is where we specify all the rendering rules. For example, we want
the nodes to be rendered as circles and the transitions to be rendered as lines
with arrows. This is where most of the complexity and flexibility of Penrose can
be seen. I will only touch upon the style rules used for our purpose here but
you should definitely check out their
[examples](https://penrose.cs.cmu.edu/examples) to get an idea of what is
possible with Penrose.

We draw the node as a circle:

```
-- draw each node as a circle
forall Node n
{
  -- draw the node as a shaded circle
  n.shape = Circle {
    r: globals.radius -- radius
    fillColor: colors.lightGray
    strokeColor: colors.mediumGray
    strokeWidth: 1
  }

  n.x = n.shape.center[0]
  n.y = n.shape.center[1]

  -- draw the node label inside the node
  n.labelText = Equation {
     string: n.label
     center: n.shape.center
     fontSize: "10px"
     fillColor: colors.darkGray
  }
}
```

We decorate the `done` node with a circular border:

```
-- draw a circle around the done node
forall Node node
where DoneNode (node)
{
  shape doneMarker = Circle {
    center : node.shape.center
    r : 1.5 * node.shape.r
    fillColor : colors.paleGray
  }
}
```

We align all the nodes horizontally:

```
forall Node u; Node v
where Successor (u, v)
{
  -- ensure all node have their centers with same y coordinate
  ensure equal(u.y, v.y)

  --  ensure successive nodes are separated by 100
  ensure equal(v.x - u.x, globals.nodeSeparation)
}
```

We draw the transitions for neighboring nodes as straight lines:

```
-- draw transitions for neighboring nodes
forall Node u; Node v; Transition t; True b
where t := Transition(u, v, b);
{
  t.shape = Line {
    start: (u.x + u.shape.r, u.y)
    end: (v.x - v.shape.r, v.y)
    endArrowhead: "straight"
  }

  -- draw the label on the transition arrow
  t.text = Equation {
    string: t.label
    center: ((t.shape.start[0] + (t.shape.end[0] - t.shape.start[0])*.5) , (t.shape.start[1] + 7))
    fontSize: "10px"
    fillColor: colors.darkGray
  }
}
```

We draw the transitions for non-neighboring nodes as curved lines. The curvature
is proportional to the distance between the nodes. Penrose provides functions
for drawing curved lines, such as
[interpolateQuadraticFromPoints](https://penrose.cs.cmu.edu/docs/ref/style/functions#computation-interpolateQuadraticFromPoints)

```
-- draw transitions for non neighboring nodes
forall Node u; Node v; Transition t; False b
where t := Transition(u, v, b);
{
  -- draw an arrow from u to v
  vec2 x0 = u.shape.center
  vec2 x2 = v.shape.center
  scalar distance = norm(x2 - x0)
  vec2 w = (x2-x0)/distance -- unit vector from u to v
  vec2 n = rot90(w) -- unit normal
  -- calculate p0, p1, p2 for quadratic interpolated curve
  -- the coefficients are derived by trial and error, looking at the results
  -- to see what looks like a good curve.
  vec2 p0 = x0
  vec2 p2 = x2 - 30*w + 10*n
  vec2 mid = (p0+p2)/2
  vec2 p1 = mid + distance * 0.15 * n

  shape t.shape = Path {
    d: interpolateQuadraticFromPoints("open", p0, p1, p2)
    endArrowhead: "straight"
  }

  layer t.shape below u.shape

  -- draw the label on the transition arrow
  t.text = Equation {
    string: t.label
    center: (p1[0], (p1[1] + 7))
    fontSize: "10px"
    fillColor: colors.darkGray
  }
}
```

Note that the layout rules in the Style definition are declarative in nature and
the syntax is somewhat similar to CSS.

## Tying it all together in Lean

We just need a few lines of Lean code to generate the NFA diagram:

```lean
/-- Generate a Penrose diagram for the NFA representation of a regex. -/
def mkNFADiagram (regex : Regex) : MetaM Html :=
  return <PenroseDiagram
      embeds={#[]}
      dsl={include_str "NFAPenrose.domain"}
      sty={include_str "NFAPenrose.style"}
      sub={nfaToSubstance regex} />
```

The `nfaToSubstance` function is responsible for generating the Substance definition:

```lean
/--
 Converts an NFA representation of a regex into a Penrose substance string.
 It generates nodes, transitions, and labels for the NFA. The nodes are created
 based on the NFA's structure, and transitions are defined based on the connections
 between nodes. Labels are applied to nodes and transitions for clarity.
 The function also handles special cases like done nodes, save nodes, character
 transitions, epsilon transitions, split nodes, and anchor nodes.
 -/

def nfaToSubstance (regex : Regex) : String := Id.run do
  let mut sub := r###"
Let TRUE := True()
Let FALSE := False()
"###
  for i in [0:regex.nfa.nodes.size] do
    sub := sub ++ createNode i
    if i > 0 then sub := sub ++ relateSuccessorNodes i

  sub := sub ++ declareStartNode regex.nfa.start

  for h : i in [0:regex.nfa.nodes.size] do
    let node := regex.nfa.nodes[i]
    match node with
    | .done =>
        sub := sub ++ labelNode i "done"
        sub := sub ++ declareDoneNode i
    | .save pos next =>
        sub := sub ++ labelNode i s!"save {pos} {next}"
        sub := sub ++ createTransition i next epsilon
    | .char ch next =>
        sub := sub ++ labelNode i s!"char '{ch}' {next}"
        sub := sub ++ createTransition i next s!"{ch}"
    | .epsilon next =>
        sub := sub ++ labelNode i s!"epsilon {next}"
        sub := sub ++ createTransition i next epsilon
    | .split next₁ next₂ =>
        sub := sub ++ labelNode i s!"split {next₁} {next₂}"
        sub := sub ++ createTransition i next₁ epsilon
        sub := sub ++ createTransition i next₂ epsilon
    | .fail =>
        sub := sub ++ labelNode i "fail"
    | .anchor a next =>
        sub := sub ++ labelNode i s!"anchor {next}"
        sub := sub ++ createTransition i next s!"{Anchor.toString a}"
    | .sparse cs next =>
        sub := sub ++ labelNode i s!"sparse {next}"
        sub := sub ++ createTransition i next s!"{Classes.toString cs}"

  return sub
```

The following one liner displays the diagram in the InfoView panel:
```
-- Place your cursor at the end of the following line to see the NFA diagram
#html mkNFADiagram sampleRegex
```

## Development workflow

During development, I extensively used the Penrose [online
editor](https://penrose.cs.cmu.edu/try/). It enables quick iteration and
feedback loop. It also has a mechanism to save the trio of
domain/style/substance file versions automatically in our github workspace, so
that we can share it with others, or refer to previous versions via a URL
containing the hash of the trio. I frequently referred to the Penrose
[documentation](https://penrose.cs.cmu.edu/docs/ref) and their extensive
[examples](https://penrose.cs.cmu.edu/examples). The team also maintains an
active [Discord](https://discord.gg/a7VXJU4dfR) server where I got helpful
responses to my queries during development.

I initially created the substance definitions manually. Once I was satified with
the visualization, I generated the substance definition programmatically from
the Lean regex representation.

## Challenges and Learnings

The online Penrose editor was very handy for quick prototyping and tweaking
various domain/style/substance definitions. However, when I tried to incorporate
these definitions in `lean-regex`, it did not work. It turned out that the
version of Penrose bundled in ProofWidgets is `3.2.0` while the online editor
uses the latest code from `main` branch. There is already an existing
[issue](https://github.com/penrose/penrose/issues/1880#issuecomment-3163544953)
in Penrose repository for this. The issue would be resolved when the Penrose
team makes a new release which is then incorporated into ProofWidgets. For now,
I used a fallback to older Penrose syntax to make it work.


For integrating the Substance definition and visualization in `lean-regex`, I
referred to the ProofWidget demo code in
[Venn.lean](https://github.com/leanprover-community/ProofWidgets4/blob/main/ProofWidgets/Demos/Venn.lean),
which shows how to invoke Penrose diagrams in Lean. The complete set of changes
came to around 280 lines of code across 3 files. This just goes to show that
ProofWidgets and Penrose are doing a lot of the heavy lifting to provide users
with ergonomic APIs and abstractions for custom visualizations.

## Conclusion

Overall, this PR gave me an opportunity to improve my knowledge of Lean,
ProofWidgets and Penrose. I am still learning these, so please let me know if
you have any feedback or suggestions for improving this visualization.

Thanks to `pandaman` for reviewing the draft of this post and encouraging me to
work on this issue.


## Useful Links

- [lean-regex PR for NFA visualization](https://github.com/pandaman64/lean-regex/pull/123)
- [ProofWidgets](https://github.com/leanprover-community/ProofWidgets4)
- [Penrose](https://penrose.cs.cmu.edu/)


