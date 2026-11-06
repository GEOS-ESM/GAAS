**Audience:** coding agents and developers working on GAAS (GEOS aerosol analysis system).
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `GAAS` @ `ea3baf0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`), key `@GAAS`.
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

# GAAS architecture

**Build status on this line:** not built; not yet ported to MAPL3. `@GEOSchem_GridComp/CMakeLists.txt` on `release/MAPL-v3` has `esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as components are ported to MAPL3."; `GAAS` appears only in a commented-out list. The `esma_add_library` block that builds `GEOS_ChemGridComp.F90` (which adds this component) is commented out (`../CMakeLists.txt`); the file itself is present. The code below is the MAPL2 code, identical to the MAPL2 line, kept for a later port. Statements about the parent describe `GEOS_ChemGridComp.F90` as it is on this line; the file is present but not compiled.

One repository, one gridded component (`GAAS_GridCompMod`), one helper module (`LDE_Mod`) and a stand-alone driver. Both documents here are in one file: placement and map first, then the nine component sections.

## Placement and map

Fixture path: `src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@GAAS`. Parent: the Chemistry gridded component, see `@GEOSchem_GridComp/docs/ARCHITECTURE.md` (`../../docs/ARCHITECTURE.md` in the fixture). No child repositories.

| File | Content |
|---|---|
| `GAAS_GridCompMod.F90` | the gridded component: `SetServices`, `Initialize_`, `Run_`, `Finalize_`, helper `extract_` |
| `LDE_Mod.F90` | Local Displacement Ensembles (LDE): builds ensembles on the lat-lon or cubed-sphere grid and projects an AOD increment onto aerosol mixing ratios; public: `LDE`, `LDE_Create`, `LDE_Destroy`, `LDE_Generate`, `LDE_Projector1c` (`LDE_Mod.F90`) |
| `ana_lde.F90`, `ana_lde.py` | stand-alone program `ana_lde.x` (3-D aerosol analysis increments from pre-computed AOD increments, per its header) and a Python wrapper installed to `bin` (`CMakeLists.txt`) |
| `GAAS_Registry.rc` | ACG (automatic code generator) specs: imports and exports (`GAAS_Registry.rc`, `ImportSpec`, `ExportSpec`) |
| `GAAS_GridComp.rc` | private config read in `SetServices`: verbose, log-transform epsilon, channel, Mie and LDE parameters |
| `GAAS_AerRegistry.rc`, `GAAS_AodRegistry.rc`, `GAAS_Mie.rc` | chemistry registries and Mie-table settings read in `Initialize_` (`Chem_RegistryCreate`, `Chem_MieCreate`) |
| `GAAS_GridComp_ExtData.yaml`, `.rc` | ExtData (external data) inputs for the off-line AOD fields |
| `AMIP/` | copy of the ExtData `.yaml` (identical to the top-level one) and an `.rc` that reads the files from a fixed MERRA-2 path instead of `das.*` templates (diff of the two `GAAS_GridComp_ExtData.rc`) |
| `lde.rc` | resources for the `ana_lde` program (`ExpId`, grid size, file names, LDE keys) |
| `docs/` | this documentation |

Build (`CMakeLists.txt`): library from `LDE_Mod.F90` and `GAAS_GridCompMod.F90` (`m_ana.F90` is commented out), depending on `MAPL Chem_Base Chem_Shared GMAO_mpeu FVdycoreCubed_GridComp NetCDF::NetCDF_Fortran`; `Chem_Base` and `Chem_Shared` come from the Chemistry `Shared` directory, which is not built on this line (`../CMakeLists.txt`, comment on `Shared`); `esma_generate_gocart_code` generates the `GAAS_*___.h` include files from `GAAS_Registry.rc`; all `*.rc` and `*.yaml` (including `AMIP/`) are installed under `etc/`; `ana_lde.x` is built and `ana_lde.py` installed to `bin`.

## 1. Purpose

GAAS assimilates an off-line aerosol optical depth (AOD) analysis into the online aerosol state: it adjusts the model AOD at one wavelength with an averaging kernel, then distributes the AOD increment over the 3-D aerosol mixing ratios with Local Displacement Ensembles (`GAAS_GridCompMod.F90`, module header and `Run_`; `LDE_Mod.F90`, `LDE_Projector1c`). It reads no aerosol tracers of its own; it modifies those of the aerosol provider through the `AERO` state.

## 2. Place in the hierarchy

- Parent: Chemistry. Added with `MAPL_AddChild(GC, NAME='GAAS', SS=GAAS_SetServices)` when `ENABLE_GAAS:` is true (default false) (`GEOS_ChemGridComp.F90`, `SetServices`).
- Requires GOCART-2G: `SetServices` asserts `enable_GOCART2G` when GAAS is on, and the connection code raises an error again if it is off (`GEOS_ChemGridComp.F90`, `SetServices`).
- Children: none (`GAAS_GridCompMod.F90`, no `MAPL_AddChild`).
- Parent connections (`GEOS_ChemGridComp.F90`, `SetServices`): `AIRDENS` and `DELP` from `CHEMENV`; `AERO` from `GOCART2G`. `RH2` and `PLE` are not connected in Chemistry (grep of `GEOS_ChemGridComp.F90`); *Unverified:* they come from outside Chemistry as unconnected imports of the parent.

## 3. Entry points

One phase each; no multi-phase registration (`GAAS_GridCompMod.F90`, `SetServices`, `MAPL_GridCompSetEntryPoint` for Initialize, Run, Finalize).

- `SetServices`: allocates the private state, loads `GAAS_GridComp.rc` and reads `verbose:`, `eps_for_log_transform_aod:` (default 0.01), `single_channel:` (default 550); registers the entry points; stores the state as `'GAAS_state'`; includes the generated import and export specs; adds the `AERO` import (`MAPL_StateItem`, `MAPL_RestartSkip`); calls `MAPL_GenericSetServices`.
- `Initialize_`: `MAPL_GenericInitialize`; creates the aerosol and AOD registries and the Mie tables (`Chem_MieCreate`); takes the number of channels from the Mie tables; decides cubed-sphere versus lat-lon from `JM == 6*IM`; builds a private "AOD grid" with the same horizontal layout whose vertical dimension is the channel count; executes the `serialize_bundle` method of `AERO` and wraps the serialized bundle as simple bundles `q_f` and `q_a`; creates the AOD simple bundles (`y_f`, `y_a`, `y_d`, `z_f`, `z_a`, `z_k`); `LDE_Create` (`GAAS_GridCompMod.F90`, `Initialize_`).
- `Run_`: returns at once, doing nothing, if the `PredictorActive` alarm is ringing, the `ReplayShutOff` alarm is ringing, or the imported `aod_a` field is undefined (that is how "analysis time" is detected). Otherwise:
  1. Sets relative humidity (`RH2`), pressure (`PLE`) and wavelength (`single_channel` x 1e-9) as attributes and fields of `AERO`, then runs its `get_monochromatic_aop` method and takes the vertically summed extinction as the background AOD `y_f`.
  2. Transforms `z_a`, `z_f` and `y_f` to log(AOD+eps), forms the adjusted analysis `y_a = z_a + (1 - z_k)(y_f - z_f)`, transforms back, and `y_d = y_a - y_f`.
  3. For each species (dust, sea salt, sulfate, nitrate, organic, black and brown carbon) runs the `get_mixRatioSum` method of `AERO` to get the species sum; saves it for the `*INC` exports.
  4. `LDE_Projector1c(E, q_a, q_f, y_f, y_d)` changes the aerosol mixing ratios.
  5. Re-reads the species sums and fills `*ANA` (after) and `*INC` (after minus before) exports, `AODANA`, `AODINC`.
  (`GAAS_GridCompMod.F90`, `Run_`.)
- `Finalize_`: `MAPL_GenericFinalize`, `LDE_Destroy`, destroys the simple bundles `y_f`, `y_a`, `y_d` (`GAAS_GridCompMod.F90`, `Finalize_`).
- `extract_`: fetches the private state and the time; the time used is the clock time plus one time step, because the clock ticks before HISTORY is written (`GAAS_GridCompMod.F90`, `extract_`, comment).

## 4. State

- Imports, from `GAAS_Registry.rc`: `DELP`, `RH2`, `PLE` (3-D); `aod_a`, `aod_f`, `aod_k` (2-D, flagged `norestart`); plus `AERO` added in `SetServices`. Exports: `AODANA`, `AODINC` (2-D); `{DU,SS,NI,SU,BC,OC,BR}ANA` and `{...}INC` (3-D, kg/kg).
- Off-line AOD inputs `aod_a` (analysis), `aod_f` (background), `aod_k` (averaging kernel) come from ExtData, 3-hourly files `das.aod_{a,f,k}.sfc.*.nc4`, offset `PT450S`, `fail_on_missing_file: false` (`GAAS_GridComp_ExtData.yaml`).
- Key coupling: `AERO` is the GOCART-2G aerosol state; GAAS drives it through attributes and `ESMF_MethodExecute` labels `serialize_bundle`, `get_monochromatic_aop`, `get_mixRatioSum`; see the parent's documents on the aerosol interface. `q_a` and `q_f` are created from the same serialized bundle, so *Inferred:* the LDE update changes the provider's aerosol mixing ratios in place (the `LDE_Projector1c` comment says `bQ_a` "may share storage with `bQ_f`").
- Exports are not consumed by other Chemistry code (grep of `GEOS_ChemGridComp.F90` for `AODANA`, `DUANA`, `AODINC` finds nothing); *Inferred:* they are for HISTORY output.
- Internal state: none (`GAAS_GridCompMod.F90`, `SetServices`, "INTERNAL STATE: (none for now)"). Restarts: none; `AERO` is `MAPL_RestartSkip`. The private `GAAS_State` holds config, Mie tables, registries, the `LDE` object, and the simple bundles.

## 5. Configuration that changes structure

- `ENABLE_GAAS:` in the Chemistry config (parent: `GEOS_ChemGridComp.F90`, `SetServices`).
- `GAAS_GridComp.rc`: `single_channel:`, `eps_for_log_transform_aod:`, `verbose:`; `top_vertical_layer`, `number_of_ensemble_members`, `stencil_radius_in_km`, `aod_weight_delta` (LDE, read in `LDE_Create`); `NUM_BANDS`, `BANDS` and the `*_OPTICS` file names (Mie).
- Extra: `AMIP/GAAS_GridComp_ExtData.rc` replaces the input file templates; which one is used is chosen by the experiment setup, not by this repository.

## 6. Dependencies

`MAPL`, `ESMF`, `Chem_Base` (`Chem_SimpleBundleMod`, `Chem_RegistryMod`, `Chem_MieMod`, `Chem_AodMod`), `Chem_Shared`, `GMAO_mpeu` (`m_StrTemplate`, `m_Random`, `m_MergeSorts`), `FVdycoreCubed_GridComp` (*Inferred:* listed in `CMakeLists.txt` for the cubed-sphere grid code; `Initialize_` uses `CubedSphereGridFactory` from MAPL), NetCDF, MPI (`GAAS_GridCompMod.F90` and `LDE_Mod.F90`, `use` lines; `CMakeLists.txt`).

## 7. Known traps

- `GAAS_GridCompMod.F90` includes generated headers (`GAAS_ImportSpec___.h` etc.); names such as `rh2`, `ple`, `aod_a`, `aodana` in `Run_` are declared there, not in the file.
- GAAS depends on the optics methods and attribute names of `AERO` (`relative_humidity_for_aerosol_optics`, `air_pressure_for_aerosol_optics`, `wavelength_for_aerosol_optics`, `monochromatic_extinction_in_air_due_to_ambient_aerosol`, `sum_of_internalState_aerosol_<token>`); renaming any in GOCART-2G breaks GAAS at run time.
- The species list in `get_aerosolSum` is fixed (DU, SS, SU, NI, CA.oc, CA.bc, CA.br); another aerosol provider would need these names.
- Alarms `PredictorActive` and `ReplayShutOff` are fetched from the clock with no existence guard; *Unverified:* they are created by the GCM driver above Chemistry.
- Grid type is detected by `JM == 6*IM` in both `Initialize_` and `LDE_Create`.

## 8. Decisions

See [`decisions.md`](decisions.md).

## 9. Observations (noted, not fixed)

- Chemistry connects `AIRDENS` from `CHEMENV` to GAAS, but `GAAS_Registry.rc` declares no `AIRDENS` import (`GEOS_ChemGridComp.F90`, `SetServices`; `GAAS_Registry.rc`). Question: is the connection ignored, or is the import declared elsewhere?
- `Finalize_` destroys `y_f`, `y_a`, `y_d` but not `z_f`, `z_a`, `z_k`; the destroy of `q_f`, `q_a` is commented out (`GAAS_GridCompMod.F90`, `Finalize_`).
- In `Run_`, `y_f`, `y_a`, `y_d` point at local allocatable arrays (`aodInt`, `y_a_`, `y_d_`) that go out of scope when `Run_` returns, while the bundles live to `Finalize_`; `Finalize_` destroys those bundles (`GAAS_GridCompMod.F90`, `Run_`, `Finalize_`). Question for maintainers: is the pointer state after return relied on?
- `Initialize_` registry and Mie calls are tagged `! REMOVE` in comments (`GAAS_GridCompMod.F90`, `Initialize_`).
- `ana_lde.py` header says it wraps `ana_lpe.x`; the built program is `ana_lde.x` (`ana_lde.py`; `CMakeLists.txt`). `lde.rc` names files under a personal home directory.
- `GAAS_Mie.rc` lists four channels (`n_channels: 4`) while `GAAS_GridComp.rc` has `NUM_BANDS: 1`; *Unverified:* which file `Chem_MieCreate` reads here (`Initialize_` passes the private config `self%CF`, which is `GAAS_GridComp.rc`).
- `README.md` holds only the title.
- Open question: `ana_lde.x` and `lde.rc` are built or installed but no GEOS code in this repository calls them; are they still used outside it?
