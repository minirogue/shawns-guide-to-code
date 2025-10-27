---
tags:
  - Architecture
---
# Dependency Graphs
When architecting an application, it can be helpful to consider your dependency graph.
This page outlines the general concepts of dependency graphs with the intent of defining terms, concepts, and properties that can be used to better reason about your dependency graph.
If you manage your dependency graph carefully, you can ensure a cleaner codebase as well as take advantage of build time optimizations like parallel module builds, compile avoidance, etc.
This page is intended as a primer for future pages about how to consider your dependency graph when architecting a codebase.
## Assumed Prior Knowledge
To understand this page, it is expected that you know

- Some basic graph theory definitions:
	- What a [graph](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)) is,
	- what a [directed graph](https://en.wikipedia.org/wiki/Directed_graph) is, and
	- what a [directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) is (which would include the definition of a [cycle](https://en.wikipedia.org/wiki/Cycle_(graph_theory))).

Reading through the linked wikipedia pages should sufficient, assuming I've written the rest of this page well enough.
Understanding other graph theory definitions and concepts (especially those specific to directed acyclic graphs) will be helpful, but I'll try to explain them where necessary.

## Assumptions of This Page

Here are the base assumptions I'll be making

- I'll be assuming that we are using a compiled language.
- A module cannot compile unless all the modules it depends on have compiled.
- The number of modules is finite[^1].

## What is a Dependency Graph

In the broadest sense, a dependency graph is a mapping of how areas of your code are interconnected.
To better define a dependency graph, we'll first need to define what we mean by "areas of your code", which we'll call "modules".
Let's define a <span class="define">module</span> as a collection of code that compiles together into a well-defined component.
Gradle modules should directly align with the content here, but in general (perhaps with some small alterations in assumptions) a module could be a file, a class, a package, etc.

A <span class="define">dependency</span> in our code occurs when one module must "know" about the details of another module.
If module `A` must "know" about the details of module `B`, then we say that `A` <span class="define">depends on</span> `B` and that `B` is a <span class="define">dependency of</span> `A`.
Often these dependencies will be explicit, such as in the `dependencies` block of a `build.gradle` file.

The <span class="define">dependency graph</span> of a collection of modules is the [directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) where the nodes represent the modules, and an edge is drawn from node `A` to node `B` if and only if `A` depends on `B`.
Here's a toy example that will be used throughout the rest of this page:

<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App --> B[Feature B]
	A --> C[Library C]
	A --> D[Library D]
	B --> D
	B --> Util[Library E]
	C --> Util
	D --> Util
```
</figure>

In this dependency graph `Application Module` depends on the modules `Feature A` and `Feature B`, `Feature B` depends on `Library D` and `Library E`, `Library E` has no dependencies, etc.

### Why is the Dependency Graph Acyclic?

The directedness of our dependency graphs arises naturally from the nature of dependencies: if module `A` depends on module `B`, then `A` needs to know about details of `B`, but `B` need not know that `A` even exists.
However, the necessity of the graph being acyclic may be less obvious.

To explore what it would mean to allow cycles in our dependency graph, let's first consider 1-cycles.
A 1-cycle would just mean that a module is dependent on itself.
We could argue that the module already inherently dependent on itself in a way (the module must "know" about its own code).
However, if we tried to explicitly declare a dependency on itself, then by our assumptions, we need it to compile before it can compile, which is clearly nonsense.
<figure>
```mermaid
graph LR
	A --> A
```
</figure>

Expanding to larger cycles, consider two modules, `A` and `B`, which both depend on each other, forming a 2-cycle:
<figure>
```mermaid
graph LR
	A --> B
	B --> A
```
</figure>
Suppose you attempt to compile module `A`.
Since module `A` depends on module `B`, we need to compile module `B` first.
But module `B` depends on `A`, so to compile `B` we need to first compile `A`.
So, which module actually gets compiled first?
There's no valid answer, so we cannot begin the compilation.

For any cycle larger than 2, we can follow the same logic to find that there is no module in the cycle that can be compiled first.
Thus, we cannot allow a cycle of any size in a dependency graph.
<figure>
```mermaid
graph LR
	A --> B
	B --> C
	C --> A
	D --> E
	E --> F
	F --> G
	G --> H
	H --> D
```
</figure>

## Useful properties of Directed Acyclic Graphs (DAGs)

Since dependency graphs are directed acyclic graphs (DAGs), it is helpful to define some terms and concepts from graph theory that will help us.

### Walks/Paths

A <span class="define">walk</span> from node `A` to node `B` along a directed graph is a sequence of edges where the first edge starts at `A`, each subsequent edge starts at the node where the previous edge ended, and the final edge ends at `B`.
One can conceptually think of this as literally standing on node `A` and walking along the directed edges from node to node until they reach node `B`.
In a DAG, all walks are also <span class="define">paths</span>, which have the additional feature that no node (or edge, for that matter) will be visited more than once.
The <span class="define">length</span> of a walk/path is the number of edges in that walk/path

<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App e1@==> B[Feature B]
	A --> C[Library C]
	A --> D[Library D]
	B e2@==> D
	B --> Util[Library E]
	C --> Util
	D --> Util
	style App stroke:#333,stroke-width:4px
	style B stroke:#333,stroke-width:4px
	style D stroke:#333,stroke-width:4px
	e1@{ animate: true }
	e2@{ animate: true }
```
<figcaption>A walk/path of length 2 from Application Module to Library D.</figcaption>
</figure>
### Height

The <span class="define">height</span> of a node in a directed acyclic graph is the maximum length of all paths that start at that node.
In a dependency graph, the height of a module is the largest number of modules that must compile in serial before that module can compile. 
The <span class="define">height</span> of a directed acyclic graph is the maximum height among the nodes in that graph.
This measure has build-time implications in a system that is capable of parallel compilation.

<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App --> B[Feature B]
	A --> C[Library C]
	A --> D[Library D]
	B --> D
	B --> Util[Library E]
	C --> Util
	D --> Util
```
<figcaption>The height of Feature B is 2. The height of the Application Module is 3. 
There are no nodes with height larger than 3, so the height of this dependency graph is 3.</figcaption>
</figure>
### Reachability
A node $B$ is <span class="define">reachable</span> from a node $A$ if there exists a path from $A$ to $B$.
In this case, we can apply the notation $A\leq B$ as a [partial ordering](https://en.wikipedia.org/wiki/Partially_ordered_set).
Note that reachability is transitive, i.e. $A\leq B$ and $B\leq C$ implies $A \leq C$ (just concatenate the path from $A$ to $B$ and the path from $B$ to $C$ to get a path from $A$ to $C$).
An important feature of reachability in a dependency graph is that if $A \leq B$, then $B$ must compile before $A$.

<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App e1@==> B[Feature B]
	A --> C[Library C]
	A --> D[Library D]
	B e2@==> D
	B --> Util[Library E]
	C --> Util
	D --> Util
	style App stroke:#333,stroke-width:4px
	style B stroke:#333,stroke-width:4px
	style D stroke:#333,stroke-width:4px
	e1@{ animate: true }
	e2@{ animate: true }
```
<figcaption>There is a path from Application Module to Library D, but no path from Library C to Library D.
So, Library D is reachable from the Application Module, but not reachable from Library C.</figcaption>
</figure>
### Degree and Reach

The <span class="define">in-degree</span> of a node in a directed graph is the number of edges that point directly to that node.
In a dependency graph, the in-degree is the number of modules that directly depend on the given module.
<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App --> B[Feature B]
	A --> C[Library C]
	A e2@==> D[Library D]
	B e1@==> D
	B --> Util[Library E]
	C --> Util
	D --> Util
	style D stroke:#333,stroke-width:4px
    e1@{ animate: true }
    e2@{ animate: true }
```
<figcaption>Library D has in-degree 2.</figcaption>
</figure>

The <span class="define">in-reach</span>[^2] of a node is the number of nodes from which it is reachable.
In a dependency graph, the in-reach of a node is the number of modules that cannot compile until that node has compiled

<figure>
```mermaid
graph TB
	App[Application Module] ==> A[Feature A]
	App ==> B[Feature B]
	A --> C[Library C]
	A ==> D[Library D]
	B ==> D
	B --> Util[Library E]
	C --> Util
	D --> Util
	style D stroke:#333,stroke-width:4px
	style A fill:#f9f,stroke:#333,stroke-width:4px
	style B fill:#f9f,stroke:#333,stroke-width:4px
	style App fill:#f9f,stroke:#333,stroke-width:4px
```
<figcaption>The Library D module has an in-reach of 3.</figcaption>
</figure>

The <span class="define">out-degree</span> of a node in a directed graph is the number of edges that point away from that node.
The out-degree is the number of modules on which the given module directly depends.
<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App --> B[Feature B]
	A --> C[Library C]
	A --> D[Library D]
	B --> D
	B --> Util[Library E]
	C --> Util
	D e1@==> Util
	style D stroke:#333,stroke-width:4px
    e1@{ animate: true }
```
<figcaption>Library D has out-degree 1.</figcaption>
</figure>

The <span class="define">out-reach</span>[^2] of a node is the number of nodes that are reachable from it.
In a dependency graph, the out-reach of a node is the number of modules that must compile before that node can compile.
<figure>
```mermaid
graph TB
	App[Application Module] --> A[Feature A]
	App --> B[Feature B]
	A ==> C[Library C]
	A ==> D[Library D]
	B --> D
	B --> Util[Library E]
	C ==> Util
	D ==> Util
	style A stroke:#333,stroke-width:4px
	style C fill:#f9f,stroke:#333,stroke-width:4px
	style D fill:#f9f,stroke:#333,stroke-width:4px
	style Util fill:#f9f,stroke:#333,stroke-width:4px
```
<figcaption>The Feature A module has out-reach of 3.</figcaption>
</figure>

[^1]: Please don't think too hard about the implications of infinite modules. My background is in mathematics, so including this assumption is more of an academic compulsion than anything... Although, I admit that I am now thinking about the horrible horrible implications of infinite modules.

[^2]: Both "in-reach" and "out-reach" are my own definitions, as I couldn't find any pre-existing definitions for these values. 
If you know of established names that define these values, please contact me, so I can update this page.
