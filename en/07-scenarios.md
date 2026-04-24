# AIMP 07 — Reference Scenarios

**Status:** Draft · **Version:** 1.0.0-draft · **Informative**

This chapter walks through four concrete end-to-end scenarios that exercise the full
protocol. All JSON fragments are simplified for readability; full request/response bodies
are in `examples/` alongside this spec.

## Scenario A — AI-Generated Poster → Cloud Print → Doorstep

**Goal:** Demonstrate the canonical "AI makes a thing and it shows up at my door" flow.

**Actors:** User, single AI agent via MCP, AIMP gateway fronting a cloud-print vendor,
courier API.

**Domain:** `manufacturing.print.2d.v1` + `logistics.shipping.v1`

**Flow:**

1. User: *"Make me an A3 cyberpunk poster and ship it. Budget $25."*
2. Agent generates a 300-dpi CMYK PNG and stores it at a signed URL.
3. `aimp.discover({"device_filter":{"domains":["manufacturing.print.2d.v1"]}})` →
   returns one device `cloudprint-us-west`.
4. `aimp.quote`:
   ```json
   {
     "device_id": "cloudprint-us-west",
     "domain": "manufacturing.print.2d.v1",
     "asset": {"url": "...", "format": "image/png", "hash_sha256": "..."},
     "payload": {
       "paper_size": "A3",
       "paper_stock": "matte_200gsm",
       "dpi": 300,
       "color_mode": "CMYK",
       "copies": 1
     },
     "logistics": {
       "carrier_preference": "standard",
       "recipient": {"name":"...", "address_lines":["..."], "postal_code":"...", "country":"US"}
     },
     "budget_limit": {"amount": 25.00, "currency": "USD"}
   }
   ```
   Gateway responds with `quote_id` and `estimated_cost.amount = 18.20`.
5. Agent checks budget, calls `aimp.execute({"quote_id": "..."})`.
6. Gateway transitions `LOCKED → EXECUTING → FULFILLING`, emits shipping webhook with
   tracking number.
7. Gateway reaches `COMPLETED`. Agent replies to user:
   *"Printed and shipped. FedEx #7721... arrives Tuesday. Charged $18.20."*

Key observations:

- No adapter code in the agent. Everything is through five MCP tools.
- Budget was enforced on the gateway, not the agent's honor system.
- Vision audit was not required for routine 2D print — paper-jam detection is sufficient.

## Scenario B — 3D-Printing a Gear with Mid-Build Human Review

**Goal:** Show a `restricted`-tier job with AI vision checks and a mandated human
checkpoint.

**Actors:** Design Agent, Audit Agent, Executor Agent (A2A), AIMP gateway, local FDM
printer, human reviewer.

**Domain:** `manufacturing.additive.fdm.v1`

**Flow:**

1. Design Agent produces `gear.gcode` from user-provided CAD.
2. Review Agent verifies geometry; signs off to Executor Agent.
3. Executor `aimp.quote` with:
   ```json
   {
     "device_id": "printer-lab-01",
     "domain": "manufacturing.additive.fdm.v1",
     "asset": {"url": "...", "format": "model/gcode", "hash_sha256": "..."},
     "payload": {
       "material_requirement": "PETG",
       "parameters": {
         "nozzle_temp_celsius": 240,
         "bed_temp_celsius": 80,
         "infill_percent": 35,
         "layer_height_mm": 0.2
       }
     }
   }
   ```
4. Quote: 2.30 USD, 73 min. Executor forwards to Finance Agent; receives
   `approval_token`.
5. `aimp.execute` with `audit_requirements`:
   ```json
   {
     "snapshot_interval_seconds": 300,
     "snapshot_channels": ["camera.top"],
     "sensors": ["extruder_temp", "chamber.temp"],
     "ai_vision_checks": ["detect_spaghetti_failure"],
     "pause_for_human_at": ["mid_build_50_percent"]
   }
   ```
6. At progress 0.50 → `AUDITING`. Gateway sends `human_action_required` to the shop
   operator's Lark / Slack.
7. Operator reviews the camera frames, confirms layer adhesion looks correct, signs
   approval. Executor calls `/resume` with `decision: "CONTINUE"`.
8. Audit Agent's vision check runs at each snapshot; at progress 0.72 it flags
   `detect_spaghetti_failure` verdict `warn`, confidence 0.61. Job continues but telemetry
   is flagged.
9. Completed at progress 1.0. Final cost 2.28 USD.

Key observations:

- Three agents cooperate; only Executor touches AIMP.
- Both automated (vision) and human checkpoints are first-class states.
- `ADJUST` on resume could have nudged nozzle temperature up by 5 °C had the reviewer
  seen stringing.

## Scenario C — Laboratory Pipetting of a Reagent Plate

**Goal:** Show a domain unrelated to printing to prove the core's generality.

**Actors:** Protocol Agent (biology-specialized), AIMP gateway in a lab, pipetting robot,
scientist reviewer.

**Domain:** `fluidics.pipette.v2`

**Flow:**

1. Protocol Agent interprets a protocol description: *"Transfer 15 µL from plate 1 row A
   into plate 2 starting at B3, slow aspirate."*
2. `aimp.discover` returns `pipette-rob-07` with `audit_channels: ["camera.workcell",
   "liquid_level"]`.
3. `aimp.quote` payload:
   ```json
   {
     "device_id": "pipette-rob-07",
     "domain": "fluidics.pipette.v2",
     "payload": {
       "operations": [
         {"step": 1, "type": "ASPIRATE", "source_well": "Plate_1_A1",
          "volume_microliters": 15.0, "aspirate_speed_ul_per_sec": 5.0},
         {"step": 2, "type": "DISPENSE", "destination_well": "Plate_2_B3",
          "volume_microliters": 15.0, "dispense_speed_ul_per_sec": 10.0},
         {"step": 3, "type": "ASPIRATE", "source_well": "Plate_1_A2",
          "volume_microliters": 15.0, "aspirate_speed_ul_per_sec": 5.0},
         {"step": 4, "type": "DISPENSE", "destination_well": "Plate_2_B4",
          "volume_microliters": 15.0, "dispense_speed_ul_per_sec": 10.0}
       ]
     },
     "audit_requirements": {
       "sensors": ["liquid_level"],
       "ai_vision_checks": ["detect_bubble_formation"]
     }
   }
   ```
4. Quote: 0.40 USD, 2 min. Executed.
5. At step 3, a bubble is detected → verdict `warn`, confidence 0.88. Job pauses to
   `AUDITING` because policy binds `warn` on hazardous reagents.
6. Scientist reviews camera frame, signs approval to continue. Protocol completes.

Key observations:

- Exactly the same core verbs, exactly the same envelope as Scenario A and B.
- Only `domain` and `payload` differ.
- Audit policy is tuned at the gateway, not hard-coded in the agent.

## Scenario D — Smart Kitchen: Sous-Vide Steak

**Goal:** A consumer / hazardous-tier analog.

**Actors:** Home user, kitchen-agent on the appliance's built-in hub, AIMP gateway
embedded in the appliance.

**Domain:** `thermodynamics.cooking.v1`

**Flow:**

1. User: *"Cook the ribeye medium-rare."*
2. Agent consults the user's preferences; chooses 55 °C core for 1 h 20 min, then
   30 s high-heat sear.
3. `aimp.quote` → 0.14 USD (energy), 1 h 25 min.
4. Budget is tiny; no approval token required by policy.
5. `aimp.execute` with:
   ```json
   "audit_requirements": {
     "sensors": ["water_bath_temp", "core_temp"],
     "ai_vision_checks": ["detect_char_on_food"]
   }
   ```
6. Job runs. At sear step, vision detects heavier char than target → verdict `warn`.
   Policy chose CONTINUE because domain risk tier is `restricted` not `hazardous` and the
   confidence is below threshold.
7. Completes. Agent tells user: *"Done. Internal 54.8 °C, surface slightly darker than
   target — want me to adjust next time?"*

Key observations:

- Even a countertop appliance can be AIMP-native; it just runs gateway + adapter on the
  same SoC.
- The vision check produces a learning signal for the next run, not just a safety gate.

---

## Interoperability Summary

All four scenarios share:

- The same five core verbs (`discover`, `quote`, `execute`, `telemetry`, `abort`).
- The same envelope (§01.3) and state machine (§01.4).
- The same auth, budget, and audit-log patterns.

They differ only in:

- The `domain` string.
- The `payload` structure (bound by each domain's JSON Schema).
- The set of sensors and `ai_vision_checks` registered for that domain.

That is the point of AIMP: **the physical world is plural, but the contract with it can
be singular**.

## Further Examples

See `examples/`:

- `01-cloud-print-poster.json` — Scenario A request/response pairs.
- `02-3d-print-gear.json` — Scenario B including audit cycle.
- `03-lab-pipette.json` — Scenario C including bubble-detection pause.
