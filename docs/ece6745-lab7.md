
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
(layout-versus-schematic) verification flow using Mentor Calibre. These
checks are performed after place-and-route to ensure the design is ready
for manufacturing:

 - **DRC Flow**: The post-PNR GDS file is checked against the foundry's
   design rules. The rules file (`.rule`) contains geometric constraints
   such as minimum metal width, minimum spacing between wires, minimum
   area for metal shapes, and enclosure rules for vias.

 - **LVS Flow**: The post-PNR Verilog netlist is first converted to
   SPICE format using `v2lvs`. Then Calibre extracts the circuit
   connectivity from the GDS layout and compares it against the SPICE
   netlist to ensure they match.

Calibre is configured using **runset files** (`.rs`), which capture all
the options that would normally be set through the Calibre GUI. The
runset files specify:

 - **Rules file**: Path to the foundry-provided rule file
 - **Run directory**: Where to store the results
 - **Layout file**: The GDS file to check/extract
 - **Top cell**: The top-level cell name in the layout
 - **Source file** (LVS only): The SPICE netlist to compare against
 - **Rule waivers** (DRC only): Which checks to skip for block-level design

By using runset files, you can run Calibre in batch mode and easily
re-run DRC/LVS after making changes. When moving to a new design, you
only need to update the top cell name in the runset file.

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

In this section, we will push a four-bit ripple carry adder through the
front-end flow and place-and-route. Since this is a small combinational
design, we will stop after PNR (step 05). We will work in
`$TOPDIR/asic/playground` with numbered step directories:

 - `01-pymtl-rtlsim`: PyMTL 2-State RTL Simulation
 - `02-synopsys-vcs-rtlsim`: Synopsys VCS 4-State RTL Simulation
 - `03-synopsys-dc-synth`: Synopsys DC Synthesis
 - `04-synopsys-vcs-ffglsim`: Synopsys VCS Fast-Functional Gate-Level Simulation
 - `05-cadence-innovus-pnr`: Cadence Innovus Place-and-Route

After completing these steps for the ripple carry adder, we will archive
the playground directory and then modify the runscripts for more complex
designs that require additional back-end steps.

### 2.1. Front-End Flow

We have provided complete runscripts for the front-end steps (01-04)
for the ripple carry adder. Simply run each step in order:

**Step 1: PyMTL RTL Simulation**

```bash
% cd $TOPDIR/asic/playground/01-pymtl-rtlsim
% ./run
```

This runs PyMTL tests with `--test-verilog` to generate synthesizable
Verilog and `--dump-vtb` to generate Verilog test benches.

**Step 2: VCS RTL Simulation**

```bash
% cd $TOPDIR/asic/playground/02-synopsys-vcs-rtlsim
% ./run
```

This compiles and runs VCS 4-state RTL simulation using the generated
Verilog and test bench.

**Step 3: Synopsys DC Synthesis**

```bash
% cd $TOPDIR/asic/playground/03-synopsys-dc-synth
% ./run
% less area.rpt
% less timing.rpt
```

This synthesizes the design. Note that for this combinational design,
the runscript uses `compile` (not `compile_ultra`) and `set_max_delay`
to constrain the propagation delay from inputs to outputs. There is no
clock constraint since there are no registers.

**Step 4: Fast-Functional Gate-Level Simulation**

```bash
% cd $TOPDIR/asic/playground/04-synopsys-vcs-ffglsim
% ./run
```

This simulates the synthesized gate-level netlist to verify functional
correctness.

### 2.2. Cadence Innovus for Place-and-Route

Change to the place-and-route directory:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
```

**Constraint and Timing Input Files**

Before starting Cadence Innovus, we need to create the timing setup file. This
"multi-mode multi-corner" (MMMC) analysis file specifies what "corner" to use
for our timing analysis. A corner specifies process, temperature, and voltage
conditions on which to optimize the design and fix timing violations. The more
corners that are considered for optimization, the better the design will perform
under various conditions when taped-out on silicon. While multiple corners can
be configured, we only use the typical corner for this lab. Use VS Code to
create a file named `setup-timing.tcl`:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
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
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% innovus
```

We need to set various variables before starting to work in Cadence Innovus.
These variables tell Cadence Innovus the location of the MMMC file, the location
of the Verilog gate-level netlist, the name of the top-level module in our
design, the location of the `.lef` files, and finally the names of the power and
ground nets. Note that you cannot block-copy-and-paste these commands as we
cannot make an alias for "innovus>" in the Innovus shell, so you will need to
enter each line one at a time without the "innovus>" header. After each command
runs, check how the current view of the block changes in the Innovus GUI window.

```
innovus> set init_mmmc_file "setup-timing.tcl"
innovus> set init_verilog   "../03-synopsys-dc-synth/post-synth.v"
innovus> set init_top_cell  "AdderRippleCarry_4b"
innovus> set init_lef_file  [list "$env(TSMC_180NM)/apr-tech.tlef" "$env(TSMC_180NM)/stdcells.lef"]
innovus> set init_pwr_net   "VDD"
innovus> set init_gnd_net   "VSS"
```

We can now use the `init_design` command to read in the Verilog, set the
design name, setup the timing analysis views, read the technology `.lef`
for layer information, and read the standard cell `.lef` for physical
information about each cell used in the design.

```
innovus> init_design
```

We start by working on floorplanning. Use the `floorPlan` command to set
the dimensions for our chip. Since the ripple carry adder is a small
combinational design, we use small margins (0.5um):

```
innovus> floorPlan -r 1.0 0.70 0.5 0.5 0.5 0.5
```

In this example, we have chosen the aspect ratio to be 1.0, the target
cell utilization to be 0.7, and we have added 0.5um of margin around the
top, bottom, left, and right of the chip. This small design does not need
a power ring.

**Power Routing**

Now we need to tell Cadence Innovus that `VDD` and `VSS` in the
gate-level netlist correspond to the physical pins labeled `VDD` and
`VSS` in the `.lef` files:

```
innovus> globalNetConnect VDD -type pgpin -pin VDD -all -verbose
innovus> globalNetConnect VSS -type pgpin -pin VSS -all -verbose
```

Route the power and ground nets:

```
innovus> sroute -nets {VDD VSS}
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

Place the input/output pins so they are close to the standard cells they
are connected to:

```
innovus> assignIoPins -pin *
```

**Signal Routing**

Use the `routeDesign` command to do a detailed routing pass:

```
innovus> routeDesign
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

Save the design and generate several output files that will be used in
subsequent steps of the flow:

 - **post-pnr.enc**: Innovus design database that can be reloaded later
   to continue working on the design or generate additional outputs
 - **post-pnr.v**: Gate-level Verilog netlist with the final placed and
   routed cells; used for gate-level simulation
 - **post-pnr.spef**: Standard Parasitic Exchange Format file containing
   extracted resistance and capacitance values for all wires; used for
   accurate timing and power analysis
 - **post-pnr.sdf**: Standard Delay Format file containing timing delays
   for all cells and interconnects; used for back-annotated gate-level
   simulation (baglsim)
 - **post-pnr.sdc**: Synopsys Design Constraints file containing the
   timing constraints; used for static timing analysis

```
innovus> saveDesign post-pnr.enc
innovus> saveNetlist post-pnr.v
innovus> rcOut -rc_corner typical -spef post-pnr.spef
innovus> write_sdf -recompute_delay_calc post-pnr.sdf
innovus> write_sdc post-pnr.sdc
```

Generate the final layout as a GDS file by merging in the GDS for the standard
cells and using a map file to assign layers in Innovus to layers in the final
GDS. GDS is the industry-standard format for IC layout data that will be used
for DRC and LVS verification, and will also be the file that is sent to the fab
that tapes out the chip.

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

**Creating the Run Script**

Now that you have successfully run all commands manually, you should put
them into a run script so you can easily re-run the flow. Create a file
named `run.tcl` containing all the Innovus commands you just entered
(without the `innovus>` prompt). Then create a shell script named `run`
that invokes Innovus with this TCL file:

```bash
#!/usr/bin/env bash
set -e
RUNDIR="$(dirname $(readlink -f "${BASH_SOURCE[0]}"))"
cd $RUNDIR
innovus -no_gui -files run.tcl -log run.log
```

Make the script executable with `chmod +x run`. For subsequent designs
(regincr and gcd-xcel), you will edit the `run.tcl` file directly rather
than manually entering commands, which is much more efficient.

Open the final layout using Klayout:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 2.3. Archive the Ripple Carry Adder

Before moving on to the next design, archive the playground directory so
you can keep a record of your work on the ripple carry adder:

```bash
% cd $TOPDIR/asic
% cp -r playground playground-addrc4b
```

This preserves all your runscripts and results for the ripple carry adder.
You will continue to use the `playground` directory for the next design,
modifying the runscripts as needed.

3. Registered Incrementer
--------------------------------------------------------------------------

In this section, we will modify the runscripts in the playground directory
to push a four-stage registered incrementer through the ASIC flow. This
design is sequential (it has registers), which requires several important
changes to the flow compared to the combinational ripple carry adder.

**Additional Steps for regincr:** For the registered incrementer, we will
perform additional back-end steps beyond PNR:

 - **Step 06**: Static timing analysis (STA) with Synopsys PrimeTime
 - **Step 07**: Back-annotated gate-level simulation with VCS
 - **Step 09**: Design rules check (DRC) with Mentor Calibre
 - **Step 10**: Layout versus schematic (LVS) with Mentor Calibre

![](img/sec01-regincr-nstage.png)

### 3.1. Front-End Flow Changes

The registered incrementer is a sequential design with a clock. This
requires several changes to the runscripts compared to the combinational
ripple carry adder.

**Step 1: PyMTL RTL Simulation**

Edit `01-pymtl-rtlsim/run`:

 - **Remove**: The pytest command for AdderRippleCarry_4b and the sed command
 - **Add**: The registered incrementer simulator command below

```bash
${SIMDIR}/tut3_verilog/regincr/regincr-sim -s 0xff 0x20 0x30 0x40 0x00
```

This runs the registered incrementer simulator with specific test values.
Unlike the adder which used pytest, this design uses a command-line
simulator. Run the script:

```bash
% cd $TOPDIR/asic/playground/01-pymtl-rtlsim
% ./run
```

**Step 2: VCS RTL Simulation**

Edit `02-synopsys-vcs-rtlsim/run`:

 - **Change**: `AdderRippleCarry_4b__pickled.v` to `RegIncr4stage__pickled.v`
 - **Change**: `AdderRippleCarry_4b_test_exhaustive_tb.v` to `RegIncr4stage_basic_tb.v`

The updated script should be:

```bash
set -e
trap 'echo "CMD: $BASH_COMMAND"' DEBUG

RUNDIR="$(dirname $(readlink -f "${BASH_SOURCE[0]}"))"
cd $RUNDIR

rm -rf simv* run.log

vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd \
  +incdir+../01-pymtl-rtlsim \
  ../01-pymtl-rtlsim/RegIncr4stage__pickled.v \
  ../01-pymtl-rtlsim/RegIncr4stage_basic_tb.v | tee run.log

./simv | tee -a run.log
```

Run the script:

```bash
% cd $TOPDIR/asic/playground/02-synopsys-vcs-rtlsim
% ./run
```

**Step 3: Synopsys DC Synthesis**

Edit `03-synopsys-dc-synth/run.tcl`. The key changes for a sequential
design are clock constraints and using `compile_ultra` instead of
`compile`:

 - **Change**: `AdderRippleCarry_4b__pickled.v` to `RegIncr4stage__pickled.v`
 - **Change**: `AdderRippleCarry_4b` to `RegIncr4stage` (in `elaborate` and `set_max_transition`)
 - **Add**: `create_clock clk -name clk -period 1.0` (after `set_load`)
 - **Add**: `set_input_delay` commands for clock-relative input timing
 - **Add**: `set_output_delay` commands for clock-relative output timing
 - **Change**: `compile` to `compile_ultra -no_autoungroup -gate_clock`

The `-no_autoungroup` option preserves the design hierarchy, and `-gate_clock`
enables clock gating optimizations for power reduction.

The updated script should be:

```tcl
set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
set_app_var report_default_significant_digits 4

analyze -format sverilog ../01-pymtl-rtlsim/RegIncr4stage__pickled.v
elaborate RegIncr4stage

set_max_transition 0.250 RegIncr4stage
set_driving_cell -lib_cell INVD1BWP7T [all_inputs]
set_load 0.005 [all_outputs]

create_clock clk -name clk -period 1.0

set_input_delay -clock clk -max 0.050 [all_inputs -exclude_clock_ports]
set_input_delay -clock clk -min 0.000 [all_inputs -exclude_clock_ports]

set_output_delay -clock clk -max 0.050 [all_outputs]
set_output_delay -clock clk -min 0.000 [all_outputs]

set_max_delay 1.0 -from [all_inputs -exclude_clock_ports] -to [all_outputs]

compile_ultra -no_autoungroup -gate_clock

write -format ddc     -hierarchy -output post-synth.ddc
write -format verilog -hierarchy -output post-synth.v
write_sdc                                post-synth.sdc

report_timing -nets      > timing.rpt
report_area   -hierarchy > area.rpt

exit
```

Run synthesis:

```bash
% cd $TOPDIR/asic/playground/03-synopsys-dc-synth
% ./run
% less area.rpt
% less timing.rpt
```

**Step 4: Fast-Functional Gate-Level Simulation**

Edit `04-synopsys-vcs-ffglsim/run`:

 - **Change**: `AdderRippleCarry_4b_test_exhaustive_tb.v` to `RegIncr4stage_basic_tb.v`

The updated script should be:

```bash
set -e
trap 'echo "CMD: $BASH_COMMAND"' DEBUG

RUNDIR="$(dirname $(readlink -f "${BASH_SOURCE[0]}"))"
cd $RUNDIR

rm -rf simv* run.log

vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd \
  +incdir+../01-pymtl-rtlsim \
  ${TSMC_180NM}/stdcells.v \
  ../03-synopsys-dc-synth/post-synth.v \
  ../01-pymtl-rtlsim/RegIncr4stage_basic_tb.v | tee run.log

./simv | tee -a run.log
```

Run the script:

```bash
% cd $TOPDIR/asic/playground/04-synopsys-vcs-ffglsim
% ./run
```

### 3.2. Cadence Innovus for Place-and-Route

The registered incrementer is a larger sequential design that requires
a more sophisticated place-and-route flow including clock tree synthesis
and power grid routing.

Change to the PNR directory:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
```

**Constraint and Timing Input Files**

The `setup-timing.tcl` file is identical to the addrc4b version since
it only references the SDC file from synthesis (which already contains
the design-specific constraints).

**PNR Script Changes**

Edit `run.tcl` to implement the PNR flow for a sequential design. The
commands shown below use the `innovus>` prompt for clarity, but you should
add them to your `run.tcl` file without the prompt. After making your
edits, run `./run` to execute the flow.

Update the top cell name and power/ground net definitions:

```
innovus> set init_top_cell  "RegIncr4stage"
innovus> set init_pwr_net   {VDD vdd}
innovus> set init_gnd_net   {VSS gnd}
```

After `init_design`, add process mode and optimization settings:

 - **setDesignMode -process 180**: Specifies the process technology node
   (180nm); affects timing calculations and design rule checking
 - **setDelayCalMode -SIAware false**: Disables signal integrity (SI)
   aware delay calculation; SI analysis models crosstalk effects between
   adjacent wires but adds complexity and is not needed for this lab
 - **setOptMode -holdTargetSlack 0.010**: Sets the target hold time slack
   to 10ps; the optimizer will insert buffers to ensure all paths meet
   this minimum slack margin
 - **setOptMode -holdFixingCells**: Specifies which buffer cells from the
   standard cell library can be used to fix hold time violations; these
   are various drive-strength buffers (BUFFD0 through BUFFD12)

```
innovus> setDesignMode -process 180
innovus> setDelayCalMode -SIAware false
innovus> setOptMode -holdTargetSlack 0.010
innovus> setOptMode -holdFixingCells {BUFFD0BWP7T BUFFD10BWP7T BUFFD12BWP7T BUFFD1BWP7T BUFFD1P5BWP7T BUFFD2BWP7T BUFFD2P5BWP7T BUFFD3BWP7T BUFFD4BWP7T BUFFD5BWP7T BUFFD6BWP7T BUFFD8BWP7T}
```

Change the floorplan margins from 0.5um to 10um for the power ring:

```
innovus> floorPlan -r 1.0 0.70 10.0 10.0 10.0 10.0
```

**Power Routing**

Connect power and ground nets. Add tiehi/tielo connections (new):

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

Add a power ring on M5 and M6 around the outside of the core area (new):

```
innovus> addRing -nets {VDD VSS} -width 2.6 -spacing 2.5 -layer [list top 6 bottom 6 left 5 right 5] -extend_corner {tl tr bl br lt lb rt rb}
```

Route power stripes:

```
innovus> addStripe -nets {VSS VDD} -layer 6 -direction horizontal -width 5.52 -spacing 16.88 -set_to_set_distance 44.8 -start_offset 22.4
innovus> addStripe -nets {VSS VDD} -layer 5 -direction vertical -width 5.52 -spacing 16.88 -set_to_set_distance 44.8 -start_offset 22.4
innovus> sroute -nets {VDD VSS}
```

**Placement**

Place and optimize the design (use `place_opt_design` instead of `place_design`):

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

Route the design:

```
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
```

Save outputs and write reports:

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

After updating `run.tcl` with all the above changes, run the script:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% ./run
```

View the layout in Klayout:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 3.3. Synopsys PrimeTime for Static Timing Analysis

Perform static timing analysis on the post-place-and-route design:

```bash
% mkdir -p $TOPDIR/asic/playground/06-synopsys-pt-sta
% cd $TOPDIR/asic/playground/06-synopsys-pt-sta
% pt_shell
```

Set up the standard cell library:

```
pt_shell> set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
pt_shell> set_app_var link_library [concat "*" $target_library]
```

Read in the design and parasitics:

```
pt_shell> read_verilog ../05-cadence-innovus-pnr/post-pnr.v
pt_shell> current_design RegIncr4stage
pt_shell> link_design
pt_shell> read_parasitics -format spef ../05-cadence-innovus-pnr/post-pnr.spef
```

Set timing constraints for the sequential design similarly to the synthesis
step:

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

Perform timing analysis and write reports:

```
pt_shell> update_timing
pt_shell> report_global_timing -delay_type max > setup-summary.rpt
pt_shell> report_global_timing -delay_type min > hold-summary.rpt
pt_shell> report_timing -nets  -delay_type max > setup-detailed.rpt
pt_shell> report_timing -nets  -delay_type min > hold-detailed.rpt
```

Exit Synopsys PT:

```
pt_shell> exit
```

**Creating the Run Script**

Now put these commands into a `run.tcl` file (without the `pt_shell>` prompt)
and create a `run` shell script:

```bash
#!/usr/bin/env bash
set -e
trap 'echo "CMD: $BASH_COMMAND"' DEBUG
RUNDIR="$(dirname $(readlink -f "${BASH_SOURCE[0]}"))"
cd $RUNDIR
pt_shell -f run.tcl | tee run.log
```

Make it executable:

```bash
% chmod +x run
```

Take a moment to review the generated reports which are similar to those
generated after the synthesis step. You should see that the design has positive
setup and hold slack with no violating paths.

### 3.4. Synopsys VCS for Back-Annotated Gate-Level Simulation

Back-annotated gate-level simulation will take into account all of the
gate and interconnect delays. This helps verify not just that the final
gate-level netlist is functionally correct, but also that it meets all
setup and hold time constraints.

Change to the working directory (already provided):

```bash
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
```

Run VCS for back-annotated gate-level simulation. Compared to ffglsim,
baglsim adds several new arguments:

 - **+neg_tchk**: Enable negative timing checks, which detect hold time
   violations where signals change too soon after the clock edge
 - **+sdfverbose**: Print verbose messages during SDF annotation to help
   debug any timing back-annotation issues
 - **-sdf max:Top.DUT:...**: Back-annotate timing delays from the SDF
   file generated by PNR; `max` specifies worst-case (slow) delays, and
   `Top.DUT` is the hierarchical path to the design under test
 - **+define+CYCLE_TIME=3.0**: Set the clock period to 3.0ns (slower than
   the synthesis constraint) to ensure the design meets timing with
   realistic gate and wire delays
 - **+define+VTB_INPUT_DELAY=0.025**: Input delay (25ps) for the testbench
   to model realistic input timing relative to the clock
 - **+define+VTB_OUTPUT_DELAY=0.025**: Output delay (25ps) for the
   testbench to check outputs at realistic times after the clock edge
 - Uses **post-pnr.v** instead of post-synth.v since we are simulating
   the placed-and-routed netlist

```bash
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
% vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
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
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
% ./simv-test
```

The simulation should pass all tests. View the waveforms:

```bash
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
% code waves.vcd
```

Zoom in and notice how the signals now change throughout the cycle. This
is because the delay of every gate and wire is now modeled.

### 3.5. Mentor Calibre for Design Rules Check (DRC)

We use Mentor Calibre for DRC verification as described in the
introduction. The runset file (`main_drc.rs`) is already provided and
configures Calibre with the design-specific settings.

Change to the working directory:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
```

Run the main DRC check:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
```

Here is an explanation of the Calibre options:

 - **`-gui`**: Use the graphical interface (required even in batch mode)
 - **`-drc`**: Run design rules check
 - **`-runset main_drc.rs`**: Use the specified runset file
 - **`-batch`**: Run in batch mode (non-interactive)

**Note:** The simple PNR flow we used in this lab will likely produce
DRC violations. A production PNR flow includes additional commands for
fixing violations, such as via generation rules, antenna violation
fixes, and metal fill insertion. When you use the automated ASIC flow
later in the course, the provided runscripts will include these
additional commands to produce DRC-clean layouts.

To view the DRC results interactively, first copy the layer properties
file and then launch Calibre DRV:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

This opens the Calibre DRV GUI which allows you to browse DRC violations.
You can click on any violation to highlight its location in the layout.
Explore the violations to understand what types of design rule checks
are being performed.

### 3.6. Mentor Calibre for Layout Versus Schematic (LVS)

We use Mentor Calibre for LVS verification as described in the
introduction. The runset file (`lvs.rs`) is already provided.

Change to the working directory:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
```

First, we need to convert the Verilog gate-level netlist to SPICE format
so that Calibre can compare it against the extracted layout. We use the
`v2lvs` tool for this conversion:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
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
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

**Note:** Similar to DRC, the simple PNR flow may produce LVS
mismatches due to missing power connections or other issues that a
production flow would handle. When you use the automated ASIC flow
later in the course, the provided runscripts will produce LVS-clean
layouts.

To view the LVS results interactively:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```

This opens the Calibre DRV GUI which shows the LVS comparison results.
Explore the results to understand how LVS compares the extracted layout
connectivity against the schematic netlist.

### 3.7. Archive the Registered Incrementer

Before moving on to the GCD accelerator, archive the playground directory:

```bash
% cd $TOPDIR/asic
% cp -r playground playground-regincr
```

You will continue to use the `playground` directory for the GCD accelerator,
modifying the runscripts as needed.

4. GCD Accelerator
--------------------------------------------------------------------------

In this section, we will modify the runscripts to push the GCD accelerator
through the ASIC flow. The GCD accelerator is similar to the registered
incrementer (both are sequential designs), but it is more complex and
requires a longer clock period.

**Additional Step for gcd-xcel:** For the GCD accelerator, we will perform
one additional step beyond what was done for regincr:

 - **Step 08**: Power analysis with Synopsys PrimeTime

This requires generating SAIF (Switching Activity Interchange Format) files
during back-annotated simulation, which capture the switching activity of
every net for accurate power estimation.

![](img/lab5-gcd.png)

### 4.1. Front-End Flow Changes

The GCD accelerator requires different test files and a longer clock
period due to its more complex logic.

**Step 1: PyMTL RTL Simulation**

Edit `01-pymtl-rtlsim/run`:

 - **Remove**: The regincr simulator command
 - **Add**: pytest command for GcdXcel testing (see below)
 - **Add**: gcd-xcel-sim command for evaluation test bench generation (see below)

```bash
pytest ${SIMDIR}/lab5_xcel/test/GcdXcel_test.py -k random_3x9 --test-verilog --dump-vtb
${SIMDIR}/lab5_xcel/gcd-xcel-sim --impl rtl --translate --dump-vtb
```

This runs both a pytest-based test (`random_3x9`) and a simulator to
generate evaluation test benches. Run the script:

```bash
% cd $TOPDIR/asic/playground/01-pymtl-rtlsim
% ./run
```

**Step 2: VCS RTL Simulation**

Edit `02-synopsys-vcs-rtlsim/run`:

 - **Change**: `rm -rf simv* run.log` to `rm -rf simv-test* simv-eval* run.log`
 - **Change**: `RegIncr4stage__pickled.v` to `GcdXcel_noparam__pickled.v`
 - **Change**: `RegIncr4stage_basic_tb.v` to `GcdXcel_noparam_test_random_3x9_tb.v`
 - **Add**: `-o simv-test` to name the test simulation executable
 - **Add**: Second simulation block for evaluation with `GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v`, it should use the `-o simv-eval` executable name

The updated script should be:

```bash
rm -rf simv-test* simv-eval* run.log

# Test simulation
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd -o simv-test \
  +incdir+../01-pymtl-rtlsim \
  ../01-pymtl-rtlsim/GcdXcel_noparam__pickled.v \
  ../01-pymtl-rtlsim/GcdXcel_noparam_test_random_3x9_tb.v | tee run.log

./simv-test | tee -a run.log

# Evaluation simulation
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd -o simv-eval \
  +incdir+../01-pymtl-rtlsim \
  ../01-pymtl-rtlsim/GcdXcel_noparam__pickled.v \
  ../01-pymtl-rtlsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v | tee -a run.log

./simv-eval | tee -a run.log
```

Run the script:

```bash
% cd $TOPDIR/asic/playground/02-synopsys-vcs-rtlsim
% ./run
```

**Step 3: Synopsys DC Synthesis**

Edit `03-synopsys-dc-synth/run.tcl`:

 - **Change**: `RegIncr4stage__pickled.v` to `GcdXcel_noparam__pickled.v`
 - **Change**: `RegIncr4stage` to `GcdXcel_noparam` (in `elaborate` and `set_max_transition`)
 - **Change**: `create_clock clk -name clk -period 1.0` to `create_clock clk -name clk -period 3.0`
 - **Change**: `set_max_delay 1.0` to `set_max_delay 3.0`

The GCD accelerator needs a longer clock period (3.0ns vs 1.0ns) because
it has more complex combinational logic. The updated script should be:

```tcl
set_app_var target_library "$env(TSMC_180NM)/stdcells.db"
set_app_var link_library   "* $env(TSMC_180NM)/stdcells.db"
set_app_var report_default_significant_digits 4

analyze -format sverilog ../01-pymtl-rtlsim/GcdXcel_noparam__pickled.v
elaborate GcdXcel_noparam

set_max_transition 0.250 GcdXcel_noparam
set_driving_cell -lib_cell INVD1BWP7T [all_inputs]
set_load 0.005 [all_outputs]

create_clock clk -name clk -period 3.0

set_input_delay -clock clk -max 0.050 [all_inputs -exclude_clock_ports]
set_input_delay -clock clk -min 0.000 [all_inputs -exclude_clock_ports]

set_output_delay -clock clk -max 0.050 [all_outputs]
set_output_delay -clock clk -min 0.000 [all_outputs]

set_max_delay 3.0 -from [all_inputs -exclude_clock_ports] -to [all_outputs]

compile_ultra -no_autoungroup -gate_clock

write -format ddc     -hierarchy -output post-synth.ddc
write -format verilog -hierarchy -output post-synth.v
write_sdc                                post-synth.sdc

report_timing -nets      > timing.rpt
report_area   -hierarchy > area.rpt

exit
```

Run synthesis:

```bash
% cd $TOPDIR/asic/playground/03-synopsys-dc-synth
% ./run
% less area.rpt
% less timing.rpt
```

**Step 4: Fast-Functional Gate-Level Simulation**

Edit `04-synopsys-vcs-ffglsim/run`:

 - **Change**: `rm -rf simv* run.log` to `rm -rf simv-test* simv-eval* run.log`
 - **Change**: `RegIncr4stage_basic_tb.v` to `GcdXcel_noparam_test_random_3x9_tb.v`
 - **Add**: `-o simv-test` to name the test simulation executable
 - **Add**: Second simulation block for evaluation with `GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v`, it should use the `-o simv-eval` executable name

The updated script should be:

```bash
rm -rf simv-test* simv-eval* run.log

# Test simulation
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd -o simv-test \
  +incdir+../01-pymtl-rtlsim \
  ${TSMC_180NM}/stdcells.v \
  ../03-synopsys-dc-synth/post-synth.v \
  ../01-pymtl-rtlsim/GcdXcel_noparam_test_random_3x9_tb.v | tee run.log

./simv-test | tee -a run.log

# Evaluation simulation
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
  +vcs+dumpvars+waves.vcd -o simv-eval \
  +incdir+../01-pymtl-rtlsim \
  ${TSMC_180NM}/stdcells.v \
  ../03-synopsys-dc-synth/post-synth.v \
  ../01-pymtl-rtlsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v | tee -a run.log

./simv-eval | tee -a run.log
```

Run the script:

```bash
% cd $TOPDIR/asic/playground/04-synopsys-vcs-ffglsim
% ./run
```

### 4.2. Cadence Innovus for Place-and-Route

The PNR script for the GCD accelerator is essentially identical to the
registered incrementer. The only change is the top cell name.

Change to the PNR directory:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
```

Edit `run.tcl`:

 - **Change**: `set init_top_cell "RegIncr4stage"` to `set init_top_cell "GcdXcel_noparam"`

Everything else (power ring, clock tree synthesis, optimization) remains
the same since both designs are sequential and use similar floorplanning
parameters.

Run place-and-route:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% ./run
```

View the final layout:

```bash
% cd $TOPDIR/asic/playground/05-cadence-innovus-pnr
% klayout -l ${TSMC_180NM}/klayout.lyp post-pnr.gds
```

### 4.3. Synopsys PrimeTime for Static Timing Analysis

Perform static timing analysis as with the registered incrementer.

```bash
% cd $TOPDIR/asic/playground/06-synopsys-pt-sta
```

Edit `run.tcl` with the following changes:

 - **Change**: `current_design RegIncr4stage` to `current_design GcdXcel_noparam`
 - **Change**: `create_clock clk -name ideal_clock1 -period 1.0` to `create_clock clk -name ideal_clock1 -period 3.0`
 - **Change**: `set_max_delay 1.0` to `set_max_delay 3.0`
 - **Change**: `set_max_transition 0.250 RegIncr4stage` to `set_max_transition 0.250 GcdXcel_noparam`

Run the script:

```bash
% ./run
```

### 4.4. Synopsys VCS for Back-Annotated Gate-Level Simulation

The back-annotated simulation for the GCD accelerator includes SAIF
generation for power analysis.

```bash
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
```

Edit `07-synopsys-vcs-baglsim/run` with the following changes:

 - **Change**: `RegIncr4stage_basic_tb.v` to `GcdXcel_noparam_test_random_3x9_tb.v`
 - **Add**: `-o simv-test` to name the test simulation executable
 - **Add**: Second simulation block for evaluation with SAIF generation
 - **Add**: `+define+VTB_DUMP_SAIF=...` to generate switching activity file for the second simulation block

Run the test simulation:

```bash
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +vcs+dumpvars+waves.vcd \
    -o simv-test \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/GcdXcel_noparam_test_random_3x9_tb.v
./simv-test
```

Run the evaluation simulation with SAIF generation (new):

```bash
vcs -sverilog -xprop=tmerge -override_timescale=1ns/1ps -top Top \
    +neg_tchk +sdfverbose \
    -sdf max:Top.DUT:../05-cadence-innovus-pnr/post-pnr.sdf \
    +define+CYCLE_TIME=3.0 \
    +define+VTB_INPUT_DELAY=0.025 \
    +define+VTB_OUTPUT_DELAY=0.025 \
    +define+VTB_DUMP_SAIF=GcdXcel_noparam_gcd-xcel-sim-rtl-random.saif \
    +vcs+dumpvars+waves.vcd \
    -o simv-eval \
    +incdir+../01-pymtl-rtlsim \
    ${TSMC_180NM}/stdcells.v \
    ../05-cadence-innovus-pnr/post-pnr.v \
    ../01-pymtl-rtlsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random_tb.v
./simv-eval
```

The `+define+VTB_DUMP_SAIF=...` option generates a SAIF file as described
in the section introduction.

Run the script:

```bash
% cd $TOPDIR/asic/playground/07-synopsys-vcs-baglsim
% ./run
```

### 4.5. Synopsys PrimeTime for Power Analysis

This is a new step that was not needed for the previous designs. Power analysis
uses the SAIF file generated during back-annotated simulation for evaluations to
calculate power consumption based on actual switching activity.

```bash
% mkdir -p $TOPDIR/asic/playground/08-synopsys-pt-pwr
% cd $TOPDIR/asic/playground/08-synopsys-pt-pwr
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

Read in the SAIF file from BAGL simulation with activity factors and the SPEF
file with parasitic capacitances:

```
pt_shell> read_saif "../07-synopsys-vcs-baglsim/GcdXcel_noparam_gcd-xcel-sim-rtl-random.saif" -strip_path "Top/DUT"
pt_shell> read_parasitics -format spef "../05-cadence-innovus-pnr/post-pnr.spef"
```

The `-strip_path "Top/DUT"` option tells PrimeTime to strip the
testbench hierarchy prefix from the signal names in the SAIF file so
they match the design hierarchy.

Set the clock period (must match the simulation):

```
pt_shell> create_clock clk -name ideal_clock1 -period 3.0
```

Perform power analysis and write reports:

```
pt_shell> update_power
pt_shell> report_power > power-summary.rpt
pt_shell> report_power -hierarchy > power-hierarchy.rpt
```

The `power-summary.rpt` file shows the total power broken down into:
- **Switching power**: Due to charging/discharging load capacitances
- **Internal power**: Due to short-circuit current during transitions
- **Leakage power**: Due to static leakage through transistors

The `power-hierarchy.rpt` file shows power broken down by module,
which helps identify power-hungry components.

Exit PrimeTime:

```
pt_shell> exit
```

**Creating the Run Script**

Now put these commands into a `run.tcl` file (without the `pt_shell>` prompt)
and create a `run` shell script:

```bash
#!/usr/bin/env bash
set -e
trap 'echo "CMD: $BASH_COMMAND"' DEBUG
RUNDIR="$(dirname $(readlink -f "${BASH_SOURCE[0]}"))"
cd $RUNDIR
pt_shell -f run.tcl | tee run.log
```

Make it executable:

```bash
% chmod +x run
```

### 4.6. Mentor Calibre for Design Rules Check (DRC)

Change to the DRC directory:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
```

Edit `main_drc.rs`:

 - **Change**: `RegIncr4stage` to `GcdXcel_noparam` (top cell name)

Run the main DRC check:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
% calibre -gui -drc -runset main_drc.rs -batch
```

As noted earlier, the simple PNR flow will produce DRC violations.
View the results interactively:

```bash
% cd $TOPDIR/asic/playground/09-mentor-calibre-drc
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -drc main_drc/drc.results
```

### 4.7. Mentor Calibre for Layout Versus Schematic (LVS)

Change to the LVS directory:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
```

Convert the Verilog netlist to SPICE format:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
% v2lvs -v ../05-cadence-innovus-pnr/post-pnr.v \
    -o post-pnr.sp \
    -lsr ${TSMC_180NM}/stdcells.sp \
    -s ${TSMC_180NM}/stdcells.sp \
    -log v2lvs.log
```

Edit `lvs.rs`:

 - **Change**: `RegIncr4stage` to `GcdXcel_noparam` (top cell name)

Run Calibre LVS:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
% calibre -gui -lvs -runset lvs.rs -batch
```

As noted earlier, the simple PNR flow may produce LVS mismatches.
View the results interactively:

```bash
% cd $TOPDIR/asic/playground/10-mentor-calibre-lvs
% cp $TSMC_180NM/calibre.layerprops ../05-cadence-innovus-pnr/post-pnr.gds.layerprops
% calibredrv -m ../05-cadence-innovus-pnr/post-pnr.gds -rve -lvs lvs_results/svdb
```
