# CLAUDE.md — working on nastran_layout

`nastran_layout` describes the layout of a stiffened structure once, in one versioned schema that the other packages
read: the stations, surfaces, members, nodes, edges, pockets, groups and calculation points (the idealised structure),
and their mapping to each finite element model (the roles of the elements, the representation of each edge in 1D,
shells or both, the recipes of the calculation points). It is a pure Python package built on `nastran_rw`
(`../nastran_rw`), which reads the models. The specification is authoritative; read it before writing code:

- [spec/SPEC.md](spec/SPEC.md): what the package does (requirements, algorithms, architecture, plan, decisions).
- [spec/NASTRAN-RW-PREREQUISITES.md](spec/NASTRAN-RW-PREREQUISITES.md): what it asks of `nastran_rw`.

Its consumers are `nastran_ssa` (`../nastran_ssa`, the analysis of stiffened structures), `nastran_smear`
(`../nastran_smear`, smeared results), `nastran_lcs` (`../nastran_lcs`, the selection of the sizing load cases) and a 3D
editor (another plugin, SPEC §3.7). Their specs say what they read from the layout.

The code is classic Python (PEP 8). The principles of `nastran_rw`'s coding rules (its `spec/CPP-LAYOUT.md` §6) are kept:
one job per module and per class, names that say what things are, comments that say what the code cannot. In case of
doubt, the spec wins over this file, and this file over habit.

## Language

- **Everything written in the repository is in English, without exception**: code, names, comments, docstrings, error
  and diagnostic messages, docs, READMEs, tests and their data, scripts, configuration files, commit messages, pull
  requests. A French word found in the repository is a bug to fix.
- The owner writes in French: answer in French in the conversation only.

## Hard rules (never break them)

1. **Pure Python.** One `py3-none-any` wheel. No compiled code in this repository; the heavy work goes to the compiled
   code that exists (the topology queries of `nastran_rw`, the sparse graphs of SciPy, NumPy). Dependencies:
   `nastran_rw`, NumPy, SciPy, pyarrow (the file format, SPEC §3.8). Adding any other dependency is the owner's
   decision.
2. **`nastran_rw` through its public API only.** `import nastran_rw as nr` and use what its `__init__` exports and its
   data contract documents; never `nastran_rw._core`, never a private name, never parse a deck here. A need it does not
   cover is asked of `nastran_rw` (`spec/NASTRAN-RW-PREREQUISITES.md`), not worked around here.
3. **The sibling repositories are read-only from here.** Read the specs, docs and stubs of `../nastran_rw`,
   `../nastran_ssa`, `../nastran_smear`, `../nastran_lcs` and `../nastran_sv` freely; never write, build or install from
   there, never run a git command that writes there.
4. **No import of the consumers.** `nastran_ssa`, `nastran_smear`, `nastran_lcs` and `nastran_sv` are never imported,
   not even optionally: they read the layout, not the reverse. No GUI dependency: the 3D editor drives the API of SPEC
   §3.7.
5. **The layout holds no loads, no properties and no methods.** It is static (SPEC §1): a thickness, a section or a
   laminate is never stored in it, and a change of one never makes a mapping stale. The recipes of the calculation
   points are data; they are never executed here.
6. **The schema is a contract.** Any change of a table, a column, its type or its meaning increments the schema version,
   updates `tests/schema_contract.json` and the schema reference of the docs, and keeps the reading of the files of the
   previous version (SPEC §3.8, §7).
7. **Every change of a built layout goes through an edit operation** (SPEC §3.7): validated, journalled, undoable. No
   public setter, no public mutable table bypasses them.
8. **Determinism.** The same inputs give the same layout, the same identifiers and the same files, whatever the number
   of threads or the order of the inputs. Identifiers are built from what defines an entity (SPEC §4.1), never from a
   row index, a hash order or a counter.
9. **Everything in English** (see Language).
10. **No test lasts more than one second.** Every test, without exception: `pytest-timeout` is set to 1 second for the
    whole suite and a test that exceeds it fails. There is no slow suite: a check that needs a large model is a
    benchmark (`benchmarks/`, not run by pytest), or it is rewritten on a smaller model that shows the same behaviour.

## Priorities

**Correctness** (closed pockets, complete mappings, recipes that give the hand values, the same level 1 in 1D, 2D and
mixed fidelity, a journal that replays, determinism), then **the targets of SPEC §5**, then simplicity. A faster but
harder to read version is kept only when it is measured and needed for a target.

## Coding rules

The reader is a Python developer of **intermediate level** (about five years of experience): at ease with Python,
NumPy arrays and broadcasting, dataclasses, type hints and pytest, but not expected to know the project, SciPy's sparse
graphs in depth, Nastran or the anatomy of stiffened structures. Idiomatic Python needs no explanation; explain what
such a reader cannot know: the Nastran rules, the structural vocabulary (a pocket, a run-out, a shear tie), the
geometric method and its source, the shape and frame of an array, the invariants, the non-obvious optimisations.

### Folders and files

The folder tree is a clean, logical architecture: a reader who knows what the package does must **guess where a piece of
code is** before searching for it.

- **A folder is named after what it contains**, with a word of the domain that a stress engineer or a developer
  recognises (`structure`, `mapping`, `points`, `infer`). Never a catch-all: no `utils`, `helpers`, `common`, `misc`,
  `core`, `lib`, `tools`, `stuff`, `other`. A function that seems to belong nowhere shows that a folder is missing or
  that the function is in the wrong place.
- **A file is named after its main class or its one job**, in `snake_case`: `pocket_boundaries.py` closes the boundary
  loops of the pockets, `edit_journal.py` holds `EditJournal`. A file that grows past one job is split.
- **One sub-package per module of SPEC §6**, at most two levels under `src/nastran_layout/`. A module uses only the
  modules below it in the SPEC §6 list (no import upwards, no cycle).
- **The same sub-path in `src/` and `tests/`**: `tests/structure/test_pocket_boundaries.py` tests
  `src/nastran_layout/structure/pocket_boundaries.py`.
- The `__init__.py` of a sub-package says in its docstring what the sub-package does, its main files and where to
  start reading; the package `__init__.py` re-exports the public API with an explicit `__all__`.
- A new folder, or a renamed one, is first added to SPEC §6 and to the tree below.

```
nastran_layout/
├── CLAUDE.md, README.md, pyproject.toml, LICENSE
├── spec/                     the specification (SPEC.md) and the requests to nastran_rw
├── src/nastran_layout/
│   ├── __init__.py           the public API
│   ├── errors.py             LayoutError and its subclasses
│   ├── diagnostics.py        the diagnostics table
│   ├── schema/               the tables, their columns and types, the schema version (SPEC §3.8)
│   ├── structure/            level 1: stations, surfaces, members, nodes, edges, pockets, groups (SPEC §3.1)
│   ├── mapping/              level 2: element roles, edge representations, staleness (SPEC §3.2)
│   ├── points/               calculation points, recipes, adapters to nastran_rw and nastran_sv (SPEC §3.3)
│   ├── build/                a layout from the user's tables (SPEC §3.4)
│   ├── infer/                a layout from the mesh, the rules (SPEC §3.5)
│   ├── check/                the checks of level 1, level 2, tables against mesh (SPEC §3.6)
│   ├── edit/                 operations, journal, undo, replay, geometry for an editor (SPEC §3.7)
│   └── io/                   the layout file, the tables in and out (SPEC §3.8)
├── tests/
│   ├── support/              builders of a stiffened barrel and a wing box in 1D, 2D and mixed fidelity
│   ├── schema_contract.json  the schema as the tests check it
│   └── <the same folders as src/nastran_layout/>
├── validation/               the layouts of the validation decks, as tables and inferred
├── benchmarks/               speed and memory measurements of SPEC §5 (not tests)
├── scripts/                  check.ps1 (format, lint, tests)
└── docs/                     the user guide and the schema reference
```

### Names

PEP 8: `snake_case` for modules, functions, methods, variables and parameters; `PascalCase` for classes, dataclasses
and enums; `UPPER_SNAKE_CASE` for constants and enum members; one leading underscore for what is private to a module or
a class.

**A name says what the thing is, in full words, so that a reader understands a line without looking elsewhere.**

- **No abbreviations** beyond the few below: `element_count`, not `n_elem`; `coordinate_system`, not `cs`;
  `member_pitch`, not `b`; `pockets_by_surface`, not `pbs`.
- **No generic names**: `data`, `values`, `result`, `res`, `tmp`, `obj`, `info`, `item`, `arr`, `x2`, `helper`,
  `manager`, `process`, `handle`, `do_*`. Say what is inside: `boundary_edges`, `reference_line`, `edges_by_member`.
- **The words of the layout are the words of the spec**, everywhere: `station`, `surface`, `member`, `segment`, `node`,
  `edge`, `pocket`, `group`, `calculation_point`, `recipe`, `mapping`, `role`, `representation`, `provenance`. Never a
  synonym (`bay` for a pocket, `stiffener` for a member) in the code.
- **Functions start with a verb that says what they do or give**: `close_pocket_boundaries`, `join_member_chains`,
  `find_web_feet`, `snap_point_to_grid_line`. A function that returns a boolean reads as a question: `is_closed`,
  `has_shell_segments`.
- **Booleans** read as a yes/no question: `is_irregular`, `is_stale`, `should_split_on_property`.
- **Collections are plural**; a mapping is named `<value>_by_<key>`: `element_ids`, `role_by_element`,
  `edges_by_member`. An index array says what it indexes: `pocket_index_of_element`.
- **The frame or the stage** is in the name when it matters: `normal_in_basic`, `position_in_pocket_frame`,
  `angle_in_degrees`, `inferred_members` vs `table_members`.
- **Identifiers**: in the code, full words (`element_id`, `property_id`, `grid_point_id`, `coordinate_system_id`); the
  stable identifiers of the layout are `member_name`, `edge_name`, `pocket_name`. The short Nastran names are kept where
  the outside world uses them: the column names of the tables (`eid`, `pid`, `nid`, the data contract of `nastran_rw`)
  and the fields of a card.
- **Mathematical names** only inside a function whose docstring states the formula with its source, and as written
  there. Short loop indices (`i`, `j`) are fine in a small scope.
- **Allowed short names**: `np`, `sparse` (`scipy.sparse`), `nr` (`nastran_rw`), `pa` and `pq` (`pyarrow`,
  `pyarrow.parquet`), `i`/`j`/`k` for loop indices, `x`/`y`/`z` for coordinates, and the Nastran words above.
- **Tests** are named by the behaviour they check: `test_run_out_makes_a_five_sided_pocket`,
  `test_replayed_journal_survives_a_remesh`.

| Unclear | Explicit |
|---|---|
| `def mk(m, r)` | `def infer_layout(model, rules)` |
| `ids = np.unique(c[:, 0])` | `member_element_ids = np.unique(chain_rows[:, ELEMENT_COLUMN])` |
| `idx`, `mask2` | `pocket_index_of_element`, `is_on_web_foot` |
| `a, b, r` | `pocket_length, pocket_width, curvature_radius` |
| `flag` | `is_irregular` |
| `d` | `edges_by_member` |

### Classes and data

- A class has **one job**. Plain data carriers are `@dataclass(frozen=True)`; a constructor establishes the
  invariants, no two-phase initialisation.
- Prefer functions to classes when there is no state to keep. A class holds state that lives across calls (a layout,
  its journal, a mapping).
- The tables of the layout are columnar (NumPy arrays or Arrow tables), never a Python object per entity: a fuselage has
  some 100 000 entities.
- No mutable global state; no mutable default arguments.
- The string options of the API (`by="pocket"`, `kind="stringer"`) are checked once, at the public entrance, against a
  tuple of allowed values or turned into an `Enum`; the error lists the allowed values. The member kinds and segment
  roles are open lists (SPEC §3.1): a user kind is accepted, never coerced.
- Avoid what an intermediate reader must look up to follow the flow: metaclasses, custom decorators, `__getattr__`
  tricks, monkey patching, deep inheritance, generators chained across modules. `@property`, `@staticmethod`,
  `functools.cached_property`, enums and dataclasses are ordinary tools.

### Functions

- A function does one thing and its name says it. A long function that reads top to bottom is better than one cut into
  fragments that must be read together.
- Type hints on every function used outside its module and every dataclass field (`numpy.typing.NDArray[np.float64]`
  for arrays).
- Return values, not output parameters; a small named dataclass, not a tuple, when the parts are not self-evident.
- Early return or raise for error cases.
- Comprehensions are fine when they hold one loop and at most one condition; otherwise write the loop.
- Named constants, never magic numbers; a tolerance is a constant with its reason, and the tolerances the user may set
  are options with that constant as default.

### NumPy, SciPy and geometric code

- **State the shape** of every array that crosses a function boundary, in the docstring or on the line that makes it:
  `pocket_frames  # (pockets, 3, 3), the rows x, y, z in the basic system`.
- Vectorise over elements, grid points and entities; a Python loop over elements or pockets is a bug on a fuselage.
  Loops over surfaces, member kinds or rules are fine. Graph work (chains, loops, connected parts) goes to
  `scipy.sparse.csgraph` or to the topology queries of `nastran_rw`.
- Fancy indexing, `np.add.at`, `np.bincount`, `np.lexsort` or a sparse construction whose intent is not evident get a
  one-line comment saying what it computes ("count the shells on each edge").
- Coordinates and lengths in float64; index arrays are int64.
- Every geometric method cites its source or states its rule (`# least-squares circle fit (Kasa): ...`); a choice the
  spec leaves open is written where it is made (`# SPEC §3.1: x is the projection of the axis on the pocket plane`).
- NaN means "no value" (a pocket that is not a quadrilateral has no single width); it is never used as a flag inside a
  computation.

### Performance code

An optimisation whose intent is not evident starts with

```python
# PERFORMANCE: <what is optimised and the invariant it relies on>
# Measured: <benchmark and gain>
```

and is removed if no gain is measured.

### Comments and docstrings

- Docstrings are prose, as in `nastran_rw`: say what the function gives, then what the reader cannot guess (the frame
  and unit of the values, the shapes, the order of the rows, what is raised). Name the parameters in backticks when they
  need explaining; no boilerplate section per parameter when the signature says it all.
- Class docstring: the role, the invariants, thread-safety, and what it keeps.
- Inside functions, the **why**, never a paraphrase of the code; cite the rule applied (`# SPEC §4.3: a cut needs a
  line of grid points`).
- Long modules are cut into sections by a comment line: `# ---- the boundary loops ----...` (to column 120).
- `TODO` and `FIXME` say what is missing and why it is acceptable for now.

### Errors and diagnostics

- A misuse of the API, a table that cannot be read or an edit that would break the layout raises `LayoutError` (a
  subclass of `nr.NastranError`) or one of its subclasses (`TableError`, `MappingError`, `EditError`, `SchemaError`),
  naming the table, row and column, or the entity and the elements at fault.
- What is approximated or left out but still gives a layout (an element of no entity, a snapped cut, an irregular
  pocket, an inferred member the rules could not classify) goes into the diagnostics table, never into a warning nor a
  print.
- Every message says what was expected, what was found, where (entity, element, grid point) and what to do.
- No bare `except`, no `except Exception` that hides an error; never a silent fallback.

### Formatting and checks

- `ruff format` formats the code and `ruff check` lints it (configured in `pyproject.toml`, PEP 8 naming included):
  4 spaces, **120 columns** as in `nastran_rw`, double quotes. Nobody formats by hand.
- Imports in three groups: the standard library, third-party (NumPy, SciPy, pyarrow, `nastran_rw`), this package.
- `scripts/check.ps1` runs the formatting check, the lint and the tests.

## Tests and checks

- Every behaviour has a pytest test, and every result a reference: a layout written by hand in the test, the hand values
  of a recipe on a barrel in pure tension, bending or shear, or a property (the same level 1 in 1D, 2D and mixed
  fidelity; an operation and its undo give back the layout bit for bit; the same identifiers whatever the order of the
  inputs).
- Models are built in memory by the builders of `tests/support/` (a stiffened barrel and a wing box, in 1D, 2D and mixed
  fidelity, with a run-out, a rail and a tapered tip); no large file in the repository. Random data uses a fixed seed.
- `tests/schema_contract.json` is checked by a test: the schema of the code equals the recorded one (hard rule 6).
- **No test above one second** (hard rule 10): small synthetic models, vectorised code. A test that approaches the
  limit is made smaller, never given a longer timeout or a skip marker. Benchmarks (`benchmarks/`) measure the targets
  of SPEC §5 and are run only before a release; they are not tests.
- While working, run only the tests of what you change (`pytest tests/structure/test_pocket_boundaries.py -k <name>`);
  run `scripts/check.ps1` once at the end of a step; never two test runs at once.
- Use the Python 3.12 that has `nastran_rw` installed (`py -3.12`).

## Way of working

- A change is done when its tests pass and `scripts/check.ps1` is green.
- A step that reveals a problem in a spec document corrects that document first; a decision that belongs to the owner
  (SPEC §10) is asked, not taken.
- A need of `nastran_rw` is added to `spec/NASTRAN-RW-PREREQUISITES.md`, never worked around (hard rule 2).
- Commits and pushes are made on the owner's request.
