# GAAS (GEOS aerosol analysis system)

**Audience:** coding agents and developers working on GAAS.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `GAAS` @ `ea3baf0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`), key `@GAAS`.

**Build status on this line:** not built; not yet ported to MAPL3. On
`release/MAPL-v3`, `@GEOSchem_GridComp/CMakeLists.txt` calls
`esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as
components are ported to MAPL3."; `GAAS` is in a commented-out list of
subdirectories. The `esma_add_library` block that builds
`GEOS_ChemGridComp.F90` is commented out (`../CMakeLists.txt`); the file
itself is present. The code in this repository is the MAPL2 code, identical
to the MAPL2 line, kept for a later port.

Single gridded component, `GAAS_GridCompMod`: reads off-line AOD (aerosol
optical depth) analysis fields, adjusts the model AOD, and spreads the
increment over the aerosol mixing ratios of GOCART-2G with Local
Displacement Ensembles (`LDE_Mod`). It is a child of the Chemistry gridded
component, enabled by `ENABLE_GAAS:` and requiring GOCART-2G.

Fixture path: `src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@GAAS`.
Parent: `@GEOSchem_GridComp/docs/ARCHITECTURE.md` (`../docs/ARCHITECTURE.md` in the fixture)
and `@GEOSchem_GridComp/AGENTS.md` (`../AGENTS.md` in the fixture).

## Documents
| File | Content |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | placement, file map, entry points, state, traps, observations |
| [`docs/decisions.md`](docs/decisions.md) | recorded design patterns and open questions |

Existing files: `README.md` (title only), `CHANGELOG.md`, `CONTRIBUTING.md`.

## Rules for agents
- Note bugs and oddities; do not fix them unless asked. Record them in
  `docs/ARCHITECTURE.md`, "Observations".
- Imports and exports are declared in `GAAS_Registry.rc` (generated code);
  edit the `.rc`, not generated headers.
- GAAS calls `AERO` methods and attribute names defined in GOCART-2G; check
  that side before changing either.
- Keep these documents current when the code changes.
- Add a line to `CHANGELOG.md` for changes (the repository enforces it in
  `.github/workflows/changelog-enforcer.yml`).

## Build and test
Not built on this line (see "Build status" above). This repository's
`CMakeLists.txt` defines the library (`LDE_Mod.F90`, `GAAS_GridCompMod.F90`),
builds the `ana_lde.x` program and installs `ana_lde.py`; it is reached only
through the parent's `esma_add_subdirectories`, which does not list `GAAS`
on this line. GAAS depends on `Chem_Base` and `Chem_Shared`
(`CMakeLists.txt`, `DEPENDENCIES`), which come from the Chemistry `Shared`
directory; that directory is also not built on this line (`../CMakeLists.txt`,
comment on `Shared`). No unit tests in this repository.
