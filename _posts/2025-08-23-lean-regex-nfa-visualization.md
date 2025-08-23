---
title: Visualizing Regular Expression NFAs in Lean
date: YYYY-MM-DD HH:MM:SS +0530
categories: [Visualization]
tags: [draft]     # TAG names should always be lowercase
---

The [lean-regex](https://github.com/pandaman64/lean-regex) project implements a formally verified regular expression engine
for the [Lean](https://lean-lang.org/) programming language. It is authored and maintained by [pandaman](https://github.com/pandaman64).
It has a few newcomer-friendly issues, one of which was about adding [NFA visualization](https://github.com/pandaman64/lean-regex/issues/96).
After discussing with `pandaman` I decided to pick up this issue. Prior to this issue, I had worked on
[implementing word boundary operators](https://github.com/pandaman64/lean-regex/issues/93),
and this seemed like a nice orthogonal diversion in terms of implementation.

In the rest of this post I'll describe how I went about implementing visualization for NFAs.

# NFAs in lean-regex
Every regular expression can be converted to an NFA using standard constructions, such as
[Thompson's construction](https://en.wikipedia.org/wiki/Thompson%27s_construction).
`lean-regex` uses a similar construction where the nodes are represented as an inductive data type
with distinct constructors for each type of node. The nodes are stored in a Lean array and
each node contains the data related to the transitions from that node.

A sample regex and its NFA look as follows:
```lean
def sampleRegex : Regex := re! r##"a|"##

#eval sampleRegex
-- when we put the cursor at the end of the previous line,
-- the following outpput is displayed in the Lean Infoview pane
/-
{
  nfa :=
    {
      nodes :=
        #[Regex.NFA.Node.done, Regex.NFA.Node.save 1 0, Regex.NFA.Node.char 'a' 1, Regex.NFA.Node.epsilon 1,
          Regex.NFA.Node.split 2 3, Regex.NFA.Node.save 0 4],
      start := 5 },
  wf := ⋯, maxTag := 1, optimizationInfo := { firstChar := none } }
-/
```

We are interested in the `nodes` array and the `start` field, using which we can build the visualization.

# Visualization using ProofWidgets and Penrose

[ProofWidgets](https://github.com/leanprover-community/ProofWidgets4) is a library that enables
custom visualizations in the Lean InfoView pane. It has many features and integrations but the one
we will be using is its support for [Penrose](https://penrose.cs.cmu.edu/). Penrose is a library
that provides a declarative DSL for diagramming. One needs to specify the types of domain objects
in a Domain file, the actual objects in a Substance file, and the drawing/styling rules in a Style file,
and the Penrose engine takes care of the layout and rendering.

The NFA rendering for the sample regex given above is as follows:
![NFA rendering for sample regex](/assets/sample-regex-nfa.png)

## Domain definition
In the case of our NFA, the types in our domain are Nodes and Transitions.
We define these in the Domain file as follows:
```
type Node
type Transition
```
Each node is displayed at a certain position relative to other nodes.
This position is determined by the offset of the node in the `nodes` array.
In the Domain file we define the constructor for the `Node` type as follows:
```
-- offset: index of the node in the nodes array
-- strOffset: string value of the offset
constructor Node(String offset)
```

Similarly, a `Transition` goes from a source `Node` to a target `Node`.
We define its constructor as follows:
```
-- source: the ssource node for the transition
-- target: the target node for the transition
-- isNeighbor: True if target node is a neighbor of the source node, False otherwise
constructor Transition(Node source, Node target, Boolean isNeighbor)
```
The `isNeighbor` flag is used in the Style file to determine the transition rendering
where the transition arrow is drawn as a straight line between neighboring nodes,
and as a curved line between non-neighboring nodes.

## Substance definition

For our sample regex, the Substance definition looks as follows
(it is programmatically generated from the regex representation):
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

As you can see, it is a straight forward translation from the `nodes` array representation.

## Style definition
The Style file is where all the rendering rules are specified. For example, the nodes should be
depicted circles and the transitions should be depicted as arrows. This is where the complexity
and flexibility of Penrose can be seen. I will only touch upon the style rules used for our purpose here
but you should check out their [examples](https://penrose.cs.cmu.edu/examples)
to get an idea of what is possible with Penrose.

Our style definition for nodes looks as follows:
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

forall Node u; Node v
where Successor (u, v)
{
  -- ensure all node have their centers with same y coordinate
  ensure equal(u.y, v.y)

  --  ensure successive nodes are separated by 100
  ensure equal(v.x - u.x, globals.nodeSeparation)
}
```
The style for transitions is as follows:
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
As you can see, the layout rules are declarative in nature, and the syntax is somewhat reminiscent of CSS.

# Development workflow
During development, I extensively used the Penrose [online editor](https://penrose.cs.cmu.edu/try/) which enables quick
iteration and feedback loop. It also has a mechanism to save the trio of domain/style/substance file versions automatically
in your github workspace, so that you can share it with others, or refer to previous versions via a URL which contains the
hash of the trio.

During development, I manually created the substance definitions. Once I was satified with the visualization, I then
proceeded to generate the substance definition programmatically.

# Challenges and Learnings
The online Penrose editor was extremely helpful for quick prototyping and tweaking various domain/style/substance definitions.
But when I tried to incorporate these definitions in `lean-regex`, it did not work. After some debugging, I realized that the version
of Penrose bundled in ProofWidgets is `3.2.0` while the online editor used the latest code from `main` branch. The Penrose team also
maintains an active [Discord](https://discord.gg/a7VXJU4dfR) server where I got helpful responses to my queries during development.

Once the issue was identified, the fix was straight forward. Integration of substance definition and visualization in `lean-regex`
codebase was simple enough. The complete set of changes came to around 280 lines of code across 3 files. This just goes to show ProofWidgets and Penrose are doing all the heavy lifting and providing us with ergonomic APIs and abstractions for custom visualizations.

# Useful Links
- [lean-regex PR for NFA visualization](https://github.com/pandaman64/lean-regex/pull/123)
- [ProofWidgets](https://github.com/leanprover-community/ProofWidgets4)
- [Penrose](https://penrose.cs.cmu.edu/)


