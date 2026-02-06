
ECE 6745 Lab 3: TinyFlow Front End
==========================================================================

In this lab, we will explore the TinyFlow front-end which takes as input
a Verilog RTL design and produces a gate-level netlist of standard cells.
The complete TinyFlow standard-cell and ASIC design flow is shown below
with the front end highlighted in red.

![](img/lab3-tinyflow.png)

The front end includes two-state RTL simulation, four-state RTL
simulation, synthesis, and fast-functional gate-level simulation. In
lecture, we discussed an approach to synthesis based on technology
mapping with dynamic programming to optimize the area of the final
gate-level netlist. In this lab, we will be instead implementing a very
simple unoptimized synthesis tool. The three key algorithms in the
unoptimized synthesis tool are shown below.

![](img/lab3-synth-flow.png){ width=50% }

We will be begin by exploring the key data structure used in the
synthesis tool: a forest of trees where the nodes are generic gates,
standard cells, or signals. We will then explore the verilog reader,
implement an unoptimized technology mapping algorithm, use the provided
gate-level netlist writer to generate the final Verilog gate-level
netlist, and then put these algorithms together into a synthesis tool.
Finally, we will go through the entire end-to-end front end flow for a
full adder.

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

Follow the same process as previous labs. Find a free workstation and log
into the workstation using your NetID and standard NetID password. Then
complete the following steps. These are the same steps as in the previous
lab with one exception. We are now installing the Verilog extension both
on the workstation and on the remote server.

 - Start VS Code
 - Install the Remote-SSH extension, Surfer, and Verilog extensions
 - Use View > Command Palette to execute Remote-SSH: Connect Current Window to Host...
 - Enter netid@ecelinux-XX.ece.cornell.edu where XX is an ecelinux server number
 - Find the Verilog extension again and install on the remote server
 - Use View > Explorer to open your home directory on ecelinux
 - Use View > Terminal to open a terminal on ecelinux
 - Start MS Remote Desktop

![](img/tut00-vscode-verilog.png)

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

Your repo contains the following files.

```
.
├── README.md
├── asic
│   └── build-fa
│       ├── 01-verilator-rtlsim
│       ├── 02-iverilog-rtlsim
│       ├── 03-tinyflow-synth
│       │   └── run.py
│       └── 04-tinyflow-ffglsim
├── rtl
│   ├── FullAdder.v
│   └── test
│       └── FullAdder-test.v
├── stdcells
└── tinyflow
    ├── synth
    │   ├── StdCellFrontEndView.py
    │   ├── TinyFrontEndDB.py
    │   ├── TinyFrontEndGUI.py
    │   ├── print_tree.py
    │   ├── substitute.py
    │   ├── techmap_unopt.py
    │   ├── tinyv.lark
    │   └── verilog_parser.py
    └── tinyflow-synth
```

Our front-end flow use the behavioral and front-end views you developed
in Project 1, Part A. Copy these views into the lab3 directory.

```bash
% cd ${HOME}/ece6745/lab3/stdcells
% cp project1-groupXX/stdcells/stdcells.v .
% cp project1-groupXX/stdcells/stdcells-fe.yml .
```

where `XX` is your group number.

To make it easier to cut-and-paste commands from this handout onto the
command line, you can tell Bash to ignore the `%` character using the
following command:

```bash
% alias %=""
```

Now you can cut-and-paste a sequence of commands from this tutorial
document and Bash will not get confused by the `%` character which begins
each line.

2. Data Structure: Forest of Trees
--------------------------------------------------------------------------

As discussed in lecture, the synthesis data structure used in our basic
front-end is a forest of trees where nodes can be generic gates, standard
cells, or signals. The synthesis algorithms create, transform, and
analyze this forest of trees.

To get started, create a build directory and start the TinyFlow synthesis
REPL.

```bash
% mkdir -p ${HOME}/ece6745/lab3/tinyflow/build
% cd ${HOME}/ece6745/lab3/tinyflow/build
% ../tinyflow-synth
```

You should see the following:

```
TinyFlow Synth REPL v0.1

Type 'help()' for available commands.
Type 'clear()' to clear the screen.
Type 'exit()' or Ctrl-D to quit.

tinyflow-synth>
```

`tinyflow-synth>` is the TinyFlow synthesis REPL prompt which will enable
you to intereactively experiment with different synthesis data structures
and algorithms. The TinyFlow synthesis REPL is basically the Python REPL
with a few extra features so most standard Python command should work as
well.

### 2.1. Nodes and Trees

The base class for all gates is `Node`. Generic gates (AND2, OR2, NAND2,
NOR2, XOR2, NOT, INV, BUF) are Nodes that represent logic operations.
Standard cell gates (INVX1, NAND2X1, NOR2X1, etc.) are Nodes read in from
your front-end view. `Signal` nodes represent inputs and wires. There are
also special `Wildcard` nodes that we will explain later. Let's create a
simple tree and explore it:

```python
tinyflow-synth> a, b, c = Signal("a"), Signal("b"), Signal("c")
tinyflow-synth> tree = AND2(OR2(a, b), c)
tinyflow-synth> print(tree)
tinyflow-synth> print(tree.type)
tinyflow-synth> print(tree.generic)
tinyflow-synth> print(tree.children)
tinyflow-synth> print(tree.children[0].type)
tinyflow-synth> print(tree.children[1])
tinyflow-synth> print(tree.eval(a=1, b=0, c=1))
```

Each node has a `type` (the gate name), `children` (its inputs), and
`generic` (True for generic gates, False for standard cell gates). Signal
nodes are leaf nodes with no children. Nodes also provide helper methods:
`is_signal()` returns `True` for Signal nodes, `is_wildcard()` returns
`True` for Wildcard nodes, and `==`/`!=` compare nodes by type.

```python
tinyflow-synth> a.is_signal()
True
tinyflow-synth> tree.is_signal()
False
tinyflow-synth> _a.is_wildcard()
True
tinyflow-synth> a == Signal("a")
True
tinyflow-synth> a == AND2(a, b)
False
tinyflow-synth> AND2(a, b) == AND2(c, c)
True
tinyflow-synth> AND2(a, b) == OR2(a, b)
False
```

Go ahead and evaluate all input combinations using `tree.eval()` and derive
the truth table for `AND2(OR2(a, b), c)`.


print_tree

### 2.2. Front-End Database

print_forest

3. Algorithm: Verilog Reader
--------------------------------------------------------------------------

4. Algorithm: Unoptimized Technology Mapping
--------------------------------------------------------------------------

### 4.1. Exactly Matching Trees

### 4.2. Partially Matching Trees

### 4.3. Capturing Subtrees

### 4.4. Replacing Trees

### 4.5. Substitutions

### 4.6. Unoptimized Technology Mapping

5. Algorithm: Gate-Level Netlist Writer
--------------------------------------------------------------------------

6. Synthesis Tool
--------------------------------------------------------------------------

full adder

7. TinyFlow Front End
--------------------------------------------------------------------------

### 7.1. Two-State RTL Simulation

### 7.2. Four-State RTL Simulation

### 7.3. Synthesis

### 7.4. Fast-Functional Gate-Level Simulation


