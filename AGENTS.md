# GAAS (GEOS aerosol analysis system)

**Audience:** coding agents and developers working on GAAS.
**Describes:** the MAPL2 line, branch `develop`.
**Verified against:** `GAAS` @ `ea3baf0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `0f2e0fb`), key `@GAAS`.

Single gridded component, `GAAS_GridCompMod`: reads off-line AOD (aerosol
optical depth) analysis fields, adjusts the model AOD, and spreads the
increment over the aerosol mixing ratios of GOCART-2G with Local
Displacement Ensembles (`LDE_Mod`). It is a child of the Chemistry gridded
component, enabled by `ENABLE_GAAS:` and requiring GOCART-2G.

Fixture path: `src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@GAAS`.
Parent: [`@GEOSchem_GridComp/docs/ARCHITECTURE.md`](../docs/ARCHITECTURE.md)
and [`@GEOSchem_GridComp/AGENTS.md`](../AGENTS.md).

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
Built as part of the Chemistry library (`CMakeLists.txt`); also builds the
`ana_lde.x` program. No unit tests in this repository. Build with the
GEOSgcm tree (NAG first, then gfortran).
