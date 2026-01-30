
ECE 6745 Project 1: TinyFlow Tape-Out<br>TinyFlow Standard Cells
==========================================================================

In this project, students will build their own TinyFlow, a very simple
standard-cell-based flow. They will develop four standard cells in TSMC
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
use VS Code to log into a specific ecelinux server, source the setup
script, and clone your remote repository from GitHub:

```bash
% source setup-ece6745.sh
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
% cd ${HOME}/ece2300/groupXX
% git pull
% tree
```

where `XX` should be replaced with your group number. Your repo contains the
following files for the views and simulation scripts for each standard cell:

```
.
└── stdcells/
    ├── verilog-sim/
    │   ├── AOI21X1-sim.v
    │   ├── INVX1-sim.v
    │   ├── NAND2X1-sim.v
    │   ├── NOR2X1-sim.v
    │   ├── TIEHI-sim.v
    │   └── TIELO-sim.v
    ├── stdcells-be.yml
    ├── stdcells-fe.yml
    ├── stdcells-rcx.yml
    ├── stdcells.gds
    ├── stdcells.sp
    └── stdcells.v
```

1. Standard-Cell Library
--------------------------------------------------------------------------

In this part, you will need to implement the following seven standard
cells:

 - **INVX1:** Canonical minimum-sized inverter, should implement logic
     function $Y = \overline{A}$, should have same drive strenght as
     INVX1

 - **NAND2X1:** Two-input NAND gate implementing logic function $Y =
     \overline{AB}$, should have same drive strenght as INVX1

 - **NOR2X1:** Two-input NOR gate implementing logic function $Y =
     \overline{A + B}$, should have same drive strength as INVX1

 - **AOI21X1:** Three-input AND-OR-INV gate implementing logic function
     $Y = \overline{AB + C}$, should have same drive strength as INVX1

 - **TIEHI:** Tie high cell, connects output to logic one, should use a
     weak PMOS to pull-up the output to VDD, gate of weak PMOS should be
     connected to a diode-connected NMOS

 - **TIELO:** Tie low cell, connects output to logic zero, should use a
     weak NMOS to pull-down the output to ground, gate of weak NMOS
     should be connected to a diode-connected PMOS

 - **FILL:** One-track wide filler cell, used to fill in blank space
     between standard cells

For each standard cell we must create the following six views:

 - **Behavioral View:** Logical function of the standard cell, used for
     gate-level simulation

 - **Schematic View:** Transistor-level representation of standard cell,
     used for functional verification and layout-vs-schematic

 - **Layout View:** Layout of standard cell, used for design-rule
     checking (DRC), layout-vs-schematic (LVS), resistance/capacitance
     extraction (RCX), and fabrication

 - **Extracted Schematic View:** Transistor-level representation with
     extracted resistance and capacitances, used for layout-vs-schematic
     (LVS) and timing characterization

 - **Front-End View:** High-level information about standard cell
     including area, input capacitance, logical function, and delay
     model; used in synthesis

 - **Back-End View:** Low-level information about standard cell including
     height, width, and pin locations; used in placement and routing

We will be using the following TinyFlow standard-cell design flow.

![](img/lab02-flow.png)

We will by begin by writing the behavioral view in Verilog and verifying
its functionality using a Verilog test bench and Icarus Verilog. We will
then write the schematic view in SPICE and verify its functionality using
a SPICE test bench and Ngspice. We will then use the KLayout design
editor to create the layout, perform a design-rules check (DRC), perform
a layout vs. schematic check (LVS), and generate an extracted schematic.
We will re-simulate the extracted transistor-level schematic using
Ngspice to characterize the propagation delay for different load
capacitances in order to create a linear delay model. Finally, we will
write the front-end and back-end views for our standard-cells in
two YAML files.

We recommend implementing all six views for one standard cell before moving on
to the next standard cell. Consider having each student implement all six views
for different standard cells. Follow the same steps as in lab 2 to implement
each standard cell.

### 2.1. Behavioral View

Think critically about the truth table for each standard cell. Are you testing
all possible inputs? 

### 2.2. Schematic View

Think critically about transistor sizing. You should aim to create balanced
cells that have equivalent drive as the canonical minimum-sized 2:1 inverter.
You should aim to have roughly equal rise and fall propagation delays.

### 2.3. Layout View

Draw stick diagrams for your layouts before drawing on KLayout. Try to route
connections to minimize the number of sites your standard cell takes up, and
then shrink it to the minimum number of sites. **Be sure to reference the lab 2
handout to learn how to run batch DRC and LVS scripts for all of your standard
cells without having to open the KLayout GUI!**

### 2.4. Extracted Schematic View

Ensure your propagation delay measurements are reasonable. You should be
anywhere from 10s of ps to 100s of ps. Be sure to double-check your extracted
schematic by re-simulating it. **For multi-input cells, `tinyflow-ngspice` will
print out multiple input->output rising and falling transition times, you will
want to pick the maximum out of all of these times to use for the delay value
for the given load capacitance.**

### 2.5. Front-End View

Think critically about the INV-NAND patterns for each cell. There may be
multiple!

### 2.6. Back-End View

Be sure to update pin locations here if you change anything in your layout.

3. Project Submissions
--------------------------------------------------------------------------

We will be running various commands to test each aspect of your standard cells:

 - Behavioral view testbenches: ensure all of your testbenches pass for all
   values of your cells' corresponding truth tables
 - Schematic view testbenches: we will run `tinyflow-ngspice` for various values
   of `cload` between 0f and 100f for all truth table values for your cells
   pre-extracted Spice schematics. We will also verify your Spice schematics
   include properly-sized transistors for balanced drive
 - Layout view: we will run DRC and LVS batch scripts to ensure all of your
   cells are DRC and LVS-clean. We will also verify your layouts include
   properly-sized transistors for balanced drive
 - Extracted schematic view: we will re-simulate your extracted Spice views
   using `tinyflow-ngspice` with all truth table values for your standard cells.
   We will also ensure your delay values are within reason as discussed above
   (10s to 100s of ps)
 - Front-end view: we will check your front-end view values to ensure they are
   reasonable and match your other views
 - Back-end view: we will check your back-end view values to ensure they are
   reasonable and match your other views
