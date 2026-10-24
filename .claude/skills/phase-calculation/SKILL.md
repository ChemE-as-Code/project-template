---
name: phase-calculation
description: Identify the phases of a pure component or mixture with a flash at given temperature, pressure and composition (vapour, liquid, liquid-liquid, vapour-liquid, vapour-liquid-liquid; no solids), through thermo with an equation of state or an activity-coefficient model. Use when phase identification or a flash is needed for a stream.
---

# Phase calculation

## Purpose and when to use

Runs a flash through thermo and reports the number of phases, their fractions and
compositions. Use it to identify the phase of a stream or mixture. It is a calculation without a
stated application range: stream properties written to `process/flowsheet.yaml` come from
`cheme process props` (see `process-design`), which uses one validated method.

## Required inputs and missing inputs

- A thermodynamic model that has been recommended (`thermodynamic-model-selection`) or given.
  Do not pick one yourself.
- Components (names or CAS numbers), the temperature and the absolute pressure with units, and
  the overall mole fractions for a mixture.
- `allow_two_liquids`: set it when liquid-liquid separation is expected, as a stated assumption.

If the temperature, pressure, composition or model is missing, ask for it.

## Workflow and tools

```python
from chemeascode.domains.process.thermodynamics.phase import phase_determination

result = phase_determination(
    components=["ethanol", "water"],
    overall_composition=[0.25, 0.75],
    temperature_value=70,
    temperature_unit="°C",
    pressure_value=1,
    pressure_unit="bar(a)",
    thermodynamic_model={"liquid_phase_model": "NRTL", "vapor_phase_model": "ideal_gas"},
    allow_two_liquids=False,
)
```

Models: `{"eos_model": "Peng-Robinson" | "Redlich-Kwong" | "Redlich-Kwong-Soave"}`, or
`{"liquid_phase_model": "NRTL" | "Wilson" | "UNIFAC", "vapor_phase_model": <an EOS> |
"ideal_gas"}`. The names are exact and case-sensitive. Needs the `props` extra
(`pip install 'cheme-as-code[props]'`). There is no CLI command for the flash.

Redlich-Kwong-Soave takes its `kij` from the ChemSep SRK file, which is not shipped (licence
unconfirmed, core#55). `cheme data status` shows whether it is installed; the user installs it
with `cheme data fetch chemsep-srk` (downloads ChemSep-LITE; `--zip lite.zip` for a copy
already downloaded). Ask before running the fetch: it downloads third-party data.

## Validation and failure handling

- `ValueError` "Invalid thermodynamic model input": compare the names with the values above and
  call again.
- The recommended model is not available: report it; use a similar one only with approval.
- A gauge pressure is refused, because no ambient pressure is known: give it as absolute.
- Check `flash_convergence_error` before using the result.
- `notes` not empty (e.g. "ChemSep SRK interaction parameters are not installed; kij = 0"): the
  result used `kij = 0` for every pair. Report it with the result, and suggest
  `cheme data fetch chemsep-srk` or another model where the interaction matters.

## Outputs and provenance

`FlashCalculationResult`: `temperature_K`, `pressure_Pa`, `gas_count`, `liquid_count`, `betas`,
`gas_composition`, `liquid_compositions`, `component_cas_numbers`, the inputs
(`provided_facts`), `notes` (assumptions such as `kij = 0`) and `source` (thermo). Report the model and the assumptions with the result.

## Required human judgment

An engineer reviews the model choice and `allow_two_liquids`. A result near a phase boundary
needs confirmation from data. For a stream at its dew or bubble point that the flash puts a
fraction of a kelvin into the other phase, do not shift its temperature: state its phase, and
`cheme process props --fill` calculates it on that phase's EOS root, marked phase-constrained,
within a metastability tolerance (`process-design`).

## References

- `thermodynamic-model-selection`, `process-design`
- Interaction parameters: thermo's ChemSep databases (PR, NRTL, Wilson) and, for SRK, the
  ChemSep file installed by `cheme data fetch chemsep-srk` (origin and licence status:
  `src/chemeascode/domains/process/thermodynamics/data/chemsep/SOURCE.md`)
