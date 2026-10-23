
Lab 6: TinyFlow Testing (Yaqi version)
==========================================================================

In this lab, you will test your TinyFlow design using both manual inputs
and automated test vectors (Not for today). We will start by creating a spreadsheet of
inputs and expected outputs before:

 - testing a few input combinations using the switches and LEDs

1. Create Test Vectors
--------------------------------------------------------------------------

Let's start by creating an Excel spreadsheet with all of the input
combinations you want to test and their expected outputs. Each row should
represent one test vector. Include columns for the select value, the
eight input bits, and the expected output bits.

Make a copy of this [test-vector spreadsheet](https://docs.google.com/spreadsheets/d/1JQzhulDAJZ4bDyIybEy0l-xraZfogiFvMUTpYtc1EVM/edit?usp=sharing)
to get started.

Determine the expected outputs from your design before testing the
hardware. Include all input and select combinations needed to verify
your design.

Login to ecelinux? or get Ivan to do it if you no longer an access ecelinux and somehow generate all your test vectors to fill the spreadsheet.

2. Manual Testing
--------------------------------------------------------------------------

Before running all of your vectors, let's check a few combinations by
hand. Pick a few rows from your spreadsheet and use the switches to apply
each test vector:

 - Power on the chip testing board making sure its plugged into the workstation
 - Power on the TinyFlow Chip using the On/Off button on the tester board, the red LED should light up
 - Use the bottom switches to set the TinyFlow block to your design
 - Use the top switches to set the eight input bits
 - Read the output bits from the top LED bar
 - Compare the observed outputs with the expected outputs in your
   spreadsheet

Try a few different input and select values. If an output does not match,
check your switch settings and expected result before continuing.

3. Automated Testing over SPI
--------------------------------------------------------------------------

Later Ivan will get this working so ignore this.

You have now successfully verified your TinyFlow tape-out!