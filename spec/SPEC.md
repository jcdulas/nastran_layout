# nastran-layout — Specification

Status: draft for the owner's review (2026-10-09). Package `nastran_layout`, repository
https://github.com/jcdulas/nastran_layout.

Companion document: what `nastran_layout` asks of `nastran_rw` ([NASTRAN-RW-PREREQUISITES.md](NASTRAN-RW-PREREQUISITES.md)).
The reading of models is `nastran_rw`'s; the analysis of stiffened structures is `nastran_ssa`'s; the smeared results
are `nastran_smear`'s; the selection of the sizing load cases is `nastran_lcs`'s.

## 1. Purpose

A **Python package** that describes the **layout of a stiffened structure** (a fuselage, a wing, a floor, a bulkhead)
once, in one versioned schema that every package reads: the reference **stations**, the **surfaces** (skins, floors,
webs), the **members** that stiffen them (stringers, frames, rails, floor beams, longerons, intercostals, spar and rib
caps, or a kind the user names), their **nodes** (where members meet), their **edges** (a member between two nodes),
the **pockets** (a region of a surface bounded by edges), the named **groups** (sides, zones), and the **calculation
points** at which the analyses read their loads.

The layout is **static**: the sizing changes thicknesses and section dimensions, never the topology nor the geometry
(decision of 2026-10-09). It is built once, before the analyses, then read; it holds no loads, no properties and no
methods.

It has two levels:

- **the idealised structure** (level 1), independent of any finite element model: what the structure is;
- **the mapping to a model** (level 2), one per model: which elements represent each entity, with which role, and how
  the loads of each calculation point are obtained from that model. A global model (GFEM) and a detailed model (DFEM) of
  the same structure share level 1 and have two mappings. A member may be represented by 1D elements, by shells, or by
  both, depending on the fidelity of the model, along the whole member or part of it (decision of 2026-10-09).

The layout is built from the user's tables, or inferred from the mesh to start the tables, and checked against the
model. It is edited through an API of atomic, validated, journalled operations, which a 3D editor (another plugin,
section 3.7) drives; the journal is replayed after a remesh so that the corrections are not lost.

The packages split the work:

| | `nastran_rw` | `nastran_layout` | `nastran_ssa` | `nastran_smear` | `nastran_lcs` |
|---|---|---|---|---|---|
| Reads decks and results; generic topology queries (edges, panels, chains, cuts) | yes | uses them | uses them | uses them | uses them |
| Stations, surfaces, members, nodes, edges, pockets, groups, calculation points; their mapping to a model | no | yes | reads it | reads it | reads it |
| Loads at the calculation points | no | gives the recipes | reads them (through `nastran_rw` or `nastran_sv`) | smears them | superposes them |
| Sections, materials, methods, reserve factors, sizing | no | no | yes | no | uses the RF as a criterion |

## 2. Constraints

- **Python only**: one `py3-none-any` wheel. The heavy work is the compiled code that exists: the topology queries of
  `nastran_rw` (C++), the sparse graphs of SciPy, the vector operations of NumPy. A need it does not cover is asked of
  `nastran_rw` (the companion document), not worked around here.
- **`nastran_rw` through its public API and data contract only** (its companion-packages.md §1): never `_core`, never a
  private name, never a deck parsed here.
- **Dependencies**: `nastran_rw` (a declared range of data contract versions, checked at import), NumPy, SciPy, pyarrow
  (the storage, section 3.8). No import of `nastran_ssa`, `nastran_smear`, `nastran_lcs` or `nastran_sv`: they read
  the layout, not the reverse. No GUI dependency.
- **Everything is written in English.** **No test lasts more than one second**; longer checks are benchmarks.
- **Platforms**: Windows x64 and Linux x64, CPython 3.12 and later.
- **Deterministic**: the same inputs give the same layout, the same identifiers and the same files, whatever the number
  of threads or the order of the inputs.

## 3. Functional requirements

### 3.1 Level 1: the idealised structure

Every entity has a **stable identifier**: a name given by the user or made by the package from the entity's content
(section 4.1), never a row index; an identifier does not change when another entity is added, removed or edited. Every
entity records its **provenance**: `table` (given by the user), `inferred` (from the mesh), `edited` (changed by an
edit, section 3.7).

All lengths are in the units of the model; the layout records the unit system the user declares for the model
(`units="mm"`), or `unknown`. It converts nothing.

**Reference axis and stations.** `axis`: the reference axis of the structure (a point and a direction, or a coordinate
system of the model): along x for a fuselage, along the span (the reference line) for a wing. A **station** is a
reference plane: `station` (name), `kind` (`frame`, `rib`, `cut`, or a user kind), `point`, `normal`, `position` (along
the axis). The planes need not be parallel (the ribs of a swept wing). Stations serve three purposes: the address of the
results (section 3.3), the cuts of the section loads, and the place of the frames and ribs; they are not the place of
the calculations.

**Surfaces.** `surface` (name), `kind` (`skin`, `floor`, `spar_web`, `rib_web`, `bulkhead`, or a user kind). A surface
is a sheet that members stiffen and that pockets divide.

**Members.** `member` (name), `kind` (`stringer`, `frame`, `rail`, `floor_beam`, `longeron`, `intercostal`, `spar_cap`,
`rib_cap`, or a user kind: the list is open), `surfaces` (the surfaces it is attached to: none for a rail standing on
floor beams, two for a spar cap between a cover and a web), `station` (for a frame or a rib: the station it lies in), the
**reference line** (an ordered polyline in the basic system: the line of the member on its surface, at the attachment),
and its **section segments**: `segment` (name), `role` (`foot`, `web`, `flange`, `lip`, `cap`, or a user role), `order`.
The segments name the parts of the section that the mapping (section 3.2) and the analyses (crippling per segment)
refer to; their dimensions are not in the layout (they change in the sizing).

**Nodes.** `node`: a point where two members or more meet, or where a member ends (a run-out), or where its
representation changes (section 3.2); `position`, `members`, `kind` (`crossing`, `junction`, `end`, `transition`).

**Edges.** `edge`: a member between two consecutive nodes: `member`, `node_a`, `node_b`, `order` along the member,
`length`, the reference polyline between the nodes.

**Pockets.** `pocket`: a region of a surface bounded by edges: `surface`, `boundary` (the edges in order around the
pocket, each with its orientation; a closed loop; a pocket of three sides at the tip of a wing, or of five where a
member ends inside, is a pocket like the others), `area`, `centroid`, the **local frame** (z the area-weighted mean
normal, oriented outward for a skin; x the projection of the axis, or of the direction of the members the user names
for that surface, on the plane normal to z; y = z × x), the **dimensions** in that frame (`length` along x and `width`
along y between the reference lines of the bounding edges; for a pocket that is not a quadrilateral, the extents and a
flag `irregular`), the **curvature** (the radius fitted along y, along x, or flat within a tolerance), and the stations
it lies between.

The faces of a station (fwd and aft, inboard and outboard) are not stored: a pocket or an edge between the stations i
and i+1 is on the aft face of i and on the fwd face of i+1, and the package gives that relation.

**Groups.** `group` (name), `kind` (`side`, `zone`, or a user kind), and its members: pockets, edges, nodes, members,
over all stations or a range of them. A side of a fuselage (`crown`, `keel`, ...) or of a wing (`upper_cover`,
`front_spar`, ...) is a group; `nastran_ssa` attaches its side types to groups by name. Groups may overlap.

### 3.2 Level 2: the mapping to a model

A mapping is made for one model (`label`, the path and the revision stamps of the elements it maps, from `nastran_rw`'s
`group_revisions`). It holds:

- **The role of each element** (`element_roles`): `eid`, `type`, the entity it belongs to (a pocket, an edge, a node),
  its `role` (`skin`, `member`, `skin_and_foot` for a shell that is both the skin and the foot of a member, `fastener`,
  `clip`, `filler`, or a user role) and, for a member, its `segment`. A skin shell and its doubler stacked on the same
  grid points both have the role `skin` of the pocket. An element of the model that belongs to no entity is listed by
  the check (section 3.6), not forced into one.
- **The representation of each edge** (`edge_representations`): `1d` (a chain of CBAR, CBEAM, CROD, CONROD, CTUBE),
  `2d` (shells, each tied to a segment), `mixed` (both at the same place: a web in shells and a flange in rods), or
  `absent` (the model does not represent this edge: a coarse model without that intercostal). A member may change its
  representation from one edge to the next; the node where it changes has the kind `transition`.
- **The recipe of each calculation point** (section 3.3).

The same level 1 may have several mappings; the package compares two mappings entity by entity (which entities each
represents, with which representation).

### 3.3 Calculation points and their recipes

The calculation points are **explicit**: the analyses read their loads at the points of the layout, and the results
are given at them (decision of 2026-10-09). The default points:

| Entity | Points | What they serve |
|---|---|---|
| pocket | `centroid` | the average fluxes of the pocket (plate buckling, strength) |
| edge | `end_a`, `end_b`, `mid` | the loads of the member along the edge (column, crippling at the worst point, beam-column) |
| node | `node` | the loads of the attachments (clips, shear ties, rail fittings) |

The user adds points (a hot spot at a rail fitting, more points along a long frame edge) and changes the defaults per
member kind or group. Each point has: `point` (identifier), `entity`, `location` (`centroid`, `end_a`, `end_b`, `mid`,
`node`, `at` with a parameter along the edge), its `position`, its **frame** (the pocket frame; for a member, x along
its tangent, z along the normal of its surface, or of the plane of its station for a frame), and its **address**: the
stations it lies between (or the station it lies in), the groups it belongs to, the face. An entity with no station
(a rail outside the stations' range) is addressed by its identifier alone.

The **recipe** of a point, in a mapping, says how its loads are obtained from that model; it is data, executed by the
consumers through `nastran_rw` (MSC results) or `nastran_sv` (results and sensitivities), never by this package:

| Representation | Recipe | What it gives |
|---|---|---|
| pocket | `average`: the skin shells of the pocket with their weights (the area of each shell over the area of the pocket, the shells stacked on the same grid points counted once in that area: a skin and its doubler, a skin and a foot add their fluxes), summed; the frame of the point | Nx, Ny, Nxy, Mx, My, Mxy, Qx, Qy in the pocket frame |
| edge `1d` | `element_end`: the element at the point and its end (or its station x/L for a CBEAM), the vector from the element axis to the reference point of the member | the forces of the bar, moved to the reference point, in the member frame |
| edge `2d` | `cut`: the grid points of the member on the line of grid points nearest the point (the actual position stored), the elements of the member on one side, the reference point, the member frame; and one such cut per segment | the resultants N, V1, V2, T, M1, M2 of the member; the loads of each segment |
| edge `mixed` | `cut` over the bars and the shells of the member | the same |
| node | `node_forces`: the elements meeting at the node whose forces are asked (fasteners, clips) | the forces at the attachment |

Whatever the representation, the consumers get the same quantities at a point of an edge: the resultants of the
member at its reference point in its frame, and the loads of its segments where the model has them (in a `1d` model
they are recovered from the resultants by the consumer, which knows the section). **Adapters** turn the recipes into
the arguments of the calls that execute them (`nastran_rw`'s `read_block(element_groups=)` and `section_cuts(cuts=)`;
`nastran_sv`'s section definitions and `Cut`s), without importing `nastran_sv` and without reading results.

### 3.4 Building a layout from tables

The user gives tables (CSV, Parquet, or DataFrames), each with its columns as in section 3.1: `stations`, `surfaces`,
`members` (the reference line as a polyline, or as an ordered list of elements or grid points of a model),
`member_segments`, `groups`, `calc_points` (additions and changes). A member given by elements or grid points is turned
into a polyline from the positions of that model, so that level 1 stays independent of it; the elements also seed its
mapping to that model. The package derives what the tables leave out: the nodes (the crossings of the reference lines
within a tolerance), the edges, the pockets (the regions of each surface the edges close), their frames, dimensions and
curvatures, the default points. The mapping to a model is then made (section 3.5, step 3).

### 3.5 Inferring a layout from the mesh

To start the tables, not to replace them: an inferred layout is reviewed (the check, the editor) before it is used
(decision of 2026-10-09: `nastran_ssa` infers nothing from the mesh; what it reads is the layout the user accepted).
The user gives **rules**: the reference axis; the surfaces by property IDs, groups or element sets; the kinds of the
members by property (a table `pid → kind, segment`), by direction (parallel to the axis: stringer, longeron, rail;
in a station plane: frame, floor beam) and by surface (on the skin, on the floor, on none); the stations by position,
or found from the frames. The steps (section 4.2):

1. the 1D members: the chains of 1D elements, joined across the nodes where they cross other members when their
   directions continue;
2. the 2D members: the shells standing on a surface (on the edges that three shells or more share, the foot of a web),
   grouped by property and continuity into members and segments;
3. the mapping: the roles, the representations (a member found both as a chain and as shells at the same place is
   `mixed`), the recipes;
4. the nodes, edges, pockets and groups, as from tables (section 3.4);
5. the diagnostics: what could not be classified, the members that end without a node, the pockets that do not close.

The inferred layout can be exported as the tables of section 3.4, for the user to correct and keep as the reference.

### 3.6 Checks

`layout.check(model, mapping)` gives a diagnostics table (code, entity, elements, message):

- **level 1**: the boundary of each pocket closed; no edge without a member; the edges of a member continuous; no two
  pockets overlapping on a surface; the stations crossing the members they should;
- **level 2**: every element of the mapped surfaces and members in exactly one entity (the elements of no entity, of
  two entities); the segments of a `2d` edge all present; the cut of each `2d` point complete (the line of grid points
  crosses the whole member);
- **the tables against the mesh**: the layout given by the user against the layout inferred with the same rules, entity
  by entity (a member in the tables and not in the mesh, a pitch that differs beyond a tolerance, a pocket that the
  mesh splits);
- **the model has changed**: the revision stamps of the mapped elements against those recorded (`group_revisions`), and
  `model_diff` for the cards that moved; a mapping whose elements changed is **stale** and is made again (section 3.7).

A change of a property (a thickness, a section, a laminate) never makes a mapping stale; a change of the elements, of
the grid points or of their connectivity does.

### 3.7 Editing, journal and the 3D editor

**Operations.** The layout is changed only through atomic operations, each validated on the entities it touches
(section 3.6, locally) and refused with its diagnostics when it would break the layout: create or delete a member
along a chain of elements or a polyline; change the kind of a member; split or join members; set the representation of
an edge; reassign elements (entity, role, segment); split or merge pockets; add, move or delete a calculation point;
create, rename or change a group; add or move a station. Each operation returns the entities it created, changed and
deleted.

**Journal.** Every operation is recorded in a journal (the operation, its arguments by stable identifiers and element
IDs, the time, the author when given). The journal gives undo and redo, and it is **replayed** on a layout built again
(after a remesh: inferred again, then the corrections replayed); an operation that no longer applies (an element gone)
is reported, never skipped silently. The entities that edits made or changed have the provenance `edited`.

**The 3D editor** is another plugin; this package gives it, and needs nothing from it:

- the geometry to draw: the reference lines of members and edges as polylines, the pockets as polygons, the nodes, the
  calculation points with their frames, and the elements of the model (through `nastran_rw`);
- the picking: element ID → entities, entity → elements;
- the provenance and the diagnostics of each entity, for its colours;
- the operations and the journal above. The editor never writes the layout folder itself.

### 3.8 Storage and schema

A layout is saved as **one folder** (decision of 2026-10-09): a `manifest.json` (the schema version, the units, the
reference axis, the list of the mappings with their model labels and revision stamps, the rules of the inference) and
one Parquet file per table of sections 3.1 to 3.3, the level 2 tables in a sub-folder per mapping
(`mappings/<label>/`), the journal in `journal.parquet`. A folder is easy to inspect and to keep under version control
beside the decks; it is written to a temporary folder beside it, then renamed, so that a reader never sees half a
layout. The **schema** (the files, the tables, their columns, their types, their meaning) is the contract with the
consumers: it is versioned, documented, recorded in a `tests/schema_contract.json` checked by the tests, and a change of
it increments the schema version (as the data contract of `nastran_rw`). A consumer may read the folder with pyarrow
alone, without this package.

### 3.9 What the consumers read

- **`nastran_ssa`**: its items are the entities: `skin_panel` = a pocket of a skin, `stringer` = an edge of a member of
  kind stringer (and of the kinds the user analyses as stringers: longeron, rail), `frame` = an edge of a member of kind
  frame or rib, `stiffened_panel` = the pockets and edges of a group between two stations, `station_shell` = a station;
  its side types attached to the groups by name; the geometry of the items (pocket dimensions and curvature, the pitch,
  the length of an edge between its nodes); the calculation points and their recipes for its loads.
- **`nastran_smear`**: the stiffened panels from the layout (the members, the pitch, the pockets on either side of an
  edge) instead of inferring them.
- **`nastran_lcs`**: the zones (`layout.zones(by="pocket" | "group" | "station")`: the elements of each), and the items
  made of several elements of mixed types (`element_roles`).
- **The 3D editor**: section 3.7.

### 3.10 Errors, diagnostics, progress

- Typed exceptions (`LayoutError`, `TableError`, `MappingError`, `EditError`, `SchemaError`), each naming the table, row
  and column, or the entity and the elements at fault.
- A diagnostics table (section 3.6).
- `progress(stage, done, total)` as in `nastran_lcs`, returning `False` cancels.

### 3.11 Python API (illustrative)

```python
import nastran_rw as nr
import nastran_layout as nl

model = nr.Model.read("fuselage_gfem.bdf")

rules = nl.Rules(
    axis=(0.0, 0.0, 0.0, 1.0, 0.0, 0.0),
    surfaces={"skin": {"kind": "skin", "pids": [100, 101]}, "floor": {"kind": "floor", "pids": [300]}},
    member_kinds="member_kinds.csv",              # pid -> kind, segment
)
draft = nl.infer(model, rules)                   # a starting point, to review
draft.check(model).to_dataframe()                # what could not be classified
draft.to_tables("layout_tables/")                # the tables the user corrects and keeps

layout = nl.Layout.from_tables("layout_tables/", units="mm")
gfem = layout.map(model, label="gfem")           # roles, representations, recipes
layout.check(model, gfem)

with layout.edit(author="jc") as edit:           # what the 3D editor calls
    edit.set_kind("M-0412", "longeron")
    edit.add_point(edge="M-0412/C40-C41", at=0.25)
layout.save("fuselage_layout/")                  # a folder: manifest.json and Parquet tables

dfem_model = nr.Model.read("fuselage_dfem.bdf")
dfem = layout.map(dfem_model, label="dfem")      # the same structure, another fidelity
layout.compare_mappings("gfem", "dfem")

cuts = gfem.recipes.to_rw()                      # the arguments of read_block and section_cuts
```

## 4. Algorithms

### 4.1 Stable identifiers

The identifier of an entity made by the package is built from what defines it: a member from its kind and the smallest
element ID of its chain or the first point of its line, an edge from its member and its two nodes, a pocket from its
surface and its boundary edges, a node from the members that meet there, a point from its entity and its location. The
names are readable (`STR-12/C40-C41`), unique, and the same for the same layout whatever the order of the inputs. A
user name always wins.

### 4.2 Inference

- **1D members** from `nastran_rw`'s `element_chains` (chains that stop where three 1D elements meet), joined into
  members across a crossing when the tangents continue within an angle; split where the property changes when the rules
  ask it.
- **2D members**: the edges shared by three shells or more (`edges_table`, `kind == "non_manifold"`) give the lines where
  a web stands on a surface; the shells off the surface on those lines, grown by `panels` (with `feature_angle`, and
  `split_on_property` when the rules ask it), give the webs, flanges and lips; their properties and positions give the
  segments.
- **Pockets**: `nastran_rw`'s `panels` of each surface, bounded by the members, then their boundary loops
  (`free_edges` of their elements) cut into the edges of the layout; the stacks of shells (a skin and its doubler, a
  skin and a foot) are one place.
- **Frames and dimensions**: the area-weighted normal, the projection of the axis; the length and the width between the
  reference lines of the bounding edges; the radius by a least-squares circle fit of the grid points of the pocket across
  and along.
- Everything is vectorised over the entities (NumPy, SciPy sparse graphs): no Python loop over the elements.

### 4.3 Calculation points of `2d` members

A cut through the grid point forces needs a line of grid points: the point of a `2d` edge is snapped to the nearest line
of grid points of the member that crosses all its segments; the actual position is stored and the snap distance reported
when it exceeds a tolerance.

## 5. Non-functional requirements

### 5.1 Performance targets (reference machine: the owner's, 12 threads)

The target volume is a **whole fuselage**: some 200 000 shells and 50 000 1D elements, 100 stations, some 20 000
pockets, 25 000 edges and 100 000 calculation points.

| Case | Target |
|---|---|
| Inference of a whole fuselage | under 1 min |
| Building from tables, and mapping to a model | under 30 s |
| Save and load of a layout folder | under 3 s |
| Complete check | under 20 s |
| One edit operation and its local validation (the 3D editor) | under 100 ms |

## 6. Architecture

| Folder | Content |
|---|---|
| `src/nastran_layout/` | `schema/` (the tables, their columns and types, the version), `structure/` (stations, surfaces, members, nodes, edges, pockets, groups: level 1), `mapping/` (roles, representations, level 2), `points/` (calculation points, recipes, adapters), `build/` (from tables), `infer/` (from the mesh, the rules), `check/`, `edit/` (operations, journal, replay), `io/` (the layout folder, the tables in and out), `errors.py`, `diagnostics.py` |
| `tests/` | one test module per module, each test under one second; small decks of a stiffened barrel and a wing box, in 1D, 2D and mixed fidelity |
| `validation/` | the layouts of the validation decks, given as tables and inferred, compared |
| `spec/`, `docs/`, `benchmarks/`, `scripts/` | the specification, the user and developer guides (the schema reference), the performance targets, the checks |

## 7. Quality and verification

- **Schema**: the contract file is checked by the tests; a layout written by version n is read by version n + 1.
- **Inference**: on the validation decks, the layout inferred with the rules equals the layout given as tables (every
  entity, every role); in 1D, 2D and mixed fidelity, the same level 1.
- **Recipes**: executed through `nastran_rw` on a barrel loaded in pure tension, bending and shear, the resultants of the
  members equal the hand values; the `1d` and the `2d` models of the same barrel give the same resultants at the same
  points within a stated tolerance.
- **Edits**: each operation and its undo give back the layout bit for bit; a journal replayed on a layout inferred
  again after a remesh gives the corrected layout, and reports the operations that no longer apply.
- **Determinism**: the same layout and the same files whatever the order of the inputs and the number of threads.

## 8. Delivery plan

1. The package skeleton (wheel, checks), the schema and its contract, the layout folder.
2. Level 1 from tables: stations, surfaces, members, nodes, edges, pockets, groups, calculation points; the level 1
   checks.
3. Level 2: the mapping of 1D, 2D and mixed members, the roles, the recipes and their adapters; the level 2 checks; the
   staleness.
4. The inference from the mesh with the rules, and its comparison with the tables.
5. The edit operations, the journal, the replay; the geometry for an editor.
6. The performance targets; the user guide; the first release. `nastran_ssa` reads it from its step 1.

## 9. Out of scope

Loads, properties, materials and methods (the consumers'); the cut-outs (doors, windows) and their surrounding
structure; solid elements; the change of the layout during an optimisation (the shape is fixed); the 3D editor itself
(another plugin, section 3.7).

## 10. Decisions and open points

Decided by the owner (2026-10-09):

| Topic | Decision |
|---|---|
| Name and place | `nastran_layout`, a package of its own: the layout is managed by a module specialised on it, read by `nastran_ssa`, `nastran_smear` and `nastran_lcs` |
| Static | The layout does not change in the optimisation: built once, then read |
| Two levels | The idealised structure, and its mapping to each model |
| Members | Of any kind, the list open (stringers, frames, rails and others that are neither) |
| Fidelity | A member is represented by 1D elements, shells, or both, per edge |
| Calculation points | Explicit, attached to the entities, not to the stations; the stations are reference planes |
| Editing | Atomic validated operations and a replayable journal; a 3D editor as another plugin |
| Storage | One folder: a `manifest.json` and one Parquet file per table; pyarrow a dependency |

Open points:

- The default list of member kinds and segment roles, and the rules of the inference that come with the package.
- The defaults of the calculation points per member kind, and the tolerance of the snap of the `2d` cuts.
- The tolerances: the crossing of reference lines, a flat pocket, the comparison of the tables with the mesh.
- Whether the adapters to `nastran_sv` live here or in `nastran_sv` (they need no import of it here).
- The technology of the 3D editor (the `stlViewer` prototype in Three.js, a plugin of `GemseoProcessBuilder`, or other):
  its own specification.
