
ECE 6745 Lab 3: TinyFlow Frontend
==========================================================================

In this lab, we will learn about the TinyFlow frontend and implement a
basic synthesis flow that transforms Verilog RTL into a gate-level netlist.

 - **Verilog Parser:** Reads Verilog RTL and converts it into an internal
     tree representation of generic gates

 - **Substitution:** Pattern matching and replacement operation that
     transforms trees by matching a find pattern and producing a new tree
     from a replace template

 - **Naive Technology Mapping:** Maps generic gates to standard cells from
     your library using simple pattern substitution

 - **Gate-Level Netlist Writer:** Outputs the mapped design as a Verilog
     gate-level netlist using your standard cells

We will be using the following TinyFlow frontend synthesis flow.

![](img/lab3-frontend.png)

We will begin by exploring the TinyFlow data structures using the REPL and
GUI. We will then implement a Verilog parser, tree printing, substitution,
and naive technology mapping. Finally, we will walk through the four-step
frontend flow (2-state simulation, 4-state simulation, synthesis,
fast-functional gate-level simulation) to verify a Full Adder design.

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

Follow the same process as previous labs. Find a free workstation and log
into the workstation using your NetID and standard NetID password. Then
complete the following steps. These are the same steps as in the first lab
with one exception. We are now installing
the VS Code Surfer extension to be able to view waveforms.

 - Start VS Code
 - Install the Remote-SSH extension and the Surfer extension
 - Use View > Command Palette to execute Remote-SSH: Connect Current Window to Host...
 - Enter netid@ecelinux-XX.ece.cornell.edu where XX is an ecelinux server number
 - Use View > Explorer to open your home directory on ecelinux
 - Use View > Terminal to open a terminal on ecelinux
 - Start MS Remote Desktop

![](img/tut00-vscode-remote-ssh.png)

![](img/tut00-vscode-surfer.png)

Now use the following commands to clone the repo we will be using for
today's lab.

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/ece6745-lab3 lab3
% cd lab3
% tree
```

Your repo contains the following directories:

```
.
├── asic/
│   └── build-fa/
│       ├── 01-verilator-rtlsim/
│       ├── 02-iverilog-rtlsim/
│       ├── 03-tinyflow-synth/
│       └── 04-iverilog-ffglsim/
├── rtl/
│   ├── FullAdder.v
│   └── test/
│       └── FullAdder-test.v
├── stdcells/
└── tinyflow/
    ├── conftest.py
    ├── pytest.ini
    ├── synth/
    │   ├── canonicalize.py
    │   ├── design_check.py
    │   ├── generate_forms.py
    │   ├── sta.py
    │   ├── StdCellFrontEndView.py
    │   ├── substitute.py
    │   ├── synth.py
    │   ├── techmap.py
    │   ├── TinyFrontEndDB.py
    │   ├── TinyFrontEndGUI.py
    │   ├── tinyv.lark
    │   ├── verilog_parser.py
    │   └── tests/
    │       └── ...
    └── tinyflow-synth
```

To make it easier to cut-and-paste commands from this handout onto the
command line, you can tell Bash to ignore the `%` character using the
following command:

```bash
% alias %=""
```

Now you can cut-and-paste a sequence of commands from this tutorial
document and Bash will not get confused by the `%` character which begins
each line.

Our frontend tools use the front-end view you developed in Project 1 Part A.
Copy your front-end view file into the lab3 folder:

```bash
% cd ${HOME}/ece6745
% cp project1-groupXX/stdcells/stdcells-fe.yml lab3/stdcells
```

2. Data Structures
--------------------------------------------------------------------------

As discussed in lecture, TinyFlow represents logic designs as trees of
gates. Our synthesis tool will form these trees from Verilog and manipulate
them through various transformations. The trees are stored in a data
container called `TinyFrontEndDB`.

To get started, create a build directory and start the TinyFlow REPL:

```bash
% mkdir -p ${HOME}/ece6745/lab3/tinyflow/build
% cd ${HOME}/ece6745/lab3/tinyflow/build
% ../tinyflow-synth

TinyFlow Synth REPL v0.1

Type 'help()' for available commands.
Type 'clear()' to clear the screen.
Type 'exit()' or Ctrl-D to quit.

tinyflow-synth>
```

The `tinyflow-synth` tool is where we will implement our synthesis
algorithms. It provides a REPL for interactive exploration.

### 2.1. Nodes and Trees

The base class for all gates is `Node`. Generic gates (AND2, OR2, NAND2,
NOR2, XOR2, NOT, INV, BUF) are Nodes that represent logic operations.
Standard cell gates (INVX1, NAND2X1, NOR2X1, etc.) are Nodes read in from
your front-end view. There are also special nodes like `Const0`, `Const1`,
and `Wildcard` that we will explain later. Let's create a simple tree and
explore it:

```python
tinyflow-synth> tree = AND2(OR2("a", "b"), "c")
tinyflow-synth> print(tree)
tinyflow-synth> print(tree.type)
tinyflow-synth> print(tree.generic)
tinyflow-synth> print(tree.children)
tinyflow-synth> print(tree.children[0].type)
tinyflow-synth> print(tree.children[1])
tinyflow-synth> print(tree.eval(a=1, b=0, c=1))
```

Each node has a `type` (the gate name), `children` (its inputs), and
`generic` (True for generic gates, False for standard cell gates). Children
can be other nodes or strings (input names).

### 2.2. Constants

TinyFlow also supports constant values using `Const0` and `Const1` (or
the shorthand `_0` and `_1`):

```python
tinyflow-synth> tree = AND2("a", _1)
tinyflow-synth> print(tree.eval(a=1))
tinyflow-synth> print(tree.eval(a=0))
```

`Const0` and `Const1` correspond to the TIELO and TIEHI cells in your
standard cell library.

### 2.3. Frontend Database and GUI

Now let's use the frontend database to manage a design. To create a
database using `TinyFrontEndDB`, we first need to have our frontend view ready. `StdCellFrontEndView`
loads the front-end view YAML file you created in Project 1 Part A. It
provides access to cell information (area, timing parameters), patterns for
technology mapping, and standard cell gate classes (INVX1, NAND2X1, etc.).

Let's create the view and database:

```python
tinyflow-synth> view = StdCellFrontEndView.parse_lib("../../stdcells/stdcells-fe.yml")
tinyflow-synth> db = TinyFrontEndDB(view)
```

The database supports visualizing its contents through a GUI. Enable the
GUI with:

```python
tinyflow-synth> db.enable_gui()
```
The GUI window will open. 

![](img/lab3-gui-blank.png)

Now add inputs, outputs, and set a tree:

```python
tinyflow-synth> db.add_inports(["a", "b", "c"])
tinyflow-synth> db.add_outports(["out"])
tinyflow-synth> db.set_tree("out", AND2(OR2("a", "b"), "c"))
```

Watch the GUI update to show your tree.


![](img/lab3-gui-types.png)

In the GUI:

 - **Green ovals:** Primary inputs
 - **Orange ovals:** Wire signals
 - **Blue ovals:** Primary outputs
 - **Grey rectangles:** Generic gates
 - **Red rectangles:** Standard cell gates

Once you are done with the GUI, you can exit the REPL by calling `exit()` or pressing Ctrl-D.

3. Synthesis Algorithms
--------------------------------------------------------------------------

In this section we will implement a couple of algorithms that will warm
you up for writing recursive functions and will be useful for Project 1
Part B. By the end of this lab, you will have a working naive technology
mapping implementation that does the job but may not guarantee minimal area cost.

### 3.1. Verilog Parser

The first step of our synthesis flow is parsing. The parser lexes the
simple Verilog syntax, forms an Abstract Syntax Tree (AST), and performs
"forresting" to translate the AST into the tree of logic form used in
TinyFlow.

<!-- TODO: @Chris to fill in parser implementation details -->


<!-- Irwin's text on verilog -->
In this part, we will be discussing the limitations of verilog we can write
for the TinyFlow. This limitation is mainly pedagogical to simplify the flow as well as limitations due to our parser (this does not mean our parser is not good).

**Rules:**

1. Only Combinational Verilog
2. Only Use `wire` keyword instead of `logic`
3. Only Gate level modeling. However NOT using the gate primitives like `and(), or(), not()` instead we will be using `assign` statements with operators: `&, |, ~, ^`.
4. No hierarchy, every module is flat.
5. No multibit wires. So you cannot declare something like `wire[7:0] foo` you need something like
    ```verilog
    wire foo0;
    wire foo1;
    ...
    wire foo7;
    ```
6.  Limitations on ports. Note this is a limitation for the physical tapeout
and we will communicate with you the number of input and output pins your
module will get.

If you are ever confused take a look at the Lark grammar that we use to see what our parser should parse.

<!-- Irwin's text on verilog -->

The database includes a method to read in Verilog by passing the path to
the Verilog file:

```python
tinyflow-synth> view = StdCellFrontEndView.parse_lib("../../stdcells/stdcells-fe.yml")
tinyflow-synth> db = TinyFrontEndDB(view)
tinyflow-synth> db.enable_gui()
tinyflow-synth> db.read_verilog("../../rtl/FullAdder.v")
```

Watch the GUI update to show the parsed trees.

### 3.2. Printing Trees

Before we implement more complex algorithms, let's warm up with a simple
recursive function. Write a standalone function `print_tree(node, indent=0)`
that prints a tree structure to the terminal. For example, given
`AND2(OR2("a", "b"), "c")`, it should print:

```
AND2
|- OR2
|  |- a
|  |- b
|- c
```

Create a new file for your function:

```bash
% cd ${HOME}/ece6745/lab3/tinyflow
% code synth/simple_print_tree.py
```

Think about the recursive structure: for each node, print its type, then
recursively print each child with increased indentation. Strings (input
names) are leaf nodes. Test your function in the REPL:

```python
tinyflow-synth> from synth.simple_print_tree import print_tree
tinyflow-synth> tree = AND2(OR2("a", "b"), "c")
tinyflow-synth> print_tree(tree)
```

### 3.3. Substitution

We are now ready to start implementing our first core algorithm: substitution.
Substitution is a core operation used throughout synthesis. It matches a
**find pattern** against a node and its subtree, and if successful, produces
a new subtree based on a **replace template**. The `Substitute` class takes
a find pattern and a replace template, both of which are trees that can
contain wildcards:

```python
sub = Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
```

`Wildcard` is a special type of Node that matches any subtree and captures
it by name. For example, `_a = Wildcard("a")` creates a wildcard that
captures into the key `"a"`. The REPL provides predefined wildcards `_a`,
`_b`, `_c`, `_d` for convenience. When the pattern matches, the captured subtrees are
substituted into the replace template. For example:

```python
tinyflow-synth> sub = Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
tinyflow-synth> result = sub.apply(AND2("x", "y"))
tinyflow-synth> print(result)
INV(NAND2(x, y))
```

In this example, the substitution matches the AND2 tree with the find
pattern, captures `{"a": "x", "b": "y"}`, and builds a new tree from the
replace template with captured subtrees in the corresponding positions.

The `apply` method works in two phases:

 1. **Match**: Recursively compare the find pattern tree against the input
    tree. Wildcards match any subtree and return a dictionary of captures
    (e.g., `{"a": subtree1, "b": subtree2}`). If types or structure don't
    match, return `None`.

 2. **Replace**: Recursively build a new tree from the replace template,
    substituting captured values from the match phase wherever wildcards
    appear.

Go ahead and implement `_match_recursive` and `_replace_recursive` in
`synth/substitute.py`. Test your implementation in the REPL:

```python
tinyflow-synth> sub = Substitute(find=AND2(_a, _b), replace=INV(NAND2(_a, _b)))
tinyflow-synth> sub.apply(AND2(OR2("a", "b"), "c"))
```

You should see `INV(NAND2(OR2(a, b), c))`. Once you are happy with your
implementation, run the substitution tests:

```bash
% cd ${HOME}/ece6745/lab3/tinyflow/build
% pytest ../synth/tests/substitute_test.py -v
```

### 3.4. Naive Technology Mapping

In Project 1 Part B you will implement an optimized version of technology
mapping. For this lab, we will implement a naive version that simply
replaces each generic gate with a corresponding standard cell.

The idea is to use your substitution implementation to define a pattern for
each generic gate. For example, to map AND2 to NAND2X1 + INVX1:

```python
Substitute(find=AND2(_a, _b), replace=INVX1(NAND2X1(_a, _b)))
```

Implement `techmap_unopt` in `synth/techmap.py`. The function takes the
database and view as arguments:

```python
def techmap_unopt(db: TinyFrontEndDB, view: StdCellFrontEndView):
```

Inside the function, define substitution rules for each generic gate type
(AND2, OR2, NOT, BUF, XOR2) and apply them to all trees in the database.
You can iterate over all trees with `db.trees.keys()`, retrieve each tree
with `db.get_tree(name)`, and update it with `db.set_tree(name, new_tree)`.

Test your implementation with the REPL and GUI. After running techmap, the
grey generic gates should become red standard cell gates:

```python
tinyflow-synth> from synth.techmap import techmap_unopt
tinyflow-synth> view = StdCellFrontEndView.parse_lib("../../stdcells/stdcells-fe.yml")
tinyflow-synth> db = TinyFrontEndDB(view)
tinyflow-synth> db.enable_gui()
tinyflow-synth> db.read_verilog("../../rtl/FullAdder.v")
tinyflow-synth> techmap_unopt(db, view)
```

4. The Frontend Flow
--------------------------------------------------------------------------

As discussed in lecture, the frontend is more than just synthesis. The
frontend flow consists of four stages: two-state simulation, four-state
simulation, synthesis, and fast-functional gate-level simulation. As
paranoid ASIC engineers, we verify our design at each step. We simulate
the RTL before synthesis to catch design bugs early, then simulate the
gate-level netlist after synthesis to ensure the transformation preserved
functionality.

### 4.1 Two-State RTL Simulation

To ensure functionality, the first step is to verify our design quickly
using two-state simulation. Two-state simulation tests only logic values
1 and 0 to ensure basic logic correctness. In this part we will use
Verilator to perform two-state simulation.

In this lab we will verify a Full Adder design. We provide the Verilog RTL
in `rtl/FullAdder.v` and a basic testbench in `rtl/test/FullAdder-test.v`.
Take a look at both files to understand the design and test structure.

Now run the two-state simulation with Verilator:

```bash
% cd $HOME/ece6745/lab3/asic/build-fa/01-verilator-rtlsim
% verilator --top Top --timing --binary -o FullAdder-test \
    ../../../rtl/FullAdder.v \
    ../../../rtl/test/FullAdder-test.v
% ./obj_dir/FullAdder_test
```

As discussed in lecture, two-state simulation has a limitation: unassigned
signals default to 0. This can hide bugs in your design. For example, if
you forget to assign an output, two-state simulation will silently use 0
instead of flagging an error.

Try this experiment: comment out the `assign g = a & b;` line in your Full
Adder and re-run the simulation. Notice that `g` silently takes the value
0 instead of producing an error. This is why we need four-state simulation
in the next step. Change the code back before continuing.

### 4.2 Four-State RTL Simulation

Four-state simulation uses four logic values: 0, 1, X (unknown), and Z
(high impedance). You get X when a signal is uninitialized, when multiple
drivers are fighting (contention), or through propagation of uncertainty
(X propagates through logic). You get Z when a wire is floating (nothing
is driving it) or from a tri-stated output.

We use four-state simulation to capture these bugs. It is slower than
two-state simulation, but it narrows down our issue search space. If your
design passes two-state but fails four-state, the problem is usually
related to X propagation or uninitialized signals.

Go ahead and run the four-state simulation with Icarus Verilog:

```bash
% cd $HOME/ece6745/lab3/asic/build-fa/02-iverilog-rtlsim
% iverilog -g2012 -I ../../../rtl -o FullAdder-test \
    ../../../rtl/test/FullAdder-test.v
% ./FullAdder-test
```

Now try the same experiment: comment out the `assign g = a & b;` line and
re-run the simulation. This time you should see the simulation catch the
error because `g` becomes X instead of silently defaulting to 0. Change
the code back before continuing.

### 4.3 Synthesis

Now that we have rigorously tested our Verilog design, we are ready to
synthesize it into a gate-level netlist. For this step, we will use the
batch processing mode of `tinyflow-synth` instead of the REPL mode we have
previous used. The batch mode takes a run script that describes the synthesis
steps.

Go ahead and edit the run script:

```bash
% cd $HOME/ece6745/lab3/asic/build-fa/03-tinyflow-synth
% code run.py
```

Populate the script with the commands to perform technology mapping:

```python
view = StdCellFrontEndView.parse_lib("../../../stdcells/stdcells-fe.yml")
db = TinyFrontEndDB(view)
db.read_verilog("../../../rtl/FullAdder.v")
techmap_unopt(db, view)
db.write_verilog("post-synth.v")
```

Now run the synthesis:

```bash
% ../../../tinyflow/tinyflow-synth -f run.py
```

This outputs the `post-synth.v` file. Open it and have a look. You should
see that all gates are now standard cells from your library, and the
module still has the same inputs and outputs as the original RTL.

### 4.4 Fast-Functional Gate-Level Simulation

Now that we have our synthesized design, as paranoid ASIC engineers we
want to double check that the synthesized design still does what we
intended. Synthesis tools may not always be correct! To verify this, we
perform fast-functional gate-level simulation (FFGL), which is four-state
simulation using the same testbench but with the synthesized design and
the behavioral view of the standard cells.

First, copy over your `stdcells.v` from your project directory to the lab3
stdcells directory:

```bash
% cp $HOME/ece6745/project1-groupXX/stdcells/stdcells.v $HOME/ece6745/lab3/stdcells/
```

Now run the fast-functional gate-level simulation:

```bash
% cd $HOME/ece6745/lab3/asic/build-fa/04-iverilog-ffglsim
% iverilog -g2012 -o FullAdder-test \
    ../../../stdcells/stdcells.v ../03-tinyflow-synth/post-synth.v \
    ../../../rtl/test/FullAdder-test.v
% ./FullAdder-test
```

If the simulation passes, your synthesized design is functionally correct.

### 4.5 Exercise: Design a Decoder

You have now walked through the frontend flow with the Full Adder. For the
final part of this lab, design your own module from scratch. Implement a
simple 2-to-4 decoder and push it through the complete four-step flow. 

First, create the build directory structure:

```bash
% mkdir -p $HOME/ece6745/lab3/asic/build-decoder
% cd $HOME/ece6745/lab3/asic/build-decoder
% mkdir 01-verilator-rtlsim 02-iverilog-rtlsim 03-tinyflow-synth 04-iverilog-ffglsim
```

Next, write the Verilog RTL for your decoder:

```bash
% cd $HOME/ece6745/lab3/rtl
% code Decoder.v
```

Then write a testbench for your decoder:

```bash
% cd $HOME/ece6745/lab3/rtl/test
% code Decoder-test.v
```

Now run through the four-step frontend flow, referring back to sections
4.1-4.4 for the commands. Remember to update the file paths to use your
decoder files instead of the Full Adder files.
