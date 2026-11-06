**Audience:** coding agents and developers working on GAAS.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `GAAS` @ `ea3baf0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`).

# GAAS decisions

### Increment applied through the aerosol provider's state
- Pattern: GAAS has no tracers; it sets attributes on `AERO` and calls its `serialize_bundle`, `get_monochromatic_aop` and `get_mixRatioSum` methods (`GAAS_GridCompMod.F90`, `Initialize_`, `Run_`). [Verified]
- Likely reason: keep the analysis independent of the provider's internal layout. [Inferred]
- Question for maintainers: is GOCART-2G the only intended provider?

### GOCART-2G required
- Pattern: Chemistry asserts GOCART-2G when GAAS is enabled (`GEOS_ChemGridComp.F90`, `SetServices`). [Verified]
- Likely reason: only GOCART-2G offers the `AERO` methods used. [Inferred]

### Analysis in log(AOD+eps) space, LDE in plain AOD
- Pattern: averaging-kernel adjustment is done in log(AOD+eps); the projector receives plain AOD `y_f` and `y_d` (`GAAS_GridCompMod.F90`, `Run_`, comments). [Verified]
- Likely reason: log transform for the kernel adjustment; the log is non-linear in concentration (see the `perform_lde_in_aod_variable` comment in `lde.rc`). [Inferred]

### One analysis step gated by the off-line field
- Pattern: no analysis unless `aod_a` is defined, and none during the predictor step or after replay shut-off (`GAAS_GridCompMod.F90`, `Run_`). [Verified]
- Likely reason: the off-line AOD files are 3-hourly (`GAAS_GridComp_ExtData.yaml`, `freq`). [Inferred]

### Private AOD grid with channels as vertical
- Pattern: `Initialize_` builds a grid with the model's horizontal layout and the Mie channel count as vertical size (`GAAS_GridCompMod.F90`, `Initialize_`). [Verified]
