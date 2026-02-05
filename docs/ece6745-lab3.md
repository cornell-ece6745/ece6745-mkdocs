
ECE 6745 Lab 3: TinyFlow Frontend
==========================================================================

In this lab, we will learn about the TinyFlow Frontend and implement 
several front end algorithms including print forest, substitution and 
canonicalize.

 - **Print Forest** Logical function of the standard cell, used for
     gate-level simulation

 - **Substitution** Transistor-level representation of standard cell,
     used for functional verification and layout-vs-schematic

 - **Canonicalize:** Layout of standard cell, used for design-rule
     checking (DRC), layout-vs-schematic (LVS), resistance/capacitance
     extraction (RCX), and fabrication

We will be using the following TinyFlow Frontend design flow.

Please fix with actual image.
![](img/T03-frontend.png)


-- using the repl to run through the whole flow
-- naive flow (naive techmapping)

-- random placement?
-- pin to pin routing?


Lab 3
1. ECE linux
2. Datastructures, TinyFrontEndDB, TinyFrontEnnView, REPL
3. Algorithms
    3.1 Print 
    3.2 Substitution
    3.3 Naive Techmapping (naive_techmap or techmap_v0)
4. Front End Flow (manual)
    4.1 Pymtl Sim
    4.2 Iverilog sim
    4.3 Synth (write in tinyflow-synth)
    4.4 FFGL sim

Lab 4.
1. ECE Linux
2. Datastructures, TinyBackEndDb, TinyBackEndView, REPL
3. Algorithms
    3.1 Naive Placement (place_random)
    3.2 Niave Route (route_)
        pick two points, manhatten, try other way, if not go up manhatten distance up to 6
4. Back End Flow (manual)
    4.1 PNR
    4.2 DRC
    4.3 LVS
5. Automated Flow

(Are we describing the whole flow or just doing specific parts, like should we have students start by writing verilog and simulating it or not)

We will begin by copying over your TinyFrontendView that you did for Lab 2 and
Project 1A and verifying its functionality using our testbench. We will then
write some verilog, with massive restrictions (as detailed later), with the
intention of running it through the TinyFlow Frontend. We will then examine
the TinyFlow codebase, understanding the data representations, and begin with
a refresher of recusive functions through the implementation of
`print_forest`. Finally, we will write and test two key parts of the frontend
flow: `substitute` and `canonicalize`.

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

Follow the same process as previous labs Find a free workstation Find a
free workstation and log into the workstation using your NetID and
standard NetID password. Then complete the following steps. These are the
same stes as in the first lab with one exception. We are now installing
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

Your repo contains the following files for the views and simulation scripts for
each standard cell:

```
.
└── stdcells/
    ├── ...
    └── stdcekks-fe.yml
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
    │   ├── tinysv.lark
    │   ├── verilog_parser.py
    │   └──tests
    │       ├── ...
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

Run the following to copy your frontend view file into the new folder. Or
just do it manually.

```bash
% cd ${HOME}/ece6745
% cp lab2/stdcells/stdcell-fe.yml lab3/stdcells
```

2. Datastructures
--------------------------------------------------------------------------

In this part, we will be examining the TinyFlow Frontend code base and begin
implementing some key functions.

The key to the TinyFlow frontend is TREES. We represent our verilog modules as a forest of trees, where each output and wire is the root of a tree, and the leaves are gates. Gates can have leaves that are trees or an input/wire Node. The unified data container class that you will use to store the intermedaite information is `TinyFrontendDB`. Furthermore the FrontEnd view that you created in lab2 is now read in as a `TinyFrontendView`

Remembering the frontend flow, the key is to take in verilog or some HDL and
output a netlist composed of standard cells from our standard cell library
(the one you made in fact). Here are the key steps of our frontend flow.

 - **Verilog Parser** The input is the subset of verilog that you write,
 which is then taken in by our verilog parser which uses Lark to create an
 Abstract Syntax Tree (AST) representation. We then have a script and that converts it into a `TinyFrontendDB`

 - **Technology Mapping** Today you will implement parts of the technology
 mapping flow. Currently our techmapping flow involves two passes:
 `canonicalize` and `techmap`. However to implement these functions we must 
 first implement some helper functions/classes such as `substitution`.

 - **Static Timing Analysis:** You are responsible for implementing the 
 static timing analysis engine during Project1 Part B. More details in
 Lecture/Project handout on how this works.

 - **Gate-Level Netlist Writer:** Given the `TinyFrontendDB` and the `TinyFrontEndView` we have provided a Gate Level Netlist Writer that dumps
 out verilog using the standard cells that you have made.

**INSERT IMAGE OF TREES**

To get started, create a build directory which will use for all of our
simulations.

```bash
% mkdir -p ${HOME}/ece6745/lab3/tinyflow/build
% cd ${HOME}/ece6745/lab3/tinyflow/build
```

We will begin by manually working with the TinyFlow Frontend with our own custom REPL.

```bash
% cd ${HOME}/ece6745/lab3/tinyflow/build
% ../tinyflow-synth

TinyFlow Synth REPL v0.1

Type 'help()' for available commands.
Type 'clear()' to clear the screen.
Type 'exit()' or Ctrl-D to quit.

tinyflow-synth> 
```

Many of these functions have already been implemented for you. Now we are going to go through an
example just to get familiar with the workflow and how we represent data. Lets start by creating 
a pair triple detector. This detects when at least two of the inputs are on.

![](img/sec02-pair-triple-detector.jpg)

First we need to create our database (referenced as db from here on out) using our FrontendView
that we copied earlier. So we parse the frontend view and then pass it in to the constructor of
the db.

```
tinyflow-synth> view = StdCellFrontEndView.parse_lib('../../stdcells/stdcell-fe.yml')
tinyflow-synth> db = TinyFrontEndDB(view)
```

Next, lets add the inputs and the outputs of our Pair Triple Detector. Let's open the gui so we can watch our trees being built.
```
tinyflow-synth> db.enable_gui()
tinyflow-synth> db.addinports("in0", "in1", "in2")
tinyflow-synth> db.addoutports("out")
```

Next, we have two options to create the trees. We can either add intermediate wires using `add_wires` and setting their respective trees and building our overall output tree using subtree. Or we could just set the overall tree.

```
tinyflow-synth> tree_out = db.get_tree("out")
tinyflow-synth> tree_out.set_tree(OR(AND(OR("in0", "in1"), "in2"), AND("in0", "in1")))
```

Our db is mutable, lets try an intermediate representation like in the picture.

```
tinyflow-synth> db.add_wires("w", "y", "x")
tinyflow-synth> tree_w = db.get_tree("w")
tinyflow-synth> tree_y = db.get_tree("y")
tinyflow-synth> tree_x = db.get_tree("x")
tinyflow-synth> tree_w.set_tree(OR("in0", "in1"))
tinyflow-synth> tree_y.set_tree(AND("w", "in2"))
tinyflow-synth> tree_x.set_tree(AND("in0", "in1"))
tinyflow-synth> tree_out.set_tree(OR("y", "x"))
```

Look at our gui and how when we choose to represent wire as subtrees we get a different structure. Think about how depending on how you might write verilog there might be a different output!

*INSERT IMAGES OF GUI*

We can exit the REPL by calling `exit()`

3. Algorithms
--------------------------------------------------------------------------

In this section we will implement a couple of algorithms that will be useful for the flow.
Some can be reused for Project 1A but others are just to get you warmed up for writing recursive functions.

### 3.1 print_tree()

Let's begin by writing an external function print_tree(). 

**MAYBE we give them teh funcion file in the repo?**

```bash
% cd ${HOME}/ece6745/lab3/tinyflow/build
% code print_tree.py
```

Looking at the function signature we see the following:
 **INSERT IMAGE**

Using some of the methods above use recursion to write `print_tree()`

### 3.2 subtitution()

We should use your pseudo code elton.

### 3.3 techmap_unopt()

In the project you will implement an optimized version to techmapping. For this lab, we will just implement a naive version. First define patterns for each generic gate using standard cells. The generic gates for which we need to define standard cell patterns are the following: `buf`, `not`, `and`, `or`, `xor`.

The way to create this function is to utilize your substiution method you just wrote. Iterate through all the possible gates and do a basic substitution to techmap our trees.

**Lets give them a shell of substitution.**

4. The Frontend Flow
--------------------------------------------------------------------------

### 4.1 pymtl-rtlsim


In this part, we will be discussing the limitations of verilog we can write
for the TinyFlow. This limitation is mainly pedagogical to simplify the flow as well as limitations due to our parser (this does not mean our parser is not good).

**Rules:**

1. Only Combinational Verilog
2. Only Use `wire` keyword instead f `logic`
3. Only Gate level modeling. However NOT using the gate primitives like `and(), or(), not()` instead we will be using `assign` statements with operators: `&, |, ~, ^`.
4. No heirarchy, every module is flat.
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

In this lab we will be implementing a FullAdder module, go ahead and write
the verilog for a full adder and test it with the corresponding PyMTL test
bench.

```bash
% cd ${HOME}/ece6745/lab3/tinyflow/build
% code FullAdder.v
```

**Talk more about test bench**

### 4.2 iverilog-rtlsim

### 4.3 tinyflow-synth

Compose a script that starts by
1. Creating a TinyFrontendView and TinyFrontendDB using the parser
2. Call your unoptimized techmapping algorithm
3. Call the function to dump out verilog

### 4.4 iverilog-ffglsim
