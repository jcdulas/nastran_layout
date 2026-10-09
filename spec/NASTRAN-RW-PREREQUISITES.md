# nastran-layout — What `nastran_rw` must add

Status: requests of 2026-10-09, for the owner and the maintainers of `nastran_rw`; nothing is delivered yet. Companion of
[SPEC.md](SPEC.md). State read on `nastran_rw` 1.1.0 (unreleased), data contract 51.

`nastran_layout` reads models only through `nastran_rw`'s public API and the columns of its data contract. Most of what
it needs exists: the topology queries of the family T3 and T4 give it the chains of 1D elements, the panels of shells,
the edges and their kind, the feature edges, the free edges as loops, the neighbours of an element, the revision stamps
of groups and the differences between two models. The requests below are few; the numbering is "layout R1" to "layout
R4", to be listed in companion-packages.md §11.

## Summary

| Item | Content | State today | Needed by `nastran_layout` step |
|---|---|---|---|
| R1 | Coordinate transforms from Python (shared with `nastran_ssa` R3) | missing | 2 |
| R2 | The boundary loops of many groups of shells in one call | one group per call | 2, 4 |
| R3 | `panels` bounded by the 1D elements the caller names | all 1D elements on shell edges bound | 4 |
| R4 | Release of 1.1.0, and a row for `nastran_layout` in the compatibility table | 1.1.0 unreleased | 1 |

Already delivered and used as they are: `element_chains` (the 1D members), `edges_table` (`kind` `non_manifold`: the feet
of the webs of 2D members), `panels` (with `feature_angle`, `split_on_property`, `element_ids`, `detail="elements"` and
the layers of the stacks), `feature_edges`, `free_edges` (ordered loops), `element_neighbors`, `node_elements`,
`property_zones`, `group_revisions` (the revision stamps of the mapped elements), `model_diff` (what moved after a
remesh), `coords_table`, the element geometry (`center`, `normal`, `area`, `length`, `frame`), the grid point positions,
and `section_cuts` with named cuts `{"nodes": ..., "side_elements": ...}`, whose resultants the recipes of the `2d`
points ask (section 3.3 of the spec; the consumers execute them).

## R1 — Coordinate transforms from Python

**Need.** The reference axis, the stations and the reference lines of the members may be given in coordinate systems of
the model (a CORD2C for a fuselage); the frames of the pockets and of the calculation points are computed in the basic
system.

**Today.** `coords_table` gives the origin and axes of each system; no call transforms points or vectors.

**Asked.** Vectorised `model.coords.to_basic(cid, xyz)`, `from_basic(cid, xyz)` and `axes_at(cid, xyz)`, for points and
vectors (the same item as `nastran_ssa`'s R3: one delivery serves both).

**Check.** Round trips to 1e-12 on CORD1R, CORD2R, CORD2C and CORD2S, nested systems included.

## R2 — The boundary loops of many groups in one call

**Need.** The boundary of each pocket (some 20 000 on a fuselage) as an ordered loop of grid points, cut afterwards into
the edges of the layout.

**Today.** `free_edges(element_ids=...)` gives the loops of the boundary of one selection of elements; 20 000 calls do
not meet the target of one minute for the whole inference.

**Asked.** `free_edges(groups={name: element IDs})`: the loops of the boundary of each group, a column `group`, one call,
the stacks of shells counted once as `panels` counts them.

**Check.** Equal, group by group, to the loops of one call per group.

## R3 — `panels` bounded by chosen 1D elements

**Need.** The pockets are bounded by the members of the layout, not by every 1D element that lies on a shell edge (a
small bracket, a rod modelling a seal, a CBAR that the user does not keep as a member).

**Today.** `panels` is bounded by every 1D element on shell edges, the edges shared by three stacks or more, the free
edges, and the property changes and feature angles when asked.

**Asked.** `panels(bounding_elements=IDs)`: only these 1D elements bound the panels (the others are ignored as
boundaries), the other boundaries unchanged; and `bounding_edges=` (pairs of grid points) for a boundary that no element
marks (a member of the layout that the model represents by no element: an `absent` edge).

**Check.** On a barrel with stringers, frames and brackets: the panels bounded by the stringers and frames only.

## R4 — Release

`nastran_layout` will accept the contracts it has tested (51 at least) and pin `nastran_rw>=1.1,<1.2`, checked at import
as `nastran_smear` does. Asked: the release of 1.1.0 and a row for `nastran_layout` in the compatibility table of
companion-packages.md §10.
