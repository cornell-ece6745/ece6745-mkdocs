
ECE 6745 Lab 7: Commercial Back-End Flow
==========================================================================

In this lab, we will be discussing the back-end of the ASIC toolflow. More
detailed tutorials will be posted on the public course website, but this lab
will at least give you a chance to take a gate-level netlist through
place-and-route, simulate the final gate-level netlist, perform static timing
and energy analysis, as well as execute DRC and LVS. The following diagram
illustrates the tool flow we will be using in ECE 6745. Notice that the Synopsys
and Cadence ASIC tools all require various views from the standard-cell library
which part of the ASIC design kit (ADK).

![](img/tut07-asic-flow-back-end.png)

The "back-end" of the flow is highlighted in red and refers to the PyMTL
simulator, Synopsys DC, and Synopsys VCS:

 - We use **Cadence Innovus** to place-and-route our design, which means
   to place all of the gates in the gate-level netlist into rows on the
   chip and then to generate the metal wires that connect all of the
   gates together. We need to provide Cadence Innovus with similar
   abstract logical and timing views used in Synopsys DC. Cadence Innovus
   takes as input the `.lib` file which is the ASCII text version of a
   `.db` file. In addition, we need to provide Cadence Innovus with
   technology information in `.lef` and `.captable` format and abstract
   physical views of the standard-cell library in `.lef` format. Cadence
   Innovus will generate an updated Verilog gate-level netlist, a `.spef`
   file which contains parasitic resistance/capacitance information about
   all nets in the design, and a `.gds` file which contains the final
   layout. The `.gds` file can be inspected using the open-source Klayout
   GDS viewer. Cadence Innovus also generates reports which can be used
   to accurately characterize area and timing.

 - We use **Synopsys VCS** for back-annotated gate-level simulation.
   Gate-level simulation involves simulating every standard-cell gate and
   helps verify that the Verilog gate-level netlist is functionally
   correct. Fast-functional gate-level simulation does not include any
   timing information, while back-annotated gate-level simulation does
   include the estimated delay of every gate and every wire.

 - We use **Synopsys PrimeTime (PT)** to perform both static timing analysis and
   power-analysis of our design. Both of these require parasitic capacitance
   information for every net in the design (which comes from Cadence Innovus),
   and power-analysis additionally requires switching activity information for
   every net in the design (which comes from the back-annotated gate-level
   simulation). Synopsys PT puts the capacitance and clock frequency together to
   estimate setup and hold time on each path of the design, while also putting
   in switching activity and voltage to estimate the power consumption
   of every net and thus every module in the design.

![](img/lab7-drc-lvs.png)

The diagram above illustrates the DRC (design-rules check) and LVS
(layout-versus-schematic) verification flow. These checks are performed
after place-and-route to ensure the design is ready for manufacturing:

 - **DRC Flow**: The post-PNR GDS file is checked against the foundry's
   design rules using Mentor Calibre. The rules file (`.rule`) contains
   geometric constraints such as minimum metal width, minimum spacing
   between wires, minimum area for metal shapes, and enclosure rules for
   vias. Any violations are reported and can be viewed interactively
   using Calibre DRV (Design Review with Verification).

 - **LVS Flow**: The post-PNR Verilog netlist is first converted to
   SPICE format using `v2lvs`. Then Mentor Calibre extracts the circuit
   connectivity from the GDS layout and compares it against the SPICE
   netlist. This ensures that what was physically implemented matches
   the intended circuit connectivity. Mismatches (opens, shorts, missing
   connections) are reported and can be debugged interactively.

We consider DRC and LVS as part of "signoff," where we essentially
"sign-off" that the block is clean and ready to be taped out or
instantiated in a larger design.

Extensive documentation is provided by Synopsys and Cadence. We have
organized this documentation and made it available to you on the Canvas
course page:

 - <https://www.csl.cornell.edu/courses/ece6745/asicdocs>

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

The first step is to access `ecelinux`. Use Microsoft Remote Desktop to
log into a specific `ecelinux` server. Then use VS Code to log into the
same specific `ecelinux` server. Once you are at the `ecelinux` prompt,
source the setup script, source the GUI setup script, clone this
repository from GitHub, and define an environment variable to keep track
of the top directory for the project.

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% mkdir -p $HOME/ece6745
% cd $HOME/ece6745
% git clone https://github.com/cornell-ece6745/ece6745-lab7 lab7
% cd lab7
% TOPDIR=$PWD
```

To make it easier to cut-and-paste commands from this handout onto the
command line, you can tell Bash to ignore the `%` character using the
following command:

```bash
% alias %=""
```

2. Ripple Carry Adder
--------------------------------------------------------------------------

In this section, we will be pushing a four-bit ripple carry adder through
the commercial back-end flow. Recall that you already pushed this design
through the front-end flow in lab 6. In this lab, we will run the
front-end steps using provided runscripts and then manually execute
commands for the back-end steps.

### 2.1. Front-End Flow

We have provided run scripts that will reproduce the front-end steps you
learned about in the previous lab:

 - Step 1: PyMTL 2-State RTL Simulation
 - Step 2: Synopsys VCS 4-State RTL Simulation
 - Step 3: Synopsys DC Synthesis
 - Step 4: Synopsys VCS Fast-Functional Gate-Level Simulation

Let's execute each of these steps using the provided run scripts:

```bash
% cd $TOPDIR/asic/playground/addrc4b
% ./01-pymtl-rtlsim/run
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-ffglsim/run
```

Verify that your design passes all simulations. Take a look at the
synthesis reports:

```bash
% cd $TOPDIR/asic/playground/addrc4b
% less ./03-synopsys-dc-synth/area.rpt
% less ./03-synopsys-dc-synth/timing.rpt
```

### 2.2. Cadence Innovus for Place-and-Route

We will be running Cadence Innovus in a separate directory to keep the
input and output files separate.

```bash
% mkdir -p $TOPDIR/asic/playground/addrc4b/05-cadence-innovus-pnr
% cd $TOPDIR/asic/playground/addrc4b/05-cadence-innovus-pnr
```

**Constraint and Timing Input Files**

Before starting Cadence Innovus, we need to create two files which will
be loaded into the tool. The first file is a "multi-mode multi-corner"
(MMMC) analysis file. This file specifies what "corner" to use for our
timing analysis. Use VS Code to create a file named `setup-timing.tcl`:

```bash
% cd $TOPDIR/asic/playground/addrc4b/05-cadence-innovus-pnr
% code setup-timing.tcl
```

The file should have the following content:

```
create_rc_corner -name typical \
   -cap_table "$env(TSMC_180NM)/typical.captable" \
   -T 25

create_library_set -name libs_typical \
   -timing [list "$env(TSMC_180NM)/stdcells.lib"]

create_delay_corner -name delay_default \
   -library_set libs_typical \
   -rc_corner typical

create_constraint_mode -name constraints_default \
   -sdc_files [list ../03-synopsys-dc-synth/post-synth.sdc]

create_analysis_view -name analysis_default \
   -constraint_mode constraints_default \
   -delay_corner delay_default

set_analysis_view \
   -setup analysis_default \
   -hold analysis_default
```

Here is an explanation of each command:

 - **`create_rc_corner`**: Creates an RC (resistance/capacitance) corner
   that defines the interconnect parasitic characteristics. The
   `-cap_table` option loads the `.captable` file which contains
   information about the resistance and capacitance of every metal layer.
   The `-T 25` option specifies an operating temperature of 25 degrees
   Celsius. We are using the "typical" captable which represents average
   process conditions.

 - **`create_library_set`**: Creates a library set that groups together
   timing libraries. The `-timing` option loads the `.lib` file which
   contains timing information for each standard cell including
   input/output capacitance and delay from every input to every output.

 - **`create_delay_corner`**: Creates a delay corner by combining a
   library set with an RC corner. This represents a specific PVT
   (process, voltage, temperature) operating condition. In this case, we
   combine the typical library with the typical RC corner to create a
   "typical" delay corner.

 - **`create_constraint_mode`**: Creates a constraint mode that specifies
   the timing constraints for the design. The `-sdc_files` option loads
   the SDC (Synopsys Design Constraints) file generated during synthesis,
   which contains clock definitions, input/output delays, and other
   timing constraints.

 - **`create_analysis_view`**: Creates an analysis view by combining a
   constraint mode with a delay corner. An analysis view represents a
   complete timing scenario that the tool will use for optimization and
   analysis.

 - **`set_analysis_view`**: Tells Cadence Innovus which analysis views to
   use for setup time analysis (`-setup`) and hold time analysis
   (`-hold`). In this case, we use the same analysis view for both.

**Initial Setup and Floorplanning**

Now we can start Cadence Innovus. Note that we are using the Cadence
Innovus GUI so you will need to use Microsoft Remote Desktop.

```bash
% cd $TOPDIR/asic/playground/addrc4b/05-cadence-innovus-pnr
% innovus
```

We need to set various variables before starting to work in Cadence Innovus.
These variables tell Cadence Innovus the location of the MMMC file, the location
of the Verilog gate-level netlist, the name of the top-level module in our
design, the location of the `.lef` files, and finally the names of the power and
ground nets. Note that you cannot block-copy-and-paste these commands as we
cannot make an alias for "innovus>" in the Innovus shell, so you will need to
enter each line one at a time without the "innovus>" header.

```
innovus> set init_mmmc_file "setup-timing.tcl"
innovus> set init_verilog   "../03-synopsys-dc-synth/post-synth.v"
innovus> set init_top_cell  "AdderRippleCarry_4b"
innovus> set init_lef_file  [list "$env(TSMC_180NM)/apr-tech.tlef" "$env(TSMC_180NM)/stdcells.lef"]
innovus> set init_pwr_net   {VDD vdd}
innovus> set init_gnd_net   {VSS gnd}
```

We can now use the `init_design` command to read in the Verilog, set the
design name, setup the timing analysis views, read the technology `.lef`
for layer information, and read the standard cell `.lef` for physical
information about each cell used in the design.

```
innovus> init_design
```

Set the process node to 180nm for technology-specific optimizations:

```
innovus> setDesignMode -process 180
```

Limit routing and pin placement to be between layers 2 and 5. Metal 1 is
reserved for local interconnect within standard cells and metal 6 and
above are reserved for the power grid:

```
innovus> setDesignMode -bottomRoutingLayer 2
innovus> setDesignMode -topRoutingLayer    5
```

Ensure that the constraints applied to VIA generation prevent any DRC
violations:

```
innovus> setViaGenMode -ignore_viarule_enclosure false
innovus> setViaGenMode -optimize_cross_via true
innovus> setViaGenMode -ignore_DRC false
innovus> setViaGenMode -disable_via_merging true
```

Turn off signal integrity analysis and configure optimization settings:

```
innovus> setDelayCalMode -SIAware false
innovus> setOptMode -usefulSkew false
innovus> setOptMode -holdTargetSlack 0.010
innovus> setOptMode -holdFixingCells {BUFFD0BWP7T BUFFD10BWP7T BUFFD12BWP7T BUFFD1BWP7T BUFFD1P5BWP7T BUFFD2BWP7T BUFFD2P5BWP7T BUFFD3BWP7T BUFFD4BWP7T BUFFD5BWP7T BUFFD6BWP7T BUFFD8BWP7T}
innovus> set report_precision 4
```

We start by working on floorplanning. Use the `floorPlan` command to set
the dimensions for our chip. Since the ripple carry adder is a small
combinational design, we use small margins:

```
innovus> floorPlan -r 1.0 0.70 0.5 0.5 0.5 0.5
```

In this example, we have chosen the aspect ratio to be 1.0, the target
cell utilization to be 0.7, and we have added 0.5um of margin around the
top, bottom, left, and right of the chip.

Set pin placement constraints, this helps prevent DRC violations:

```
innovus> setPinConstraint -global -pinTemplate {2 3 4 5} -depth 0.73
```

**Power Routing**

Now we need to tell Cadence Innovus that `VDD` and `VSS` in the
gate-level netlist correspond to the physical pins labeled `VDD` and
`VSS` in the `.lef` files:

```
innovus> globalNetConnect VDD -type pgpin -pin VDD -all -verbose
innovus> globalNetConnect VSS -type pgpin -pin VSS -all -verbose
innovus> globalNetConnect VDD -type tiehi -pin VDD -all -verbose
innovus> globalNetConnect VSS -type tielo -pin VSS -all -verbose
```

Route the power and ground nets:

```
innovus> sroute -nets {VDD VSS}
```

Add well tap cells on the sides of the block to ensure good connection to the
well:

```
innovus> set_well_tap_mode -cell TAPCELLBWP7T
innovus> addWellTap -cellInterval 29.68
```

**Placement**

Place all of the standard cells:

```
innovus> place_design
```

You should be able to see the standard cells placed in the rows along
with preliminary routing to connect all of the standard cells together.
You can toggle the visibility of metal layers by pressing the number keys
on the keyboard.

Add tie-hi/tie-lo cells which are used to connect constant values to
either VDD or ground:

```
innovus> addTieHiLo -cell "TIEHBWP7T TIELBWP7T"
```

Place the input/output pins so they are close to the standard cells they
are connected to:

```
innovus> assignIoPins -pin *
```

**Signal Routing**

Set routing settings to fix antenna violations by adding antenna diodes:

```
innovus> setNanoRouteMode -route_detail_fix_antenna true
innovus> setNanoRouteMode -route_antenna_cell_name "ANTENNABWP7T"
innovus> setNanoRouteMode -route_antenna_diode_insertion true
```

Now use the `routeDesign` command to do a detailed routing pass:

```
innovus> routeDesign
```

Fix setup time violations by reducing the delay of the slow paths:

```
innovus> optDesign -postRoute -setup
```

Fix hold time violations by increasing the delay of the fast paths:

```
innovus> optDesign -postRoute -hold
```

Fix design rule violations:

```
innovus> optDesign -postRoute -drv
```

Extract the parasitic resistance and capacitances:

```
innovus> extractRC
```

**Final Output and Reports**

Add filler cells to complete each row of standard cells:

```
innovus> setFillerMode -core {FILL1BWP7T FILL2BWP7T FILL4BWP7T FILL8BWP7T FILL16BWP7T FILL32BWP7T FILL64BWP7T}
innovus> addFiller
```

Check the final physical design, ensure there are no reported violations:

```
innovus> verifyConnectivity
innovus> verify_drc
innovus> verify_antenna
```

Save the design and generate outputs:

```
innovus> saveDesign post-pnr.enc
innovus> saveNetlist post-pnr.v
innovus> rcOut -rc_corner typical -spef post-pnr.spef
innovus> write_sdf -recompute_delay_calc post-pnr.sdf
innovus> write_sdc post-pnr.sdc
```

Generate the final layout as a `.gds` file:

```
innovus> streamOut post-pnr.gds -units 1000 -merge "$env(TSMC_180NM)/stdcells.gds" -mapFile "$env(TSMC_180NM)/gds_out.map"
```

Generate timing and area reports:

```
innovus> report_timing -late  -path_type full_clock -net > timing-setup.rpt
innovus> report_timing -early -path_type full_clock -net > timing-hold.rpt
innovus> report_area > area.rpt
```

Exit Cadence Innovus:

```
innovus> exit
```

Open the final layout using Klayout:

```bash
% cd $TOPDIR/asic/playground/addrc4b/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 2.3. Synopsys PrimeTime for Static Timing Analysis

We use Synopsys PrimeTime to perform static timing analysis on the
post-place-and-route design.

```bash
% mkdir -p $TOPDIR/asic/playground/addrc4b/06-synopsys-pt-sta
% cd $TOPDIR/asic/playground/addrc4b/06-synopsys-pt-sta
% pt_shell
```

Set up the standard cell library:

```
pt_shell> set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
```

Read in the gate-level netlist and link the design:

```
pt_shell> read_verilog   "../05-cadence-innovus-pnr/post-pnr.v"
pt_shell> current_design AdderRippleCarry_4b
pt_shell> link_design
```

Read in the parasitic information:

```
pt_shell> read_parasitics -format spef ../05-cadence-innovus-pnr/post-pnr.spef
```

Set timing constraints. Since the ripple carry adder is purely
combinational, we constrain the delay from inputs to outputs:

```
pt_shell> set_max_delay 1.0 -from [all_inputs] -to [all_outputs]
pt_shell> set_max_transition 0.250 AdderRippleCarry_4b
pt_shell> set_driving_cell -lib_cell INVD1BWP7T [all_inputs]
pt_shell> set_load 0.005 [all_outputs]
```

Perform timing analysis and generate reports:

```
pt_shell> update_timing
pt_shell> report_timing -nets -delay_type max
pt_shell> report_timing -nets -delay_type min
```

Exit Synopsys PT:

```
pt_shell> exit
```

### 2.4. Synopsys VCS for Back-Annotated Gate-Level Simulation

Back-annotated gate-level simulation (BAGL sim) uses the Standard Delay
Format (SDF) file generated by Cadence Innovus to annotate realistic gate
and interconnect delays onto the gate-level simulation. This helps verify
not just functional correctness, but also that the design meets all setup
and hold timing constraints under realistic delay conditions.

Change to the working directory (already provided):

```bash
% cd $TOPDIR/asic/playground/addrc4b/07-synopsys-vcs-baglsim
```

Before running the simulation, we need to calculate the clock insertion
source latency. This value accounts for any clock network delay that was
added during clock tree synthesis (CTS). Even though the ripple carry
adder is combinational and doesn't have a clock tree, we still need to
check for any clock latency that may have been specified. The
`calc-clk-ins-src-lat` script (already provided in this directory)
extracts this value from the SDC file:

```bash
% cd $TOPDIR/asic/playground/addrc4b/07-synopsys-vcs-baglsim
% clk_ins_src_lat=$(./calc-clk-ins-src-lat ../05-cadence-innovus-pnr/post-pnr.sdc)
% echo $clk_ins_src_lat
```

Now run VCS to compile the back-annotated simulation:

```bash
% cd $TOPDIR/asic/playground/addrc4b/07-synopsys-vcs-baglsim
% vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=1.0 \
    +define+VTB_CLK_INS_SRC_LAT=${clk_ins_src_lat} \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +vcs+dumpvars+waves.vcd \
    -o simv-test \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/AdderRippleCarry_4b_test_exhaustive_tb.v
```

Here is an explanation of the key VCS options:

 - **`-sverilog`**: Enable SystemVerilog language features
 - **`-xprop=tmerge`**: Enable X-propagation mode for more accurate
   simulation of unknown values
 - **`-override_timescale=1ns/1ps`**: Set the simulation timescale to
   1ns time unit with 1ps precision
 - **`-top Top`**: Specify the top-level module name
 - **`+neg_tchk`**: Enable negative timing checks (for hold time)
 - **`+sdfverbose`**: Print detailed information about SDF annotation
 - **`-sdf max:Top.DUT:...`**: Annotate the SDF file to the DUT instance
   using maximum (worst-case) delays
 - **`+define+CYCLE_TIME=1.0`**: Set the clock period to 1.0ns
 - **`+define+VTB_CLK_INS_SRC_LAT=...`**: Set the clock insertion source
   latency to account for clock tree delay
 - **`+define+VTB_INPUT_DELAY=0.025`**: Set the input delay for the
   test vectors
 - **`+define+VTB_OUTPUT_DELAY=0.025`**: Set the output checking delay

Run the compiled simulator:

```bash
% cd $TOPDIR/asic/playground/addrc4b/07-synopsys-vcs-baglsim
% ./simv-test
```

The simulation should pass all tests. You can view the waveforms to see
the realistic gate and wire delays:

```bash
% cd $TOPDIR/asic/playground/addrc4b/07-synopsys-vcs-baglsim
% code waves.vcd
```

Zoom in on the waveforms and notice how signals now change throughout
the cycle rather than instantaneously. This is because every gate and
wire delay is now modeled in the simulation.

### 2.5. Mentor Calibre for Design Rules Check (DRC)

Design Rules Check (DRC) verifies that the final layout meets all of the
geometric constraints specified by the foundry. These rules ensure that
the design can be manufactured reliably. Common DRC rules include minimum
width, minimum spacing, minimum area, and enclosure requirements for each
metal and via layer.

We use Mentor Calibre for DRC verification. The runset files (`.rs`)
are already provided and configure Calibre with the design-specific
settings and rule file locations.

Change to the working directory (already provided):

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
```

Run the main DRC check:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
```

Here is an explanation of the Calibre options:

 - **`-gui`**: Use the graphical interface (required even in batch mode)
 - **`-drc`**: Run design rules check
 - **`-runset main_drc.rs`**: Use the specified runset file which
   contains the design name, layout file location, and rule file
 - **`-batch`**: Run in batch mode (non-interactive)

The `main_drc.rs` runset file specifies:

 - The DRC rule file location (`${TSMC_180NM}/main_drc.rule`)
 - The layout file to check (`post-pnr.gds`)
 - The top cell name
 - Any rule waivers for block-level design (e.g., minimum metal coverage
   rules that only apply at chip level)

Run the antenna DRC check:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% calibre -gui -drc -runset antenna_drc.rs -batch
```

Antenna rules check for long metal wires connected to transistor gates
that could accumulate charge during manufacturing and damage the gate
oxide. The antenna check verifies that either the wire length is within
limits or that antenna diodes have been added to protect the gates.

There should be no violations if the place-and-route tool was configured
correctly. To view the DRC results interactively, first copy the layer
properties file and then launch Calibre DRV:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

This opens the Calibre DRV GUI which allows you to browse DRC violations.
You can click on any violation to highlight its location in the layout.

**Exploring DRC Violations**

To better understand how DRC works, let's intentionally introduce a
violation. Open the GDS file in Klayout in edit mode:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% klayout -e -l ${TSMC_180NM}/klayout.lyp ../05-cadence-innovus-pnr/post-pnr.gds
```

In Klayout, select a metal layer (e.g., M2) and draw a very thin piece
of metal that violates the minimum width rule. Save the modified GDS
file. Then rerun the DRC check:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
```

Now view the results interactively:

```bash
% cd $TOPDIR/asic/playground/addrc4b/09-mentor-calibre-drc
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

You should see the new DRC violation. Click on it to highlight where
the violation occurs in the layout. This demonstrates how DRC catches
manufacturing rule violations. After exploring, restore the original
GDS file by re-running the place-and-route step or copying a backup.

### 2.6. Mentor Calibre for Layout Versus Schematic (LVS)

Layout Versus Schematic (LVS) verifies that the final physical layout
matches the original gate-level netlist. LVS extracts the circuit
connectivity from the layout and compares it against the schematic
(netlist) to ensure they are functionally equivalent.

Change to the working directory (already provided with the `lvs.rs`
runset file):

```bash
% cd $TOPDIR/asic/playground/addrc4b/10-mentor-calibre-lvs
```

First, we need to convert the Verilog gate-level netlist to SPICE format
so that Calibre can compare it against the extracted layout. We use the
`v2lvs` tool for this conversion:

```bash
% cd $TOPDIR/asic/playground/addrc4b/10-mentor-calibre-lvs
% v2lvs -v ../05-cadence-innovus-pnr/post-pnr.v \
    -o post-pnr.sp \
    -lsr ${TSMC_180NM}/stdcells.sp \
    -s ${TSMC_180NM}/stdcells.sp \
    -log v2lvs.log
```

Here is an explanation of the v2lvs options:

 - **`-v`**: Input Verilog netlist file
 - **`-o`**: Output SPICE netlist file
 - **`-lsr`**: Library SPICE reference file for standard cells
 - **`-s`**: SPICE subcircuit definition file
 - **`-log`**: Log file for conversion messages

Now run Calibre LVS:

```bash
% cd $TOPDIR/asic/playground/addrc4b/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

The `lvs.rs` runset file specifies:

 - The LVS rule file location
 - The layout file (`post-pnr.gds`)
 - The source netlist file (`post-pnr.sp`)
 - The top cell name for both layout and source

**Note:** For this small ripple carry adder design, LVS will actually report a
mismatch. This is expected because the design is too small to include a power
ring, and without a power ring, Cadence Innovus was unable to create block-level
pins for VDD and VSS. The LVS tool detects that the power and ground connections
in the layout don't match the netlist's expectations for block-level power pins.
This is a good example of how LVS catches connectivity issues. In a real design
that will be integrated into a larger chip, the block would be placed in such a
way to allow the VDD/VSS M1 rails to align with those of the top-level.

To view the LVS results interactively:

```bash
% cd $TOPDIR/asic/playground/addrc4b/10-mentor-calibre-lvs
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```

This opens the Calibre DRV GUI which shows the LVS comparison results.
Click on the mismatches to see where the VDD/VSS connectivity issues
occur in the layout.

3. Registered Incrementer
--------------------------------------------------------------------------

In this section, we will be pushing a four-stage registered incrementer
through the commercial back-end flow.

![](img/sec01-regincr-nstage.png)

### 3.1. Front-End Flow

Run the front-end steps using the provided run scripts:

```bash
% cd $TOPDIR/asic/playground/regincr
% ./01-pymtl-rtlsim/run
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-ffglsim/run
```

Verify that your design passes all simulations and examine the synthesis
reports:

```bash
% cd $TOPDIR/asic/playground/regincr
% less ./03-synopsys-dc-synth/area.rpt
% less ./03-synopsys-dc-synth/timing.rpt
```

### 3.2. Cadence Innovus for Place-and-Route

Create the working directory and start Cadence Innovus:

```bash
% mkdir -p $TOPDIR/asic/playground/regincr/05-cadence-innovus-pnr
% cd $TOPDIR/asic/playground/regincr/05-cadence-innovus-pnr
```

**Constraint and Timing Input Files**

Create the `setup-timing.tcl` file:

```bash
% cd $TOPDIR/asic/playground/regincr/05-cadence-innovus-pnr
% code setup-timing.tcl
```

The file should have the following content:

```
create_rc_corner -name typical \
   -cap_table "$env(TSMC_180NM)/typical.captable" \
   -T 25

create_library_set -name libs_typical \
   -timing [list "$env(TSMC_180NM)/stdcells.lib"]

create_delay_corner -name delay_default \
   -library_set libs_typical \
   -rc_corner typical

create_constraint_mode -name constraints_default \
   -sdc_files [list ../03-synopsys-dc-synth/post-synth.sdc]

create_analysis_view -name analysis_default \
   -constraint_mode constraints_default \
   -delay_corner delay_default

set_analysis_view \
   -setup analysis_default \
   -hold analysis_default
```

**Initial Setup and Floorplanning**

Start Cadence Innovus:

```bash
% cd $TOPDIR/asic/playground/regincr/05-cadence-innovus-pnr
% innovus
```

Set up the design variables:

```
innovus> set init_mmmc_file "setup-timing.tcl"
innovus> set init_verilog   "../03-synopsys-dc-synth/post-synth.v"
innovus> set init_top_cell  "RegIncr4stage"
innovus> set init_lef_file  [list "$env(TSMC_180NM)/apr-tech.tlef" "$env(TSMC_180NM)/stdcells.lef"]
innovus> set init_pwr_net   {VDD vdd}
innovus> set init_gnd_net   {VSS gnd}
```

Initialize the design:

```
innovus> init_design
innovus> setDesignMode -process 180
```

Limit routing and pin placement to be between layers 2 and 5:

```
innovus> setDesignMode -bottomRoutingLayer 2
innovus> setDesignMode -topRoutingLayer    5
```

Configure VIA generation and optimization settings:

```
innovus> setViaGenMode -ignore_viarule_enclosure false
innovus> setViaGenMode -optimize_cross_via true
innovus> setViaGenMode -ignore_DRC false
innovus> setViaGenMode -disable_via_merging true
innovus> setDelayCalMode -SIAware false
innovus> setOptMode -usefulSkew false
innovus> setOptMode -holdTargetSlack 0.010
innovus> setOptMode -holdFixingCells {BUFFD0BWP7T BUFFD10BWP7T BUFFD12BWP7T BUFFD1BWP7T BUFFD1P5BWP7T BUFFD2BWP7T BUFFD2P5BWP7T BUFFD3BWP7T BUFFD4BWP7T BUFFD5BWP7T BUFFD6BWP7T BUFFD8BWP7T}
innovus> set report_precision 4
```

Create the floorplan with 10um margins for the power ring:

```
innovus> floorPlan -r 1.0 0.70 10.0 10.0 10.0 10.0
innovus> setPinConstraint -global -pinTemplate {2 3 4 5} -depth 0.73
```

**Power Routing**

Connect power and ground nets:

```
innovus> globalNetConnect VDD -type pgpin -pin VDD -all -verbose
innovus> globalNetConnect VSS -type pgpin -pin VSS -all -verbose
innovus> globalNetConnect VDD -type tiehi -pin VDD -all -verbose
innovus> globalNetConnect VSS -type tielo -pin VSS -all -verbose
```

Route the power and ground nets:

```
innovus> sroute -nets {VDD VSS}
```

Route a power ring on M5 and M6 around the outside of the core area:

```
innovus> addRing -nets {VDD VSS} -width 2.6 -spacing 2.5 -layer [list top 6 bottom 6 left 5 right 5] -extend_corner {tl tr bl br lt lb rt rb}
```

Route power stripes:

```
innovus> addStripe -nets {VSS VDD} -layer 6 -direction horizontal -width 5.52 -spacing 16.88 -set_to_set_distance 44.8 -start_offset 22.4
innovus> addStripe -nets {VSS VDD} -layer 5 -direction vertical -width 5.52 -spacing 16.88 -set_to_set_distance 44.8 -start_offset 22.4
innovus> sroute -nets {VDD VSS}
```

Add well tap cells:

```
innovus> set_well_tap_mode -cell TAPCELLBWP7T
innovus> addWellTap -cellInterval 29.68
```

**Placement**

Place and optimize the design:

```
innovus> place_opt_design
```

Add tie-hi/tie-lo cells and assign I/O pins:

```
innovus> addTieHiLo -cell "TIEHBWP7T TIELBWP7T"
innovus> assignIoPins -pin *
```

**Clock-Tree Synthesis**

Since this is a sequential design, we need to synthesize the clock tree:

```
innovus> create_ccopt_clock_tree_spec
innovus> set_ccopt_property update_io_latency false
innovus> clock_opt_design
```

Post-CTS optimization:

```
innovus> optDesign -postCTS -setup
innovus> optDesign -postCTS -hold
```

**Signal Routing**

Configure antenna fixing and route the design:

```
innovus> setNanoRouteMode -route_detail_fix_antenna true
innovus> setNanoRouteMode -route_antenna_cell_name "ANTENNABWP7T"
innovus> setNanoRouteMode -route_antenna_diode_insertion true
innovus> routeDesign
```

Post-route optimization:

```
innovus> optDesign -postRoute -setup
innovus> optDesign -postRoute -hold
innovus> optDesign -postRoute -drv
innovus> extractRC
```

**Final Output and Reports**

Add filler cells and verify the design:

```
innovus> setFillerMode -core {FILL1BWP7T FILL2BWP7T FILL4BWP7T FILL8BWP7T FILL16BWP7T FILL32BWP7T FILL64BWP7T}
innovus> addFiller
innovus> verifyConnectivity
innovus> verify_drc
innovus> verify_antenna
```

Save outputs:

```
innovus> saveDesign post-pnr.enc
innovus> saveNetlist post-pnr.v
innovus> rcOut -rc_corner typical -spef post-pnr.spef
innovus> write_sdf -recompute_delay_calc post-pnr.sdf
innovus> write_sdc post-pnr.sdc
innovus> streamOut post-pnr.gds -units 1000 -merge "$env(TSMC_180NM)/stdcells.gds" -mapFile "$env(TSMC_180NM)/gds_out.map"
innovus> report_timing -late  -path_type full_clock -net > timing-setup.rpt
innovus> report_timing -early -path_type full_clock -net > timing-hold.rpt
innovus> report_area > area.rpt
innovus> exit
```

View the layout in Klayout:

```bash
% cd $TOPDIR/asic/playground/regincr/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 3.3. Synopsys PrimeTime for Static Timing Analysis

Perform static timing analysis on the post-place-and-route design:

```bash
% mkdir -p $TOPDIR/asic/playground/regincr/06-synopsys-pt-sta
% cd $TOPDIR/asic/playground/regincr/06-synopsys-pt-sta
% pt_shell
```

Set up the standard cell library:

```
pt_shell> set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
```

Read in the design and parasitics:

```
pt_shell> read_verilog   "../05-cadence-innovus-pnr/post-pnr.v"
pt_shell> current_design RegIncr4stage
pt_shell> link_design
pt_shell> read_parasitics -format spef ../05-cadence-innovus-pnr/post-pnr.spef
```

Set timing constraints for the sequential design:

```
pt_shell> create_clock clk -name ideal_clock1 -period 1.0
pt_shell> set_input_delay -clock ideal_clock1 -max 0.050 [all_inputs -exclude_clock_ports]
pt_shell> set_input_delay -clock ideal_clock1 -min 0     [all_inputs -exclude_clock_ports]
pt_shell> set_output_delay -clock ideal_clock1 -max 0.050 [all_outputs]
pt_shell> set_output_delay -clock ideal_clock1 -min 0     [all_outputs]
pt_shell> set_max_delay 1.0 -from [all_inputs -exclude_clock_ports] -to [all_outputs]
pt_shell> set_max_transition 0.250 RegIncr4stage
pt_shell> set_input_transition 0 [all_inputs]
pt_shell> set_load 0.005 [all_outputs]
```

Perform timing analysis:

```
pt_shell> update_timing
pt_shell> report_global_timing -delay_type max
pt_shell> report_global_timing -delay_type min
pt_shell> report_timing -nets -delay_type max
pt_shell> report_timing -nets -delay_type min
```

Exit Synopsys PT:

```
pt_shell> exit
```

### 3.4. Synopsys VCS for Back-Annotated Gate-Level Simulation

Back-annotated gate-level simulation will take into account all of the
gate and interconnect delays. This helps verify not just that the final
gate-level netlist is functionally correct, but also that it meets all
setup and hold time constraints.

Change to the working directory (already provided):

```bash
% cd $TOPDIR/asic/playground/regincr/07-synopsys-vcs-baglsim
```

Before running the simulation, we need to calculate the clock insertion
source latency. This value accounts for any clock network delay that was
added during clock tree synthesis (CTS). The `calc-clk-ins-src-lat`
script (already provided in this directory) extracts this value from
the SDC file:

```bash
% cd $TOPDIR/asic/playground/regincr/07-synopsys-vcs-baglsim
% clk_ins_src_lat=$(./calc-clk-ins-src-lat ../05-cadence-innovus-pnr/post-pnr.sdc)
% echo $clk_ins_src_lat
```

Run VCS for back-annotated gate-level simulation. Note that we use a
slower clock period (3.0ns) than the original constraint to ensure the
design meets timing with realistic gate and wire delays:

```bash
% cd $TOPDIR/asic/playground/regincr/07-synopsys-vcs-baglsim
% vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
    +define+VTB_CLK_INS_SRC_LAT=${clk_ins_src_lat} \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +vcs+dumpvars+waves.vcd \
    -o simv-test \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/RegIncr4stage_basic_tb.v
```

Run the compiled simulator:

```bash
% cd $TOPDIR/asic/playground/regincr/07-synopsys-vcs-baglsim
% ./simv-test
```

The simulation should pass all tests. View the waveforms:

```bash
% cd $TOPDIR/asic/playground/regincr/07-synopsys-vcs-baglsim
% code waves.vcd
```

Zoom in and notice how the signals now change throughout the cycle. This
is because the delay of every gate and wire is now modeled.

### 3.5. Mentor Calibre for Design Rules Check (DRC)

Change to the working directory (already provided with the `.rs` runset
files):

```bash
% cd $TOPDIR/asic/playground/regincr/09-mentor-calibre-drc
```

Run the main DRC check and antenna DRC check:

```bash
% cd $TOPDIR/asic/playground/regincr/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
% calibre -gui -drc -runset antenna_drc.rs -batch
```

There should be no violations if the place-and-route tool was configured
correctly. To view the DRC results interactively:

```bash
% cd $TOPDIR/asic/playground/regincr/09-mentor-calibre-drc
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

As with the ripple carry adder, you can experiment by opening the GDS
in Klayout edit mode (`klayout -e`), adding a thin piece of metal to
violate minimum width rules, and then rerunning DRC to see the violation
highlighted in the interactive viewer.

### 3.6. Mentor Calibre for Layout Versus Schematic (LVS)

Change to the working directory (already provided with the `lvs.rs`
runset file):

```bash
% cd $TOPDIR/asic/playground/regincr/10-mentor-calibre-lvs
```

Convert the Verilog netlist to SPICE format:

```bash
% cd $TOPDIR/asic/playground/regincr/10-mentor-calibre-lvs
% v2lvs -v ../05-cadence-innovus-pnr/post-pnr.v \
    -o post-pnr.sp \
    -lsr ${TSMC_180NM}/stdcells.sp \
    -s ${TSMC_180NM}/stdcells.sp \
    -log v2lvs.log
```

Run Calibre LVS:

```bash
% cd $TOPDIR/asic/playground/regincr/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

The LVS should report "CORRECT" indicating that the layout matches the
schematic. To view the LVS results interactively:

```bash
% cd $TOPDIR/asic/playground/regincr/10-mentor-calibre-lvs
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```

You can experiment with LVS violations by opening the GDS in Klayout
edit mode (`klayout -e`), deleting a portion of a signal wire to create
an open circuit, and then rerunning LVS to see the mismatch highlighted
in the interactive viewer. See the GCD accelerator section for a
detailed walkthrough of this exercise.

4. GCD Accelerator
--------------------------------------------------------------------------

In this section, we will be pushing the GCD accelerator through the
commercial back-end flow. This is the same GCD accelerator we explored in
lab 5 and pushed through the front-end flow in lab 6.

![](img/lab5-gcd.png)

### 4.1. Front-End Flow

Run the front-end steps using the provided run scripts:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel
% ./01-pymtl-rtlsim/run
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-ffglsim/run
```

Verify that your design passes all simulations. Take a close look at the
timing report. If you do not meet timing then you must resynthesize your
design with a longer clock period to ensure you meet timing. **Do not
continue to the next step unless your design meets timing!**

```bash
% cd $TOPDIR/asic/playground/gcd-xcel
% less ./03-synopsys-dc-synth/area.rpt
% less ./03-synopsys-dc-synth/timing.rpt
```

### 4.2. Cadence Innovus for Place-and-Route

Create the working directory:

```bash
% mkdir -p $TOPDIR/asic/playground/gcd-xcel/05-cadence-innovus-pnr
% cd $TOPDIR/asic/playground/gcd-xcel/05-cadence-innovus-pnr
```

**Constraint and Timing Input Files**

Create the `setup-timing.tcl` file:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/05-cadence-innovus-pnr
% code setup-timing.tcl
```

The file should have the following content:

```
create_rc_corner -name typical \
   -cap_table "$env(TSMC_180NM)/typical.captable" \
   -T 25

create_library_set -name libs_typical \
   -timing [list "$env(TSMC_180NM)/stdcells.lib"]

create_delay_corner -name delay_default \
   -library_set libs_typical \
   -rc_corner typical

create_constraint_mode -name constraints_default \
   -sdc_files [list ../03-synopsys-dc-synth/post-synth.sdc]

create_analysis_view -name analysis_default \
   -constraint_mode constraints_default \
   -delay_corner delay_default

set_analysis_view \
   -setup analysis_default \
   -hold analysis_default
```

**Initial Setup and Floorplanning**

Start Cadence Innovus:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/05-cadence-innovus-pnr
% innovus
```

Set up the design variables:

```
innovus> set init_mmmc_file "setup-timing.tcl"
innovus> set init_verilog   "../03-synopsys-dc-synth/post-synth.v"
innovus> set init_top_cell  "GcdXcel_noparam"
innovus> set init_lef_file  [list "$env(TSMC_180NM)/apr-tech.tlef" "$env(TSMC_180NM)/stdcells.lef"]
innovus> set init_pwr_net   {VDD vdd}
innovus> set init_gnd_net   {VSS gnd}
```

Initialize the design:

```
innovus> init_design
innovus> setDesignMode -process 180
```

Limit routing layers:

```
innovus> setDesignMode -bottomRoutingLayer 2
innovus> setDesignMode -topRoutingLayer    5
```

Configure VIA generation and optimization settings:

```
innovus> setViaGenMode -ignore_viarule_enclosure false
innovus> setViaGenMode -optimize_cross_via true
innovus> setViaGenMode -ignore_DRC false
innovus> setViaGenMode -disable_via_merging true
innovus> setDelayCalMode -SIAware false
innovus> setOptMode -usefulSkew false
innovus> setOptMode -holdTargetSlack 0.010
innovus> setOptMode -holdFixingCells {BUFFD0BWP7T BUFFD10BWP7T BUFFD12BWP7T BUFFD1BWP7T BUFFD1P5BWP7T BUFFD2BWP7T BUFFD2P5BWP7T BUFFD3BWP7T BUFFD4BWP7T BUFFD5BWP7T BUFFD6BWP7T BUFFD8BWP7T}
innovus> set report_precision 4
```

Create the floorplan with a larger margin for the power ring:

```
innovus> floorPlan -r 1.0 0.70 10.0 10.0 10.0 10.0
innovus> setPinConstraint -global -pinTemplate {2 3 4 5} -depth 0.73
```

**Power Routing**

Connect power and ground nets:

```
innovus> globalNetConnect VDD -type pgpin -pin VDD -all -verbose
innovus> globalNetConnect VSS -type pgpin -pin VSS -all -verbose
innovus> globalNetConnect VDD -type tiehi -pin VDD -all -verbose
innovus> globalNetConnect VSS -type tielo -pin VSS -all -verbose
```

Route the power grid:

```
innovus> sroute -nets {VDD VSS}
```

Add a power ring around the design:

```
innovus> addRing \
  -nets {VDD VSS} -width 2.6 -spacing 2.5 \
  -layer [list top 6 bottom 6 left 5 right 5] \
  -extend_corner {tl tr bl br lt lb rt rb}
```

Add power stripes:

```
innovus> addStripe \
  -nets {VSS VDD} -layer 6 -direction horizontal \
  -width 5.52 -spacing 16.88 \
  -set_to_set_distance 44.8 -start_offset 22.4
innovus> addStripe \
  -nets {VSS VDD} -layer 5 -direction vertical \
  -width 5.52 -spacing 16.88 \
  -set_to_set_distance 44.8 -start_offset 22.4
```

Reroute power and ground:

```
innovus> sroute -nets {VDD VSS}
```

Add well tap cells:

```
innovus> set_well_tap_mode -cell TAPCELLBWP7T
innovus> addWellTap -cellInterval 29.68
```

**Placement**

Place and optimize the design:

```
innovus> place_opt_design
```

Add tie-hi/tie-lo cells:

```
innovus> addTieHiLo -cell "TIEHBWP7T TIELBWP7T"
```

Assign I/O pins:

```
innovus> assignIoPins -pin *
```

**Clock-Tree Synthesis**

Create the clock tree:

```
innovus> create_ccopt_clock_tree_spec
innovus> set_ccopt_property update_io_latency false
innovus> clock_opt_design
```

Optimize timing:

```
innovus> optDesign -postCTS -setup
innovus> optDesign -postCTS -hold
```

**Signal Routing**

Set up routing for antenna fix:

```
innovus> setNanoRouteMode -route_detail_fix_antenna true
innovus> setNanoRouteMode -route_antenna_cell_name "ANTENNABWP7T"
innovus> setNanoRouteMode -route_antenna_diode_insertion true
```

Route the design:

```
innovus> routeDesign
```

Post-route optimization:

```
innovus> optDesign -postRoute -setup
innovus> optDesign -postRoute -hold
innovus> optDesign -postRoute -drv
```

Extract parasitics:

```
innovus> extractRC
```

**Final Output and Reports**

Add filler cells:

```
innovus> setFillerMode -core {FILL1BWP7T FILL2BWP7T FILL4BWP7T FILL8BWP7T \
  FILL16BWP7T FILL32BWP7T FILL64BWP7T}
innovus> addFiller
```

Verify the design:

```
innovus> verifyConnectivity
innovus> verify_drc
innovus> verify_antenna
```

Save outputs:

```
innovus> saveDesign post-pnr.enc
innovus> saveNetlist post-pnr.v
innovus> rcOut -rc_corner typical -spef post-pnr.spef
innovus> write_sdf -recompute_delay_calc post-pnr.sdf
innovus> write_sdc post-pnr.sdc
innovus> streamOut post-pnr.gds \
  -units 1000 \
  -merge "$env(TSMC_180NM)/stdcells.gds" \
  -mapFile "$env(TSMC_180NM)/gds_out.map"
```

Generate reports:

```
innovus> report_timing -late  -path_type full_clock -net > timing-setup.rpt
innovus> report_timing -early -path_type full_clock -net > timing-hold.rpt
innovus> report_area > area.rpt
innovus> exit
```

View the layout in Klayout:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 4.3. Synopsys PrimeTime for Static Timing Analysis

Perform static timing analysis:

```bash
% mkdir -p $TOPDIR/asic/playground/gcd-xcel/06-synopsys-pt-sta
% cd $TOPDIR/asic/playground/gcd-xcel/06-synopsys-pt-sta
% pt_shell
```

Set up and read the design:

```
pt_shell> set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
pt_shell> read_verilog   "../05-cadence-innovus-pnr/post-pnr.v"
pt_shell> current_design GcdXcel_noparam
pt_shell> link_design
pt_shell> read_parasitics -format spef ../05-cadence-innovus-pnr/post-pnr.spef
```

Set timing constraints (use the same clock period as synthesis):

```
pt_shell> create_clock clk -name ideal_clock1 -period 3.0
pt_shell> set_input_delay -clock ideal_clock1 -max 0.050 [all_inputs -exclude_clock_ports]
pt_shell> set_input_delay -clock ideal_clock1 -min 0     [all_inputs -exclude_clock_ports]
pt_shell> set_output_delay -clock ideal_clock1 -max 0.050 [all_outputs]
pt_shell> set_output_delay -clock ideal_clock1 -min 0     [all_outputs]
pt_shell> set_max_delay 3.0 -from [all_inputs -exclude_clock_ports] -to [all_outputs]
pt_shell> set_max_transition 0.250 GcdXcel_noparam
pt_shell> set_input_transition 0 [all_inputs]
pt_shell> set_load 0.005 [all_outputs]
```

Perform timing analysis:

```
pt_shell> update_timing
pt_shell> report_global_timing -delay_type max
pt_shell> report_global_timing -delay_type min
pt_shell> report_timing -nets -delay_type max
pt_shell> report_timing -nets -delay_type min
```

Exit Synopsys PT:

```
pt_shell> exit
```

### 4.4. Synopsys VCS for Back-Annotated Gate-Level Simulation

Change to the working directory (already provided):

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/07-synopsys-vcs-baglsim
```

Before running the simulation, we need to calculate the clock insertion
source latency. This value accounts for any clock network delay that was
added during clock tree synthesis (CTS). The `calc-clk-ins-src-lat`
script (already provided in this directory) extracts this value from
the SDC file:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/07-synopsys-vcs-baglsim
% clk_ins_src_lat=$(./calc-clk-ins-src-lat ../05-cadence-innovus-pnr/post-pnr.sdc)
% echo $clk_ins_src_lat
```

Run VCS for back-annotated gate-level simulation with a test:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/07-synopsys-vcs-baglsim
% vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
    +define+VTB_CLK_INS_SRC_LAT=${clk_ins_src_lat} \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +vcs+dumpvars+waves.vcd \
    -o simv-test \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/GcdXcel_noparam_test_random_3x9_tb.v
% ./simv-test
```

The simulation should pass all tests. Now run the evaluation with SAIF
generation for power analysis. The SAIF (Switching Activity Interchange
Format) file records the switching activity of every net in the design,
which is needed for accurate power analysis:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/07-synopsys-vcs-baglsim
% vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
    +define+VTB_CLK_INS_SRC_LAT=${clk_ins_src_lat} \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +define+VTB_DUMP_SAIF=GcdXcel_noparam_gcd-xcel-sim-rtl-random.saif \
    +vcs+dumpvars+waves.vcd \
    -o simv-eval \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v
% ./simv-eval
```

View the waveforms:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/07-synopsys-vcs-baglsim
% code waves.vcd
```

### 4.5. Synopsys PrimeTime for Power Analysis

Perform power analysis using the activity factors from back-annotated
gate-level simulation:

```bash
% mkdir -p $TOPDIR/asic/playground/gcd-xcel/08-synopsys-pt-pwr
% cd $TOPDIR/asic/playground/gcd-xcel/08-synopsys-pt-pwr
% pt_shell
```

Set up the standard cell library and enable power analysis:

```
pt_shell> set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var power_enable_analysis true
```

Read in the design:

```
pt_shell> read_verilog   "../05-cadence-innovus-pnr/post-pnr.v"
pt_shell> current_design GcdXcel_noparam
pt_shell> link_design
```

Set the clock period:

```
pt_shell> create_clock clk -name ideal_clock1 -period 3.0
```

Read in the SAIF file with activity factors and the SPEF file with
parasitic capacitances:

```
pt_shell> read_saif "../07-synopsys-vcs-baglsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random.saif" -strip_path "Top/DUT"
pt_shell> read_parasitics -format spef "../05-cadence-innovus-pnr/post-pnr.spef"
```

Perform power analysis:

```
pt_shell> update_power
pt_shell> report_power
pt_shell> report_power -hierarchy
```

Exit Synopsys PT:

```
pt_shell> exit
```

### 4.6. Mentor Calibre for Design Rules Check (DRC)

Design Rules Check (DRC) verifies that the final layout meets all of the
geometric constraints specified by the foundry. These rules ensure that
the design can be manufactured reliably. Common DRC rules include minimum
width, minimum spacing, minimum area, and enclosure requirements for each
metal and via layer.

Change to the working directory (already provided with the `.rs` runset
files):

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/09-mentor-calibre-drc
```

Run the main DRC check:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
```

The `main_drc.rs` runset file specifies:

 - The DRC rule file location (`${TSMC_180NM}/main_drc.rule`)
 - The layout file to check (`post-pnr.gds`)
 - The top cell name (`GcdXcel_noparam`)
 - Block-level rule waivers (e.g., minimum metal coverage rules that only
   apply at chip level)

Run the antenna DRC check:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/09-mentor-calibre-drc
% calibre -gui -drc -runset antenna_drc.rs -batch
```

Antenna rules check for long metal wires connected to transistor gates
that could accumulate charge during manufacturing and damage the gate
oxide. The antenna check verifies that either the wire length is within
limits or that antenna diodes have been added to protect the gates.

There should be no violations if the place-and-route tool was configured
correctly. To view the DRC results interactively:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/09-mentor-calibre-drc
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

As with the previous designs, you can experiment by opening the GDS
in Klayout edit mode (`klayout -e`), adding a thin piece of metal to
violate minimum width rules, and then rerunning DRC to see the violation
highlighted in the interactive viewer.

### 4.7. Mentor Calibre for Layout Versus Schematic (LVS)

Layout Versus Schematic (LVS) verifies that the final physical layout
matches the original gate-level netlist. LVS extracts the circuit
connectivity from the layout and compares it against the schematic
(netlist) to ensure they are functionally equivalent.

Change to the working directory (already provided with the `lvs.rs`
runset file):

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
```

First, we need to convert the Verilog gate-level netlist to SPICE format
so that Calibre can compare it against the extracted layout:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% v2lvs -v ../05-cadence-innovus-pnr/post-pnr.v \
    -o post-pnr.sp \
    -lsr ${TSMC_180NM}/stdcells.sp \
    -s ${TSMC_180NM}/stdcells.sp \
    -log v2lvs.log
```

Now run Calibre LVS:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

The `lvs.rs` runset file specifies:

 - The LVS rule file location
 - The layout file (`post-pnr.gds`)
 - The source netlist file (`post-pnr.sp`)
 - The top cell name for both layout and source (`GcdXcel_noparam`)

The LVS should report "CORRECT" indicating that the layout matches the
schematic. To view the LVS results interactively:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```

This opens the Calibre DRV GUI which shows the LVS comparison results.
If there are mismatches, you can click on them to highlight where the
discrepancy occurs in the layout.

**Exploring LVS Violations**

To better understand how LVS works, let's intentionally introduce a
mismatch. Open the GDS file in Klayout in edit mode:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% klayout -e -l ${TSMC_180NM}/klayout.lyp ../05-cadence-innovus-pnr/post-pnr.gds
```

In Klayout, find a signal wire and delete a small portion of it to
create an open circuit. Save the modified GDS file. Then rerun LVS:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

Now view the results interactively:

```bash
% cd $TOPDIR/asic/playground/gcd-xcel/10-mentor-calibre-lvs
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```

The LVS should now report a mismatch. Click on the violation to see
where the broken connection occurs. This demonstrates how LVS catches
connectivity errors between the layout and netlist that would result
in a non-functional chip. After exploring, restore the original GDS
file by re-running the place-and-route step or copying a backup.

