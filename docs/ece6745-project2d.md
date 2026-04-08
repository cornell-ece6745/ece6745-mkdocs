
ECE 6745 Project 2: Accelerator Tape-Out<br>ASIC Evaluation
==========================================================================

In this project, students will leverage what they learned in the first
project to transition to using a commercial standard-cell library and
commercial electronic design automation tools for simulation, synthesis,
place-and-route, static-timing analysis, power analysis, design rule
checking (DRC), and layout-vs-schematic checking (LVS). Students will
develop a simple accelerator in Verilog RTL and evaluate the potential
benefit of using this accelerator in the context of a RISC-V processor.
Students will then combine just their accelerator with an SPI interface
and use the commercial library and tools to turn this accelerator+SPI
into complete chip layout in TSMC 180nm. We have some project ideas here:

 - <https://cornell-ece6745.github.io/ece6745-mkdocs/ece6745-project2-ideas>

The project includes three parts:

 - Part A: Software & Testing
 - Part B: Accelerator RTL Milestone
 - Part C: Accelerator RTL Design
 - Part D: ASIC Evaluation
 - Part E: Tape-out and Report

All parts must be done in a group of 2-3 students. You can confirm your
group on Canvas (Click on People, then Groups, then search for your name
to find your project group).

!!! warning "All students must contribute to and understand all submitted work!"

    It is not acceptable for one student to do all of part A and a
    different student to do all of part B/C. It is not acceptable for one
    student to only focus on one module of part B/C and not understand
    anything about the other modules. All students must contribute and
    understand all aspects of all parts. The instructors will also survey
    the Git commit log on GitHub to confirm that all students are
    contributing equally. If you are using a "pair programming" style,
    then both students must take turns using their own account so both
    students have representative Git commits. Students should create
    commits after finishing each step of the project, so their
    contribution is clear in the Git commit log. It is fine for the Git
    commit log to indicate that a student took the lead on just one part,
    but the student is still responsible for reviewing and understanding
    all aspects included as part of the submission. **A student whose
    contribution is limited as represented by the Git commit log will
    receive a significant deduction to their project score.**

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
% git clone git@github.com:cornell-ece6745/project2-groupXX
% cd project2-groupXX
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
the following directories.

```
.
├── sim
│   ├── vc
│   ├── tut3_verilog
│   ├── lab5_xcel
│   ├── proc
│   ├── sram
│   ├── cache
│   ├── pmx
│   └── proj2
├── app
│   ├── scripts
│   ├── ece6745
│   ├── simple
│   ├── ubmark
│   └── proj2
└── asic
    ├── designs
    ├── steps
    │   └── block
    │       ├── 01-pymtl-rtlsim
    │       ├── 02-synopsys-vcs-rtlsim
    │       ├── 03-synopsys-dc-synth
    │       ├── 04-synopsys-vcs-fflgsim
    │       ├── 05-cadence-innovus-pnr
    │       ├── 06-synopsys-pt-sta
    │       ├── 07-synopsys-vcs-baglsim
    │       ├── 08-synopsys-pt-pwr
    │       ├── 09-mentor-calibre-drc
    │       ├── 10-mentor-calibre-lvs
    │       └── 11-summarize-results
    └── playground
        └── proj2
            ├── 01-pymtl-rtlsim
            ├── 02-synopsys-vcs-rtlsim
            ├── 03-synopsys-dc-synth
            ├── 04-synopsys-vcs-ffglsim
            ├── 05-cadence-innovus-pnr
            ├── 06-synopsys-pt-sta
            ├── 07-synopsys-vcs-baglsim
            ├── 08-synopsys-pt-pwr
            ├── 09-mentor-calibre-drc
            └── 10-mentor-calibre-lvs
```

1. ASIC Evaluation
--------------------------------------------------------------------------

You should use the automated ASIC block flow to push through three
designs:

 - Accelerator in isolation (`proj2.yml`)
 - Processor running baseline software (`proj2-px-null.yml`)
 - Processor with accelerator running accelerator software (`proj2-px-xcel.yml`)

We have provided you design YAML files for each of these designs. You can
use the automated ASIC block flow like this:

```bash
% mkdir -p $TOPDIR/asic/build-design
% cd $TOPDIR/app/build-design
% pyhflow ../designs/design.yml
% ./01-pymtl-rtlsim/run
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-fflgsim/run
% ./05-cadence-innovus-pnr/run
% ./06-synopsys-pt-sta/run
% ./07-synopsys-vcs-baglsim/run
% ./08-synopsys-pt-pwr/run
% ./09-mentor-calibre-drc/run
% ./10-mentor-calibre-lvs/run
% ./11-summarize-results/run
```

where `design` is either `proj2`, `proj2-px-null`, or `proj2-px-xcel`.
Make sure each step works without errors before moving on to the next
step. Note that you will need to update the `proj2.yml` file to run your
tests and the `proj2-xcel-sim` appropriately. You must have
`proj2-xcel-sim` working since this enables you to evaluate your
accelerator in isolation. You may also need to update `proj2-px-null.yml`
and `proj2-px-xcel.yml` if you have included additional C test and
evaluation programs.

As mentioned in previous parts you need to ensure your accelerator in
isolation is approximately less than 300kum^2. You need to make sure all
three designs meet timing with a 10ns clock cycle constraint and pass all
checks.

2. Evaluation Document
--------------------------------------------------------------------------

Prepare a short document with your preliminary project 2 evaluation and
submit it on Canvas. Your document should be similar to the following PDF
example which is based on the GCD accelerator.

 - [ASIC Evaluation Document for GCD Accelerator](img/ece6745-proj2d.pdf)

Include your project title, group number, and group members at the top of
the first page.

**Section 1: Evaluation**

Use 10-point times or palantino font, single-spaced, with 1" margins.
Include 1-2 pages of text discussing your current evaluation results.

Start with a discussion of your accelerator block-level results. Do not
just mention the total area, cycle time, and power. Be sure to dive into
the results to discuss the detailed area break-down, where the critical
path goes, and the detailed power break-down.

Then do a comparative analysis of your processor baseline design vs
processor+accelerator alternative design. Be sure to discuss area and
cycle time before discussing the total execution time in ns and the
energy for your evaluation program. Do not just discuss the results but
dive into what these results mean and why each design achieves those
results. Why does one design have more area? Why does one design have
better performance?

**Section 2: Accelerator Block-Level Results**

- Format this section on one page exactly as shown in the PDF example.

- Include screenshot of chip plot from Innovus. You must hide the
  routing. You must highlight different parts of your accelerator in
  different colors as discussed in lab.

- Include screenshot of final layout using Klayout. This should show all
  layers. You use the standard layer property file.

- Include the summarize results. You need some kind of actual evaluation
  so you will need to modify proj2-xcel-sim to run an evaluation of your
  accelerator in isolation.

**Section 3: Processor Baseline Design Results**

- Format this section on one page exactly as shown in the PDF example.

- Include screenshot of chip plot from Innovus. You must hide the
  routing. You must highlight the register file in red and the rest of
  the processor in yellow.

- Include screenshot of final layout using Klayout. This should show all
  layers. You use the standard layer property file.

- Include the summarize results. You must include the results for your
  evaluation microbenchmark.

**Section 4: Processor+Accelerator Alternative Design Results**

- Format this section on one page exactly as shown in the PDF example.

- Include screenshot of chip plot from Innovus. You must hide the
  routing. You must highlight the register file in red and the rest of
  the processor in yellow. You must highlight the accelerator in other
  colors ... at least highlight the entire processor in cyan but you can
  use more colors if you want to show different parts. Obviously do not
  reuse yellow or red.

- Include screenshot of final layout using Klayout. This should show all
  layers. You use the standard layer property file.

- Include the summarize results. You must include the results for your
  evaluation microbenchmark.

3. Project Code Submission
--------------------------------------------------------------------------

To submit your code you simply push your code to GitHub. You can push
your code as many times as you like before the deadline. Students are
responsible for going to the GitHub website for your repository, browsing
the source code, and confirming that the code they want to submit is on
GitHub. Be sure to verify your code is passing all of your simulations on
`ecelinux`. Your submission will be assessed for code quality and
functionalty. You should be continuing to improve your testing strategy.

**Make sure the main branch of the code on GitHub can be used to
reproduce the results in your evaluation document as follows.**

Here is how we will be testing your final code submission for part D.
First, we will clone your repo and create an environment variable for the
top of your repo.

```bash
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/project2-groupXX
% cd project2-groupXX
% TOPDIR=$PWD
```

Then, we will push your accelerator through the flow in isolation.

```bash
% mkdir -p $TOPDIR/asic/build-proj2-px-null
% cd $TOPDIR/app/build-proj2-px-null
% pyhflow ../designs/proj2-px-null
% ./run-flow
```

Finally, we will push your processor running the baseline software
through the flow.

```bash
% mkdir -p $TOPDIR/asic/build-proj2-px-null
% cd $TOPDIR/app/build-proj2-px-null
% pyhflow ../designs/proj2-px-null
% ./run-flow
```

Finally, we will push your processor with accelerator through the flow
running the accelerator software.

```bash
% mkdir -p $TOPDIR/asic/build-proj2-px-xcel
% cd $TOPDIR/app/build-proj2-px-xcel
% pyhflow ../designs/proj2-px-xcel
% ./run-flow
```

!!! warning "You do not need to finalize your accelerator!"

    Students must submit a preliminary evaluation of their accelerator
    but they will almost certainly continue to improve their accelerator
    as they push towards tape-out. The key is to get a preliminary
    version version of their accelerator ready and through the rigorous
    comparative evaluation process. Studetns can continue to
    incrementally add complexity as they push towards tape-out.

