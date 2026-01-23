
ECE 6745 Lab 1: Full-Custom Inverter Design
==========================================================================

In this lab, we will be designing a basic CMOS inverter from scratch by writing
and simulating a reference schematic, using the KLayout design editor to create
the layout, performing a design-rules check (DRC), performaing a layout vs.
schematic check (LVS), and re-simulating the RC-extracted inverter. You will
also be tasked with implementing a CMOS buffer based on your work on the
inverter.

0. BEFORE YOU BEGIN
--------------------------------------------------------------------------

**Be sure to fully read through [tut0](ece6745-tut00-remote-access.md),
[tut1](ece6745-tut01-linux.md), and [tut2](ece6745-tut02-git.md) before
beginning this lab.**

On the ECELinux, source the course setup scripts and clone the repo. Be sure to
open your remote desktop GUI as well.

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% mkdir -p $HOME/ece6745
% cd $HOME/ece6745
% git clone git@github.com:cornell-ece6745/ece6745-lab1.git lab01
% cd lab01
% TOPDIR=$PWD
```

Take a minute to look through the included files:

```
.
└── lab01/
    ├── buf/
    │   ├── buf-rcx-sim.sp (extracted buffer simulation script template - TODO FOR YOU)
    │   ├── buf-sim.sp (buffer simulation script template - TODO FOR YOU)
    │   ├── buf.gds (blank buffer layout file - TODO FOR YOU)
    │   └── buf.sp (reference buffer spice file template - TODO FOR YOU)
    ├── inv/
    │   ├── inv-rcx-sim.sp (extracted inverter simulation script template - TODO FOR YOU)
    │   ├── inv-sim.sp (inverter simulation script template - TODO FOR YOU)
    │   ├── inv.gds (blank inverter layout file - TODO FOR YOU)
    │   └── inv.sp (reference inverter spice file template - TODO FOR YOU)
    └── pdk/
        ├── d25/
        │   └── tinyflow-180nm.lyd25 (script for viewing your layout in a more 3-dimensional way)
        ├── drc/
        │   └── tinyflow-180nm.lydrc (DRC script for ensuring all design rules are followed for your layout)
        ├── lvs/
        │   └── tinyflow-180nm.lylvs (LVS script for ensuring your layout matches the reference spice schematic)
        ├── klayoutrc (setup file for KLayout with all necessary settings - sourced automatically by setup script)
        ├── tinyflow-180nm.lyp (layer information for the tinyflow-180nm process)
        └── tinyflow-180nm.lyt (technology manager information for the tinyflow-180nm process)
```

1. KLayout Introduction
--------------------------------------------------------------------------

[KLayout](https://www.klayout.de/) is a powerful, open-source layout tool that
allows users to both view and edit layout files. Additionally, it can perform
design-rules checking, layout vs. schematic checking, and RC extraction among
other features. In this lab, we will be using all of these features so that you
can become familiar with making your own custom circuits!

To open on the ECELinux server, first open KLayout in edit mode (`-e`) with the
blank canvas for the inverter your are going to draw (`inv/inv.gds`)

```bash
% cd $TOPDIR
% klayout -e inv/inv.gds
```

There will be a pop-up the first time you open KLayout asking about whether to
use full-hierarchy mode. Select the radio button for "Don't show this window
again" and then select "Yes" in the bottom-right corner.

The KLayout GUI should now look like this:

![](img/lab01-klayout-blank.png)

Take a minute to become familiar with the user interface (some items are omitted
as they are not used for our class):

 - Toolbar (upper) - includes back/forward (undo/redo) buttons, as well as
   buttons for selecting various tools such as selecting a feature, moving a
   feature, drawing a feature, measuring a feature (ruler), etc.
 - Cells window (upper left) - displays names of all the cells in the current
   layout file that is open. In this case, we only have one cell in our layout
   file, but we will have more cells in future labs/projects which constitute an
   entire *library* of cells! Right click -> Show As New Top to select the cell
   to view in the main window
 - Layers window (middle right) - displays all available layers which elements
   can be placed on, take a minute to correlate these names with what you have
   seen in lecture. You can right click on a layer to set visibility options for
   it and other layers - **this is very useful accurately and quickly editing
   layouts!**

You should be able to scroll in on the main layout window and see a grid of
dotted lines (as in the image above), this represents the **lambda grid**, where
the side length of each such box is 1 lambda (0.09um in our case). This
represents the minimum unit on which you can draw a feature. If you scroll out,
the lines should disappear and a grid of interspersed dots only on the
intersection of these boxes should be present instead. **When drawing a feature
with a specific dimension of lambda, be sure that you are counting on the boxes
contained within the dotted lines, not the interspersed dots on the
intersections!** The intersection of the two solid lines represents the origin.

!!! question "Activity 1: Open KLayout"

    Use the information in this section and the above images to open KLayout
    in editor mode on the blank `inv/inv.gds` layout canvas.


2. Writing A Spice Schematic for Your Inverter and Simulating It
--------------------------------------------------------------------------

Normally, circuit designers will write a reference **Spice schematic** of the
intended circuit first and then simulate it to ensure the high-level
functionality is correct before moving onto layout. Spice is a language used to
define how various hardware elements are connected, along with any parameters
such as RC parasitics for devices. In this schematic, we will not include
extracted RC parasitics, we will instead extract these values from our layout
and re-simulate using the same testbench later in the lab.

Open up the blank reference Spice file for your inverter (`inv/inv.sp`). Add a
line for each device (transistor) in your inverter circuit inside the
sub-circuit using the following format:

 - For the PMOS: `M_P <D> <G> <S> <B> PMOS L=<length>U W=<width>U`
 - For the NMOS: `M_N <D> <G> <S> <B> NMOS L=<length>U W=<width>U`

where D, G, S, and B represent the drain, gate, source, and base connections for
the transistor, respectively. You should replace these with the correct pin name
(A, Y, VDD, VSS) for those connections as per your knowledge of how CMOS
transistors are connected from lecture. You should also fill in numerical values
for the length and width of the transistor. **For this inverter, we are defining
the width to be 8-lambda wide**, this is a design requirement we are providing
to you, but custom-circuit designers will test their design for a wide variety
of parameters, including modifying this width to achieve their design goals of
power, performance, or area. **Additionally, the definition of lambda states
that the gate-width (transistor length) for a given process is equal to
2-lambda.** Convert the lambda measurement of these values to micrometers using
our conversion factor of 1 lambda = 0.09um (the U suffix on the end of the
values denotes that the numerical value should be interpreted in micro-units).
Save the file.

!!! question "Activity 2a: Write the reference Spice schematic for your
inverter"

    Use the information in this section to fill in the blank Spice schematic 
    in `inv/inv.sp`.

Now that we have written our reference Spice schematic, we need to test it to
make sure it is functionally correct. To check this functionality, we provide a
**Spice deck**, or testbench, for our Spice circuit, which will simulate the
circuit given specific input stimuli. We run the simulation using an open-source
tool called [Ngspice](https://ngspice.sourceforge.io/). Go ahead and paste in
your Spice circuit into the `inv/inv-sim.sp` file where it says to. Take a
minute to browse the testbench as well and understand how it works.

**Additionally, we need to make the following changes to the pasted Spice
circuit to ensure it is compatible with the transistor models we will be
using:**

 - Replace the `PMOS` identifier with `sky130_fd_pr__pfet_01v8`
 - Replace the `NMOS` identifier with `sky130_fd_pr__nfet_01v8`
 - Replace the `M_P` identifier with `XM_P`, and the `M_N` identifier with
   `XM_N`

To allow the simulation to work, we need to provide pre-characterized device
models for the transistors, and we use the open-source
[Sky130](https://skywater-pdk.readthedocs.io/en/main/) models. The above changes
modify the Spice to work with these models. 

We are now ready to run our simulation. Execute the following in your terminal
(it should take a few seconds to run):

```bash
% cd $TOPDIR
% ngspice inv/inv-sim.sp
```

The simulation will open a new plot window in your remote desktop viewer,
plotting both the input voltage (at A) vs. time as well as the output voltage
(at Y) vs. time.

!!! question "Activity 2b: Simulate your inverter schematic"

    Use the information in this section to simulate the inverter using Ngspice.
    View the output plot and save a picture of it. Does the behavior of the 
    inverter look correct? Think about it's high-level functionality.

3. Laying Out and Performing DRC on a PMOS
--------------------------------------------------------------------------

Now that you have written your schematic, let's go ahead and start laying out
our PMOS for the inverter!

Take some time now to draw the inverter in the following image **as shown
exactly** in your `inv/inv.gds` file. To draw a new feature, select the layer
from the Layers window, and then click the Box tool from the upper toolbar,
click to set one corner, and then click again to set the opposite corner.
**Notice how the width of the transistor is 8-lambda (boxes) as we mentioned
above, and the gate width (transistor length) is 2-lambda. This matches our
expectations from writing the schematic!**

Compare the relative spacing and widths of the various features to what is shown
in the [design rules manual (DRM)](ece6745-design-rule-manual.md). We can see
how all such spacings and widths are at least the minimum requirement for the
associated rule in the DRM.

![](img/lab01-klayout-pmos.png)

!!! question "Activity 3a: Lay out the PMOS"

    Use the information in this section and the above images to lay out the
    PMOS.

Let's go ahead and run DRC to ensure these requirements are met, go to Tools ->
DRC -> select the `tinyflow-180nm.lydrc` file from the list (it should be the
only file listed under "Edit DRC Script").

![](img/lab01-klayout-enter-drc.png)

A new window should open with the DRC results, the left side shows all the
performed DRC checks, with the numbers corresponding to the associated rules in
the DRM. If your design is DRC-clean, then all the numbers should be green and
the topcell name under "By Cell" should be green.

![alt text](img/lab01-klayout-drc-browser.png)

If the topcell name is black, scroll down to find the rule number(s) that is
also black and cross-reference it with the DRM. If you click on the violated
rule, details will also be populated in the Info window in the bottom right.
Edit your layout to fix the DRC *violation*.

![alt text](img/lab01-klayout-drc-vio.png)

!!! question "Activity 3b: Perform DRC on the PMOS"

    Use the information in this section and the above images to perform DRC on 
    the PMOS.

4. Laying Out the Full Inverter
--------------------------------------------------------------------------

Let's go ahead and do the layout for the full inverter according to the below
image.

![alt text](img/lab01-klayout-full-inverter.png)

Feel free to delete your PMOS from the previous step, or add onto it.
Overlapping boxes on the same layer does not matter in the end, as it is all
part of one coherent polygon in the final layout! **Be sure to follow the
dimensions exactly! Otherwise you may fail DRC!**

Once your layout is finished, add the pin labels on the `metal1 label` layer by
selecting the layer and then selecting the Text tool from the toolbar. Once the
label is placed on appropriate metal1 net, double-click it to edit the name. The
pin names should be as follows:

 - A (input connecting to the gates of both transistors)
 - Y (output from the drains of both transistors)
 - VDD (positive voltage power rail)
 - VSS (negative voltage power rail)

!!! question "Activity 4a: Lay out the full inverter"

    Use the information in this section and the above images to lay out the
    full inverter.

After the pin labels are added, run DRC as before to ensure your design passes
all design rule checks. **Ensure your design is DRC-clean before moving onto the
next step! Be sure to save the layout as well by going to File -> Save, DO NOT
DO CTRL+S AS IT BREAKS THE VIEW!**

You can also view your inverter in a semi-three-dimensional view called 2.5D.
**First, make sure your layout is fully-visible in the layout viewer.** Then, go
to Tools -> 2.5d View -> select the `tinyflow-180nm.lyd25` file under "Edit 2.5d
Script". A new window should pop up with a 2.5D viewer which you can use the
mouse to scroll around and view the inverter from different angles!

![alt text](img/lab01-klayout-25d.png)

!!! question "Activity 4b: Perform DRC on the inverter"

    Use the information in this section and the above images to perform DRC on 
    the inverter, as well as view it in 2.5D.

5. Performing LVS on the Full Inverter
--------------------------------------------------------------------------

Layout vs. schematic checks compare the layout you just created with the
reference Spice schematic you wrote earlier to ensure that the layout drawing
matches the intended high-level functionality. The LVS tool in KLayout will then
*extract* a Spice schematic from the drawn layout, along with RC parasitics, and
compare this extracted Spice schematic with the reference one.

Make sure your desired cell to check via LVS is active in the viewer (important
if multiple such cells are in the same layout file as will happen in later
projects and labs). Open the LVS tool in your KLayout window similarly to DRC,
but now selecting LVS from the Tools menu instead of DRC and selecting the
`tinyflow-180nm.lylvs` file to run the LVS script.

After running the script, and if LVS passes, you should see all green in the
window that pops up:

![alt text](img/lab01-klayout-lvs-browser.png)

If you see any red stop signs, this means LVS failed. You can view the
violations by clicking the drop-down arrows under Objects in the Cross Reference
window to see what is failing the check. Edit your layout and/or reference Spice
file to fix the violations.

Once all violations are resolved, take a minute to view the extracted Spice
output from the LVS tool in `inv/inv-rcx.sp`. This file looks similar to the
reference Spice file, except that it includes additional information for RC
parasitics in the parameters AS, AD, PS, and PD.

!!! question "Activity 5: Perform LVS on the inverter"

    Use the information in this section and the above images to write the 
    reference Spice circuit and perform LVS on the inverter. View the extracted 
    Spice netlist once LVS has passed.

6. Simulating the Inverter with Ngspice
--------------------------------------------------------------------------

Congratulations! You now have a fully DRC and LVS-clean inverter cell! For our
final step, we need to make sure that the layed-out inverter functionality
matches the original high-level functionality we defined in our Spice schematic.

Go ahead and paste in your extracted Spice circuit into the `inv/inv-rcx-sim.sp`
file where it says to.

**Additionally, we need to make the following changes to the pasted Spice
circuit to ensure it is compatible with the transistor models we will be
using:**

 - Replace the `PMOS` identifier with `sky130_fd_pr__pfet_01v8`
 - Replace the `NMOS` identifier with `sky130_fd_pr__nfet_01v8`
 - Replace the `M$1` identifier with `XM$1`, and the `M$2` identifier with
   `XM$2`

We are now ready to run our simulation. Execute the following in your terminal
(it should take a few seconds to run):

```bash
% cd $TOPDIR
% ngspice inv/inv-rcx-sim.sp
```

The simulation will open a new plot window in your remote desktop viewer,
plotting both the input voltage (at A) vs. time as well as the output voltage
(at Y) vs. time.

!!! question "Activity 6: Simulate the inverter"

    Use the information in this section and the above images to simulate the
    extracted inverter in Ngspice. How does this plot compare to the 
    pre-extracted simulation? Qualitatively, how does the rise time of the 
    output compare to the fall time qualitatively (compare this to your 
    lecture notes)?

7. Writing a Schematic, Laying Out, and Simulating a CMOS Buffer
--------------------------------------------------------------------------

Many students wonder why we can't just make a buffer by "flipping" the NMOS and
PMOS in the inverter. Let's go ahead and do just that! Do everything that you
did for the inverter but for a buffer (in the `buf` directory) by writing the
reference schematic and simulating it, laying it out, performing DRC and LVS,
and then re-running the simulation script for the extracted Spice. When doing
layout, you can start by copying-and-pasting your layout from the layout viewer
in `inv/inv.gds` to the blank layout in `buf/buf.gds` and redesigning it.

**When laying out the buffer, keep the base of the NMOS tied to VSS and the base
of the PMOS tied to VDD.**

!!! question "Activity 7: For the buffer, write the reference schematic and
simulate, lay out, perform DRC and LVS, and re-simulate the extracted Spice"

    Use the information in this section and the above images to write the 
    reference schematic and simulate, lay out, perform drc and lvs, and 
    re-simulate the extracted version of a CMOS buffer. Take a look at the 
    plots from the simulation, what is happening here and why? Save a picture 
    of the plot.
