
ECE 6745 Lab 8: SRAMs and ASIC Automated Flow
==========================================================================

In this section, we will introduce the SRAM compiler flow which takes a PDK's
transistor models and generates an SRAM array complete with the bitcells and all
peripheral circuitry for accessing the read and write lines. Since SRAMs are
commonly used in varying sizes, it is convenient to be able to simply generate
one given input specifications.

Additionally, we will go over the ASIC automated flow. In the previous sections,
we manually entered commands for each tool to take a design from RTL
to layout. Flow scripts can help automate the process but copying and modifying
these flow scripts for every design is tedious and error prone. An agile
hardware design flow demands automation to simplify rapidly exploring the area,
energy, timing design space of one or more designs. In this section, we will
introduce a simple tool called pyhflow which takes as input a _step templates_
and a _design YAML_ and generates appropriate flow scripts.

The following diagrams illustrate the seven primary tools we have already
seen in the previous discussion sections. Notice that the ASIC tools all
require various views from the standard-cell library.

![](img/tut08-asic-flow.png)
![](img/lab7-drc-lvs.png)

Extensive documentation is provided by Synopsys, Cadence, and Siemens for these
ASIC tools. We have organized this documentation and made it available to you on
the public course webpage:

 - <https://www.csl.cornell.edu/courses/ece6745/asicdocs>

!!! warning "Students MUST work in pairs!"

  You MUST work in pairs for this lab, as having too many instances of
  Innovus open at once can cause the `ecelinux` servers to crash. So
  find a partner and work together at a workstation to complete today's
  lab.

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
% git clone git@github.com:cornell-ece6745/ece6745-lab8 lab8
% cd lab8
% export TOPDIR=$PWD
```

1. SRAM Compiler Flow
--------------------------------------------------------------------------

2. pyhflow For Generating Flows
--------------------------------------------------------------------------

pyhflow is based on the idea of _step templates_ which are located in the
`asic/steps/block` directory.

```bash
% cd $TOPDIR/asic/steps/block
% tree
.
├── 00-openram-memgen
│   └── run
├── 01-pymtl-rtlsim
│   └── run
├── 02-synopsys-vcs-rtlsim
│   └── run
├── 03-synopsys-dc-synth
│   ├── dc-warnings.tcl
│   ├── run
│   └── run.tcl
├── 04-synopsys-vcs-ffglsim
│   └── run
├── 05-cadence-innovus-pnr
│   ├── innovus-warnings.tcl
│   ├── run
│   ├── run.tcl
│   └── setup-timing.tcl
├── 06-synopsys-pt-sta
│   ├── run
│   └── run.tcl
├── 07-synopsys-vcs-baglsim
│   ├── calc-clk-ins-src-lat
│   └── run
├── 08-synopsys-pt-pwr
│   ├── run
│   └── run.tcl
├── 09-mentor-calibre-drc
│   ├── run_antenna.rs
│   ├── run_main.rs
│   ├── print-drc-summary
│   ├── run
│   └── run-interactive
├── 10-mentor-calibre-lvs
│   ├── run.rs
│   ├── print-lvs-summary
│   ├── run
│   └── run-interactive
└── 11-summarize-results
    ├── run
    └── summarize-results

```

Each step is a directory with a run script and possibly other scripts.
The key difference from the scripts we used in the previous tutorials, is
that these scripts are templated using the Jinja2 templating system:

 - <https://jinja.palletsprojects.com>

Open the `run.tcl` script in the `03-synopsys-dc-synth` step template
which uses Synopsys DC for synthesis.

```
% cd $TOPDIR/asic/steps/block/03-synopsys-dc-synth
% code run.tcl
```

Notice how the `run.tcl` script is templated based on the design name and
the target clock period.

```
analyze -format sverilog ../01-pymtl-rtlsim/{{design_name}}__pickled.v
elaborate {{design_name}}

create_clock clk -name ideal_clock1 -period {{clock_period}}
```

The `{{ }}` directive is the standard syntax for template variable
substitution using Jinja2.

The pyhflow program takes as input a design YAML file which specifies:

 - what steps make up the flow
 - key/value pairs for variables to substitute into scripts
 - list of tests
 - list of evals

Take a look at the provided design YAML file for the GCD accelerator.

```bash
% cd $TOPDIR/asic/designs
% cat lab5-gcd-xcel.yml

steps:
 - block/01-pymtl-rtlsim
 - block/02-synopsys-vcs-rtlsim
 - block/03-synopsys-dc-synth
 - block/04-synopsys-vcs-ffglsim
 - block/05-cadence-innovus-pnr
 - block/06-synopsys-pt-sta
 - block/07-synopsys-vcs-baglsim
 - block/08-synopsys-pt-pwr
 - block/09-mentor-calibre-drc
 - block/10-mentor-calibre-lvs
 - block/11-summarize-results

design_name  : GcdXcel_noparam
clock_period : 3.0
dump_vcd     : true

pymtl_rtlsim:
 - pytest ../../../sim/lab5_xcel/test/GcdXcel_test.py --test-verilog --dump-vtb
 - ../../../sim/lab5_xcel/gcd-xcel-sim --impl rtl --input random --stats --translate --dump-vtb
 - ../../../sim/lab5_xcel/gcd-xcel-sim --impl rtl --input small --stats --translate --dump-vtb
 - ../../../sim/lab5_xcel/gcd-xcel-sim --impl rtl --input zeros --stats --translate --dump-vtb

tests:
 - GcdXcel_noparam_test_basic
 - GcdXcel_noparam_test_zeros
 - GcdXcel_noparam_test_equal
 - GcdXcel_noparam_test_divides
 - GcdXcel_noparam_test_common_factors
 - GcdXcel_noparam_test_coprime
 - GcdXcel_noparam_test_powers_of_two
 - GcdXcel_noparam_test_larger_numbers
 - GcdXcel_noparam_test_swap_ordering
 - GcdXcel_noparam_test_fibonacci
 - GcdXcel_noparam_test_sub_stress
 - GcdXcel_noparam_test_random_0x0
 - GcdXcel_noparam_test_random_5x0
 - GcdXcel_noparam_test_random_0x5
 - GcdXcel_noparam_test_random_3x9

evals:
 - GcdXcel_noparam_gcd-xcel-sim-rtl-random
 - GcdXcel_noparam_gcd-xcel-sim-rtl-small
 - GcdXcel_noparam_gcd-xcel-sim-rtl-zeros

```

This design YAML file specifies the generated flow should use all eleven steps.
We run PyMTL 2-state RTL sim, VCS 4-state RTL sim, FFGL sim, and BAGL sim on all
tests and evals, but we only do energy analysis on the evals. The evals usually
come from running an interactive simulator like `gcd-xcel-sim`. All pyhflow does
is use the YAML file to figure out what to substitute into the templated steps
and then copy the run scripts into the current working directory. You can also
override parameters on pyhflow command line.

### 2.1. Running ASIC Flow with One Test

Let's go ahead and use pyhflow to generate the flow scripts for the
GCD accelerator.

```bash
% mkdir -p $TOPDIR/asic/build-gcd-xcel
% cd $TOPDIR/asic/build-gcd-xcel
% pyhflow --one-test ../designs/lab5-gcd-xcel.yml
```

The `--one-test` command line option tells pyhflow to only include the
first test and no evals in the flow scripts. This is a useful way to get
started with a single test and reduces the overall runtime of the flow.
Once we know that everything works with one test we can circle back and
regenerate the flow scripts with all of the tests and evals.

Let's see how the step template has been filled in for the Synopsys DC
synthesis step.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% cat 03-synopsys-dc-synth/run.tcl
...
analyze -format sverilog ../01-pymtl-rtlsim/GcdXcel_noparam__pickled.v
elaborate GcdXcel_noparam
create_clock clk -name ideal_clock1 -period 3.0
```

Notice how the name of the source Verilog RTL File, the top-level
modulename, and the clock period have all been filled in.

After generating a flow, we always recommend explicitly running at least
the first two steps to ensure there are no errors. You can run the
four-state RTL simulation as follows.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% ./02-synopsys-vcs-rtlsim/run
```

Make sure the step can find the source files and passes the test. Then
run synthesis as follows.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% ./03-synopsys-dc-synth/run
```

Carefully look at the output from the synthesis step (also stored in the
`run.log` file). Look for the output after `Running PRESTO HDLC` for any
warnings to ensure that all of your Verilog RTL is indeed synthesizable.
Scan through the rest of the logs to ensure there are no worrying
warnings or errors.

Once you have explicitly run the first two steps to ensure there are no
errors, you can run the remaining steps.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% ./04-synopsys-vcs-ffglsim/run
% ./05-cadence-innovus-pnr/run
% ./06-synopsys-pt-sta/run
% ./07-synopsys-vcs-baglsim/run
% ./08-synopsys-pt-pwr/run
% ./09-mentor-calibre-drc/run
% ./10-mentor-calibre-lvs/run
% ./11-summarize-results/run
```

If all looks good, then you would regenerate the flow with all of the tests
and evals; however, we will stick to just running one test though to save
time in this discussion section. **pyhflow will also create a `run-flow`
script which will run all of the steps in sequence for you, but only use
this if you are confident there are no errors!**

For the results to be valid, the following must be true:

 - all PyMTL 2-state RTL simulations pass
 - all four-state RTL simulations pass
 - all fast-functional gate-level simulations pass
 - all back-annotated gate-level simulations pass
 - place-and-route setup slack is positive
 - place-and-route hold slack is positive
 - static timing analysis (PrimeTime) setup slack is positive
 - static timing analysis (PrimeTime) hold slack is positive
 - DRC contains no violations
 - LVS contains no violations

If your design does not meet timing after synthesis but _does_ meet timing after
place-and-route then these are still valid results. It just means Synopsys DC
was conservative and/or Cadence Innovus did a good job further optimizing the
design. If PrimeTime STA passes timing but Innovus fails timing, **this is not
valid - you must go back and modify your design to fix this.** Additionally,
your design must be DRC and LVS clean - use the `run-interactive` scripts in the
corresponding step directories to debug any violations. If you have a DRC or LVS
violation, you likely need to change your clock period constraint - placement
and routing can produce vastly different results depending on its value.

You will notice that the step template scripts contain many extra
commands compared to the scripts we used in the previous lab. These
additional commands are necessary to ensure that more complex designs
pass DRC and LVS checks. You can look in the ASIC tool documentation
referenced above to find out what each command does.

In addition to the main DRC deck, there is now a separate _antenna_ DRC
deck. Antenna violations occur when long metal wires connected to
transistor gates accumulate charge during the manufacturing process.
During fabrication, each metal layer is deposited and etched one at a
time, and exposed metal wires can act like antennas that collect charge
from the plasma used in the etching process. If too much charge builds
up on a wire connected to a thin gate oxide, it can damage or destroy
the transistor gate. The antenna DRC deck checks for wires that exceed
the maximum allowed antenna ratio. The place-and-route step template
includes additional commands to automatically detect and fix antenna
violations by inserting diodes that provide a discharge path for the
accumulated charge.

### 2.2. Interactive Debugging

Let's start Cadence Innovus in interactive mode and then load the design.

```bash
% cd $TOPDIR/asic/build-gcd-xcel/05-cadence-innovus-pnr
% innovus
innovus> source post-pnr.enc
```

You can use Cadence Innovus to analyze the static timing of any path in
the design. For example, let's look at the static timing for a path in
the first stage:

```bash
innovus> report_timing -path_type full_clock -net \
  -from v/gcd/dpath/b_reg/q_reg_0_ \
  -to v/gcd/dpath/a_reg/q_reg_15_
```

You can use the Amoeba workspace to help visualize how modules are mapped
across the chip. Choose _Windows > Workspaces > Amoeba_ from the menu.
However, we recommend using the design browser to help visualize how
modules are mapped across the chip. Here are the steps:

 - Choose _Windows > Workspaces > Design Browser + Physical_ from the menu
 - Hide all of the metal layers by pressing the number keys
 - Browse the design hierarchy using the panel on the left
 - Right click on a module, click _Highlight_, select a color

Go ahead and highlight each stage in a different color.

You can use the following steps in Cadence Innovus to display where the
critical path is on the actual chip.

 - Choose _Timing > Debug Timing_ from the menu
 - Click _OK_ in the pop-up window
 - Right click on first path in the _Path List_
 - Choose _Highlight > Only This Path > Color_

Finally, you can use Klayout to capture a screen shot demonstrating that
you have successfully taken a design from RTL to layout.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% klayout -l $TSMC_180NM/klayout.lyp \
  05-cadence-innovus-pnr/post-pnr.gds
```

You can use _Display > Full Hierarchy_ to show all of the layout
including the layout inside the standard cells. You can use _Display >
Decrement Hierarchy_ and _Display > Decrement Hierarchy_ to show/hide the
layout inside the standard cells to focus on the routing. Consider hiding
M7, VIA7, M8, VIA8, and M9 to just show the clock and signal routing. Try
toggling _View > Show Cell Frames_ to show/hide the standard cell
bounding boxes.

### 2.3. Key Reports

Let's look at some reports. Let's start by looking at the synthesis
resources report.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% cat 03-synopsys-dc-synth/resources.rpt
...
===============================================================================
|                    |                  | Current            | Set            |
| Cell               | Module           | Implementation     | Implementation |
===============================================================================
| lt_x_1             | DW_cmp           | apparch (area)     |                |
===============================================================================
```

This means that Synopsys DC is using a DesignWare module named
`DW_cmp`. You can read the datasheet here:

 - <https://web.csl.cornell.edu/courses/ece6745/asicdocs/dwbb_datasheets>

Notice that DesignWare provides different microarchitectures. For example the
adder has the following microarchitectures: a ripple-carry adder, a
carry-look-ahead adder, a delay optimized parallel-prefix adder, and an
area-optimized parallel-prefix adder. Synopsys DC will choose the appropriate
microarchitecture based on power, performance, and area targets for the current
design.

Now let's look at the post-place-and-route setup and hold time reports from
using Synopsys PrimeTime.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% cat 06-synopsys-pt-sta/timing-setup.rpt
% cat 06-synopsys-pt-sta/timing-hold.rpt
```

We can also look at the detailed area report.

```bash
% cd $TOPDIR/asic/build-gcd-xcel
% cat 05-cadence-innovus-pnr/area.rpt
```

3. Case Studies
--------------------------------------------------------------------------

Now that we know how to push a design through the automated flow, let's
consider two different case studies: (1) decreasing the clock period
constraint; and (2) flattening the design.

### 3.1. Decreasing the Clock Period Constraint

We can use pyhflow to regenerate the flow with a different clock period
by either: (1) changing the design YAML file (i.e., `lab5-gcd-xcel.yml`);
or (2) specifying the clock period on the pyhflow command line. Let's use
the second approach. If you look at the setup timing report you will see
with a 1ns clock period you have maybe 550ps of positive slack. So a good
starting point would be to maybe try a clock period of 1ns - 550ps =
450ps. Let's try 400ps.

```
% mkdir -p $TOPDIR/asic/build-gcd-xcel-decrease-clk
% cd $TOPDIR/asic/build-gcd-xcel-decrease-clk
% pyhflow --one-test --clock_period=0.400 ../designs/lab5-gcd-xcel.yml
% ./run-flow
```

Notice how we are working in a new build directory. You can use multiple
build directories to build different blocks through the flow and/or for
design-space exploration. Also notice how we are using `--one-test` so we
can quickly experiment with pushing the design through the flow with a
single test and no evals. You could continue to decrease the clock period
in 100ps increments until the design no longer meets timing, but for now
we will just stick with the shorter 400ps clock period. **Do not be too
zealous and push the tools to try and meet a clock period constraint that
is way too small! This can cause the tools to freak out and run
forever.**

Now compare the results from the longer and shorter clock periods. Start
by looking at the summary statistics. How does the number of standard
cells and area compare? We can also look at what kind of adder
implementation Synopsys DC chose to meet the shorter clock period
constraint.

```bash
% cd $TOPDIR/asic
% cat build-gcd-xcel/03-synopsys-dc-synth/resources.rpt
% cat build-gcd-xcel-decrease-clk/03-synopsys-dc-synth/resources.rpt
```

We can also compare the critical path; you should be able to see
that the design with the shorter clock period has many fewer levels of
logic on the critical path.

```bash
% cd $TOPDIR/asic
% cat build-gcd-xcel/06-synopsys-pt-sta/timing-setup.rpt
% cat build-gcd-xcel-decrease-clk/06-synopsys-pt-sta/timing-setup.rpt
```

### 3.2. Flattening the Design

Let's modify our scripts to flatten our design and see how this impacts
various metrics. We can run pyhflow to instantiate the flow scripts and
then modify these flow scripts in the build directory. Use the shortest
clock period that still meets timing from the previous case study.

```
% mkdir -p $TOPDIR/asic/build-gcd-xcel-flatten
% cd $TOPDIR/asic/build-gcd-xcel-flatten
% pyhflow --one-test --clock_period=XX ../designs/lab5-gcd-xcel.yml
% code 03-synopsys-dc-synth/run.tcl
```

Where `XX` is the shortest clock period which meets timing. We are
currently using the following command in `03-synopsys-dc-synth/run.tcl`
to synthesize our design.

```
compile_ultra -no_autoungroup -gate_clock
```

Change this by removing `-no_autoungroup`.

```
compile_ultra -gate_clock
```

Now run the flow.

```
% cd $TOPDIR/asic/build-gcd-xcel-flatten
% ./run-flow
```

Revisit the post-synthesis gate-level netlist without flattening.

```bash
% cd $TOPDIR/asic
% less build-gcd-xcel-decrease-clk/03-synopsys-dc-synth/post-synth.v
```

Notice how the original gate-level netlist preserves the logical
hierarchy. Now look at the post-synthesis gate-level netlist with
flattening.

```bash
% cd $TOPDIR/asic
% less build-gcd-xcel-flatten/03-synopsys-dc-synth/post-synth.v
```

Now notice how all of the logical hierarchy is gone and all of the gates
are in a single "flat" module. Compare the area without and with
flattening.

```bash
% cd $TOPDIR/asic
% cat build-gcd-xcel-decrease-clk/05-cadence-innovus-pnr/area.rpt
% cat build-gcd-xcel-flatten/05-cadence-innovus-pnr/area.rpt
```

Because the flattened module lacks logical hierarchy we cannot see the
hierarchical breakdown. The advantage of flattening is that it can
improve the area and also potentially enable a shorter clock period, but
the disadvantage is that it significantly complicates our ability to
deeply understand the area, energy, and timing of our designs and thus
effectively explore an entire design space. So we will primarily turn off
flattening in this course.

Note that if we wanted to make it easier to experiment with flattening,
we could modify the synthesis step template like this:

```
{% if flatten is defined and flatten %}
compile_ultra -gate_clock
{% else %}
compile_ultra -no_autoungroup -gate_clock
{% endif %}
```

Then in your design YAML file you can add this to control whether
flattening is turned on or off; or we can specify the value of the
flatten parameter as a pyhflow command line option (i.e., with
`--flatten=true`).

```
flatten : true
```

Students should feel free to modify the step templates and/or the design
YAML files for their labs and/or projects to experiment with the ASIC
flow.

4. Processor Comparative Analysis
--------------------------------------------------------------------------

Now that we understand the automated flow, let's use it to compare two
processor configurations: one with a GCD accelerator and one without
(i.e., a null accelerator). This kind of comparative analysis is common
in hardware design as we want to understand the area and power overhead
of adding an accelerator to a processor.

### 4.1. Processor with GCD Accelerator

Let's start by pushing the processor with the GCD accelerator through
the full flow. Take a look at the provided design YAML file.

```bash
% cd $TOPDIR/asic/designs
% cat lab8-proc-gcd-xcel.yml
```

Notice that the design name is `ProcMemXcel_v0_GcdXcel` and the clock
period is 8.2ns. The initialization commands cross-compile the
`ubmark-vvadd-test` and `ubmark-vvadd-eval` microbenchmarks and then run
the processor/memory/accelerator simulator with the GCD accelerator
implementation.

Let's use pyhflow to generate the flow scripts and run the flow.

```bash
% mkdir -p $TOPDIR/asic/build-proc-gcd-xcel
% cd $TOPDIR/asic/build-proc-gcd-xcel
% pyhflow ../designs/lab8-proc-gcd-xcel.yml
```

As before, we recommend explicitly running at least the first step manually to
ensure there are no errors.

```bash
% cd $TOPDIR/asic/build-proc-gcd-xcel
% ./01-pymtl-rtlsim/run
```

Make sure the step can find the source files and passes the test. Then
run VCS 4-state RTL simulation and then synthesis.

```bash
% cd $TOPDIR/asic/build-proc-gcd-xcel
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
```

Carefully look at the output from the synthesis step. Look for the
output after `Running PRESTO HDLC` for any warnings to ensure that all
of your Verilog RTL is indeed synthesizable. Once you have verified
there are no errors, run the remaining steps.

```bash
% cd $TOPDIR/asic/build-proc-gcd-xcel
% ./04-synopsys-vcs-ffglsim/run
% ./05-cadence-innovus-pnr/run
% ./06-synopsys-pt-sta/run
% ./07-synopsys-vcs-baglsim/run
% ./08-synopsys-pt-pwr/run
% ./09-mentor-calibre-drc/run
% ./10-mentor-calibre-lvs/run
% ./11-summarize-results/run
```

### 4.2. Processor with Null Accelerator

Now let's push the processor with the null accelerator (i.e., no
accelerator) through the full flow. Take a look at the provided design
YAML file.

```bash
% cd $TOPDIR/asic/designs
% cat lab8-proc-null-xcel.yml
```

Notice that the design name is `ProcMemXcel_v0_NullXcel` and the clock period is
also 8.0ns (this is slightly different than for the GcdXcel version, and this is
due to how sensitive the placement and routing algorithms are to clock
constraints, using 8.2ns actually causes the run to fail - try it out!). The
initialization commands are similar except they use the null accelerator
implementation instead of the GCD accelerator.

Let's use pyhflow to generate the flow scripts and run the flow.

```bash
% mkdir -p $TOPDIR/asic/build-proc-null-xcel
% cd $TOPDIR/asic/build-proc-null-xcel
% pyhflow ../designs/lab8-proc-null-xcel.yml
```

As before, we recommend explicitly running at least the first step manually to
ensure there are no errors.

```bash
% cd $TOPDIR/asic/build-proc-null-xcel
% ./01-pymtl-rtlsim/run
```

Once you have verified there are no errors, run the remaining steps.

```bash
% cd $TOPDIR/asic/build-proc-null-xcel
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-ffglsim/run
% ./05-cadence-innovus-pnr/run
% ./06-synopsys-pt-sta/run
% ./07-synopsys-vcs-baglsim/run
% ./08-synopsys-pt-pwr/run
% ./09-mentor-calibre-drc/run
% ./10-mentor-calibre-lvs/run
% ./11-summarize-results/run
```

### 4.3. Comparing Area Results

Now that both designs have been pushed through the full flow, let's
compare the area results. Start by looking at the detailed area reports
from Cadence Innovus.

```bash
% cd $TOPDIR/asic
% cat build-proc-gcd-xcel/05-cadence-innovus-pnr/area.rpt
% cat build-proc-null-xcel/05-cadence-innovus-pnr/area.rpt
```

Compare the total area and the hierarchical area breakdown between the
two designs. The processor with the GCD accelerator should have a larger
area due to the additional hardware for the GCD unit. Look at the
hierarchical breakdown to understand exactly how much area the GCD
accelerator adds compared to the null accelerator.

We can also compare the synthesis resources reports to see what
DesignWare modules each design uses.

```bash
% cd $TOPDIR/asic
% cat build-proc-gcd-xcel/03-synopsys-dc-synth/resources.rpt
% cat build-proc-null-xcel/03-synopsys-dc-synth/resources.rpt
```

### 4.4. Comparing Power Results

Let's compare the power reports from Synopsys PrimeTime.

```bash
% cd $TOPDIR/asic
% cat build-proc-gcd-xcel/08-synopsys-pt-pwr/ProcMemXcel_v0_GcdXcel_pmx-sim-gcd-rtl-ubmark-vvadd-eval.rpt
% cat build-proc-null-xcel/08-synopsys-pt-pwr/ProcMemXcel_v0_NullXcel_pmx-sim-null-rtl-ubmark-vvadd-eval.rpt
```

Compare the total power and the breakdown into internal, switching, and
leakage power between the two designs. The processor with the GCD
accelerator will likely have higher leakage power due to the additional
area, but the dynamic power (internal + switching) may differ depending
on the workload.

We can also compare the timing reports to see if adding the GCD
accelerator impacts the critical path.

```bash
% cd $TOPDIR/asic
% cat build-proc-gcd-xcel/06-synopsys-pt-sta/timing-setup.rpt
% cat build-proc-null-xcel/06-synopsys-pt-sta/timing-setup.rpt
```

Think about the tradeoffs involved. The GCD accelerator adds area and
leakage power, but for workloads that use GCD it could significantly
reduce the number of cycles required and thus reduce the total energy
consumed. This is a classic area/energy tradeoff in hardware design.

