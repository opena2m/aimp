# AIMP 02 — Domain Extensions

**Status:** Draft · **Version:** 1.0.0-draft · **Normative**

The AIMP core (§01) knows nothing about nozzles, joints, pipettes, or lasers. All
domain-specific semantics live in **Domain Schemas**, discoverable and versioned
independently of the core. This chapter specifies how to write, register, and version
those schemas.

## 1. Namespace Convention

Every domain is identified by a reverse-dotted, lowercase namespace followed by an
explicit version suffix:

```
<category>.<subcategory>[.<specialization>...].v<major>
```

Examples:

```
manufacturing.additive.fdm.v1           FDM (fused deposition) 3D printing
manufacturing.additive.slm.v1           Selective laser melting
manufacturing.subtractive.cnc_mill.v1   3-axis CNC milling
manufacturing.print.2d.v1               Inkjet / laser 2D printing
kinematics.robotic_arm.v1               6-DOF industrial arms
kinematics.mobile_robot.v1              AGV / wheeled robots
kinematics.drone.v1                     Aerial platforms
fluidics.pipette.v2                     Lab liquid handling
fluidics.mixing.v1                      Bioreactors / chemical mixers
thermodynamics.cooking.v1               Smart kitchen appliances
thermodynamics.reaction.v1              Chemical reactors
optics.imaging.v1                       Cameras / microscopes for capture
sensing.spectroscopy.v1                 Spectrometers
logistics.shipping.v1                   Courier handoff
```

### 1.1 Categories (normative top level)

The spec reserves these top-level category names:

```
manufacturing   additive, subtractive, forming, assembly, print
kinematics      arms, mobile, aerial, underwater
fluidics        pipette, pumping, mixing
thermodynamics  heating, cooling, reaction, cooking
optics          imaging, projection
sensing         spectroscopy, metrology, weight, gas
logistics       shipping, storage, sortation
```

New top-level categories MUST go through the registry process (§4). Sub-levels are open.

### 1.2 Version Rules

- The `.v<N>` suffix is a **major** version. Breaking changes increment it.
- Minor additions (new optional fields) SHOULD reuse the existing major and are signaled
  through the schema's own `$id` + `schemaVersion`.
- Gateways MUST list every supported `(namespace, version)` pair in `discover`.
- Clients MUST pin to a specific version in `execute`. They MAY negotiate at `quote` time
  using `["manufacturing.additive.fdm.v1", "manufacturing.additive.fdm.v2"]`.

## 2. Anatomy of a Domain Schema

A Domain Schema is a JSON Schema 2020-12 document declaring the permitted structure of the
`payload` field inside a `quote` or `execute` request (§01.6). It SHOULD also declare
optional `audit_requirements` extensions and consumable types.

Every Domain Schema MUST provide at minimum:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://aimp.dev/schemas/manufacturing.additive.fdm.v1.json",
  "title": "FDM 3D Printing Payload",
  "aimp_domain": "manufacturing.additive.fdm.v1",
  "type": "object",
  "required": ["asset", "material_requirement", "parameters"],
  "properties": {
    "asset": {
      "type": "object",
      "required": ["url", "format"],
      "properties": {
        "url":    {"type": "string", "format": "uri"},
        "format": {"type": "string", "enum": ["model/gcode", "model/stl", "model/3mf"]},
        "hash_sha256": {"type": "string"}
      }
    },
    "material_requirement": {
      "type": "string",
      "enum": ["PLA", "PETG", "ABS", "TPU", "Nylon"]
    },
    "parameters": {
      "type": "object",
      "properties": {
        "nozzle_temp_celsius":   {"type": "number", "minimum": 150, "maximum": 300},
        "bed_temp_celsius":      {"type": "number", "minimum": 0,   "maximum": 120},
        "infill_percent":        {"type": "number", "minimum": 0,   "maximum": 100},
        "layer_height_mm":       {"type": "number", "minimum": 0.05,"maximum": 0.8}
      }
    }
  }
}
```

Machine-readable copies of the reference domains live in `schemas/domains/`.

## 3. Required Fields in Every Domain

Any published Domain Schema MUST include, in addition to its own domain-specific fields:

- **`asset`** *(when applicable)* — a pointer to the digital input (G-code, CAD, image,
  recipe, trajectory file). Schemas MUST enumerate acceptable formats via MIME or an
  explicit `enum`.
- **`parameters`** — the tunable knobs the AI is allowed to set. Bounds MUST be
  enforceable (`minimum`, `maximum`, `enum`, `pattern`).
- **`material_requirement` / `consumable_requirements`** *(when applicable)* —
  what the machine will consume. The gateway uses this to compute `resource_consumption`
  in the quote.

## 4. The Domain Registry

To avoid fragmentation, AIMP-conformant gateways SHOULD resolve unknown domains through a
registry:

```
GET https://registry.aimp.dev/domains/<namespace>/<version>
```

The registry serves the canonical JSON Schema document. Gateways MAY cache registry
responses. Vendors publishing new domains MUST supply:

1. The JSON Schema document.
2. A short prose description (≤2 pages).
3. At least one end-to-end example request and expected response.
4. Safety and hazard class (§04.4).

The registry is out-of-band for gateways operating in isolated networks; those gateways
MUST ship the schemas they support locally.

## 5. Extensibility Rules

- A Domain Schema MUST NOT rename or repurpose any core envelope field (§01.3).
- A Domain Schema MAY define **custom audit channels** (e.g., a spectroscopy domain adds
  `audit_channels: ["spectrometer.raw"]`). Gateways MUST report them in `discover`.
- A Domain Schema MAY require **additional sensor readings** by listing them under
  `required_sensors`. The gateway MUST reject `execute` requests for devices that do not
  expose those sensors.
- Domain Schemas SHOULD classify themselves by a **risk tier** (§04.4): `routine`,
  `restricted`, `hazardous`. Risk tier drives default human-in-the-loop policy.

## 6. Multi-Step Operations

Some domains (robotics, fluidics) are sequences of primitive actions. The RECOMMENDED
pattern is an `operations` array:

```json
"payload": {
  "operations": [
    {"step": 1, "type": "MOVE_TO_COORDINATE",
     "coordinates": {"x": 105.2, "y": 45.0, "z": 10.5}, "precision_mm": 0.1},
    {"step": 2, "type": "ASPIRATE_FLUID", "volume_microliters": 50},
    {"step": 3, "type": "MOVE_TO_COORDINATE",
     "coordinates": {"x": 140.0, "y": 45.0, "z": 10.5}, "precision_mm": 0.1},
    {"step": 4, "type": "DISPENSE_FLUID", "volume_microliters": 50}
  ]
}
```

Each step type is itself constrained by the Domain Schema's `$defs`.

For any multi-step operation, the domain MAY specify **per-step audit gates** by
including `audit_after: true` on individual steps.

## 7. Illustrative Domain Coverage

The following table shows how distinct physical scenarios fit the same core:

| Scenario | Domain | Key payload fields | Typical audit channels |
|---|---|---|---|
| 2D poster print | `manufacturing.print.2d.v1` | `paper_size`, `color_mode`, `dpi` | `camera.output_tray` |
| FDM 3D print | `manufacturing.additive.fdm.v1` | `nozzle_temp`, `infill_percent` | `camera.top`, `extruder_temp` |
| CNC milling | `manufacturing.subtractive.cnc_mill.v1` | `tool_id`, `feed_rate`, `spindle_rpm` | `vibration`, `spindle_load` |
| Pick-and-place arm | `kinematics.robotic_arm.v1` | `end_effector_pose`, `grip_force_N` | `camera.workcell`, `force_torque` |
| Lab pipetting | `fluidics.pipette.v2` | `source_well`, `dest_well`, `volume_ul` | `liquid_level`, `camera.rack` |
| Sous-vide cooking | `thermodynamics.cooking.v1` | `target_temp`, `duration` | `core_temp`, `camera.bath` |
| Parcel handoff | `logistics.shipping.v1` | `carrier`, `service_level`, `recipient` | `waybill_barcode_scan` |

## 8. Authoring Checklist for New Domains

Before publishing a new Domain Schema to the registry:

- [ ] Namespace is reverse-dotted, lowercase, version-suffixed.
- [ ] JSON Schema validates against draft 2020-12.
- [ ] All numeric parameters have both `minimum` and `maximum` when physically meaningful.
- [ ] All enumerations are closed; open strings are reserved for truly opaque identifiers.
- [ ] `required_sensors` declared if the domain relies on sensor feedback.
- [ ] Risk tier declared.
- [ ] A reference example request exists and passes validation.
- [ ] An adapter implementation is available (reference or vendor).

## 9. Next Reading

- §03 AI Protocol Integration — how Domain Schemas project onto MCP tool schemas.
- §04 Security & Cost — how risk tiers and consumables map to budget policy.
- §05 Audit & Telemetry — how domain-specified audit channels are streamed.
