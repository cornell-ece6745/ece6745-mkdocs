
ECE 6745 Lab 9: Commercial Chip Flow
==========================================================================

In this lab, we will be learning about chip-level RTL design and the
commercial chip ASIC flow. In the previous lab, we pushed designs
through the _block-level_ ASIC flow which synthesizes, places, and
routes a design but does not include the I/O pads, seal ring, or fill
structures needed for tapeout. In this lab, we extend that block flow
into a full _chip-level_ flow.

 - **Chip-Level RTL Design:** A real chip needs I/O cells around the
    perimeter for signal buffering, ESD protection, and level shifting
    between core and I/O voltage domains. We will study how the GCD
    accelerator is wrapped with I/O pads, a two-flop reset
    synchronizer, and an SPI minion to create a chip-level design. This
    is the same pattern you will use to wrap your own accelerator for
    Project 2.

 - **Padring, Seal Ring, and Fill:** Tapeout requires physical
    structures beyond the core logic. The _padring_ places I/O cells
    and bond pads around the chip perimeter. The _seal ring_ is a guard
    structure along the die edge that prevents cracks from wafer dicing
    from propagating into the active circuit area. _Metal, poly, and
    via fill_ insert dummy patterns to maintain uniform density for CMP
    (chemical-mechanical polishing) planarity.

 - **Chip-Level Verification:** The chip flow extends DRC to include
    wirebond and metal-fuse checks, and LVS must now account for I/O
    cell SPICE models. The chip flow has 14 steps compared to the block
    flow's 11 steps.

Extensive documentation is provided by Synopsys, Cadence, and Siemens for
these ASIC tools. We have organized this documentation and made it
available to you on the public course webpage:

 - <https://www.csl.cornell.edu/courses/ece6745/asicdocs>

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

!!! warning "Students MUST work in pairs!"

    You MUST work in pairs for this lab, as having too many instances of
    Innovus open at once can cause the `ecelinux` servers to crash. So
    find a partner and work together at a workstation to complete today's
    lab.

Follow the same process as previous labs. Find a free workstation and log
into the workstation using your NetID and standard NetID password. Then
complete the following steps.

 - Start VS Code
 - Install the Remote-SSH extension, Surfer, Python, and Verilog extensions
 - Use View > Command Palette to execute Remote-SSH: Connect Current Window to Host...
 - Enter netid@ecelinux-XX.ece.cornell.edu where XX is an ecelinux server number
 - Use View > Explorer to open your home directory on ecelinux
 - Use View > Terminal to open a terminal on ecelinux
 - Start MS Remote Desktop

Now use the following commands to clone the repo we will be using for
today's lab.

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/ece6745-lab9 lab9
% cd lab9
% tree
```

To make it easier to cut-and-paste commands from this handout onto the
command line, you can tell Bash to ignore the `%` character using the
following command:

```bash
% alias %=""
```

2. Chip-Level RTL Design
--------------------------------------------------------------------------

Before looking at the ASIC flow, we need to understand how a design is
structured for chip-level integration. In the block flow, we synthesize
and place-and-route the accelerator alone. In the chip flow, we wrap the
accelerator with I/O pads, a reset synchronizer, and an SPI minion to
create a complete chip-level design. This is the same pattern you will
replicate for your own Project 2 accelerator.

### 2.1. Chip Top Module

Take a look at the chip top module for the GCD accelerator.

```bash
% cd ${HOME}/ece6745/lab9/sim/lab5_xcel
% code GcdXcelChip.v
```

Here is the module interface:

```verilog
module lab5_xcel_GcdXcelChip
(
  input  logic clk,
  input  logic reset,

  // SPI interface (off-chip side)

  input  logic cs,
  input  logic sclk,
  input  logic mosi,
  output logic miso,

  // Parity outputs (off-chip side)

  output logic minion_parity,
  output logic adapter_parity,
  output logic clk_out
);
```

The module has five inputs (`clk`, `reset`, `cs`, `sclk`, `mosi`) and
four outputs (`miso`, `minion_parity`, `adapter_parity`, `clk_out`).
The `cs`, `sclk`, and `mosi` signals are the SPI bus from an off-chip
master. The `miso` signal carries data back to the master. The
`minion_parity` and `adapter_parity` outputs are for error detection.
The `clk_out` output drives the on-chip clock back off-chip for
monitoring.

The first thing the module does is declare on-chip signals and
instantiate I/O pads:

```verilog
  InputIO #(.is_clk(1)) clk_io   ( .off_chip( clk   ), .on_chip( on_chip_clk   ) );
  InputIO               reset_io ( .off_chip( reset ), .on_chip( on_chip_reset ) );
  InputIO               cs_io    ( .off_chip( cs    ), .on_chip( on_chip_cs    ) );
  InputIO               sclk_io  ( .off_chip( sclk  ), .on_chip( on_chip_sclk  ) );
  InputIO               mosi_io  ( .off_chip( mosi  ), .on_chip( on_chip_mosi  ) );

  OutputIO               miso_io            ( .on_chip( on_chip_miso            ), .off_chip( miso            ) );
  OutputIO               minion_parity_io   ( .on_chip( on_chip_minion_parity   ), .off_chip( minion_parity   ) );
  OutputIO               adapter_parity_io  ( .on_chip( on_chip_adapter_parity  ), .off_chip( adapter_parity  ) );
  OutputIO #(.is_clk(1)) clk_out_io         ( .on_chip( on_chip_clk_out         ), .off_chip( clk_out         ) );
```

Notice the `is_clk` parameter on the clock pads. This tells the pad
model to use the special clock I/O cell (`PDDW0812SCDG`) instead of the
standard I/O cell (`PDDW0812CDG`). The clock I/O cell uses a Schmitt trigger to switch states at two different threshold voltages instead of one, to allow for greater noise reduction which is important for high-frequency signals such as the clock.

### 2.2. I/O Pad Behavioral Models

Take a look at the I/O pad behavioral models.

```bash
% cd ${HOME}/ece6745/lab9/sim/io_cells
% code InputIO.v
% code OutputIO.v
```

Here is the `InputIO` module:

```verilog
module InputIO #(
  parameter is_clk = 0
) (
  input  wire off_chip,
  output wire on_chip
);

  `ifndef SYNTHESIS
    assign on_chip = off_chip;
  `else
    generate
      if( is_clk ) begin : CLK_IO
        PDDW0812SCDG io (
          .I   ( 1'b0     ),
          .OEN ( 1'b1     ),
          .PAD ( off_chip ),
          .C   ( on_chip  ),
          .IE  ( 1'b1     ),
          .PE  ( 1'b1     ),
          .DS  ( 1'b0     )
        );
      end else begin : NOT_CLK_IO
        PDDW0812CDG io (
          .I   ( 1'b0     ),
          .OEN ( 1'b1     ),
          .PAD ( off_chip ),
          .C   ( on_chip  ),
          .IE  ( 1'b1     ),
          .PE  ( 1'b1     ),
          .DS  ( 1'b0     )
        );
      end
    endgenerate
  `endif

endmodule
```

The key pattern here is `ifndef SYNTHESIS`. During simulation, the pad
is a simple wire passthrough (`assign on_chip = off_chip`). During
synthesis, it instantiates the real TSMC I/O cell. The I/O cell pins
are:

 - `PAD`: The external bond pad connection
 - `C`: The core-side connection
 - `I`: Output data (driven by on-chip logic)
 - `OEN`: Output enable (active low); set to `1'b1` for input mode
 - `IE`: Input enable; set to `1'b1` to enable the input buffer
 - `PE`: Pull enable; set to `1'b1` to enable a pull-up/pull-down
 - `DS`: Drive strength select

The `OutputIO` module is similar but configures the cell in output mode:
`OEN` is `1'b0` (output enabled), `I` drives the on-chip signal out,
and `IE` is `1'b0` (input buffer disabled).

### 2.3. Reset Synchronizer

The chip top module includes a two-flop reset synchronizer:

```verilog
  logic reset_sync_pre;
  logic reset_sync;

  always_ff @(posedge on_chip_clk) begin
    reset_sync_pre <= on_chip_reset;
    reset_sync     <= reset_sync_pre;
  end
```

The off-chip reset signal arrives asynchronously relative to the on-chip
clock. Sampling an asynchronous signal with a single flip-flop can cause
_metastability_ -- the flip-flop output can oscillate or settle to an
unpredictable value. The two-flop synchronizer reduces the probability
of metastability to a negligible level by giving the first flop's output
a full clock cycle to resolve before the second flop samples it. All
downstream logic uses `reset_sync` instead of the raw `on_chip_reset`.

### 2.4. SPI Minion

The chip communicates with the outside world via SPI (Serial Peripheral
Interface). Take a look at the SPI minion module:

```bash
% cd ${HOME}/ece6745/lab9/sim/spi
% code minion.v
```

The SPI minion has two sub-components:

 - `Minion_PushPull`: Handles the SPI protocol -- synchronizes the
   asynchronous `cs`, `sclk`, and `mosi` signals to the chip clock,
   uses shift registers for serial-to-parallel (MOSI) and
   parallel-to-serial (MISO) conversion, and generates push/pull
   enables on SPI transaction boundaries.

 - `Minion_Adapter`: Converts the push/pull interface into standard
   val/rdy streams (`send_msg` for requests arriving from the master,
   `recv_msg` for responses going back to the master).

The SPI minion is parameterized with `BIT_WIDTH=38` so that each 40-bit
SPI word carries a complete xcel transaction. The 40-bit SPI word format
is:

```
  MOSI (master to chip, 40 bits):
    {val_wrt(1), val_rd(1), type_(1), addr(5), data(32)}

  MISO (chip to master, 40 bits):
    {val(1), spc(1), type_(1), 0(5), data(32)}
```

The two extra bits (`val_wrt`/`val_rd` on MOSI, `val`/`spc` on MISO`)
are the SPI flow-control bits. The 38-bit payload maps directly to the
xcel request/response messages.

Back in `GcdXcelChip.v`, the wiring between the SPI minion and the GCD
accelerator is straightforward:

```verilog
  // SPI send -> xcel req (direct mapping)
  assign xcel_req     = spi_send_msg;
  assign xcel_req_val = spi_send_val;
  assign spi_send_rdy = xcel_req_rdy;

  // Xcel resp -> SPI recv (pad to 38 bits)
  assign spi_recv_msg = { 5'b0, xcel_resp };
  assign spi_recv_val = xcel_resp_val;
  assign xcel_resp_rdy = spi_recv_rdy;
```

The SPI `send_msg` (38 bits) maps directly to `xcel_req_t` which is
`{type_(1), addr(5), data(32)}`. The `xcel_resp_t` (33 bits) is
zero-padded to 38 bits for the SPI `recv_msg`.

### 2.5. PyMTL Wrapper

Take a look at the PyMTL wrapper for the chip module:

```bash
% cd ${HOME}/ece6745/lab9/sim/lab5_xcel
% code GcdXcelChip.py
```

```python
from pymtl3 import *
from pymtl3.passes.backends.verilog import *

class GcdXcelChip( VerilogPlaceholder, Component ):
  def construct( s ):
    s.cs              = InPort (1)
    s.sclk            = InPort (1)
    s.mosi            = InPort (1)
    s.miso            = OutPort(1)
    s.minion_parity   = OutPort(1)
    s.adapter_parity  = OutPort(1)
    s.clk_out         = OutPort(1)
```

This is a thin `VerilogPlaceholder` wrapper that tells PyMTL to import
the Verilog module for simulation. The port list must match the Verilog
module interface exactly (excluding `clk` and `reset` which PyMTL
handles automatically). You will create a similar `.py` wrapper for your
own chip module.

### 2.6. Test Harness and SPI Master

Take a look at the chip-level test harness:

```bash
% cd ${HOME}/ece6745/lab9/sim/lab5_xcel/test
% code GcdXcelChip_test.py
```

The test harness structure is:

```
  StreamSourceFL    SpiMasterFL      GcdXcelChip      SpiMasterFL    StreamSinkFL
  (XcelReqMsg) --> (bit-bang SPI) --> (SPI wires) --> (SPI wires) --> (XcelRespMsg)
```

```python
class TestHarness( Component ):

  def construct( s, ChipType ):

    s.src    = StreamSourceFL( XcelReqMsg )
    s.sink   = StreamSinkFL( XcelRespMsg )
    s.master = SpiMasterFL()
    s.chip   = ChipType

    # src -> SpiMasterFL (xcel req)
    s.src.ostream //= s.master.recv

    # SpiMasterFL -> sink (xcel resp)
    s.sink.istream //= s.master.send

    # SpiMasterFL <-> chip (SPI wires)
    s.master.cs   //= s.chip.cs
    s.master.sclk //= s.chip.sclk
    s.master.mosi //= s.chip.mosi
    s.master.miso //= s.chip.miso
```

The `SpiMasterFL` (in `sim/spi/SpiMasterFL.py`) is a functional-level
model that bit-bangs the SPI protocol: it takes xcel request messages,
serializes them into SPI transactions, and deserializes the SPI
responses back into xcel response messages.

To understand what the test sends over SPI, look at the `xgcd` helper
function defined in `GcdXcelFL_test.py`:

```python
def xgcd( a, b, result ):
  return [
    xreq( 'wr', 0, a ), xresp( 'wr', 0 ),
    xreq( 'wr', 1, b ), xresp( 'wr', 0 ),
    xreq( 'rd', 2, 0 ), xresp( 'rd', result ),
  ]
```

Each GCD computation is a sequence of three xcel transactions:

 1. Write `a` to xcel register 0
 2. Write `b` to xcel register 1
 3. Read the GCD result from xcel register 2

The `xreq`/`xresp` helpers create the appropriate `XcelReqMsg` and
`XcelRespMsg` objects. The messages are interleaved as
request/response pairs: `msgs[::2]` extracts all requests (sent by the
source) and `msgs[1::2]` extracts all expected responses (checked by the
sink).

The test case table in `GcdXcelFL_test.py` defines 15 test cases:

```python
test_case_table = mk_test_case_table([
  (                  "msgs                 src_delay sink_delay"),
  [ "basic",          basic_msgs,          0,        0,         ],
  [ "zeros",          zeros_msgs,          0,        0,         ],
  [ "equal",          equal_msgs,          0,        0,         ],
  [ "divides",        divides_msgs,        0,        0,         ],
  [ "common_factors", common_factors_msgs, 0,        0,         ],
  [ "coprime",        coprime_msgs,        0,        0,         ],
  [ "powers_of_two",  powers_of_two_msgs,  0,        0,         ],
  [ "larger_numbers", larger_numbers_msgs, 0,        0,         ],
  [ "swap_ordering",  swap_ordering_msgs,  0,        0,         ],
  [ "fibonacci",      fibonacci_msgs,      0,        0,         ],
  [ "sub_stress",     sub_stress_msgs,     0,        0,         ],
  [ "random_0x0",     random_msgs,         0,        0,         ],
  [ "random_5x0",     random_msgs,         5,        0,         ],
  [ "random_0x5",     random_msgs,         0,        5,         ],
  [ "random_3x9",     random_msgs,         3,        9,         ],
])
```

Each test case specifies the message list plus source and sink delays
(to test backpressure). The `random_*` variants use the same random
messages but with different delay patterns.

The key insight is that `GcdXcelChip_test.py` _imports_ this same
`test_case_table`:

```python
from lab5_xcel.test.GcdXcelFL_test import test_case_table
```

This means the exact same logical tests work at both the functional
level (direct xcel interface) and the chip level (over SPI). The only
difference is the test harness: the chip test interposes the
`SpiMasterFL` between the source/sink and the design under test. This
is an important design pattern -- you should reuse your existing FL
test cases when creating your own chip-level tests.

The chip test also supports the `--dump-xmsgs` flag, which generates
transaction log files recording each xcel message sent/received over
SPI. These transaction logs are not used in the ASIC flow itself, but
they will be essential for future chip testing -- when we have
fabricated silicon, we can replay these transaction logs through a
real SPI master to verify that the physical chip produces the correct
responses.

### 2.7. Simulator

Take a look at the simulator script:

```bash
% cd ${HOME}/ece6745/lab9/sim/lab5_xcel
% code gcd-xcel-sim
```

The simulator supports three implementations:

 - `--impl fl`: Functional-level GCD accelerator (no RTL)
 - `--impl rtl`: RTL GCD accelerator with direct xcel interface
 - `--impl rtl-chip`: RTL GCD accelerator wrapped in chip-level I/O

For the chip flow, we use `--impl rtl-chip` which instantiates
`GcdXcelChip` and uses the `ChipTestHarness` with the `SpiMasterFL`.
The key flags for the ASIC flow are:

```bash
% cd ${HOME}/ece6745/lab9/sim/lab5_xcel
% ./gcd-xcel-sim --impl rtl-chip --input random --translate --dump-vtb --dump-xmsgs
```

 - `--translate`: Converts the PyMTL RTL to pickled Verilog for synthesis
 - `--dump-vtb`: Generates Verilog test benches for VCS simulation
 - `--dump-xmsgs`: Generates SPI transaction logs for future chip testing

3. Chip Flow Overview
--------------------------------------------------------------------------

Now that we understand the chip-level RTL design, let's look at the
chip ASIC flow that will take this design all the way to layout.

### 3.1. Step Templates

The chip flow step templates are located in the `asic/steps/chip`
directory:

```bash
% cd ${HOME}/ece6745/lab9/asic/steps/chip
% tree
.
├── 00-padring-gen
├── 01-pymtl-rtlsim
├── 02-synopsys-vcs-rtlsim
├── 03-synopsys-dc-synth
├── 04-synopsys-vcs-ffglsim
├── 05-cadence-innovus-pnr
├── 06-synopsys-pt-sta
├── 07-synopsys-vcs-baglsim
├── 08-synopsys-pt-pwr
├── 09-mentor-calibre-seal
├── 10-mentor-calibre-fill
├── 11-mentor-calibre-drc
├── 12-mentor-calibre-lvs
└── 13-summarize-results
```

Compared to the block flow from the previous lab (11 steps), the chip
flow has 14 steps. The three new steps are:

 - `00-padring-gen`: Generates the I/O padring placement files
 - `09-mentor-calibre-seal`: Generates and merges the seal ring
 - `10-mentor-calibre-fill`: Generates and merges metal/poly/via fill

Several existing steps are also modified to handle I/O cells:

 - `03-synopsys-dc-synth`: Adds `iocells.db` to the target library
 - `05-cadence-innovus-pnr`: Uses a fixed padring-aware floorplan with
   I/O cell placement and separate VDD/VSS power rings
 - `11-mentor-calibre-drc`: Adds wirebond and metal-fuse DRC checks
 - `12-mentor-calibre-lvs`: Uses `iocells.sp` and `lvs-devices.sp` for
   I/O cell SPICE models

### 3.2. Design YAML

Take a look at the design YAML file for the chip-level GCD accelerator:

```bash
% cd ${HOME}/ece6745/lab9/asic/designs
% code lab9-gcd-xcel-chip.yml
```

```
steps:
 - chip/00-padring-gen
 - chip/01-pymtl-rtlsim
 - chip/02-synopsys-vcs-rtlsim
 - chip/03-synopsys-dc-synth
 - chip/04-synopsys-vcs-ffglsim
 - chip/05-cadence-innovus-pnr
 - chip/06-synopsys-pt-sta
 - chip/07-synopsys-vcs-baglsim
 - chip/08-synopsys-pt-pwr
 - chip/09-mentor-calibre-seal
 - chip/10-mentor-calibre-fill
 - chip/11-mentor-calibre-drc
 - chip/12-mentor-calibre-lvs
 - chip/13-summarize-results

design_name  : GcdXcelChip_noparam
clock_period : 4.5
dump_vcd     : true

pymtl_rtlsim: |
  pytest ../../../sim/lab5_xcel/test/GcdXcelChip_test.py \
    --test-verilog --dump-vtb --dump-xmsgs | tee -a run.log
  ...

tests:
  - GcdXcelChip_noparam_test_basic
  - GcdXcelChip_noparam_test_zeros
  ...

evals:
 - GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-zeros
 - GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-small
 - GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-random
```

The design YAML specifies all 14 chip flow steps (note the `chip/`
prefix). The design name is `GcdXcelChip_noparam` and the clock period
is 4.5ns. Note the clock period is slower than the 3.0ns used in the
block flow because I/O cell delays add overhead.

The `pymtl_rtlsim` section runs both the pytest test suite and the
simulator with three input patterns (random, small, zeros). The `tests`
list and `evals` list name all 15 tests and 3 evaluations that will be
run through subsequent simulation steps.

### 3.3. Padring Configuration

Take a look at the padring configuration file:

```bash
% cd ${HOME}/ece6745/lab9/asic/steps/chip/00-padring-gen
% code padring-config.yaml
```

```
chip:
  pitch: 60
  pad_height: 102.4
  corner_size: 130
  sealring_width: 15
  width: 950
  height: 950

cells:
  io: PDDW0812CDG
  clk: PDDW0812SCDG
  poc: PVDD2POC
  io_vdd: PVDD2CDG
  core_vdd: PVDD1CDG
  common_vss: PVSS3CDG
  corner: PCORNER
  pad: PAD60LU_SL_SM
```

The chip dimensions are 950um x 950um. The `pad_height` (102.4um) is the
height of the I/O cells, the `corner_size` (130um) is the size of the
corner cells, and `sealring_width` (15um) is the width of the seal ring.
The `pitch` (60um) is the spacing between bond pads.

The pad assignments for each side are:

```
# Left side pads (bottom to top) -- SPI bus grouped together
left:
  - CGVSS
  - IOVDD
  - cs
  - sclk
  - mosi
  - miso
  - CRVDD
  - CGVSS

# Right side pads (bottom to top) -- remaining signals
right:
  - CGVSS
  - IOVDD
  - clk
  - reset
  - clk_out
  - minion_parity
  - adapter_parity
  - UNBONDED
```

Each side has a mix of signal pads and power/ground pads. The pad types
are:

 - `CGVSS`: Common ground pad
 - `IOVDD`: I/O voltage power pad
 - `CRVDD`: Core voltage power pad
 - `POC`: Power-on-control pad (exactly one required)
 - `UNBONDED`: Unbonded I/O VDD pad (used for pad count balance)
 - Signal names (e.g., `cs`, `sclk`): Signal I/O pads

The top and bottom sides contain only power/ground pads. The SPI bus
signals are grouped on the left side for clean routing, while the clock,
reset, and parity signals are on the right side. This setup will likely not
be the final padring we use for the tapeouts, but it provides a reasonable
organization for the purposes of this lab and early trials for your own
accelerators.

4. Generating the Flow and Padring
--------------------------------------------------------------------------

### 4.1. Running pyhflow

Let's generate the chip flow scripts:

```bash
% mkdir -p ${HOME}/ece6745/lab9/asic/build-gcd
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% pyhflow ../designs/lab9-gcd-xcel-chip.yml
```

Just like the block flow, pyhflow copies the step template directories
into the build directory and applies Jinja2 template substitution. For
example, let's see how the synthesis script was filled in:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 03-synopsys-dc-synth/run.tcl
```

Notice how `{{ design_name }}` has been replaced with
`GcdXcelChip_noparam` and `{{ clock_period }}` has been replaced with
`4.5`:

```
analyze -format sverilog ../01-pymtl-rtlsim/GcdXcelChip_noparam__pickled.v
elaborate GcdXcelChip_noparam
...
create_clock [get_ports clk] -name ideal_clock1 -period 4.5
```

### 4.2. Padring Generation (Step 00)

Let's run the padring generation step:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./00-padring-gen/run
```

The `run-padring-gen.py` script reads the `padring-config.yaml` and
generates two output files:

 - `padring_gen.io`: An I/O assignment file that tells Cadence Innovus
   where to place the I/O cells (used during `init_design`)
 - `padring_gen.tcl`: A Tcl script that places bond pads at computed
   positions and adds route blockages in the I/O cell regions

These files are consumed later during the place-and-route step.

5. RTL Simulation
--------------------------------------------------------------------------

### 5.1. PyMTL 2-State RTL Simulation (Step 01)

Run the PyMTL RTL simulation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./01-pymtl-rtlsim/run
```

This step runs the chip-level test suite with `--test-verilog --dump-vtb
--dump-xmsgs` to:

 1. Verify the design passes all 15 functional tests
 2. Generate pickled Verilog (`GcdXcelChip_noparam__pickled.v`) for synthesis
 3. Generate Verilog test benches (`.v`) for VCS simulation
 4. Generate transaction log files (`.xmsgs`) for debugging

Check the output to make sure all tests pass. The step also runs the
three evaluation simulations (random, small, zeros).

### 5.2. VCS 4-State RTL Simulation (Step 02)

Run the VCS 4-state RTL simulation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./02-synopsys-vcs-rtlsim/run
```

The 4-state simulation catches X-propagation issues that the 2-state
PyMTL simulation cannot detect. Check the output to make sure all 18
simulations (15 tests + 3 evals) pass with no X warnings.

6. Synthesis
--------------------------------------------------------------------------

### 6.1. Synopsys DC Synthesis (Step 03)

Run synthesis:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./03-synopsys-dc-synth/run
```

Take a look at the synthesis script:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 03-synopsys-dc-synth/run.tcl
```

The key difference from the block flow is that the `target_library` now
includes `iocells.db` in addition to `stdcells.db`:

```tcl
set_app_var target_library [list \
  "$env(TSMC_180NM)/stdcells.db" \
  "$env(TSMC_180NM)/iocells.db" \
]
```

Another difference is the dual operating conditions. The I/O cells and
core cells operate in different voltage domains and have different
timing characteristics, so we apply separate operating conditions:

```tcl
set_operating_conditions \
  -library tph018nv3_sltc \
  -object_list [get_cells "v/*io/*io"] \
  NCCOM
set_operating_conditions \
  -library tcb018gbwp7ttc \
  NCCOM
```

The first command sets the operating conditions for I/O cells (matched
by the pattern `v/*io/*io`), and the second sets the default operating
conditions for all core cells.

Carefully look at the output from synthesis. Look for the output after
`Running PRESTO HDLC` for any warnings to ensure that all of your
Verilog RTL is indeed synthesizable. Then check the timing and area
reports:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% cat 03-synopsys-dc-synth/timing.rpt
% cat 03-synopsys-dc-synth/area.rpt
% cat 03-synopsys-dc-synth/resources.rpt
```

Verify that the timing report shows positive setup slack.

### 6.2. Fast-Functional Gate-Level Simulation (Step 04)

Run the fast-functional gate-level simulation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./04-synopsys-vcs-ffglsim/run
```

This step verifies that the synthesized gate-level netlist is
functionally correct using zero-delay simulation. Check that all 18
tests pass.

7. Place and Route
--------------------------------------------------------------------------

### 7.1. Cadence Innovus PNR (Step 05)

Run place and route:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./05-cadence-innovus-pnr/run
```

Take a look at the PNR script:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 05-cadence-innovus-pnr/run.tcl
```

There are several key differences from the block flow.

**Initialization.** The LEF files now include I/O cells and bond pads:

```tcl
set lef_files [list \
  "$env(TSMC_180NM)/apr-tech.tlef" \
  "$env(TSMC_180NM)/stdcells.lef" \
  "$env(TSMC_180NM)/iocells.lef" \
  "$env(TSMC_180NM)/bondpads.lef" \
]
```

And the design uses an I/O assignment file generated by the padring
step:

```tcl
set init_io_file   "../00-padring-gen/padring_gen.io"
```

**Floorplanning.** Instead of the block flow's automatic aspect-ratio
floorplan (`floorPlan -r 1.0 0.70 ...`), the chip flow uses fixed
coordinates that account for the padring:

```tcl
floorPlan -noSnapToGrid -b 15 15 935 935 15 15 935 935 175 175 775 775
```

The `-b` option specifies the chip boundary (15 to 935um, leaving 15um
for the seal ring on each side) and the core area (175 to 775um, leaving
room for the I/O cells between the chip boundary and the core).

**I/O Cell Placement.** The padring generation Tcl script is sourced to
place bond pads at the computed positions:

```tcl
source ../00-padring-gen/padring_gen.tcl
```

This is unique to the chip flow -- the block flow has no I/O cell
placement.

**Power Routing.** The chip flow uses separate power rings for VDD and
VSS on different metal layers to allow for direct routing to the I/O cells:

```tcl
# Route a power ring on M5 for VDD
addRing \
  -nets VDD -width 11.5 -spacing 9.6 -offset 12.8 \
  -layer [list top 5 bottom 5 left 5 right 5] ...

# Route a power ring on M6 for VSS
addRing \
  -nets VSS -width 11.5 -spacing 9.6 -offset 33.9 \
  -layer [list top 6 bottom 6 left 6 right 6] ...
```

The block flow uses a single combined ring. The chip flow also includes
special `sroute` commands to connect the power rings to the I/O cell
pad pin geometries:

```tcl
sroute \
  -allowJogging 1 \
  -connect padPin \
  -allowLayerChange 1 \
  -nets {VDD VSS} \
  -padPinPortConnect { allPort allGeom } \
  -padPinTarget { ring }
```

**Outputs.** The chip flow generates two gate-level netlists:
`post-pnr.v` (standard) and `post-pnr-lvs.v` (physical netlist
excluding filler and pad cells, used for LVS). It also adds `add_gui_text`
commands for LVS pin recognition. The GDS merge includes `iocells.gds`
and `bondpads.gds` in addition to `stdcells.gds`.

### 7.2. Setup Timing

Take a look at the timing setup file:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 05-cadence-innovus-pnr/setup-timing.tcl
```

The key difference from the block flow is that `iocells.lib` is included
in the library set:

```tcl
create_library_set -name libs_typical \
    -timing [list "$env(TSMC_180NM)/stdcells.lib" \
                 "$env(TSMC_180NM)/iocells.lib"]
```

### 7.3. Examining Reports

Check the timing and area reports:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% cat 05-cadence-innovus-pnr/timing-setup.rpt
% cat 05-cadence-innovus-pnr/timing-hold.rpt
% cat 05-cadence-innovus-pnr/area.rpt
```

Verify that both setup and hold slack are positive. Also check the log
output for `verifyConnectivity`, `verify_drc`, and `verify_antenna`
to make sure there are no errors.

Take a closer look at the hierarchical area report:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% cat 05-cadence-innovus-pnr/area.rpt
```

The area report shows a breakdown by module. Here is a simplified view
of the key modules:

```
 Hinst Name              Module Name                                      Total Area
 ---------------------------------------------------------------------------------------
 GcdXcelChip_noparam                                                      77282.579
   v/clk_io              InputIO_is_clk1                                   5113.171
   v/reset_io            InputIO_3                                         5113.171
   v/cs_io               InputIO_2                                         5113.171
   v/sclk_io             InputIO_1                                         5113.171
   v/mosi_io             InputIO_0                                         5113.171
   v/miso_io             OutputIO_2                                        5121.952
   v/minion_parity_io    OutputIO_1                                        5135.123
   v/adapter_parity_io   OutputIO_0                                        5106.586
   v/clk_out_io          OutputIO_is_clk1                                  5124.147
   v/minion              spi_Minion_BIT_WIDTH38_N_SAMPLES4                 21627.110
     v/minion/adapter1   spi_helpers_Minion_Adapter_...                    15563.968
     v/minion/minion     spi_helpers_Minion_PushPull_...                    6063.142
   v/xcel                lab5_xcel_GcdXcel                                  9349.357
     v/xcel/gcd          tut3_verilog_gcd_GcdUnit                          4432.109
     v/xcel/mngr         lab5_xcel_GcdXcelMngr                             4910.662
```

Notice how the area breaks down across the chip:

 - **I/O pads:** ~46,000 um^2 (9 pads at ~5,100 um^2 each) -- this
   dominates the total area at about 60%
 - **SPI minion:** ~21,600 um^2 (~28% of total area) -- the SPI
   interface is surprisingly large, with the adapter queues
   (`adapter1`) consuming ~15,600 um^2 and the push/pull shift
   registers (`minion`) consuming ~6,100 um^2
 - **GCD accelerator:** ~9,300 um^2 (~12% of total area) -- the actual
   compute logic is a small fraction of the total chip

This is a common pattern in chip design: the I/O infrastructure and
communication interfaces often dominate the area, while the actual
compute logic is relatively small. When you push your own accelerator
through the chip flow, you should look at this area breakdown to
understand how much area your accelerator adds compared to the I/O and
SPI overhead.

8. Static Timing Analysis
--------------------------------------------------------------------------

### 8.1. Synopsys PT STA (Step 06)

Run the PrimeTime static timing analysis:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./06-synopsys-pt-sta/run
```

Take a look at the STA script:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 06-synopsys-pt-sta/run.tcl
```

As with synthesis, the `target_library` includes `iocells.db`. The
script reads the post-PNR netlist, SPEF parasitics, and SDC constraints,
then reports setup and hold timing.

Check the timing reports:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% cat 06-synopsys-pt-sta/timing-setup.rpt
% cat 06-synopsys-pt-sta/timing-hold.rpt
```

PrimeTime is the golden signoff tool for timing analysis. Compare these
results to the Innovus timing reports. If PrimeTime reports negative
slack but Innovus showed positive slack, **this is not valid** -- you
must go back and modify your design (e.g., increase the clock period) to
fix this.

9. Back-Annotated Simulation and Power
--------------------------------------------------------------------------

### 9.1. Back-Annotated Gate-Level Simulation (Step 07)

Run the back-annotated simulation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./07-synopsys-vcs-baglsim/run
```

This step uses the SDF timing file from PNR for cycle-accurate
simulation with real gate delays. It also calculates the clock insertion
source latency from the post-PNR SDC file using the
`calc-clk-ins-src-lat` helper script. For the evaluation simulations, it
generates SAIF (Switching Activity Interchange Format) files that record
switching activity for power analysis.

Check that all 18 simulations pass.

### 9.2. Synopsys PT Power Analysis (Step 08)

Run the power analysis:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./08-synopsys-pt-pwr/run
```

The power analysis script reads the SAIF switching activity from the
back-annotated simulations and computes dynamic power. As with other
steps, the `target_library` includes `iocells.db` so that I/O cell
power is accounted for.

Check the power summary for each evaluation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ls 08-synopsys-pt-pwr/*summary*
% ls 08-synopsys-pt-pwr/*detailed*
```

The summary reports show total power (internal + switching + leakage)
and the detailed reports break it down by module hierarchy.

10. Seal Ring and Fill
--------------------------------------------------------------------------

These are new steps unique to the chip flow that prepare the design for
manufacturing.

### 10.1. Seal Ring Generation (Step 09)

Run the seal ring generation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./09-mentor-calibre-seal/run
```

A _seal ring_ is a guard structure of metal and via stacks that runs
around the entire perimeter of the die. During wafer dicing (cutting
individual chips from the wafer), micro-cracks can form at the die
edges. The seal ring prevents these cracks from propagating into the
active circuit area, protecting the chip's functionality.

The step runs two Tcl scripts using `calibredrv`:

 1. `run-gen-sealring.tcl`: Generates a seal ring GDS by placing corner
    cells at the four corners (rotated 0/90/180/270 degrees) and edge
    cells along all four sides from the TSMC seal ring library. It also
    creates a die area polygon on layer 62/0 that defines the chip
    boundary for DRC.

 2. `run-merge-sealring.tcl`: Merges the post-PNR GDS with the seal
    ring GDS, creating `post-sealed.gds`.

### 10.2. Metal, Poly, and Via Fill (Step 10)

Run the fill generation:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./10-mentor-calibre-fill/run
```

During CMP (chemical-mechanical polishing), regions with low metal
density get over-polished, causing thickness variations that can affect
timing and yield. _Fill_ inserts dummy metal, poly, and via patterns to
equalize density across the chip. The fill patterns are electrically
inert -- they don't connect to any signals.

The step generates three types of fill using Calibre DRC rule files:

```bash
calibre -drc -hier -64 -turbo -hyper run-via-fill.rule    # generates vfill.gds
calibre -drc -hier -64 -turbo -hyper run-poly-fill.rule   # generates pfill.gds
calibre -drc -hier -64 -turbo -hyper run-metal-fill.rule  # generates mfill.gds
```

Each rule file specifies the sealed GDS as input and includes the
appropriate PDK fill rule deck. After generating the fill, the
`run-merge-fill.tcl` script merges all three fill GDS files with the
sealed design to create the final `post-filled.gds`. This is the GDS
that goes to DRC and LVS verification.

11. Design Rule Checking
--------------------------------------------------------------------------

### 11.1. Calibre DRC (Step 11)

Run the DRC checks:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./11-mentor-calibre-drc/run
```

The key difference from the block flow is that the chip flow runs _four_
types of DRC checks instead of two:

 - `main-drc`: Standard design rule checks (spacing, width, enclosure,
   overlap, minimum area, etc.)
 - `antenna-drc`: Checks for antenna violations where long metal wires
   connected to transistor gates accumulate charge during fabrication
 - `wirebond-drc`: Checks rules specific to wirebond pad geometry and
   spacing (new in chip flow)
 - `metal-fuse-drc`: Checks rules for metal fuse structures used in
   chip identification or repair (new in chip flow)

You can also run individual DRC types:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./11-mentor-calibre-drc/run main
% ./11-mentor-calibre-drc/run antenna
% ./11-mentor-calibre-drc/run wirebond
% ./11-mentor-calibre-drc/run metal-fuse
```

The `print-summary` script at the end shows the violation count for each
DRC type. All four types should show zero violations.

### 11.2. Interactive DRC Debugging

If there are DRC violations, you can use the interactive Calibre DRV
viewer to visualize them. You must specify which DRC type to debug:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd/11-mentor-calibre-drc
% ./run-interactive main
```

The usage is `./run-interactive {main|antenna|wirebond|metal-fuse}`.
This launches the Calibre Results Viewing Environment with the
post-filled GDS layout and the DRC results for the specified check type.
You can click on individual violations to see their location in the
layout. If you have DRC violations, you likely need to change your clock
period constraint; placement and routing can produce different results
depending on its value.

12. Layout vs. Schematic
--------------------------------------------------------------------------

### 12.1. Calibre LVS (Step 12)

Run the LVS check:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./12-mentor-calibre-lvs/run
```

Take a look at the LVS run script:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% code 12-mentor-calibre-lvs/run
```

The key differences from the block flow are:

 1. The Verilog-to-SPICE conversion (`v2lvs`) uses three SPICE
    libraries instead of one:

    ```bash
    v2lvs \
      -v ../05-cadence-innovus-pnr/post-pnr-lvs.v \
      -o post-pnr.sp \
      -lsr ${TSMC_180NM}/stdcells.sp \
      -lsr ${TSMC_180NM}/iocells.sp \
      -lsr ${TSMC_180NM}/lvs-devices.sp \
      ...
    ```

    The block flow only uses `stdcells.sp`. The chip flow adds
    `iocells.sp` (I/O cell SPICE models) and `lvs-devices.sp`
    (additional device models needed for I/O cells).

 2. The input netlist is `post-pnr-lvs.v` (not `post-pnr.v`). This
    physical netlist excludes filler cells, pad cells, and tap cells
    that would cause LVS mismatches.

 3. The LVS runs on `post-filled.gds` (the final GDS with seal ring
    and fill), and uses `lvs.hcell` to identify hierarchical cells
    that should be treated as black boxes.

The `print-summary` script checks for a CORRECT result. If LVS fails,
common causes include missing power connections and unmatched I/O cells.

### 12.2. Interactive LVS Debugging

If LVS fails, you can use the interactive viewer to debug mismatches:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd/12-mentor-calibre-lvs
% ./run-interactive
```

13. Results Summary
--------------------------------------------------------------------------

### 13.1. Summarize Results (Step 13)

Run the results summary:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd
% ./13-summarize-results/run
```

The summary script aggregates metrics from all previous steps:

```
 timestamp            = 2026-03-20 03:54:04
 design_name          = GcdXcelChip_noparam
 clock_period         = 4.0
 pymtl-rtlsim         = 1/1 passed
 vcs-rtlsim           = 18/18 passed
 synth_setup_slack    = 0.0099 ns
 synth_num_stdcells   = 1195
 synth_area           = 74507.847 um^2
 ffglsim              = 18/18 passed
 pnr_setup_slack      = 0.0074 ns
 pnr_hold_slack       = 0.0388 ns
 pnr_num_stdcells     = 1255
 pnr_area             = 77282.579 um^2
 sta_setup_slack      = 0.0600 ns
 sta_hold_slack       = 0.0194 ns
 baglsim              = 18/18 passed
 main-drc             = no violations found
 antenna-drc          = no violations found
 wirebond-drc         = no violations found
 metal-fuse-drc       = no violations found
 lvs                  = no violations found

 GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-zeros
  - exec_time = 74708 cycles
  - power     = 15.9000 mW
  - energy    = 4751.4288 nJ

 GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-small
  - exec_time = 74708 cycles
  - power     = 16.0000 mW
  - energy    = 4781.3120 nJ

 GcdXcelChip_noparam_gcd-xcel-sim-rtl-chip-random
  - exec_time = 76036 cycles
  - power     = 15.9000 mW
  - energy    = 4835.8896 nJ
```

For the results to be valid, the following must all be true:

 - All PyMTL 2-state RTL simulations pass
 - All four-state RTL simulations pass
 - All fast-functional gate-level simulations pass
 - All back-annotated gate-level simulations pass
 - Place-and-route setup slack is positive
 - Place-and-route hold slack is positive
 - PrimeTime setup slack is positive
 - PrimeTime hold slack is positive
 - All four DRC types pass (zero violations)
 - LVS passes (CORRECT)

If your design does not meet timing after synthesis but _does_ meet
timing after place-and-route then these are still valid results. If
PrimeTime STA passes timing but Innovus fails timing, **this is not
valid -- you must go back and modify your design to fix this.**

14. Interactive Debugging and Visualization
--------------------------------------------------------------------------

### 14.1. Viewing the Chip Layout

You can use Klayout to view the chip layout at various stages:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd/05-cadence-innovus-pnr
% klayout -l $TSMC_180NM/klayout.lyp post-pnr.gds
```

You should see the core logic in the center surrounded by I/O cells
around the perimeter. You can use _Display > Full Hierarchy_ to show
all layout detail including standard cell internals.

To see the final layout with seal ring and fill:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd/10-mentor-calibre-fill
% klayout -l $TSMC_180NM/klayout.lyp post-filled.gds
```

### 14.2. Interactive Innovus

You can load the design in Cadence Innovus for interactive debugging:

```bash
% cd ${HOME}/ece6745/lab9/asic/build-gcd/05-cadence-innovus-pnr
% innovus
innovus> source post-pnr.enc
```

Use _Windows > Workspaces > Design Browser + Physical_ to browse the
design hierarchy. Right click on a module and choose _Highlight_ to
color different modules. You can use _Timing > Debug Timing_ to
visualize the critical path on the layout.

15. To Do On Your Own
--------------------------------------------------------------------------

Now that you understand the chip-level flow, you need to create a
chip-level wrapper for your own Project 2 accelerator and push it
through the chip flow. Your accelerator already has a working xcel
req/resp val/rdy interface; you just need to wrap it with I/O pads, a
reset synchronizer, and an SPI minion.

**Step 1: Create your chip top module.** Create a file called
`<YourXcel>Chip.v` in your accelerator's directory, following the same
pattern as `GcdXcelChip.v`:

 - Instantiate `InputIO` pads for `clk` (with `is_clk=1`), `reset`,
   `cs`, `sclk`, and `mosi`
 - Instantiate `OutputIO` pads for `miso`, `minion_parity`,
   `adapter_parity`, and `clk_out` (with `is_clk=1`)
 - Add the two-flop reset synchronizer
 - Instantiate the SPI minion with `BIT_WIDTH=38` and `N_SAMPLES=4`
 - Wire the SPI `send_msg` directly to your accelerator's xcel request
 - Wire your accelerator's xcel response (zero-padded to 38 bits) to
   the SPI `recv_msg`
 - Instantiate your accelerator using the synchronized clock and reset

**Step 2: Create the PyMTL wrapper.** Create `<YourXcel>Chip.py`
following `GcdXcelChip.py`:

```python
from pymtl3 import *
from pymtl3.passes.backends.verilog import *

class YourXcelChip( VerilogPlaceholder, Component ):
  def construct( s ):
    s.cs              = InPort (1)
    s.sclk            = InPort (1)
    s.mosi            = InPort (1)
    s.miso            = OutPort(1)
    s.minion_parity   = OutPort(1)
    s.adapter_parity  = OutPort(1)
    s.clk_out         = OutPort(1)
```

**Step 3: Create the test harness.** Create `<YourXcel>Chip_test.py`
following `GcdXcelChip_test.py`. Reuse the `test_case_table` from your
FL tests and wire the `StreamSourceFL -> SpiMasterFL -> Chip ->
SpiMasterFL -> StreamSinkFL` test harness.

**Step 4: Add `--impl rtl-chip` to your simulator.** Update your
simulator script to support the `rtl-chip` implementation that uses your
chip wrapper and the `ChipTestHarness`.

**Step 5: Create a design YAML.** Create a design YAML file following
`lab9-gcd-xcel-chip.yml`. Set the `design_name` to your chip module
name (e.g., `YourXcelChip_noparam`), choose an appropriate
`clock_period`, and list your tests and evals. Use the `chip/` step
list.

**Step 6: Run the chip flow.** Generate and run the flow:

```bash
% mkdir -p ${HOME}/ece6745/lab9/asic/build-your-xcel
% cd ${HOME}/ece6745/lab9/asic/build-your-xcel
% pyhflow ../designs/your-xcel-chip.yml
% ./00-padring-gen/run
% ./01-pymtl-rtlsim/run
% ./02-synopsys-vcs-rtlsim/run
% ./03-synopsys-dc-synth/run
% ./04-synopsys-vcs-ffglsim/run
% ./05-cadence-innovus-pnr/run
% ./06-synopsys-pt-sta/run
% ./07-synopsys-vcs-baglsim/run
% ./08-synopsys-pt-pwr/run
% ./09-mentor-calibre-seal/run
% ./10-mentor-calibre-fill/run
% ./11-mentor-calibre-drc/run
% ./12-mentor-calibre-lvs/run
% ./13-summarize-results/run
```

Run the steps one at a time to make sure there are no errors. Verify
that all checks in the summary pass. If your design does not meet
timing, try increasing the clock period. If you have DRC or LVS
violations, use the interactive debugging scripts to diagnose the issue.

