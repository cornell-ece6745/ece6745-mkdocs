
ECE 6745 Project 1: TinyFlow Tape-Out<br>TinyFlow Front-End
==========================================================================

In Project 1, you are building TinyFlow, a simple standard-cell-based
ASIC flow. In Part A, you developed standard cells with six views. In
this part (Part B), you will implement the front-end synthesis algorithms
and use the four-step testing flow to verify your designs. You will
implement substitution, canonicalization, technology mapping, and static
timing analysis. You will then run your designs through RTL simulation,
synthesis, and gate-level simulation to ensure correctness.

The project includes three parts:

 - Part A: TinyFlow Standard Cells
 - Part B: TinyFlow Front-End
 - Part C: TinyFlow Back-End

Continue working with your group from Part A. You can confirm your
group on Canvas (Click on People, then Groups, then search for your name
to find your project group).

!!! warning "All students must contribute to all parts!"

    It is not acceptable for one student to do all of Part A and a
    different student to do all of part B. It is not acceptable for one
    student to exclusively work on one algorithm while the other student
    exclusively works on a different algorithm. All students must
    contribute to all parts. The instructors will also survey the Git
    commit log on GitHub to confirm that all students are contributing
    equally. If you are using a "pair programming" style, then both
    students must take turns using their own account so both students
    have representative Git commits. Students should create commits after
    finishing each step of the project, so their contribution is clear in
    the Git commit log. **A student's whose contribution is limited as
    represented by the Git commit log will receive a significant
    deduction to their project score.**

This handout assumes that you have read and understand the course
tutorials, that you have attended the lab sections, and that you have
completed lab 3. To get started, use VS Code to log into a specific
ecelinux server, source the setup script, and navigate to your repository:

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% cd ${HOME}/ece6745/project1-groupXX
% git pull
% tree
```

where `XX` should be replaced with your group number. Your repo contains
the following files relevant to Part B:

```
.
├── stdcells
│   ├── ...
│   └── stdcells-fe.yml
└── tinyflow
    ├── conftest.py
    ├── pytest.ini
    ├── synth
    │   ├── StdCellFrontEndView.py
    │   ├── TinyFrontEndDB.py
    │   ├── TinyFrontEndGUI.py
    │   ├── canonicalize.py
    │   ├── design_check.py
    │   ├── sta.py
    │   ├── substitute.py
    │   ├── synth.py
    │   ├── techmap.py
    │   ├── tinyv.lark
    │   ├── verilog_parser.py
    │   └── tests
    │       └── ...
    └── tinyflow-synth
```

Go ahead and create a build directory where you will run the synthesis
tools and tests:

```bash
% cd ${HOME}/ece6745/project1-groupXX
% mkdir -p tinyflow/build
% cd tinyflow/build
```

1. Synthesis Overview
--------------------------------------------------------------------------

The front-end ASIC flow takes a Verilog RTL design and produces a
gate-level netlist. The flow includes simulation steps to verify
correctness at each stage:

 - **Two-State RTL Simulation:** Verify the RTL design using fast
   simulation where all signals are 0 or 1

 - **Four-State RTL Simulation:** Verify the RTL design using simulation
   where signals can be 0, 1, X, or Z to catch subtle bugs

 - **Synthesis:** Transform the RTL into a gate-level netlist using your
   standard cell library

 - **Fast-Functional Gate-Level Simulation:** Verify the synthesized
   netlist is functionally equivalent to the RTL (no timing)

Synthesis itself consists of several passes that transform the design:

 - **Verilog Parser:** Parse Verilog RTL into an internal representation
   of the design's logic

 - **Canonicalize:** Convert generic gates to a canonical NAND/INV form
   to simplify technology mapping

 - **Technology Mapping:** Map the canonical trees to standard cells
   (INVX1, NAND2X1, NOR2X1, AOI21X1, etc.) using dynamic programming
   to minimize area

 - **Static Timing Analysis:** Analyze all paths through the design to
   find the critical path and maximum delay

 - **Gate-Level Netlist Writer:** Output the final standard-cell
   gate-level netlist

2. Synthesis Data Structures
--------------------------------------------------------------------------

<!-- TODO: Discuss with team - this section may be better placed in lab 3,
     with project referencing lab for basics and focusing on new concepts
     (wildcards, constants, patterns for techmap). -->

TinyFlow represents logic using expression trees stored in a frontend
database. This section describes the key data structures you will work
with when implementing the synthesis algorithms.

### 2.1. Forest and Trees

A design is represented as a **forest** of expression trees. Each primary
output has one tree that computes its logic function. Intermediate signals
(wires) can also have trees. The `TinyFrontEndDB` class is the container
that holds the forest:

```
TinyFrontEndDB
├── _inputs:  { "a", "b", "c" }           # primary input names
├── _outputs: { "out1": <tree>, ... }     # output name -> tree
└── _wires:   { "tmp": <tree>, ... }      # wire name -> tree
```

You can access trees using `db.get_tree(name)` and modify them using
`db.set_tree(name, tree)`. The `db.trees` property returns a dict of all
trees (both outputs and wires).

### 2.2. Nodes

Each node in a tree represents a logic gate. The `Node` base class has
these key attributes:

 - **type:** Name of the gate (e.g., `"AND2"`, `"INVX1"`)
 - **children:** List of inputs, which can be other Nodes or strings
   (primary inputs / wire references)
 - **generic:** `True` for generic gates, `False` for library gates

Trees are recursive structures. A node's children are either leaf strings
(primary inputs) or other nodes (gates):

```
        NAND2                Node: type="NAND2", children=[Node, Node]
        /   \
      INV   INV              Node: type="INV", children=["a"]
       |     |
      "a"   "b"              String: primary input
```

The `repr()` of a tree shows its structure: `NAND2(INV(a), INV(b))`.

### 2.3. Generic Gates

Generic gates represent logic operations parsed from Verilog RTL. They
have `generic = True` and no physical implementation. TinyFlow supports
these generic gates:

 - **Two-input gates:** AND2, OR2, NAND2, NOR2, XOR2, XNOR2
 - **Single-input gates:** NOT, INV, BUF

The NOT and INV gates both compute logical inversion but serve different
purposes: NOT comes from Verilog (`~` operator), while INV is the
canonical inverter used after canonicalization.

```
Node (base class)
 ├── AND2      (a1 AND a2)
 ├── OR2       (a1 OR a2)
 ├── NAND2     (NOT (a1 AND a2))
 ├── NOR2      (NOT (a1 OR a2))
 ├── XOR2      (a1 XOR a2)
 ├── XNOR2     (a1 XNOR a2)
 ├── NOT       (NOT a)  <- from Verilog
 ├── INV       (NOT a)  <- canonical form
 └── BUF       (a)      <- buffer/passthrough
```

### 2.4. Wildcards and Constants

Wildcards and constants are special nodes used to write substitution
patterns. They are not part of actual designs but are used to match
and transform trees.

**Wildcards** match any subtree and capture it by name:

```python
_a = Wildcard('a')   # matches any subtree, captures as 'a'
_b = Wildcard('b')   # matches any subtree, captures as 'b'
```

**Constants** match constant logic values:

```python
_0 = Const0()        # matches constant 0
_1 = Const1()        # matches constant 1
```

These are used in substitution rules. For example, to replace AND with
NAND-INV:

```python
Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
```

The wildcards `_a`, `_b`, `_c`, `_d` and constants `_0`, `_1` are
predefined in `TinyFrontEndDB` for convenience.

### 2.5. Library Gates

Library gates represent physical standard cells from your cell library.
They have `generic = False` and include physical information:

 - **area_cost:** Area of the cell (in lambda^2)
 - **patterns:** List of generic gate patterns the cell can implement

For example, `NOR2X1` can implement multiple patterns:

```
NOR2X1
├── area_cost: 2048
└── patterns:
    ├── INV(NAND2(INV(_a), INV(_b)))    # NOR via De Morgan's
    └── INV(NAND2(INV(_b), INV(_a)))    # same, inputs swapped
```

The patterns tell techmap which generic gate combinations can be
"covered" by a single library cell. Technology mapping uses these
patterns to find the minimum-area implementation.

Library gate classes (INVX1, NAND2X1, NOR2X1, AOI21X1, etc.) are
dynamically created when you load the standard cell view.

### 2.6. Standard Cell View

The `StdCellFrontEndView` provides information about your standard cell
library. It is loaded from `stdcells-fe.yml` and passed to `TinyFrontEndDB`:

```python
view = StdCellFrontEndView("../stdcells/stdcells-fe.yml")
db = TinyFrontEndDB(view)
```

The view provides information needed by synthesis algorithms:

 - **For techmap:** Cell area and patterns (which generic gate
   combinations each cell implements)

 - **For STA:** Timing model parameters:
   - `Cgate`: Input capacitance (in fF)
   - `tau_d`: Intrinsic delay (in ps)
   - `tau_t`: Load-dependent delay factor (in ps/fF)

The delay through a gate is computed as: `delay = tau_d + tau_t * load`
where `load` is the total capacitance driven by the gate's output

3. Synthesis Algorithms
--------------------------------------------------------------------------

In this section, you will walk through the synthesis flow step by step,
implementing the core algorithms and verifying them interactively using
the TinyFlow REPL and GUI.

### 3.1. Verilog Parser

As discussed in lecture, Verilog parsing is the first phase in frontend
synthesis. It converts designs described in Verilog RTL into an internal
representation - specifically, trees of generic gates in TinyFlow. We
provide the parser for you. In lab 3, you explored how to build trees manually using
the REPL. Now let's see how the parser does this automatically from
Verilog.

Start the TinyFlow REPL from your build directory:

```bash
% cd ${HOME}/ece6745/project1-groupXX/tinyflow/build
% ../tinyflow-synth
```

You should see the TinyFlow REPL prompt. Go ahead and set up the standard
cell view and database, then enable the GUI so you can visualize the
trees as you work:

```python
tinyflow-synth> view = StdCellFrontEndView("../../stdcells/stdcells-fe.yml")
tinyflow-synth> db = TinyFrontEndDB(view)
tinyflow-synth> db.enable_gui()
```

The GUI window will open showing an empty forest. Now let's parse a
Verilog file to see how the parser converts RTL into generic gate trees:

```python
tinyflow-synth> db.read_verilog("path/to/design.v")
```

Watch the GUI update as the parser creates trees for each output. Each
tree is composed of generic gates (AND2, OR2, NOT, etc.) that represent
the logic from your Verilog `assign` statements. You can also inspect
the trees in the REPL:

```python
tinyflow-synth> db.print_forest()
tinyflow-synth> db.print_tree("out")
```

Take a moment to compare the Verilog source with the tree representation.
Think about how the parser maps Verilog operators (`&`, `|`, `~`, `^`) to
generic gates (AND2, OR2, NOT, XOR2).

### 3.2. Substitution

Substitution is a core operation used throughout our synthesis framework
in TinyFlow. It matches a **find pattern** against a node and, if
successful, produces a new node based on a **replace template**. Wildcards
in the pattern capture subtrees, which are then substituted into the
template. For example, to transform an AND gate into NAND-INV form:

```python
tinyflow-synth> sub = Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
tinyflow-synth> result = sub.apply(AND2("x", "y")) # Returns INV(NAND2(x, y))
```

The wildcards `_a` and `_b` in the pattern capture the inputs `"x"` and
`"y"`, then those captured values are substituted into the replace
template to produce `INV(NAND2(x, y))`.

The substitution works in two phases:

 1. **Match:** Recursively compare the find pattern tree against the input
    node. Wildcards match any subtree and capture it by name. If types
    or structure don't match, return `None`.

 2. **Replace:** Recursively build a new tree from the replace template,
    substituting captured values wherever wildcards appear.

Let's walk through a more complex example where wildcards capture entire
subtrees, not just strings. Consider matching the pattern
`INV(NAND2(_a, _b))` against the node `INV(NAND2(AND2("x", "y"), NOT("z")))`:

 - Step 1: Pattern INV matches Node INV (same type)
 - Step 2: Recurse on child: `NAND2(_a, _b)` vs `NAND2(AND2("x", "y"), NOT("z"))`
   - NAND2 matches NAND2 (same type)
   - Recurse on children[0]: `_a` (wildcard) matches `AND2("x", "y")` →
     capture `{a: AND2("x", "y")}`
   - Recurse on children[1]: `_b` (wildcard) matches `NOT("z")` →
     capture `{b: NOT("z")}`
 - Step 3: Merge captures → `{a: AND2("x", "y"), b: NOT("z")}`

Notice that `_a` and `_b` captured entire subtrees, not just leaf strings.
This is essential for synthesis transformations where we need to preserve
complex subexpressions.

Go ahead and implement the `_match_recursive` and `_replace_recursive`
methods in `synth/substitute.py`. Here are some hints to guide your
implementation:

 - Wildcards (`isinstance(pattern, Wildcard)`) match anything and return
   `{pattern.name: node}`
 - Constants (`Const0`, `Const1`) match their respective constant values
 - Non-Node leaves (strings) must be equal to match
 - Nodes must have the same `type` and matching children
 - If the same wildcard appears twice, both captures must be equal
   (compare using `repr()`)

Use the REPL and GUI to test your implementation as you develop it. Try
different patterns and see if they match as expected:

```python
tinyflow-synth> sub = Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
tinyflow-synth> tree = AND2("x", "y")
tinyflow-synth> result = sub.apply(tree)
tinyflow-synth> print(result)
INV(NAND2(x, y))
```

Once your implementation is working, run the substitution tests:

```bash
% cd ${HOME}/ece6745/project1-groupXX/tinyflow/build
% pytest ../synth/tests/substitute_test.py -v
```

All 22 tests should pass when your implementation is correct.

### 3.3. Canonicalize

With the ability to substitute patterns with a template, we can now build
the first step of the technology mapping phase: canonicalization. In order
to map between generic gates and a given standard cell library, we need to
lower both sides to a common representation and match against each other.
Canonicalization is that step. The input is a tree with generic gates
(AND2, OR2, NOR2, XOR2, XNOR2, NOT, BUF) and the output is a logically
equivalent tree using only NAND2 and INV gates.

Go ahead and implement the `canonicalize` function in
`synth/canonicalize.py`. You will need to define a substitution rule for
each generic gate type (one rule per gate), apply the rules to the nodes
in a tree, and apply it to all trees in the database. The recommended approach is to
write a recursive function to apply the rules to a tree, but you can
implement this however you want.

To test your implementation, use the REPL and GUI to visualize trees
before and after canonicalization:

```python
tinyflow-synth> view = StdCellFrontEndView("../../stdcells/stdcells-fe.yml")
tinyflow-synth> db = TinyFrontEndDB(view)
tinyflow-synth> db.enable_gui()
tinyflow-synth> db.add_inports(["a", "b", "c"])
tinyflow-synth> db.add_outports(["out"])
tinyflow-synth> db.set_tree("out", OR2(AND2("a", "b"), "c"))
tinyflow-synth> db.print_forest()
```

Look at the GUI to see the tree with generic gates. Now run canonicalize
and observe the transformation:

```python
tinyflow-synth> canonicalize(db, view)
tinyflow-synth> db.print_forest()
```

The trees should now contain only NAND2 and INV gates and the tree logic should be
equivalent to the pre-canonicalized tree. Once your implementation is working, run
the canonicalize tests:

```bash
% cd ${HOME}/ece6745/project1-groupXX/tinyflow/build
% pytest ../synth/tests/canonicalize_test.py -v
```

All 20 tests should pass when your implementation is correct.

### 3.4. Techmap

Technology mapping replaces the canonical NAND2/INV tree with library
cells from your standard cell library. The goal is to find the
minimum-area implementation.

This is where the **patterns** field from your Part A front-end view
comes into play. Each library cell has patterns describing which
NAND2/INV combinations it can implement. For example, NOR2X1 has a
pattern that matches `INV(NAND2(INV(_a), INV(_b)))`. When techmap finds
this pattern in the canonical tree, it can replace all four gates with
a single NOR2X1 cell.

**Input:** Canonicalized tree (NAND2/INV gates only)

**Output:** Tree with library cells (INVX1, NAND2X1, NOR2X1, AOI21X1, etc.)
with minimum area (optimal for trees)

The algorithm uses two-phase dynamic programming:

 - **Phase 1 (bottom-up):** For each node, try all patterns and compute
   the minimum cost. Store the optimal cost and pattern for later.

 - **Phase 2 (top-down):** Starting from the primary output tree roots,
   apply the optimal pattern stored for each node and recursively
   process children.

The DP recurrence is:

```
opt_cost(node) = min over patterns p { p.area_cost + sum(opt_cost(child)) }
```

**Your task:** Implement `_techmap_phase1` and `_techmap_phase2` in
`synth/techmap.py`.

**Hints:**

 - Use `repr(node)` as the key into `opt_cost` and `opt_pattern` dicts
 - Iterate over `view.patterns` to try all possible pattern matches
 - In phase 1, process children before the current node (bottom-up)
 - In phase 2, apply the pattern first, then recurse on children
 - Leaf nodes (strings) have zero cost and pass through unchanged

**Interactive:** After canonicalizing, run techmap and observe the GUI:

```python
tinyflow-synth> techmap(db, view)
tinyflow-synth> db.print_forest()
tinyflow-synth> db.report_area()
```

The trees should now contain library cells (INVX1, NAND2X1, NOR2X1, etc.)
instead of generic gates.

**Testing:** Run the techmap tests from your build directory:

```bash
% cd ${HOME}/ece6745/project1-groupXX/tinyflow/build
% pytest ../synth/tests/techmap_test.py -v
```

All 30 tests should pass when your implementation is correct.

### 3.5. STA

Static timing analysis (STA) computes the delay through the mapped
netlist and finds the critical path. This is where the **timing model**
from your Part A front-end view is used.

Recall the first-order delay model from Part A:

```
gate_delay = tau_d + tau_t * load
```

where `tau_d` is the intrinsic (parasitic) delay, `tau_t` is the
load-dependent delay factor, and `load` is the capacitance the gate
drives. The load includes the input capacitance (`Cgate`) of all gates
connected to the output.

**Input:** Mapped tree with library cells

**Output:** Critical path delay and the path itself

The algorithm has three phases:

 - **Phase 1 (compute loads):** Traverse the trees and accumulate the
   load capacitance on each gate. A gate's load is the sum of `Cgate`
   for all gates it drives. Primary outputs have a fixed output load.

 - **Phase 2 (compute arrivals):** Propagate arrival times from inputs
   to outputs. Primary inputs have arrival time 0. For each gate:
   `arrival = max(children arrivals) + gate_delay`.

 - **Phase 3 (find critical path):** Find the output with maximum
   arrival time, then backtrace through the tree following the critical
   pin at each node.

**Your task:** Implement `_compute_loads`, `_compute_arrivals`, and
`_find_critical_path` in `synth/sta.py`.

**Hints:**

 - Use `view.get_cgate(gate_type, pin_idx)` to get input capacitance
 - Use `view.get_parasitic_delay()` and `view.get_load_delay_factor()`
   for timing parameters
 - Store results on nodes: `node.load_cap` for load, `node.timing` for
   arrival time and critical pin (use `Timing` namedtuple)
 - Primary inputs have arrival time 0
 - For arrivals, use memoization via `node.timing` to avoid recomputation

**Interactive:** After techmap, run STA and view timing results:

```python
tinyflow-synth> sta(db, view)
tinyflow-synth> db.report_timing()
tinyflow-synth> db.report_summary()
```

The timing report shows the critical path delay and the gates on the
critical path.

**Testing:** Run the STA tests from your build directory:

```bash
% cd ${HOME}/ece6745/project1-groupXX/tinyflow/build
% pytest ../synth/tests/sta_test.py -v
```

All 18 tests should pass when your implementation is correct.

### 3.6. Gate-Level Netlist Writer

The final step is to write the mapped netlist to a Verilog file. This is
provided for you:

```python
tinyflow-synth> db.write_verilog("design_mapped.v")
```

Open the output file and verify it contains library cell instantiations
(INVX1, NAND2X1, NOR2X1, AOI21X1) connected by wires. This gate-level
netlist can be used for fast-functional simulation to verify correctness
and will be the input to the back-end flow in Part C.

4. TinyFlow Front-End
--------------------------------------------------------------------------

In this section, you will use the four-step testing flow to verify your
synthesis implementation.

### 4.1. 01-pymtl-rtlsim

TODO

### 4.2. 02-iverilog-rtlsim

TODO

### 4.3. 03-tinyflow-synth

TODO

### 4.4. 04-iverilog-ffglsim

TODO

5. Optional Extensions
--------------------------------------------------------------------------

TODO

6. Project Submission
--------------------------------------------------------------------------

TODO
