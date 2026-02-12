
ECE 6745 Lab 4: TinyFlow Back End
==========================================================================

In this lab, we will explore the TinyFlow back end which takes as input
a gate-level netlist of standard cells and produces a placed and routed
layout. The complete TinyFlow standard-cell and ASIC design flow is shown
below with the back end highlighted in red.

TODO: tinyflow diagram with back end highlighted

The back end includes floorplanning, placement, routing, and filler cell
insertion. In this lab, we will be implementing unoptimized versions of
placement and routing. The key algorithms in the unoptimized back-end
flow are shown below.

TODO: back-end flow diagram

1. Logging Into `ecelinux`
--------------------------------------------------------------------------

TODO: Same boilerplate as lab3. Clone lab4 repo, copy stdcells views.

```bash
% source setup-ece6745.sh
% source setup-gui.sh
% mkdir -p ${HOME}/ece6745
% cd ${HOME}/ece6745
% git clone git@github.com:cornell-ece6745/ece6745-lab4 lab4
% cd lab4
% tree
```

TODO: repo tree structure

TODO: copy stdcells (stdcells-be.yml, stdcells.gds, behavioral view, gate-level netlist from lab3)

2. Data Structure: Back-End Database
--------------------------------------------------------------------------

As discussed in lecture, the back end takes a gate-level netlist and
produces a physical layout. The key data structure is the back-end
database which manages cells (standard cell instances), nets (connections
between pins), a 2D placement grid of sites, and a 3D routing grid of
nodes. The PnR algorithms -- floorplanning, placement, and routing --
read and modify this database. We provide students the database, and
students are responsible for writing the algorithms.

In this section, we will manually build a small design from scratch in
the REPL to understand how these data structures work together. We will
add cells, create nets, set up a floorplan, place cells, and route nets
by hand before moving on to automated algorithms.

To get started, create a build directory and start the TinyFlow PnR
REPL.

```bash
% mkdir -p ${HOME}/ece6745/lab4/tinyflow/build
% cd ${HOME}/ece6745/lab4/tinyflow/build
% ../tinyflow-pnr
```

### 2.1. Database, Floorplan, and the Grid

Before we can place cells or route wires, we need to set up the chip's
physical foundation: the placement grid and the routing grid. The
placement grid is a 2D array of sites where cells can be placed. The
routing grid is a 3D array of nodes where wires can be routed across
multiple metal layers. Both are created when we call `floorplan` on the
database.

Let's create a back-end library view, an empty database, and a small
floorplan with 4 rows and 10 sites per row.

```python
tinyflow-pnr> view = StdCellBackEndView.parse_lef('../../stdcells/stdcells-be.yml')
tinyflow-pnr> db = TinyBackEndDB(view)
tinyflow-pnr> db.floorplan(4, 10)
tinyflow-pnr> db.enable_gui()
```

`StdCellBackEndView` is the back-end counterpart to the front-end view
from Lab 3. It provides the physical information needed for place and
route: site dimensions, cell layouts (pin locations, cell widths), and
metal layer definitions.

`TinyBackEndDB` is the central database for the back end. It stores all
cells, nets, pins, IO ports, and manages both the placement grid and
routing grid. All PnR algorithms read and modify this database.

`db.floorplan(num_rows, num_sites_per_row)` initializes the chip's
physical grid. It creates a 2D array of placement sites (rows x columns)
where standard cells will be placed, and a 3D routing grid of nodes
across all metal layers where wires will be routed. Nothing can be placed
or routed until this is called. You should see the GUI show two panels:
the top panel is the placement pane showing the grid in sites, and the
bottom panel is the routing pane showing a top-down view of the 3D
routing grid.

Let's explore what `db.floorplan` created. First, check the dimensions
of the placement grid and the routing grid.

```python
tinyflow-pnr> db.get_num_rows()
tinyflow-pnr> db.get_num_cols()
tinyflow-pnr> db.get_grid_size_i()
tinyflow-pnr> db.get_grid_size_j()
```

The placement grid has `num_rows` rows and `num_cols` sites per row --
these are the coordinates you use when placing cells. The routing grid is
finer-grained: `grid_size_i` tracks in the vertical direction and
`grid_size_j` tracks in the horizontal direction, across multiple metal
layers.

Each placement site is represented by a **Site** object. A site is the
smallest unit of space where a cell can be placed. Each rectangular box
in the placement panel GUI corresponds to one site. Let's inspect one in REPL:

```python
tinyflow-pnr> db.get_site_at(0, 0)
tinyflow-pnr> db.get_site_at(0, 0).occupied_by
```

The `occupied_by` attribute is `None` since no cells have been placed
yet.

Each routing grid point is represented by a **Node** object, addressed
by `(i, j, k)` where `i` is the row track, `j` is the column track, and
`k` is the metal layer (1=M1, 2=M2, etc.). In the routing panel of the
GUI, nodes are the intersections of the grid lines. Each node tracks what
occupies it: a net's wire, a cell pin, or an IO port. Let's inspect one:

```python
tinyflow-pnr> db.get_node_at(0, 0, 1)
tinyflow-pnr> db.get_occupancy(0, 0, 1)
```

The node is unoccupied since no wires have been routed yet.

### 2.2. Cells, Nets, and IO Ports

Now that we have a floorplan, let's add cells, create nets to connect
them, and place IO ports. We will build a simple two-inverter chain:
input `a` drives `inv1`, whose output connects to `inv2`, whose output
drives output `y`.

A **Cell** represents an instance of a standard cell from the library
(e.g., INVX1, NAND2X1). When we add a cell to the database, it
automatically creates **Pin** objects based on the cell's layout
definition. Let's add two inverters:

```python
tinyflow-pnr> db.add_cell('inv1', 'INVX1')
tinyflow-pnr> db.add_cell('inv2', 'INVX1')
tinyflow-pnr> db.get_cells()
```

Each cell has pins that we can inspect. Pins are not yet placed on the
grid since we haven't placed the cells yet. Note that the GUI only shows
placed cells, so you won't see them in the GUI here:

```python
tinyflow-pnr> inv1 = db.get_cell('inv1')
tinyflow-pnr> inv1.get_pins()
tinyflow-pnr> inv1.get_pin('A')
tinyflow-pnr> inv1.get_pin('A').get_node()
```

Now let's connect these cells with **Nets**. A net represents a logical
connection between pins. For our inverter chain, we need three nets: `a`
connects the input to `inv1`, `w` connects `inv1`'s output to `inv2`'s
input, and `y` connects `inv2`'s output to the chip output.

```python
tinyflow-pnr> inv2 = db.get_cell('inv2')
tinyflow-pnr> db.add_net('a', [inv1.get_pin('A')])
tinyflow-pnr> db.add_net('w', [inv1.get_pin('Y'), inv2.get_pin('A')])
tinyflow-pnr> db.add_net('y', [inv2.get_pin('Y')])
tinyflow-pnr> db.get_nets()
```

Notice that nets `a` and `y` each have only one pin for now. We will add
their IO port pins in the next step. Let's verify the connectivity:

```python
tinyflow-pnr> db.get_net('w').pins
```

Finally, let's add **IO ports**. An IO port sits at a fixed position on
the chip boundary on M4 (k=4) and acts as a pin for an external signal.
When we call `db.add_ioport`, it automatically connects to the net with
the same name.

```python
tinyflow-pnr> db.add_ioport('a', ??, 0)
tinyflow-pnr> db.add_ioport('y', ??, ??)
```

The IO ports should now appear on the boundary in the GUI as teal
squares. Let's verify
that they were automatically added to their nets:

```python
tinyflow-pnr> db.get_net('a').pins
tinyflow-pnr> db.get_net('y').pins
```

Each net now has two pins: the cell pin and the IO port.

### 2.3. Manually Placing and Routing

With cells, nets, and IO ports created, let's physically place the cells
on the grid and manually route a net.

`cell.set_place(row, col)` places a cell with its origin at site
`(row, col)`. The cell occupies one or more sites to the right depending
on its width. If any of those sites are already occupied by another cell,
the database raises an error. Cells in even rows are oriented normally; cells in odd rows are flipped
vertically so that adjacent rows share VDD and VSS power rails.

```python
tinyflow-pnr> inv1.set_place(0, 2)
tinyflow-pnr> inv2.set_place(2, 5)
```

The cells should now appear in the placement panel as boxes labeled with
the cell name. In the routing panel, each placed cell shows as a shadow
box with its pin locations marked as yellow squares. When a cell is
placed, its pins get assigned grid coordinates on metal layer 1 (M1):

```python
tinyflow-pnr> inv1.is_placed()
tinyflow-pnr> inv1.get_pin('Y').get_node()
tinyflow-pnr> inv2.get_pin('A').get_node()
```

Now that our cells are placed, we want to connect their pins with wires.
A wire is represented as a **Line**, a straight segment between two 3D
grid points `(i, j, k)`. Exactly one coordinate changes: `i` or `j` for
a wire on the same metal layer, or `k` for a via between layers.

Cell pins live on M1 (k=1), but routing happens on M2 and above. In this
example we will route on M2. To connect two cell pins, we need to:

1. Go up from M1 to M2 at the source pin (via)
2. Route on M2 to reach the destination (wire segments)
3. Come back down from M2 to M1 at the destination pin (via)

Since our two inverters are at different rows, the M2 route requires two
segments forming an L-shape. Let's route net `w` which connects `inv1.Y`
to `inv2.A`:

```python
tinyflow-pnr> src = inv1.get_pin('Y').get_node()
tinyflow-pnr> dst = inv2.get_pin('A').get_node()
tinyflow-pnr> net_w = db.get_net('w')
tinyflow-pnr> net_w.add_route_segment([
tinyflow-pnr>   Line((src[0], src[1], 1), (src[0], src[1], 2)),  # via up at source
tinyflow-pnr>   Line((src[0], src[1], 2), (src[0], dst[1], 2)),  # horizontal on M2
tinyflow-pnr>   Line((src[0], dst[1], 2), (dst[0], dst[1], 2)),  # vertical on M2
tinyflow-pnr>   Line((dst[0], dst[1], 2), (dst[0], dst[1], 1)),  # via down at dest
tinyflow-pnr> ])
```

The route should now appear in the routing panel of the GUI. You can
verify the node occupancy along the route:

```python
tinyflow-pnr> db.get_occupancy(src[0], src[1], 2)
tinyflow-pnr> net_w.get_route()
```

### 2.4. Reading Verilog

In practice, we don't manually add cells and nets. The front end
generates a gate-level Verilog netlist, and `db.read_verilog` reads it to
create all cells, pins, and nets automatically. It also stores the list
of input and output port names (accessible via `db.get_input_names()` and
`db.get_output_names()`) which are used later to place IO ports during
floorplanning. Here is how you would initialize the database from a
gate-level netlist and verify what was created:

```python
tinyflow-pnr> view = StdCellBackEndView.parse_lef('../../stdcells/stdcells-be.yml')
tinyflow-pnr> db = TinyBackEndDB(view)
tinyflow-pnr> db.read_verilog('../../path/to/post-synth.v')
tinyflow-pnr> db.get_cells()
tinyflow-pnr> db.get_nets()
tinyflow-pnr> db.get_input_names()
tinyflow-pnr> db.get_output_names()
```

3. Algorithm: Floorplan
--------------------------------------------------------------------------

In Section 2, we called `db.floorplan` directly with the number of rows
and sites per row. In practice, the floorplan algorithm computes the grid
dimensions and places IO ports on the chip boundary.

There are two approaches. `floorplan_fixed` takes explicit chip width and
height in micrometers along with IO port locations, and converts those
physical dimensions into grid coordinates. `floorplan_auto` computes the
chip dimensions automatically from the total cell area and a target
utilization. In this lab, we will implement `floorplan_fixed`.

### 3.1. Fixed Floorplan

`floorplan_fixed` takes the chip width and height in micrometers, along
with a dictionary of IO port locations also in micrometers. It converts
all physical dimensions into grid coordinates using the back-end library
view.

!!! note "Function: `floorplan_fixed(db, view, width_um, height_um, io_locs)`"

    **Args:**

    - `db` -- TinyBackEndDB with cells and nets loaded
    - `view` -- StdCellBackEndView with site dimensions and lambda
    - `width_um` -- Chip width in micrometers
    - `height_um` -- Chip height in micrometers
    - `io_locs` -- Dict mapping port names to (x_um, y_um) locations

    **Returns:** None (modifies db in place)

The algorithm works as follows:

1. Get the technology's lambda value in micrometers from
   `view.get_lambda_um()` and the site dimensions in lambda from
   `view.get_site()`
2. Convert the chip width and height from micrometers to lambda by
   dividing by lambda_um
3. Compute the number of rows and columns by dividing the lambda
   dimensions by the site height and site width
4. Call `db.floorplan(num_rows, num_cols)` to initialize the placement
   and routing grids
5. Convert each IO port location from micrometers to grid coordinates
   and call `db.add_ioport` for each port

Go ahead and implement `floorplan_fixed` in `floorplan.py`. Use
`logging.info` to print useful information such as the computed number of
rows and columns. Once you are done, you can
test your implementation with the following:

```python
tinyflow-pnr> view = StdCellBackEndView.parse_lef('../../stdcells/stdcells-be.yml')
tinyflow-pnr> db = TinyBackEndDB(view)
tinyflow-pnr> db.read_verilog('../../path/to/post-synth.v')
tinyflow-pnr> io_locs = { 'a': (??, ??), 'b': (??, ??), 'y': (??, ??) }
tinyflow-pnr> floorplan_fixed(db, view, ??, ??, io_locs)
tinyflow-pnr> db.get_num_rows()
tinyflow-pnr> db.get_num_cols()
tinyflow-pnr> db.get_ioport('a')
```

4. Algorithm: Unoptimized Placement
--------------------------------------------------------------------------

Placement assigns each cell to a location on the placement grid. The
goal is to find positions for all cells such that no two cells overlap.
In an optimized flow, placement also minimizes wire length and improves
timing. In this lab, we will implement `place_unopt`, which randomly
assigns cells to grid positions without any optimization. In the
project, we will build on this random placement using simulated annealing
to iteratively improve cell positions.

### 4.1. Random placement

The random placement we implement in this lab is a naive version that
simply places cells at random positions without any optimization. If we
recall the standard cells we drew in Part A, cells can have different
widths. To ensure no overlap, we use a coarse grid: instead of
considering every site column, we divide the columns into slots that are
each as wide as the widest cell. Since every slot is at least as wide as
any cell, placing one cell per slot guarantees no overlap. The algorithm
shuffles these coarse grid positions and assigns one to each cell.

The algorithm works as follows:

1. Find the maximum cell width across all cells
2. Compute the number of coarse columns by dividing the total columns by
   the max cell width
3. Generate all (row, coarse_col) positions and shuffle them randomly
4. For each cell, take the next position and place it at
   (row, coarse_col * max_width)

!!! note "Function: `place_unopt(db)`"

    **Args:**

    - `db` -- TinyBackEndDB with floorplan initialized

    **Returns:** None (modifies db in place)

Go ahead and implement `place_unopt` in `place_unopt.py`. You can use
`logging.info` to print useful information such as the number of cells
placed or cell's placed location.

Once you are done, test your implementation in the REPL:

```python
tinyflow-pnr> view = StdCellBackEndView.parse_lef('../../stdcells/stdcells-be.yml')
tinyflow-pnr> db = TinyBackEndDB(view)
tinyflow-pnr> db.read_verilog('../../path/to/post-synth.v')
tinyflow-pnr> io_locs = { 'a': (??, ??), 'b': (??, ??), 'y': (??, ??) }
tinyflow-pnr> floorplan_fixed(db, view, ??, ??, io_locs)
tinyflow-pnr> place_unopt(db)
tinyflow-pnr> db.enable_gui()
tinyflow-pnr> db.get_cells()[0].is_placed()
tinyflow-pnr> db.get_cells()[0].get_place()
```

You should see all cells placed on the grid in the GUI. Each cell
appears as a labeled box in the placement panel, with pin locations
shown as yellow squares in the routing panel.

5. Algorithm: Unoptimized Routing
--------------------------------------------------------------------------

Routing draws the metal wiring and vias that connect the pins of each
net. In Section 2, we manually routed a net by creating Line segments
for vias and wires. In this section, we will automate that process. In
this lab, we will implement a naive router that connects pins using
manhattan L-shaped routes. Similar to Lab 3, we will start by
implementing small modular functions and build up towards a working
multi-net router. In the project, we will replace this with an optimized
router.

### 5.1. Check Lines

Before we can route a net, we need a way to check whether a proposed
route is valid. When routing a net, we construct a **route** for that
net. A route is a list of **Lines**, where each Line is a straight
segment between two grid points with exactly one coordinate changing
(a wire in the x or y direction on one layer, or a via in the z
direction between layers). Together, the Lines in a route form the
complete wiring for one net.

Each Line passes through a sequence of grid points. If any of those
points is already occupied by another net's wire, we cannot use that
route. For example, if two nets try to use the same node on M2, that
would be a short circuit.

A Line provides `get_points()` which returns all grid points along the
segment. We can check each point using `db.get_occupancy(i, j, k)`,
which returns the net occupying that node or `None` if it is available.
A point is a collision only if it is occupied by a different net. Points
occupied by the current net are allowed since a net can cross its own
wires.

!!! note "Function: `check_lines(db, net, lines)`"
    **Args:**

    - `db` -- TinyBackEndDB for occupancy checks
    - `net` -- Current net (allowed to use its own nodes)
    - `lines` -- List of Line segments to check

    **Returns:** True if all lines are clear, False if any collision

Go ahead and implement `check_lines` in `single_route_unopt.py`.

To test, manually add a route to one net, then check if the same lines
collide with a different net:

```python
tinyflow-pnr> net_a = db.get_net('??')
tinyflow-pnr> net_b = db.get_net('??')
tinyflow-pnr> lines = [Line((??, ??, 2), (??, ??, 2))]
tinyflow-pnr> net_a.add_route_segment(lines)
tinyflow-pnr> check_lines(db, net_b, lines)
tinyflow-pnr> check_lines(db, net_a, lines)
```

The first check should return `False` since the nodes are occupied by
`net_a`. The second check should return `True` since a net is allowed
to cross its own wires.


### 5.2. Manhattan Route (2D, Single Layer)

Now we implement a simple routing function that only routes on the M2
layer. Given two pin locations on M1, `manhattan_route_m2` connects them
with an L-shaped route on M2. An L-shaped (manhattan) route uses at most
two wire segments that meet at a right angle: one in the x direction and
one in the y direction.

The algorithm first tries one L-shape ordering, then the other:

1. Try x-then-y: construct Lines for via up (M1 to M2), x-direction
   wire, y-direction wire, via down (M2 to M1). Use `check_lines` to
   test for collisions. If clear, commit with
   `net.add_route_segments(lines)` and return `True`.
2. Try y-then-x: construct Lines for via up, y-direction wire,
   x-direction wire, via down. Check and commit if clear.
3. If both orderings collide, return `False`.

!!! note "Function: `manhattan_route_m2(db, net, start, end)`"
    **Args:**

    - `db` -- TinyBackEndDB for occupancy checks
    - `net` -- Net to add route to
    - `start` -- Starting (i, j, k) tuple on M1
    - `end` -- Ending (i, j, k) tuple on M1

    **Returns:** True if route found and committed, False if both orderings blocked

Go ahead and implement `manhattan_route_m2` in `single_route_unopt.py`.

TODO: REPL exercises -- route between two points, observe in GUI.

### 5.3. Manhattan Route (3D, Multi-Layer)

Now we generalize `manhattan_route_m2` to try multiple metal layers.
Instead of only routing on M2, `manhattan_route` tries each routing
layer from M6 down to M2. For each layer, it tries both L-shape
orderings (x-then-y and y-then-x) just like before. If a layer is
blocked, it moves on to the next layer. If all layers and orderings
are exhausted, it returns `False`.

!!! note "Function: `manhattan_route(db, net, start, end)`"
    **Args:**

    - `db` -- TinyBackEndDB for occupancy checks
    - `net` -- Net to add route to
    - `start` -- Starting (i, j, k) tuple
    - `end` -- Ending (i, j, k) tuple

    **Returns:** True if route found and committed, False if all options blocked

Go ahead and implement `manhattan_route` in `single_route_unopt.py`.

TODO: REPL exercises.

### 5.4. Single-Net Routing (`single_route_unopt`)

Now we put the pieces together to route a single net. So far,
`manhattan_route` connects two pins. But a net can have more than two
pins. For example, a net driving a fan-out of three gates has four pins
(one driver and three loads). `single_route_unopt` takes a net name,
gets all of its pin locations, and connects them sequentially: pin 0 to
pin 1, pin 1 to pin 2, and so on. If any pair fails to route, it
returns `False`. If all pairs succeed, it marks the net as routed and
returns `True`.

!!! note "Function: `single_route_unopt(db, net_name)`"
    **Args:**

    - `db` -- TinyBackEndDB with placed design
    - `net_name` -- Name of the net to route

    **Returns:** True if routing succeeded, False otherwise

Go ahead and implement `single_route_unopt` in `single_route_unopt.py`.

TODO: REPL exercise -- route a single net by name, verify in GUI.

### 5.5. Multi-Net Routing (`multi_route_unopt`)

Finally, we route all nets in the design. `multi_route_unopt` iterates
over every net with two or more pins and calls `single_route_unopt` for
each one. It tracks which nets failed to route and reports the results.

!!! note "Function: `multi_route_unopt(db)`"
    **Args:**

    - `db` -- TinyBackEndDB with placed design

    **Returns:** True if all nets routed, False otherwise

Go ahead and implement `multi_route_unopt` in `multi_route_unopt.py`.

TODO: REPL exercise -- route all nets, check results.

6. Algorithm: Add Filler
--------------------------------------------------------------------------

After placement and routing, some sites on the grid may still be empty.
In a real chip, every site must be filled to satisfy manufacturing
design rules and maintain continuous power rails. `add_filler` walks
through every site in the placement grid and marks any unoccupied site
as a filler cell.

!!! note "Function: `add_filler(db)`"
    **Args:**

    - `db` -- TinyBackEndDB with placed and routed design

    **Returns:** None (modifies db in place)

Go ahead and implement `add_filler` in `add_filler.py`. You can access
the placement grid using `db.get_core()`, which returns a 2D array of
sites. Each site has `site._get_occupancy()` to check if a cell is
placed there, and `site.add_filler()` to mark it as a filler cell.

TODO: REPL exercise.

7. TinyFlow Back End
--------------------------------------------------------------------------

TODO: End-to-end batch flow. Write run.py script.

```bash
% cd $HOME/ece6745/lab4/asic/build-fa/XX-tinyflow-pnr
% code run.py
```

```python
view = StdCellBackEndView.parse_lef('../../stdcells/stdcells-be.yml')
db = TinyBackEndDB(view)
db.read_verilog('../XX-tinyflow-synth/post-synth.v')
db.enable_gui()

floorplan_fixed(db, view, 30.0, 30.0, io_locs)
place_unopt(db)
promote_pins(db)
multi_route_unopt(db)
add_filler(db)

db.check_design()
db.report_summary()
db.stream_out('../../stdcells/stdcells.gds', 'out.gds')
```

TODO: Run batch script, view GDS output in KLayout.

TODO: DRC, LVS

TODO: Screenshots of final layout.
