
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

TODO: Intro. Given die width and height in um, need to derive lambda
coordinates. Explain site/row structure. IO pins must be on chip edge.

### 3.1. Fixed Floorplan

TODO: Explain floorplan_fixed -- given width/height in um and IO locations.
Students understand the conversion from um to lambda grid coordinates.

```python
tinyflow-pnr> io_locs = { ... }
tinyflow-pnr> floorplan_fixed(db, view, width_um, height_um, io_locs)
```

TODO: REPL exercises. Verify grid size, check IO placement on boundary.

4. Algorithm: Unoptimized Placement
--------------------------------------------------------------------------

TODO: Intro. We just need to place all cells on the floorplan without
overlap. No optimization -- just find an unoccupied site and place.

### 4.1. Implementing `place_unopt`

TODO: Explain coarse grid approach. Divide columns by max cell width.
Shuffle positions, assign cells.

TODO: Student task scaffold in place_unopt.py

TODO: REPL exercise -- run place_unopt, verify cells placed, check GUI.

5. Pin Promotion
--------------------------------------------------------------------------

TODO: Placeholder. Cell pins live on M1, routing on M2+. Need to elevate
pins before routing. May or may not include in lab.

TODO: Explain promote_pins concept. M1 to M2 via, occupancy reservation.

6. Algorithm: Unoptimized Routing
--------------------------------------------------------------------------

TODO: Intro. Routing connects pins of each net using metal wire segments
(Lines). We build up from basic checks to full multi-net routing.

### 6.1. Check Lines

TODO: Explain Line class -- represents a straight wire segment
(i,j,k) to (i,j,k) where exactly one coordinate changes.

TODO: Student implements check_lines -- validate that a proposed Line
does not conflict with occupied nodes from other nets.

TODO: REPL exercises testing check_lines.


### 6.2. Manhattan Route (2D, Single Layer)

TODO: Explain L-shaped manhattan routing on a single metal layer.
Given two points on the same layer, route with an L-shape (horizontal
then vertical, or vertical then horizontal).

TODO: Student implements 2D manhattan route.

TODO: REPL exercises -- route between two points, observe in GUI.

### 6.3. Manhattan Route (3D, Multi-Layer)

TODO: Extend to 3D -- routing across metal layers with vias.
M1 to M2 via at source, route on M2, M2 to M1 via at destination
(or similar).

TODO: Student implements 3D manhattan route.

TODO: REPL exercises.

### 6.4. Single-Net Routing (`single_route_unopt`)

TODO: Putting it together. single_route_unopt routes all pins of a
single net using manhattan routing.

TODO: Student implements single_route_unopt.

TODO: REPL exercise -- route a single net by name, verify in GUI.

### 6.5. Multi-Net Routing (`multi_route_unopt`)

TODO: Route all nets. Iterate over nets, call single_route_unopt for
each. Track failures.

TODO: Student implements multi_route_unopt (or provided?).

TODO: REPL exercise -- route all nets, check results.

7. Algorithm: Add Filler
--------------------------------------------------------------------------

TODO: Fill empty sites with filler cells for manufacturing rules.

TODO: Student implements add_filler.

TODO: REPL exercise.

8. TinyFlow Back End
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
