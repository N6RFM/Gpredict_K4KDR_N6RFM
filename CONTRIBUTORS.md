# Contributors to GPredict_K4KDR_N6RFM

This file credits contributions specific to this fork, in addition to
the full upstream Gpredict authors list (see the About dialog or
AUTHORS file for the original project's contributors).

## K4KDR

Original author of the Alpha-5 catalog number decoding feature
(NORAD ID >= 100000 support), written for the K4KDR/gpredict fork.
See https://github.com/K4KDR/gpredict.

## N6RFM

- Carried the Alpha-5 catalog number feature forward onto a clean
  base of current upstream Gpredict master.
- Diagnosed, root-caused, and fixed the rotor display timer
  regression (upstream commit dfb7b02) that left the Rotator
  Control window's Az/El "Read" fields permanently stuck.
- Diagnosed, root-caused, and fixed a self-deadlock and stale-socket
  reuse bug in the radio control's auto-disengage-on-error path,
  which prevented reconnection after a rigctld-compatible flowgraph
  (e.g. GNU Radio Companion) restarted.
- Raised the Cycle spinbox's hardcoded value ceiling and changed the
  rotor/radio default Cycle value.
- Maintains this fork: https://github.com/N6RFM/Gpredict_K4KDR_N6RFM

## Claude (Anthropic)

Assisted with debugging, root cause analysis (including building
headless, automated sandbox reproductions and using git bisect to
identify the exact regressing commits), writing and verifying the
fixes, and preparing upstream bug reports
(https://github.com/csete/gpredict/issues/423 and /424).
