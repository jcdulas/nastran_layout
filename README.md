# nastran_layout

The layout of a stiffened structure, described once for every analysis: the stations, the surfaces, the members
(stringers, frames, rails, floor beams, longerons, intercostals, spar and rib caps, or a kind you name), the nodes where
they meet, the edges between nodes, the pockets the edges close, the named groups (sides, zones), and the calculation
points at which the analyses read their loads. A pure Python package built on `nastran_rw`, which reads the models.

## What it does

- **Two levels.** The idealised structure, independent of any finite element model; and its mapping to each model:
  which elements represent each entity, with which role, and how the loads of each calculation point are obtained. A
  global model and a detailed model of the same structure share the first level.
- **Any fidelity.** A member is represented by 1D elements, by shells, or by both, edge by edge; the calculation points
  give the same quantities whatever the representation.
- **Built from tables or inferred from the mesh.** The user's tables are the reference; the inference from the mesh
  starts them, and the check compares the two.
- **Static.** The sizing changes thicknesses and sections, never the layout: it is built once, then read. It holds no
  loads, properties or methods.
- **Edited through journalled operations.** Every change is validated and recorded; the journal gives undo and redo,
  and is replayed after a remesh so that the corrections are not lost. A 3D editor (another plugin) drives these
  operations.
- **One versioned folder.** A JSON manifest and one Parquet table per entity, its schema a documented contract: the
  consumers can read it with pyarrow alone.

It is read by `nastran_ssa` (the analysis of stiffened structures: its items are the pockets and edges), `nastran_smear`
(the smeared running loads of stiffened panels) and `nastran_lcs` (the zones and the items of the selection of the
sizing load cases).

## Example (planned API)

```python
import nastran_rw as nr
import nastran_layout as nl

model = nr.Model.read("fuselage_gfem.bdf")
draft = nl.infer(model, nl.Rules(axis=(0.0, 0.0, 0.0, 1.0, 0.0, 0.0), member_kinds="member_kinds.csv"))
draft.to_tables("layout_tables/")                # reviewed and corrected by the user

layout = nl.Layout.from_tables("layout_tables/", units="mm")
gfem = layout.map(model, label="gfem")
layout.check(model, gfem).to_dataframe()
layout.save("fuselage_layout/")
```

## Status

Specification only (2026-10-09): no code yet. See [spec/SPEC.md](spec/SPEC.md) and its delivery plan (section 8), and
[spec/NASTRAN-RW-PREREQUISITES.md](spec/NASTRAN-RW-PREREQUISITES.md) for what it asks of `nastran_rw`.

## Install

Python 3.12 or later on Windows x64 or Linux x64, with `nastran_rw` 1.1 (data contract 51) installed. `nastran_rw` is
not on PyPI: install its wheel first, then this package (`pip install -e .`). Its dependencies are NumPy, SciPy and
pyarrow.

## Develop

The coding rules are in [CLAUDE.md](CLAUDE.md): classic Python with explicit names, everything in English, no test
above one second, `nastran_rw` through its public API only, the schema as a contract.

## License

MIT, see [LICENSE](LICENSE).
