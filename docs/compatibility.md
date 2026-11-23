# Headset Compatibility Log

Real-world results pairing helmet headsets with each other and, later, with the bt_rider hub.

The hub's core idea: every headset, whatever its brand or generation, can pair with a **phone** using the Bluetooth Hands-Free Profile (HFP). If the hub acts as the phone (HFP Audio Gateway) for each headset, any headsets can be bridged together.

## Headsets under test

| ID | Headset | Intercom type | Phone pairing (HFP) | Owner / bike | Notes |
|---|---|---|---|---|---|
| H1 | Smart HJC 10B (by Sena), firmware v1.0 | Sena Bluetooth intercom only. No mesh. | Yes | Rider 1, 2015 Kawasaki KLR 650 | Sena-equivalent: SMH5. Has Universal Intercom, so it can present itself as a phone to another headset. |
| H2 | Sena Spider ST1 | Sena Mesh only. No Bluetooth intercom. | Yes | Rider 2, 2025 Honda CRF300L | Pairs only with other Sena mesh units (Spider RT1, 50S, 50R, 30K, Momentum EVO, or a Sena headset fitted with the +Mesh adapter). |

## Pairing results

| Pair | Method | Result | Date | Notes |
|---|---|---|---|---|
| H1 ↔ H2 | Native intercom | ❌ Not possible | 2026-11 | The two units use incompatible intercom systems: Bluetooth intercom on H1, mesh on H2. |
| H1 ↔ H2 | H1 Universal Intercom (H1 acts as a phone) with H2 in phone pairing mode | ⏳ To test | | Hold the 10B centre button ~8 s until the prompt "Universal intercom pairing", then put the Spider in phone pairing mode. |
| H1 ↔ H2 | Sena +Mesh adapter bridging H1 (Bluetooth intercom) into Sena Mesh with H2 | ⏳ Adapter ordered | 2026-11 | Sena officially lists the Spider ST1 as compatible with +Mesh-equipped headsets. Reviewers report +Mesh works with the SMH5, which HJC calls equivalent to the 10B. No official confirmation for the 10B itself. Update H1 firmware before testing. |
| H1 → hub | Hub as HFP Audio Gateway | ⏳ Phase 2 | | |
| H2 → hub | Hub as HFP Audio Gateway | ⏳ Phase 2 | | |

## Lessons learned

- Two units from the **same brand**, a few years apart, can be unable to talk to each other. Sena's Bluetooth intercom and Sena Mesh are separate systems.
- Phone pairing (HFP) is the only interface every headset shares. This is the basis for the hub design.

## How to add an entry

Record the exact model, firmware version (from the manufacturer's app), the method you tried, the result, and the approximate range or audio quality if it worked.
