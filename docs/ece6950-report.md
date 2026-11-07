
Final Report
==========================================================================

Each group must submit a final report which documents your chip testing
efforts throughout the semester. Please format your document with page
numbers, single-spacing, 10-point font, 1" margins. You must use Times
New Roman or Palatino font. Clearly mark each section with a numbered
section header. All diagrams, tables, and figures should be numbered with
a caption and be included in the main narrative of the text as opposed to
placed at the end of the document. You must include the sections below.
Please clearly number and label all sections. You might want to have a
bold "paragraph title" at the beginning of each paragraph (e.g.,
"Design:", "Pre-Silicon Verification", etc) as shown below. Make sure
your figures, diagrams, and photos are all well formatted. You need to
carefully size them so they are not so big that there are pages and pages
of figures, diagrams, and photos but also not so small that you cannot
see anything. Consider carefully grouping several figures, diagrams, and
photos onto a single page to help structure the report. All reports must
be submitted on Canvas in PDF format.

The final report should four sections:

 - Section 1: Introduction (one paragraph)
 - Section 2: Tiny Flow Tape-Out (five paragraphs)
 - Section 3: Accelerator Tape-Out (five paragraphs)
 - Section 4: Conclusion (one paragraph)

This handout tells you exactly what to focus on for each paragraph. You
can make it a little longer if you like, or have more paragraphs if you
like, but this handout explains the minimum expectations.

### Section 1: Introduction

 - The introduction should be 1 paragraph and should briefly summarize
   the two chips you taped out last spring: your tiny flow tape-out and
   your accelerator tape-out. Mention that your tiny flow tapeout is a
   100x100um block on a larger chip in TSMC 180nm, while your accelerator
   tapeout is your own 1x1mm chip in TSMC 180nm. Be sure to mention that
   your tiny flow tape-out is simple and uses your own RTL-to-GDS tiny
   flow, while your accelerator tape-out is more complex and uses an
   RTL-to-GDS commercial flow. You should also include a brief summary of
   the chip testing results (e.g., your block on the tiny flow tape-out
   was exhaustively tested and fully functional across all 256 inputs,
   accelerator tape-out was fully functional at the nominal voltage of
   1.8V and nominal frequency of XMHz, also mention a fast operating
   point and a slow operating point where the accelreator was fully
   functional).

### Section 2: Tiny Flow Tape-Out

 - **Design:** Include one paragraph that summarizes your tiny flow block
   design. You should revisit your ECE 6745 Project 1 report. Include a
   figure with the Verilog for your tiny flow block design. Just include
   the the contents of the module not the interface. The code must fit on
   one page. If the code would be too small to fit on one page, then just
   include the most important Verilog snippets. Here is an example of the
   [Verilog code](img/chip-test-tiny-flow-verilog.png) for the ripple
   carry adder tiny flow block. You can also include a block diagram if
   that would help (i.e., a block diagram of a four-bit ripple carry
   adder).

 - **Pre-Silicon Verification:** Include one paragraph about your
   pre-silicon verification strategy. Here you want to discuss how you
   used exhaustive testing to ensure your tiny flow block was fully
   operational. It will probably be a short paragraph.

 - **RTL-to-GDS Flow:** Include one paragraph that discusses the
   RTL-to-GDS tiny flow you implemented last spring which enabled taking
   your design from Verilog to layout. Include the figure from last
   spring which illustrate the overall [tiny
   flow](img/lab4-tinyflow.png). Include one sentence each on the
   algorithm you used for technology mapping, static timing analysis,
   placement, and routing.

 - **Tape-Out:** Include a figure of your final place-and-routed tiny
   flow block. Here is an example of what the tiny flow block looks like
   for the [ripple carry adder](img/chip-test-tiny-flow-block.png).
   Explain how 25 tiny flow blocks were integrated into a tape-out with
   additional power routing and muxing logic at the top-level. Mention
   the tiny flow chip's pin out which includes I/O VDD, core VDD, ground,
   5-bit block select input, 8-bit block data input, and 8-bit block data
   output. You should include the [tiny flow chip
   plot](img/chip-test-tiny-flow-chip.png) and a zoomed in [die photo of
   the tiny flow chip](img/chip-test-tinyflow-die-photo2.jpg).

 - **Post-Silicon Chip Testing:** Include one paragraph that discusses
   the tiny flow chip testing platform and results. You should include a
   photo of the [tiny flow chip testing
   board](img/chip-test-tinyflow-board1.jpeg). You should explain that
   the board enables manually testing inputs and also automatically
   testing inputs and verifying outputs using the USB-to-GPIO adapter.
   Then you should mention the chip testing results (your block on the
   tiny flow tape-out was exhaustively tested and fully functional across
   all 256 inputs).

### Section 3: Accelerator Tape-Out

 - **Design:** Include one paragraph that summarizes your accelerator
   design. You should revisit your ECE 6745 Project 2 report, and you
   probably want to include a figure or block diagram from that report.
   You should also include the [block diagram](img/lab9-gcd-xcel-chip.png)
   that shows how the accelerator is connected to the SPI minion and
   debug unit for the final chip design.

 - **Pre-Silicon Verification:** Include one paragraph about your
   pre-silicon verification strategy. You should revisit your ECE 6745
   Project 2 report. Be sure to discuss your directed and random testing.
   Focus just on testing your chip as opposed to testing the
   processor+accelerator composition. Discuss how you were able to reuse
   the tests for your accelerator in isolation to test the
   SPI+accelerator chip. This will be a much longer paragraph than for
   your tiny flow tape-out.

 - **RTL-to-GDS Flow:** Include one paragraph that discusses the
   commercial RTL-to-GDS flow you used last spring which enabled taking
   your design from Verilog to layout. Include the figure from last
   spring which illustrate the overall [commercial
   flow](img/T08-full-flow.png). Be sure to mention which commercial
   tools were used for two-state RTL simulation, four-state RTL
   simulation, synthesis, fast-functional GL simulation, place-and-route,
   static timing analysis, back-annotated GL simulation, power analysis,
   design rule checking, and layout vs. schematic.

 - **Tape-Out:** Include one paragraph that discusses the accelerator
   tape-out. Include a figure of your final place-and-routed GDS. Use the
   chip plots which are in the shared [Google drive
   folder](https://drive.google.com/drive/u/1/folders/141DUeYL12ZFTsCqUEtgXhruSLXaaV0aP).
   Include a figure of the [chip pin out](img/proj2-tapeout-pinout.png)
   and discuss the different pins of the chip (i.e., I/O VDD, core VDD,
   ground, clock in, clock out, reset, SPI, debug mode, debug out).
   Mention that the chip was packaged using wire bonding in a 68-pin
   ceramic pin-grid array (CPGA) package, and that we only used 32 of
   those pins. You should include the a zoomed out die photo of your chip
   (i.e., we should be able to see all of the bond wires) and a zoomed in
   die photo of your chip (i.e., we should be able to see the corner chip
   art). If you were able to get a good photo of metal 5 highlighted then
   include that photo along with the corresponding M5/M6 chip plot and
   discuss.

 - **Post-Silicon Chip Testing:** Include one paragraph that discusses the accelerator
   chip testing platform and results. Include the [chip testing system
   diagram](img/chip-test-lab5-system.png). You should include a photo of
   your accelerator chip testing board. You should annotate the following
   components on the chip testing board: accelerator chip; power LED;
   clock in and clock out; USB-to-SPI adapter; reset button; debug mode
   switch; and debug mode LED. You should explain that the board enables
   debug mode and explain what debug mode is and how it works. You should
   explain how you did the accelerator testing by using xmsgs. Then you
   should include a figure of your shmoo plot and summarize the region of
   operation where the chip is fully functional. Explain why the chip
   starts to fail at low voltage and/or high frequency.

### Section 4: Conclusion

 - The conclusion should be one paragraph. Summarize what you have
   accomplished over the past year. **You must also clearly discuss the
   most important lessons learned through the tape-out experience! Be
   specific!**

