
ECE 6745 Project 1: TinyFlow Tape-Out<br>TinyFlow Standard Cells
==========================================================================

In this project, students will build their own TinyFlow, a very simple
standard-cell-based flow. They will develop seven standard cells in TSMC
180nm and the corresponding standard cell behavioral, schematic, layout,
extracted schematic, front-end, and back-end views. They will then
implement simple algorithms for synthesis (technology mapping via tree
covering, static timing analysis) and place-and-route (simulated
annealing, 3D maze routing). Finally they will combine this work with
open-source Verilog RTL and gate-level simulators and an open-source
LVS/DRC tool to create the complete TinyFlow. Even though their TinyFlow
will only support a very small combinational subset of Verilog, this
project still gives students a unique hands-on opportunity to appreciate
every step required in more sophisticated commercial tools. Each group
will create a tiny block using their TinyFlow and these blocks will be
aggregated into a single unified tape-out on the TSMC 180nm technology
node.

The project includes three parts:

 - Part A: TinyFlow Standard Cells
 - Part B: TinyFlow Front-End
 - Part C: TinyFlow Back-End

All parts must be done in a group of 2-3 students. You can confirm your
group on Canvas (Click on People, then Groups, then search for your name
to find your project group).

!!! warning "All students must contribute to all parts!"

    It is not acceptable for one student to do all of Part A and a
    different student to do all of part B. It is not acceptable for one
    student to exclusively work on the layout while the other student
    exclusively works on SPICE characterization. All students must
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
tutorials and that you have attended the lab sections. To get started,
use VS Code to log into a specific `ecelinux` server, use MS Remote
Desktop to log into the same `ecelinux` server, source the setup scripts,
and clone your remote repository from GitHub:

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% xclock &
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/project1-groupXX
% cd project1-groupXX
% tree
```

where `XX` should be replaced with your group number. You can both pull
and push to your remote repository. If you have already cloned your
remote repository, then use git pull to ensure you have any recent
updates before working on your lab assignment.

```bash
% cd ${HOME}/ece6745/project1-groupXX
% git pull
% tree
```

where `XX` should be replaced with your group number. Your repo contains
the following files for the views and simulation scripts for each
standard cell:

```
.
└── stdcells/
    ├── verilog-test/
    │   ├── AOI21X1-test.v
    │   ├── INVX1-test.v
    │   ├── NAND2X1-test.v
    │   ├── NOR2X1-test.v
    │   ├── TIEHI-test.v
    │   └── TIELO-test.v
    ├── stdcells-be.yml
    ├── stdcells-fe.yml
    ├── stdcells-rcx.yml
    ├── stdcells.gds
    ├── stdcells.sp
    └── stdcells.v
```

Go ahead and create a build directory where you will run your simulations
when verifying and characterizing your standard cells.

```
% cd ${HOME}/ece6745/project1-groupXX
% mkdir -p stdcells/build
% cd stdcells/build
```

1. Standard-Cell Library
--------------------------------------------------------------------------

In this part, you will need to implement the following seven standard
cells.

| Name    | Logic Function        | Rise/Fall Time | Drive Strength   |
|---------|-----------------------|----------------|------------------|
| INVX1   | $Y = \overline{A}$    | Similar        |                  |
| NAND2X1 | $Y = \overline{AB}$   | Similar        | Similar to INVX1 |
| NOR2X1  | $Y = \overline{A+B}$  | (see below)    | (see below)      |
| AOI21X1 | $Y = \overline{AB+C}$ | (see below)    | (see below)      |
| TIEHI   | $Y = 1$               | n/a            | n/a              |
| TIELO   | $Y = 0$               | n/a            | n/a              |
| FILL    | n/a                   | n/a            | n/a              |

The INVX1 and NAND2X1 standard cells should be sized to have roughly
equal rise and fall times assuming the effective resistance of PNMOS
transistor is twice the effective resistance of an NMOS transistor with
the same width. The NAND2X1 standard cell should be size to have roughly
the same drive strength (i.e., effective resistance) as the INVX1
standard cell.

For NOR2X1 and AOI21X1 standard cells, students can choose to one of two
options.

 - Option 1: Size these gates such that they have roughly equal rise and
   fall times and roughly the sae drive strength (i.e., effective
   resistance) as the INVX1 standard cell; this will require using
   fingers.

 - Option 2: Do not use fingers meaning the gates will likely have
   non-equal rise and fall times. You should size the pull-down network
   to have similar effective resistance as compared to an INVX1 standard
   cell. The pull-up network will likely have a different effective
   resistance as compared to an INVX1 standard cell.

The TIEHI standard cell should use a weak PMOS to pull-up the output to
VDD; gate of weak PMOS should be connected to a diode-connected NMOS. The
TIELO standard cell should use a weak NMOS to pull-down the output to
ground, gate of weak NMOS should be connected to a diode-connected PMOS.

The FILL cell should be a one-track wide filler cell with a vertical
strip of polysilcon to meet poly density design rules.

Our standard cell library will include the following six views:

 - **Behavioral View (BV):** Logical function of the standard cell, used
     for gate-level simulation

 - **Schematic View (SV):** Transistor-level representation of standard
     cell, used for functional verification and layout-vs-schematic

 - **Layout View (LV):** Layout of standard cell, used for design-rule
     checking (DRC), layout-vs-schematic (LVS), resistance/capacitance
     extraction (RCX), and fabrication

 - **Extracted Schematic View (ESV):** Transistor-level representation
     with extracted resistance and capacitances, used for
     layout-vs-schematic (LVS) and timing characterization

 - **Front-End View (FEV):** High-level information about standard cell
     including area, input capacitance, logical function, and delay
     model; used in synthesis

 - **Back-End View (BEV):** Low-level information about standard cell
     including height, width, and pin locations; used in placement and
     routing

The following table shows which views need to be implemented for which
standard cells.

| Name    | BV     | SV     | LV     | ESV    | FEV      | BEV    |
|---------|--------|--------|--------|--------|----------|--------|
| INVX1   | yes    | yes    | yes    | yes    | yes      | yes    |
| NAND2X1 | yes    | yes    | yes    | yes    | yes      | yes    |
| NOR2X1  | yes    | yes    | yes    | yes    | yes      | yes    |
| AOI21X1 | yes    | yes    | yes    | yes    | yes      | yes    |
| TIEHI   | yes    | yes    | yes    | yes (b)| yes (c)  | yes    |
| TIELO   | yes    | yes    | yes    | yes (b)| yes (c)  | yes    |
| FILL    | yes (a)| no     | yes    | no     | yes (c,d)| yes (d)|

 - (a) The behavioral view for the FILL standard cell is an empty Verilog
   module.

 - (b) Extracted schematic simulation for the TIEHI and TIELO standard
   cells is only for functional verification; these gates do not switch
   so there is no timing characterization.

 - (c) The front-end view for the TIEHI, TIELO, and FILL standard cells
   does not include a linear delay model nor any patterns.

 - (d) The front-end and back-end views for the FILL standard cell do not
   include any pins.

We will be using the following TinyFlow standard-cell design flow.

![](img/T02-flow.png)

We will by begin by writing the behavioral view in Verilog and verifying
its functionality using a Verilog test bench and Icarus Verilog. We will
then write the schematic view in SPICE and verify its functionality using
a SPICE test bench and TinyFlow-Ngspice. Instead of using Ngspice
directly, we will use our TinyFlow-Ngspice wrapper script which makes it
much easier to run SPICE simulatoins. We will then use the KLayout design
editor to create the layout, perform a design-rules check (DRC), perform
a layout vs. schematic check (LVS), and generate an extracted schematic.
We will re-simulate the extracted transistor-level schematic using
TinyFlow-Ngspice to characterize the propagation delay for different load
capacitances in order to create a linear delay model. Finally, we will
write the front-end and back-end views for our standard-cell inverter in
two YAML files. We will also implement three _auxillary standard cells_
(i.e., TIEHI, TIELO, FILL).

We recommend implementing all six views for one standard cell before
moving on to the next standard cell. Consider having each student
implement all six views for different standard cells.

### 2.1. Behavioral View

Start by working out the truth table for a standard cell. Once you have
your truth table, then implement the logic equation which corresponds to
this truth table in a Verilog module in `stdcells.v`. The Verilog module
must have the exact module name and port names as what is used in the
other views. For example, the interface for the INVX1 standard cell is as
follows.

```verilog
module INVX1
(
  input  A,
  output Y
);

...

endmodule
```

The Verilog implementation should use a single assign statement with
simple bitwise operators.

We have provided test benches to test each Verilog module. The test
benches are structured as shown below.

```verilog
module Top;

  //----------------------------------------------------------------------
  // Instantiate design under test
  //----------------------------------------------------------------------

  logic A;
  logic Y;

  INVX1 dut
  (
    .A (A),
    .Y (Y)
  );

  //----------------------------------------------------------------------
  // check
  //----------------------------------------------------------------------

  task check
  (
    input logic A_,
    input logic Y_
  );

    A = A_;

    #1;

    $display( "%d: %b > %b", 8'($time), A, Y );

    if ( Y !== Y_ ) begin
      ... print error message ...
      $finish();
    end

  endtask

  //----------------------------------------------------------------------
  // main
  //----------------------------------------------------------------------

  initial begin

    $dumpfile("INVX1-test.vcd");
    $dumpvars();
    $display("");

    //     A     Y
    check( 1'b0, 1'b1 );

    $display("");
  end

endmodule
```

The `check` task is used to set the inputs and check that the output is
as expected. We provide you a single check for each standard cell. You
are reponsible for adding additional checks to ensure you have
exhaustively tested all inputs, i.e., you need one check for every row in
your truth table. You can run a test bench as follows.

```bash
% cd ${HOME}/ece6745/project1-groupXX/stdcells/build
% iverilog -Wall -g2012 -I .. -o INVX1-test ../verilog-test/INVX1-test.v
% ./INVX1-test
```

### 2.2. Schematic View

Start by drawing the schematic view and then sizing the transistors
appropriately based on whether you are targetting equal rise and fall
times and your target drive strength. Once you have your drawing, then
implement this schematic view in a SPICE subcircuit in `stdcells.sp`. The
SPICE subcircuit must have the exact subcircuit name and port names as
what is used in the other views. For example, the interface for the INVX1
standard cell is as follows.

```
.SUBCKT INVX1 A Y VDD VSS

...

.ENDS INVX1
```

Recall from from lecture that the SPICE syntax is as follows:

 - PMOS: `M_P <D> <G> <S> <B> PMOS L=<length>U W=<width>U`
 - NMOS: `M_N <D> <G> <S> <B> NMOS L=<length>U W=<width>U`

For standard cells with more than one NMOS and PMOS. You will need to
make sure each transistor has a unique name (e.g., `M_N1`, `M_N2`, etc).
If your schematic view needs a temporary internal wire/net, simply use a
new name like `net1`; SPICE will assume this implicitly refers to a new
temporary internal wire/net.

Once you have a completed the schematic view, you will now simulate it
with TinyFlow-Ngspice. The following shows the syntax of how to use the
new `tinyflow-ngspice` wrapper script.

```bash
tinyflow-ngspice --spice=<SPICE_FILE> --cell=<CELLNAME> --cload=<CLOAD> \
  --inputs=<INPUTS_SPEC>
```

Let's break down each input (all four are required):

 - `--spice=<SPICE_FILE>`: replace `<SPICE_FILE>` with the path to
   `stdcells.sp` where you have defined your SPICE schematic (e.g.,
   `../stdcells.sp`)

 - `--cell=<CELLNAME>`: replace `<CELLNAME>` with the name of your cell
   to test (e.g., `INVX1`)

 - `--cload=<CLOAD>`: replace `<CLOAD>` with the capacitance to put on
   the output pin of your cell, you can use regular integers followed by
   a unit suffix (e.g., `10f` for 10 femtofarads)

 - `--inputs=<INPUTS_SPEC>`: replace `<INPUTS_SPEC>` with a string
   surrounded by quotes of the following form:
   `<PIN1>:<VAL0>-<VAL1>...<VALN>;<PIN2>:<VAL0>-<VAL1>...<VALN>` where
   `<PINN>` specifies one of the input pins for your cell (e.g. A, B,
   etc.), and `<VALN>` specifies a 0 or 1 to assert on that input pin.
   Each of these values are separated by a dash (-), such that each input
   value will be set for one timestep (2ns) before the next value is set.
   All input pins and their corresponding values are separated by a
   semicolon (;), and input pin names are separated from their values
   list by a colon (:). All input pins as specified by your standard cell
   must be present, and they must all have the same number of
   transitions.

Use TinyFlow-Ngspice to simulate the INVX1 standard cell with a 0fF load
and an input toggling between zero and one.

```bash
% cd ${HOME}/ece6745/lab2/stdcells/build
% tinyflow-ngspice --spice=../stdcells.sp --cell=INVX1 --cload=0f \
    --inputs="A:0-1-0-1-0"
```

When the command has completed, you will see an output to the console
specifying the transitions of both the input and output pins of your
standard cell throughout the simulation. It will also print rising and
falling delays if captured from each input pin to the output; this will
be discussed later when characterizing the cell. Additionally, your build
directory will contain a new subdirectory `ngspice-results/` with more
subdirectories within it named with a concatenation of the inputs you
specified. Each of these subdirectories will contain the following:

 - `-tb.sp`: the Spice testbench used to simulate your schematic
 - `.csv`: CSV data generated by Ngspice for each pin of your standard cell
 - `.png`: an image plotting the voltage of each pin of your standard cell over
   time
 - `.txt`: a text file containing the same output as what was printed to the
   console

Use TinyFlow-Ngspice to verify functionality of the schematic view for
each standard cell. Look at the text output and the waveform plot.

```bash
% cd ${HOME}/ece6745/lab2/stdcells/build
% code ngspice-results/INVX1-50f-stdcells-A_0_1_0_1_0/INVX1-50f-stdcells-A_0_1_0_1_0.png
```
### 2.3. Layout View

Start by drawing a stick diagram for your layout to make a plan for how
you want to place your transitors and use polysilicon and metal 1 to
connect these transistors together. Once you have a stick diagram you can
check to make sure your transistors will fit in a fixed hight standard
cell. If the transistors will not fit then you need to either consider
using fingers OR supporting non-equal rise and fall times and/or reducing
the drive strength. If you do use fingers you do _not_ need to go back
and add fingers to the schematic view. Klayout LVS is smart enough to
know that a wide transistor in the schematic can map to fingers in the
layout.

Once you are happy with your stick diagram, go ahead and use KLayout to
draw the corresponding standard cell in `stdcells.gds`. When switching to
a new cell in the KLayout GUI, make sure to right-click it from the
_Cells_ panel on the left and select _Show As New Top_. Try to minimize
the width of your standard cell; when you are done you can shrink it to
take the minimum number of _sites_ (i.e., vertical routing tracks). Use
DRC and LVS to complete physical verification of your layout.

### 2.4. Extracted Schematic View

The extracted schematic view is generated automatically when you run LVS.
You will need to copy the contents of the generated SPICE file into
`stdcells-rcx.sp`. Now you can use TinyFlow-Ngspice to do functional
verification and timing characterization of your standard cell.

Start by creating a spreadsheet with one row for every possible input
transition. For example, a two-input standard cell (i.e., NAND2X1 and
NOR2X1) would use the following table:

| A    | B    | Y    | intercept | slope |
|------|------|------|-----------|-------|
| 0    | 0->1 |      |           |       |
| 0    | 1->0 |      |           |       |
| 1    | 0->1 |      |           |       |
| 1    | 1->0 |      |           |       |
| 0->1 | 0    |      |           |       |
| 1->0 | 0    |      |           |       |
| 0->1 | 1    |      |           |       |
| 1->0 | 1    |      |           |       |

A three-input standard cell (i.e., AOI21X1) needs a table with 24 rows.
For each row, run a simulation using TinyFlow-Ngspice with a load
capacitance of 10fF and record what happens to the output in the table
(i.e, does it stay at 0, stay at 1, transition from 0 to 1, transition
from 1 to 0).

If the output transitions, firt ensure the propagation delay measurements
are reasonable. You should be anywhere from 10s of ps to 100s of ps. Then
rerun the TinyFlow-Ngspice simulation with a load capacitance of 20fF and
then 30fF. Record the propagation delay for each simulation. You now have
three points for load capacitance and propagatoin delay. Then use a
spreadsheet to determine a linear regression through these three points.
Confirm that the linear regression is a good fit, and record the
corresponding intercept (i.e., parasitic delay in ps) and slope (i.e.,
load delay factor in ps/fF) in the table.

Here is an example for how to generate the data necessary to analyze the
fall time of the INVX1 standard cell.

```bash
% cd ${HOME}/ece6745/lab2/stdcells/build

% tinyflow-ngsice --spice=../stdcells-rcx.sp --cell=INVX1 --cload=10fF \
    --inputs="A:0-1"

% tinyflow-ngsice --spice=../stdcells-rcx.sp --cell=INVX1 --cload=20fF \
    --inputs="A:0-1"

% tinyflow-ngsice --spice=../stdcells-rcx.sp --cell=INVX1 --cload=30fF \
    --inputs="A:0-1"
```

Once you are done look at all of your data. Our goal is to create a
single path- and value-independent linear model for the propagation delay
(i.e., the worst case delay through the standard cell across all possible
input transitoins). Find the row with the maximum parasitic delay and
then check that the corresponding load delay factor is either also the
maximum or at least very close to the maximum across all rows. We will
use this parasitic delay and load delay factor for the linear delay model
of the standard cell.

### 2.5. Front-End View

The front-end view is stored in the `stdcells-fe.yml` file. This YAML
file will use much of the information we have obtained in previous steps,
and combine it together for the front-end tools to use. This view
primarily contains timing, logical, and basic area information for each
cell.

The following is a template for the front-end view for the INVX1
standard cell.

```
 - name: INVX1
   area_cost: 0 # lambda^2

   pins:

     - name: A
       type: input
       cgate: 0 # fF

     - name: Y
       type: output
       function: # tree of generic gates

   parasitic_delay:   0 # ps
   load_delay_factor: 0 # ps/fF

   patterns:
     - # list of equivalent trees of generic NAND and INV gates
```


Fill in the `area_cost` value as the area of the standard cell in
lambda-squared, starting from the bottom-left corner of the intersecting
black lines indicating the horizontal and vertical tracks, and ending at
the top-right corner at the intersection of the black lines for the
horizontal and vertical tracks.

Each cell also has pin information under the "pin" key. The pin
information itself will be a list of data for each input and output pin,
where each pin has a name and type (input or output).

For input pins, you will need to specify a value for the gate capacitance
(`cgate`) in fF associated with that pin. The calculation for gate
capacitance is as follows:

```
Cgate = 1.60 * (total gate width in lambda) * 0.09
```

The value of 1.60 fF/um is a reasonable value for a generic 180nm
process. You need to convert it to just units of fF by multiplying by the
total gate width in um associated with the given input pin. You will need
to find this total gate width in units of lambda by looking at your
layout and comparing against your SPICE schematic. The multiplication by
0.09 is to convert lambda to microns.

For output pins, you will need to specify the function which determines
the logical functionality of this pin as a function of the input pins.
You can specify it using a tree of _generic gates_. Here is a list of the
valid generic gates:

 - `BUF`, `NOT`, `INV`
 - `AND2`, `OR2`, `XOR2`
 - `NAND2`, `NOR2`, `XNOR2`

So for example, here is an example tree for a more complicated
three-input standard cell: `NAND2(INV(A),OR(INV(B),C))`.

You will need to fill in your values for `load_delay_factor` and
`parasitic_delay` as calculated when simulating the extracted schematic
view. **Note that `load_delay_factor` MUST be in units of ps/fF, and
`parasitic_delay` MUST be in units of ps. Go back and double check you
performed the conversions if necessary.**

Finally, you will need to specify a list of patterns **using only INV()
and NAND2() logical gates** that matches the functionality of your gate
for all of its inputs.

### 2.6. Back-End View

The back-end view is stored in the `stdcells-be.yml` file. This YAML file
will use much of the information we have obtained in previous steps, and
combine it together for the back-end tools to use. This file primarily
contains physical data for each cell including width, height, and pin
locations.

We give you the following template for the back-end view for the INVX1
standard cell.

```
  - name: INVX1
    size:
      width:  0 # lambda
      height: 0 # lambda
    pins:
      - name: A
        loc:  (0,0) # (x,y) lambda
      - name: Y
        loc:  (0,0) # (x,y) lambda
```

Each cell has a "size" sub-dictionary, with keys for width and height.
Fill these values in using the same lower-left and upper-right track
intersection points used for calculating area in the front-end YAML.
These width and height values should therefore be in units of lambda.
Next is the pin list, similar again to the front-end view. In this file,
each pin once again has a name, but also an X and Y location. Look at
your layout, and get the X and Y location of the pin marker for each pin
relative to the lower-left track intersection. These values should once
again be in units of lambda.

3. Optional Extensions
--------------------------------------------------------------------------

Students are welcome to add another standard cell to their standard cell
library. Students should focus on adding a combinational logic gate since
TinyFlow does not support sequential logic gates (e.g., flip-flops).
Students should not implement a gate with increased drive strength (e.g.,
INVX2, INVX4) since TinyFlow cannot take advantage of these larger cells.
Some ideas include:

 - BUF
 - NAND3X1, NOR3X1, NAND4X1, NOR4X1
 - AND2X1, OR2X1
 - OAI21X1, AOI22X1, OAI22X1
 - XOR2, XNOR2

Students will need to add this new gate to all six views.

4. Project Submissions
--------------------------------------------------------------------------

To submit your code you simply push your code to GitHub. You can push
your code as many times as you like before the deadline. Students are
responsible for going to the GitHub website for your repository, browsing
the source code, and confirming the code on GitHub is the code they want
to submit is on GitHub. Be sure to verify your code is passing all of
your simulations on `ecelinux`.

Here is how we will be testing your final code submission for Part A.
First, we will create a build directory.

```bash
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/project1-groupXX
% mkdir -p project1-groupXX/stdcells/build
% cd project1-groupXX/stdcells/build
```

We will check the behavioral views are functionally correct.

```bash
% iverilog -Wall -g2012 -I .. -o INVX1-test ../verilog-test/INVX1-test.v
% ./INVX1-test

% iverilog -Wall -g2012 -I .. -o NAND2X1-test ../verilog-test/NAND2X1-test.v
% ./NAND2X1-test

% iverilog -Wall -g2012 -I .. -o NOR2X1-test ../verilog-test/NOR2X1-test.v
% ./NOR2X1-test

% iverilog -Wall -g2012 -I .. -o AOI21X1-test ../verilog-test/AOI21X1-test.v
% ./AOI21X1-test

% iverilog -Wall -g2012 -I .. -o TIEHI-test ../verilog-test/TIEHI-test.v
% ./TIEHI-test

% iverilog -Wall -g2012 -I .. -o TIELO-test ../verilog-test/TIELO-test.v
% ./TIELO-test
```

We will check if the schematic views are functionally correct.

```bash
% tinyflow-ngspice --spice=../stdcells.sp --cell=INVX1   --cload=0f \
                   --inputs="A:0-1-0-1"

% tinyflow-ngspice --spice=../stdcells.sp --cell=NAND2X1 --cload=0f \
                   --inputs="A:0-1-0-1;B:0-0-1-1"

% tinyflow-ngspice --spice=../stdcells.sp --cell=NOR2X1  --cload=0f \
                  --inputs="A:0-1-0-1;B:0-0-1-1"

% tinyflow-ngspice --spice=../stdcells.sp --cell=AOI21X1 --cload=0f \
                  --inputs="A:0-1-0-1-0-1-0-1;B:0-0-1-1-0-0-1-1;C:0-0-0-0-1-1-1-1"
```

We will check to make sure the layout view is DRC and LVS clean.

```bash
% tinyflow-batch-drc
% tinyflow-batch-lvs
```

We will check if the extracted schematic views are functionally correct.

```bash
% tinyflow-ngspice --spice=../stdcells-rcx.sp --cell=INVX1   --cload=0f \
                   --inputs="A:0-1-0-1"

% tinyflow-ngspice --spice=../stdcells-rcx.sp --cell=NAND2X1 --cload=0f \
                   --inputs="A:0-1-0-1;B:0-0-1-1"

% tinyflow-ngspice --spice=../stdcells-rcx.sp --cell=NOR2X1  --cload=0f \
                  --inputs="A:0-1-0-1;B:0-0-1-1"

% tinyflow-ngspice --spice=../stdcells-rcx.sp --cell=AOI21X1 --cload=0f \
                  --inputs="A:0-1-0-1-0-1-0-1;B:0-0-1-1-0-0-1-1;C:0-0-0-0-1-1-1-1"
```

We will also simulate all possible transistors for all standard cells. We
will check your front-end and back-end view values to ensure they are
reasonable and match your other views.

