
ECE 6745 Project 2: Accelerator Tape-Out<br>Accelerator
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
 - Part B: Accelerator
 - Part C: Evaluation
 - Part D: Tape-out

All parts must be done in a group of 2-3 students. You can confirm your
group on Canvas (Click on People, then Groups, then search for your name
to find your project group).

!!! warning "All students must contribute to and understand all submitted work!"

    It is not acceptable for one student to do all of part A and a
    different student to do all of part B. It is not acceptable for one
    student to only focus on one module of part B and not understand
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

1. Accelerator RTL Model & Testing
--------------------------------------------------------------------------

You should develop a hardware accelerator for the baseline software you
developed in the previous part. The accelerator should implement the
accelerator protocol you developed as part of your accelerator FL model.
Feel free to go back and update the baseline software and/or the
accelerator FL model as you continue to improve your project and push
towards tapeout.

You must use the design principles you learned in ECE 4750 including
modularity, hierarchy, encapsulation, regularity, and extensibility. Do
not implement your entire accelerator in a single module. Consider
separating your accelerator into a manager unit to handle the accelerator
request/reponse messages and a separate accelerator unit which has the
actual accelerator hardware. Consider decomposing your accelerator unit
into a datapath and control unit and consider further decomposing your
datapath into a structural composition of submodules.

Consider using the ASIC playgroud to manually push your accelerator
through the commercial front-end flow to: (1) ensure your design is fully
synthesizable; and (2) ensure your design will fit on the tapeout. Each
group's chip will be 950x950um but not all of this area is available for
your accelerator. The I/O pads used to connect your chip to the package
will be arranged in a ring around the outside of the chip. These I/O pads
are about 150um deep, and we will need another 50um of space for the power
rings. So students will only have approximately 550x550um of area for
their accelerator. Keep in mind the post-synthesis area estimates are
just for the standard cells. If we assume a standard-cell density of 70%
in the back-end flow, then you will need to divide this number by 0.7.
Compare the result to 550x550 = ~300Kum^2 to see if your design has a good
chance of fitting on the tapeout.

### 1.1. Developing Tests

It is hard to envision a compelling accelerator that will not require the
development of kind of submodules. You must unit test these submodules.
Write your own PyMTL test benches to ensure your submodules are fully
correct before moving on to test the composition of submodules. You can
probably test your accelerator unit without the accelerator manager.

Once all of your submodules and the accelerator unit have been thoroughly
tested, you can reuse the tests you developed for your accelerator FL
model to test your accelerator RTL model. Recall that these tests were
written in `sim/proj2/test/Proj2XcelFL_test.py`. You will probably want
to add more tests cases that are specifically designed to trigger corner
cases in your accelerator RTL model.

### 1.2. Running Tests

You can run your accelerator RTL tests as follows.

```bash
% mkdir -p $HOME/ece6745/project2-groupXX/sim/build
% cd $HOME/ece6745/project2-groupXX/sim/build
% pytest ../proj2/test/Proc2Xcel_test.py
```

You can run all of your acclerator unit and integration tests across both
the FL and RTO models as follows.

```bash
% mkdir -p $HOME/ece6745/project2-groupXX/sim/build
% cd $HOME/ece6745/project2-groupXX/sim/build
% pytest ../proj2
```

2. Accelerator Software & Testing
--------------------------------------------------------------------------

You already developed an the accelerator software required to drive your
accelerator in the previous part using the accelerator FL model. You may
need to update this accelerator software if you made any changes to your
accelerator protocol.

### 2.1. Developing Tests

You can reuse the tests you developed for your accelerator software in
the previous part. Recall that these tests were written in
`app/proj2/proj2-xcel-test.c`. You will probably want to add more tests
cases that are specifically designed to trigger corner cases in your
accelerator RTL model.

### 2.2. Running Tests

You can compile and run your accelerator software tests natively as
follows.

```bash
% mkdir -p $HOME/ece6745/project2-groupXX/app/build-native
% cd $HOME/ece6745/project2-groupXX/app/build-native
% ../configure
% make proj2-xcel-test
% ./proj2-xcel-test
```

You can cross-compile and run your accelerator software tests on the
TinyRV1 ISA simulator as follows.

```bash
% mkdir -p $HOME/ece6745/project2-groupXX/app/build
% cd $HOME/ece6745/project2-groupXX/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-xcel-test
% ../../sim/pmx/pmx-sim --xcel-impl proj2-fl ./proj2-xcel-test
```

You can cross-compile and run your accelerator software tests on the
actual processor, cache, and accelerator RTL as follows.

```bash
% mkdir -p $HOME/ece6745/project2-groupXX/app/build
% cd $HOME/ece6745/project2-groupXX/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-xcel-test
% ../../sim/pmx/pmx-sim --proc-impl rtl --cache-impl rtl \
    --xcel-impl proj2-rtl ./proj2-xcel-test
```

3. Project Submission
--------------------------------------------------------------------------

To submit your code you simply push your code to GitHub. You can push
your code as many times as you like before the deadline. Students are
responsible for going to the GitHub website for your repository, browsing
the source code, and confirming that the code they want to submit is on
GitHub. Be sure to verify your code is passing all of your simulations on
`ecelinux`. Your submission will be assessed for code quality and
functionalty. You should be continuing to improve your testing strategy.

Here is how we will be testing your final code submission for part B.
First, we will clone your repo and create an environment variable for the
top of your repo.

```bash
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/project2-groupXX
% cd project2-groupXX
% TOPDIR=$PWD
```

Then, we will run your tests for your baseline software both natively,
cross-compiled running on the ISA simulator, and cross-compiled running
on the processor+cache RTL model.

```
% mkdir -p $TOPDIR/app/build-native
% cd $TOPDIR/app/build-native
% ../configure
% make proj2-baseline-test
% ./proj2-baseline-test

% mkdir -p $TOPDIR/app/build
% cd $TOPDIR/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-baseline-test
% ../../sim/pmx/pmx-sim ./proj2-baseline-test

% mkdir -p $TOPDIR/app/build
% cd $TOPDIR/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-baseline-test
% ../../sim/pmx/pmx-sim --proc-impl rtl --cache-impl rtl ./proj2-baseline-test
```

Then, we will run your tests for your accelerator FL and RTL models.

```
% mkdir -p $TOPDIR/sim/build
% cd $TOPDIR/sim/build
% pytest ../proj2/test/Proj2XcelFL_test.py
% pytest ../proj2/test/Proj2Xcel_test.py
```

Then, we will run your tests for your accelerator software both natively,
cross-compiled running on the ISA simulator, and cross-compiled running
on the processor+cache+xcel RTL model.

```
% mkdir -p $TOPDIR/app/build-native
% cd $TOPDIR/app/build-native
% ../configure
% make proj2-xcel-test
% ./proj2-xcel-test

% mkdir -p $TOPDIR/app/build
% cd $TOPDIR/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-xcel-test
% ../../sim/pmx/pmx-sim --xcel-impl proj2-fl ./proj2-xcel-test

% mkdir -p $TOPDIR/app/build
% cd $TOPDIR/app/build
% ../configure --host=riscv64-unknown-elf
% make proj2-xcel-test
% ../../sim/pmx/pmx-sim --proc-impl rtl --cache-impl rtl \
    --xcel-impl proj2-rtl ./proj2-xcel-test
```

Note that if your tests take a while to run you will need to modify the
GitHub Actions workflow YAML file to increase the timeout and/or increase
the `--max-cycles` command line option to `pmx-sim`.

!!! warning "You do not need to finalize your accelerator!"

    Students must submit an initial version of their accelerator but they
    will almost certainly continue to improve their accelerator as they
    push towards tape-out. The key is to get a _very simple initial
    version_ of their accelerator ready for submission. It is ok if this
    _very simple initial version_ of the accelerator is not optimized, or
    if the _very simple initial version_ of the accelerator does not
    support all of the desired functionality. Start small; start simple;
    then you can continue to incrementally add complexity as you push
    towards tape-out.

